---
description: "Level 3 project: an event-driven order service with a transactional outbox to Kafka, idempotent consumers, a resilient payment client, and metrics."
---

# Project — An Event-Driven Order Service

This project connects everything in Level 3. You will build an `orders` service that
accepts orders over REST, calls a payment provider safely, publishes `OrderPlaced` events
to Kafka through a transactional outbox, consumes `PaymentSettled` events idempotently,
and exposes health and business metrics.

Parts of this project need infrastructure: PostgreSQL and Kafka (via Docker). Those parts
were **not** run for this course, so no output is shown for them; the design, code, and
the tests you should write are given instead. The Spring mechanisms it relies on — proxies,
transactions, events, lifecycle, Actuator — were each demonstrated with real output in the
earlier lessons.

## Architecture

```text
POST /api/orders ──► OrderController ──► OrderService (@Transactional)
                                           ├─ PaymentsApi.authorize()   (RestClient, timeouts, retry, circuit breaker)
                                           ├─ orders.save(order)
                                           └─ outbox.save(OrderPlaced)  ── same DB transaction
                     OutboxRelay (@Scheduled, single-instance) ──► Kafka "orders.placed"
Kafka "payments.settled" ──► PaymentSettledListener ──► OrderService.markPaid()  (idempotent)
```

Dependencies: Web, Validation, Data JPA, PostgreSQL, Flyway, Kafka, Actuator, AspectJ
(for resilience annotations), and Testcontainers for tests.

## Schema

```sql
-- V1__orders_and_outbox.sql
create table orders (
    id           uuid primary key,
    customer_id  varchar(64)   not null,
    total_cents  bigint        not null check (total_cents > 0),
    status       varchar(20)   not null,
    payment_ref  varchar(64),
    created_at   timestamptz   not null,
    version      bigint        not null default 0
);

create table outbox (
    id            bigint generated always as identity primary key,
    aggregate_id  uuid         not null,
    type          varchar(100) not null,
    payload       jsonb        not null,
    created_at    timestamptz  not null default now(),
    published_at  timestamptz
);
create index idx_outbox_unpublished on outbox (id) where published_at is null;

create table processed_message (
    message_id    varchar(100) primary key,
    processed_at  timestamptz  not null default now()
);
```

`processed_message` is the idempotency ledger for consumed events.

## Placing an order

```java
@Service
public class OrderService {
    private final OrderRepository orders;
    private final OutboxRepository outbox;
    private final PaymentGateway payments;
    private final JsonMapper json;
    private final Counter placed;

    // constructor injecting all of the above, plus MeterRegistry for the counter

    public OrderView place(PlaceOrder cmd) {
        UUID id = UUID.randomUUID();
        // 1. Remote call OUTSIDE the transaction: no DB connection held while waiting.
        PaymentAuthorization auth = payments.authorize(id, cmd.totalCents());
        // 2. Short transaction: order + outbox row commit together.
        return saveOrder(id, cmd, auth);
    }

    @Transactional
    protected OrderView saveOrder(UUID id, PlaceOrder cmd, PaymentAuthorization auth) { ... }
}
```

Stop. That `saveOrder` call is a **self-invocation** — `@Transactional` would be ignored,
and the order and outbox rows would be saved in two separate repository transactions.
This is exactly the trap from Level 2. The correct structure puts the transactional part
in its own bean:

```java
@Service
class OrderService {
    private final PaymentGateway payments;
    private final OrderWriter writer;
    // ...
    public OrderView place(PlaceOrder cmd) {
        UUID id = UUID.randomUUID();
        PaymentAuthorization auth = payments.authorize(id, cmd.totalCents());
        OrderView view = writer.save(id, cmd, auth);
        placed.increment();
        return view;
    }
}

@Component
class OrderWriter {
    private final OrderRepository orders;
    private final OutboxRepository outbox;
    private final JsonMapper json;
    // constructor ...

    @Transactional
    OrderView save(UUID id, PlaceOrder cmd, PaymentAuthorization auth) {
        Order order = orders.save(Order.placed(id, cmd.customerId(), cmd.totalCents(), auth.reference()));
        var event = new OrderPlaced(id.toString(), cmd.customerId(), cmd.totalCents(), order.getCreatedAt());
        outbox.save(OutboxMessage.of(id, "OrderPlaced", json.writeValueAsString(event)));
        return OrderView.from(order);
    }
}
```

Now the proxy wraps `OrderWriter.save`, and the order and its outbox row commit or roll
back together. (Note the method is package-private: with CGLIB proxies, Spring can
intercept package-private methods when caller and proxy are in the same package, but
making transactional entry points `public` avoids surprises.)

## The payment client

An HTTP interface over a `RestClient` with 1 s connect / 2 s read timeouts (lesson 07),
plus:

