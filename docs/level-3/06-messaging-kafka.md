---
description: "Use Kafka from Spring Boot: KafkaTemplate producers, @KafkaListener consumers, partitions and keys, offsets, retries, dead-letter topics, and the outbox."
---

# Messaging with Apache Kafka

When one service needs to tell others that something happened — an order was placed, a
payment failed — calling each of them synchronously couples everyone's availability
together. Publishing an event to a broker lets producers and consumers evolve and fail
independently. Spring for Apache Kafka makes both sides straightforward; the design
questions (ordering, duplicates, failures) are what this lesson is really about.

RabbitMQ with Spring AMQP is the other common choice. It is a traditional message queue
(messages are removed once acknowledged, with rich routing), whereas Kafka is a
replicated log (messages are retained and consumers track their own position). Most
concepts below — idempotent consumers, dead-lettering, the outbox — apply to both.

## Kafka in five ideas

1. A **topic** is a named log of records, split into **partitions**.
2. A record has a **key**, a value, and headers. Records with the same key go to the
   same partition, and **order is guaranteed only within a partition**.
3. Consumers join a **consumer group**; each partition is read by exactly one consumer
   in the group, so partitions are the unit of parallelism.
4. Each group tracks an **offset** per partition — how far it has read. Records are not
   deleted when read; retention is time- or size-based.
5. Delivery is **at-least-once** by default: after a crash, records since the last
   committed offset are delivered again. Consumers must tolerate duplicates.

## Running Kafka locally

This lesson's code needs a broker, which was not run for the course. With Docker:

```bash
docker run -d --name kafka -p 9092:9092 apache/kafka:latest
```

(The official `apache/kafka` image starts a single-node KRaft broker with defaults
suitable for local development.) Alternatively, add Boot's Docker Compose support and a
`compose.yaml`, and Boot starts it for you when the app starts.

## Producing

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-kafka</artifactId>
</dependency>
```

(That is the Boot 4 starter, which brings `spring-kafka` plus Boot's Kafka
auto-configuration module. In Boot 3 you add `org.springframework.kafka:spring-kafka`
directly. Initializr's "Spring for Apache Kafka" option picks the right one.)

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JacksonJsonSerializer
      acks: all
      properties:
        enable.idempotence: true
```

```java
public record OrderPlaced(String orderId, String customerId, BigDecimal total, Instant at) { }

@Service
class OrderEvents {
    private final KafkaTemplate<String, OrderPlaced> kafka;

    OrderEvents(KafkaTemplate<String, OrderPlaced> kafka) { this.kafka = kafka; }

    CompletableFuture<SendResult<String, OrderPlaced>> publish(OrderPlaced event) {
        return kafka.send("orders.placed", event.orderId(), event);   // key = orderId
    }
}
```

`JacksonJsonSerializer`/`JacksonJsonDeserializer` are the Jackson 3 variants in Spring
Kafka 4 (the version Boot 4.1 manages); the older `JsonSerializer`/`JsonDeserializer`
(Jackson 2) still exist there but are deprecated, and are what you use with Boot 3.

`acks: all` waits until all in-sync replicas have the record; idempotence prevents
duplicates caused by producer retries. `send` is asynchronous; check the future (or at
least log failures) — an unobserved failed send is a lost event.

Keying by `orderId` means all events for one order land in one partition, in order.

## Consuming

```yaml
spring:
  kafka:
    consumer:
      group-id: shipping
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JacksonJsonDeserializer
      properties:
        spring.json.trusted.packages: com.example.orders.events
```

```java
@Component
class ShippingListener {
    private final ShipmentService shipments;

    ShippingListener(ShipmentService shipments) { this.shipments = shipments; }

    @KafkaListener(topics = "orders.placed", concurrency = "3")
    void on(OrderPlaced event) {
        shipments.createFor(event.orderId());   // must be idempotent
    }
}
```

`concurrency = "3"` runs three consumers in this instance; with six partitions and two
instances, each consumer gets one partition. More consumers than partitions leaves some
idle.

## Failures: retries and dead letters

If the listener throws, Spring Kafka's `DefaultErrorHandler` retries the record in place
(by default a few times with no backoff), then logs and skips it. For production, retry
with backoff and then publish the failed record to a **dead-letter topic** for inspection:

