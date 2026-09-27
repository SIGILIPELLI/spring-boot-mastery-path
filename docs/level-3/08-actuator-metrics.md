---
description: "Operate Spring Boot with Actuator: health groups and Kubernetes probes, custom health indicators, Micrometer metrics, and endpoint security."
---

# Actuator, Health & Metrics

A service that cannot tell you whether it is healthy, or how it is performing, is a
service you operate blindfolded. Spring Boot Actuator adds production endpoints for
health, metrics, configuration, and more; Micrometer supplies the metrics API underneath.

## Adding Actuator

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

By default only `/actuator/health` is exposed over HTTP. Expose more deliberately:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
```

Useful endpoints: `health`, `info`, `metrics`, `prometheus`, `loggers` (read and change log
levels), `env` and `configprops` (with values sanitized), `conditions` (the
auto-configuration report), `mappings`, `beans`, `threaddump`, `heapdump`. Several of these
reveal internals — treat them as admin interfaces.

## Worked example: real responses

With Actuator added to the Level 1 project and the configuration above (minus
`prometheus`), these are the actual responses from Boot 4.1:

```bash
$ curl localhost:8080/actuator/health
{"groups":["liveness","readiness"],"status":"UP"}

$ curl localhost:8080/actuator/health/readiness
{"status":"UP"}
```

After a few requests, the built-in HTTP server metric:

```bash
$ curl localhost:8080/actuator/metrics/http.server.requests
{"availableTags":[{"tag":"exception","values":["none"]},{"tag":"method","values":["GET"]},
 {"tag":"error","values":["none"]},{"tag":"uri","values":["/actuator/health/**","/actuator/health","/api/tasks"]},
 {"tag":"outcome","values":["SUCCESS"]},{"tag":"status","values":["200"]}],
 "baseUnit":"seconds","measurements":[{"statistic":"COUNT","value":4.0},
 {"statistic":"TOTAL_TIME","value":0.06834950000000001},{"statistic":"MAX","value":0.057555}],
 "name":"http.server.requests"}
```

(Line breaks added.) Notice the `uri` tag holds the **route template**, not the raw path —
`/api/tasks/{id}` rather than `/api/tasks/42` — which keeps the number of distinct time
series bounded. Drill down with `?tag=uri:/api/tasks&tag=status:200`.

## Health: liveness vs readiness

On Kubernetes (and similar platforms) there are two different questions:

- **Liveness** — "is this process broken beyond repair?" If it fails, the platform
  *restarts* the container. It should almost never depend on external systems; a database
  outage should not cause every instance to restart in a loop.
- **Readiness** — "should this instance receive traffic right now?" If it fails, the
  platform stops routing to it, without restarting. Include dependencies the instance
  genuinely cannot serve without.

With probes enabled, Boot exposes `/actuator/health/liveness` and
`/actuator/health/readiness`, backed by the application's `AvailabilityState`. Readiness
becomes `ACCEPTING_TRAFFIC` only after startup completes and flips to `REFUSING_TRAFFIC`
during graceful shutdown. Add indicators to the readiness group explicitly:

```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: readinessState, db
```

## Custom health indicators

```java
@Component
class PaymentProviderHealth implements HealthIndicator {
    private final PaymentsApi payments;

    PaymentProviderHealth(PaymentsApi payments) { this.payments = payments; }

    @Override
    public Health health() {
        try {
            payments.ping();
            return Health.up().build();
        } catch (Exception e) {
            return Health.down().withDetail("reason", e.getClass().getSimpleName()).build();
        }
    }
}
```

The bean name minus `HealthIndicator`/`Health` suffix becomes the component name
(`paymentProvider`). Keep checks fast and cheap: health endpoints are polled every few
seconds by several systems. In Boot 4, the health API moved to its own module
(`org.springframework.boot.health.contributor.HealthIndicator`); in Boot 3 it is
`org.springframework.boot.actuate.health.HealthIndicator`.

## Custom metrics with Micrometer

Micrometer is to metrics what SLF4J is to logging: one API, many backends (Prometheus,
OTLP, Datadog, CloudWatch…).

```java
@Service
class OrderService {
    private final Counter ordersPlaced;
    private final Timer checkoutTimer;