```java
@Component
class PaymentGateway {
    private final PaymentsApi api;
    PaymentGateway(PaymentsApi api) { this.api = api; }

    @Retryable(includes = ResourceAccessException.class, maxRetries = 2, delay = 200, multiplier = 2, jitter = 50)
    @CircuitBreaker(name = "payments")
    PaymentAuthorization authorize(UUID orderId, long cents) {
        return api.authorize(new AuthorizeRequest(orderId.toString(), cents));   // orderId doubles as idempotency key
    }
}
```

The order id is sent as the idempotency key, so a retried authorization after a timeout
cannot double-charge — the provider returns the original result. Because the order id is
generated *before* the call, a client retry of the whole `POST` is a different order;
accept an `Idempotency-Key` header on the endpoint if clients need end-to-end retry safety.

## The outbox relay

```java
@Component
class OutboxRelay {
    private final OutboxRepository outbox;
    private final KafkaTemplate<String, String> kafka;
    // constructor ...

    @Scheduled(fixedDelay = 500)
    @SchedulerLock(name = "outbox-relay")        // ShedLock: one instance at a time
    @Transactional
    void relay() {
        for (OutboxMessage m : outbox.findTop100ByPublishedAtIsNullOrderByIdAsc()) {
            kafka.send("orders.placed", m.getAggregateId().toString(), m.getPayload()).join();
            m.markPublished(Instant.now());
        }
    }
}
```

`.join()` waits for the broker acknowledgment before marking the row published. If the
relay crashes between the send and the commit, the message is sent again next time —
at-least-once, which the consumers must tolerate. Keying by order id keeps each order's
events ordered.

## Consuming payment settlements idempotently

```java
@Component
class PaymentSettledListener {
    private final ProcessedMessages processed;
    private final OrderRepository orders;
    // constructor ...

    @KafkaListener(topics = "payments.settled", groupId = "orders")
    @Transactional
    void on(PaymentSettled event, @Header(KafkaHeaders.RECEIVED_KEY) String key) {
        if (!processed.markIfNew(event.messageId())) return;   // insert ... on conflict do nothing
        orders.findById(UUID.fromString(event.orderId()))
              .ifPresent(o -> o.markPaid(event.settledAt()));   // dirty checking
    }
}
```

`markIfNew` inserts into `processed_message` and returns whether a row was inserted.
Because it runs in the same transaction as the order update, a duplicate delivery either
sees the ledger row and skips, or races and fails on the primary key — never double-applies.

## Observability

- Actuator with probes; readiness includes `db`. Liveness does not include Kafka or the
  payment provider.
- Metrics: `orders.placed` counter; an `orders.outbox.pending` gauge
  (`outbox.countByPublishedAtIsNull()`, cached for a few seconds); the auto-configured
  `http.server.requests`, HikariCP, Kafka client, and `http.client.requests` metrics.
- Alert when `orders.outbox.pending` keeps growing — it means the relay or Kafka is stuck.

## Tests to write

1. **Unit**: `Order` state transitions (`placed → paid`, paying twice is a no-op).
2. **`@WebMvcTest`**: validation errors and 503 problem detail when the circuit is open
   (mock `PaymentGateway` to throw `CallNotPermittedException`).
3. **`@RestClientTest`**: 402 from the provider maps to `PaymentRejectedException`.
4. **Testcontainers (PostgreSQL + Kafka)**: placing an order results in exactly one record
   on `orders.placed` (consume it with a test consumer and Awaitility); publishing the same
   `PaymentSettled` twice marks the order paid once.
5. **Outbox atomicity**: make `outbox.save` throw and assert no order row exists.
6. **Proxy check**: assert `AopUtils.isAopProxy(orderWriter)` so a refactor that breaks the
   transactional boundary fails a test.

## How It Actually Works

The reliability of this design comes from *where each guarantee lives*:

- **Atomicity of "order + intent to publish"** comes from one database transaction — made
  real by the `OrderWriter` proxy's `TransactionInterceptor`.
- **Delivery to Kafka** comes from the relay's retry-until-acknowledged loop — at-least-once.
- **Exactly-once effect** at consumers comes from idempotency — the ledger table's primary
  key, checked in the same transaction as the state change.
- **Protection from a slow payment provider** comes from timeouts (bounded waiting),
  retries with an idempotency key (transient failures), and the circuit breaker (sustained
  failures) — and from keeping the remote call outside the transaction so no database
  connection is held while waiting.

None of these relies on Kafka transactions or distributed transactions (XA). That is
deliberate: local transactions plus idempotency are simpler to reason about and operate.

## Exercise (stretch goals)

1. Implement the service with Docker Compose for PostgreSQL and Kafka, and all six tests.
2. Replace the polling relay with Spring Modulith's event externalization or Debezium, and
   write down the trade-offs.
3. Add a `GET /api/orders/{id}` with Caffeine caching and `@TransactionalEventListener`-based
   eviction when an order is paid.
4. Load-test `POST /api/orders` while the fake payment provider is slow, with and without the
   circuit breaker, and compare the service's thread usage and error rates.