```java
@Bean
DefaultErrorHandler errorHandler(KafkaTemplate<Object, Object> template) {
    var recoverer = new DeadLetterPublishingRecoverer(template);    // → orders.placed-dlt by default
    var handler = new DefaultErrorHandler(recoverer, new ExponentialBackOff(500, 2.0));
    handler.addNotRetryableExceptions(ValidationException.class);  // don't retry poison pills
    return handler;
}
```

Boot wires a single `CommonErrorHandler` bean into its listener container factory. For
non-blocking retries (moving the failed record to retry topics so the partition keeps
flowing), Spring Kafka offers `@RetryableTopic`.

## Idempotent consumers

At-least-once delivery means `createFor("o-123")` may be called twice. Make it safe:

```java
@Transactional
public void createFor(String orderId) {
    if (shipments.existsByOrderId(orderId)) return;     // plus a unique constraint as the real guard
    shipments.save(new Shipment(orderId));
}
```

The unique constraint on `shipment.order_id` is what actually guarantees one shipment even
if two deliveries race; the `exists` check just avoids a noisy exception.

## Worked example: the outbox pattern

The dangerous code is the obvious one:

```java
@Transactional
public Order place(PlaceOrder cmd) {
    Order order = orders.save(...);
    kafka.send("orders.placed", ...);   // what if the DB commit fails after this? or Kafka is down?
    return order;
}
```

The database and Kafka cannot share a transaction. Either the event is sent for an order
that was rolled back, or the order commits and the event is lost. The **transactional
outbox** fixes this:

1. In the *same* database transaction as the order, insert a row into an `outbox` table
   (`id, aggregate_id, type, payload, created_at, published_at`).
2. A separate relay reads unpublished rows in order, sends them to Kafka, and marks them
   published (a `@Scheduled` poller is enough to start; change-data-capture tools such
   as Debezium read the database log instead).
3. Consumers are idempotent, because the relay may send a row twice if it crashes between
   sending and marking.

The order and the intent to publish now commit or roll back together. Level 3's project
implements exactly this.

## How It Actually Works

**Producer.** `KafkaTemplate.send` serializes key and value, and the Kafka client's
partitioner hashes the key (murmur2) modulo the partition count to choose a partition.
Records are batched per partition in memory and sent by a background I/O thread; the
returned future completes when the broker acknowledges according to `acks`. With
idempotence on, the producer attaches a producer id and sequence numbers so the broker can
discard duplicates caused by retries.

**Consumer.** For each `@KafkaListener`, Spring creates a
`ConcurrentMessageListenerContainer` holding N `KafkaMessageListenerContainer`s, each with
its own `KafkaConsumer` and thread (consumers are not thread-safe). Each thread loops:
`poll()` a batch, invoke your method per record, and commit offsets according to the
container's `AckMode` — by default `BATCH`, meaning after all records from a poll are
processed. The group coordinator on the broker assigns partitions to consumers and
**rebalances** when consumers join, leave, or stop polling for longer than
`max.poll.interval.ms` — a slow listener can therefore get kicked out of its group and
cause duplicate processing. Keep listeners fast or lower `max.poll.records`.

## Common mistakes

- **Ignoring the send future**, losing events silently.
- **Publishing inside a DB transaction** and assuming atomicity. Use an outbox.
- **Non-idempotent consumers.** Duplicates will happen.
- **Random or null keys** where order matters.
- **Trusting all packages in the JSON deserializer** (`spring.json.trusted.packages: "*"`),
  which lets message headers choose classes to instantiate.
- **Infinite retries of a poison message**, blocking the partition forever.

## Exercise

1. Run Kafka locally, publish `OrderPlaced` events keyed by order id, and consume them in a
   listener that logs partition and offset (`@Header(KafkaHeaders.RECEIVED_PARTITION)`).
2. Make the listener throw for a specific order id, configure the dead-letter handler, and
   find the record in `orders.placed-dlt`.
3. Kill the consumer mid-batch (stop the app) and restart it. Show that some records are
   processed again, and that your idempotent `createFor` handles it.
4. Write an integration test with Testcontainers' Kafka module (needs Docker) that
   publishes an event and asserts, with Awaitility, that a shipment row appears.