    OrderService(MeterRegistry registry) {
        this.ordersPlaced = Counter.builder("orders.placed")
                .description("Orders successfully placed")
                .register(registry);
        this.checkoutTimer = Timer.builder("orders.checkout")
                .publishPercentileHistogram()
                .register(registry);
    }

    OrderView place(PlaceOrder cmd) {
        return checkoutTimer.record(() -> {
            OrderView view = doPlace(cmd);
            ordersPlaced.increment();
            return view;
        });
    }
}
```

- **Counter** — things that only go up (orders, errors). Rate is computed by the backend.
- **Timer** — durations and counts; with histograms the backend can compute percentiles
  across instances.
- **Gauge** — a current value sampled on read (queue depth, pool usage).
- **Tags** add dimensions (`outcome=success`). Never tag with unbounded values like user
  ids or order ids — every distinct value creates a new time series.

`@Timed` and `@Observed` annotations do the same declaratively (they need AOP on the
classpath). The Observation API (`@Observed`, `ObservationRegistry`) produces metrics *and*
trace spans from one instrumentation point, which Level 4 builds on.

For Prometheus, add `micrometer-registry-prometheus` and scrape `/actuator/prometheus`.

## Securing Actuator

- Serve it on a separate port (`management.server.port: 8081`) that is not exposed
  publicly, and/or
- Protect it with Spring Security — `/actuator/health/**` open to the platform,
  everything else requiring an admin role.
- Never expose `heapdump`, `env`, or `threaddump` publicly. A heap dump contains every
  secret in memory.

## How It Actually Works

Each Actuator endpoint is a bean annotated `@Endpoint` with `@ReadOperation`/
`@WriteOperation` methods. At startup, endpoint discovery filters them by
`management.endpoints.web.exposure.*` and access settings, and a web adapter maps each
operation to an HTTP route under `/actuator` — for Spring MVC, via a dedicated handler
mapping. The same endpoints can be exposed over JMX.

The health endpoint asks a `HealthContributorRegistry` for all `HealthIndicator` beans,
calls them, and combines statuses with a `StatusAggregator` (by default the worst status
wins: `DOWN` beats `OUT_OF_SERVICE` beats `UP`). Groups are just named subsets of
contributors with their own aggregation and detail settings. Boot auto-registers
indicators for what it finds: a `DataSource` gets a `db` indicator, Redis a `redis` one,
and so on.

Metrics flow through Micrometer's `MeterRegistry`. Boot creates a composite registry and
adds one registry per backend it finds on the classpath. Instrumentation for Tomcat,
Spring MVC, JVM memory and GC, HikariCP, and executors is auto-configured; the
`http.server.requests` timer you saw is recorded by an observation filter around every
request.

## Common mistakes

- **Liveness checks that include the database**, causing restart storms during an outage.
- **Exposing all endpoints** (`include: "*"`) on a public port.
- **High-cardinality tags**, overwhelming the metrics backend.
- **Health checks that do real work** (a full query, a payment call) on every poll.
- **Measuring averages only.** Use histograms/percentiles for latency; averages hide the
  slow tail users feel.

## Exercise

1. Add Actuator, enable probes, and call `health`, `health/liveness`, and
   `health/readiness`. Stop your database (if using PostgreSQL) and observe which ones
   change.
2. Add the `orders.placed` counter and a timer to your service and read them via
   `/actuator/metrics/orders.placed`.
3. Add `micrometer-registry-prometheus` and inspect `/actuator/prometheus`. Find the
   `http_server_requests_seconds` series for one endpoint.
4. Move Actuator to port 8081 and secure everything except health with an admin role.
