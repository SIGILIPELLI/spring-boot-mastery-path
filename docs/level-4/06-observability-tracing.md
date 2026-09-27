---
description: "Make Spring Boot observable with Micrometer Observation and Tracing, OpenTelemetry export, trace IDs in logs, and correlated logs, metrics, and traces."
---

# Observability: Logs, Metrics & Tracing

With one service, logs and a few metrics are enough to debug most problems. With several
services, a slow request might have spent its time in any of them. **Distributed tracing**
follows a request across service boundaries; combined with metrics and logs that share a
trace id, you can go from "p99 latency is up" to the exact slow span and its log lines.

## The three signals

| Signal | Answers | In Spring Boot |
|--------|---------|---------------|
| Metrics | *Is something wrong, and how much?* | Micrometer → Prometheus/OTLP |
| Traces | *Where in the request did the time go?* | Micrometer Tracing → OpenTelemetry/Zipkin |
| Logs | *What exactly happened here?* | SLF4J/Logback, with trace ids in MDC |

## Setting up tracing

Micrometer Tracing is a facade over a tracer implementation (OpenTelemetry or Brave). In
Boot 4, the simplest route is the OpenTelemetry starter:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-opentelemetry</artifactId>
</dependency>
```

(In Boot 3, add `micrometer-tracing-bridge-otel` and `opentelemetry-exporter-otlp`.)

```yaml
management:
  tracing:
    sampling:
      probability: 0.1            # sample 10%; default is 0.1, use 1.0 locally
  opentelemetry:
    tracing:
      export:
        otlp:
          endpoint: http://otel-collector:4318/v1/traces
```

The exact property names for OTLP export have moved between Boot versions — confirm them
in the reference documentation for your release. Send data to an **OpenTelemetry
Collector**, which forwards it to whichever backend you use (Jaeger, Tempo, Zipkin, or a
commercial vendor), so the application does not need to know. Running a collector and
backend needs Docker and was not done for this course.

## What you get automatically

Because Boot's instrumentation is built on the **Observation API**, one observation
produces both a timer metric and a span:

- Incoming HTTP requests (`http.server.requests`), with the route as the span name.
- Outgoing calls through Boot-built `RestClient`/`WebClient` builders — which also
  **propagate** trace context in the W3C `traceparent` header, so the next service
  continues the same trace.
- JDBC via the datasource-micrometer integration (if added), Kafka producer and consumer
  (with observation enabled on the template/listener), `@Scheduled` methods, and more.
- Trace and span ids placed in the logging MDC, and included in Boot's default log format
  once tracing is on. Structured JSON logs (Level 1, lesson 09) include them as fields.

## Custom observations

```java
@Service
class PricingService {
    private final ObservationRegistry registry;

    PricingService(ObservationRegistry registry) { this.registry = registry; }

    Price quote(Cart cart) {
        return Observation.createNotStarted("pricing.quote", registry)
                .lowCardinalityKeyValue("cart.size.bucket", cart.size() > 10 ? "large" : "small")
                .highCardinalityKeyValue("cart.id", cart.id())
                .observe(() -> computePrice(cart));
    }
}
```

**Low-cardinality** key-values become metric tags *and* span attributes; **high-cardinality**
ones (ids) go only on spans, where they are fine. `@Observed(name = "pricing.quote")` does
the same declaratively via AOP.

## Worked example: following one slow request

1. An alert fires: p99 of `http.server.requests{uri="/api/orders"}` doubled.
2. In the tracing backend, filter traces for that route above the p99 threshold. The
   waterfall shows `POST /api/orders` → `http post payments/charges` taking most of the time.
3. The payment span has an attribute showing retries; its child spans show three attempts,
   the first two timing out at 2 s each.
4. Copy the trace id, search the logs: the payment client's WARN lines for those retries
   carry the same trace id, with the error details.
5. Conclusion: the provider degraded; the retry policy multiplied latency. Adjust
   the circuit breaker thresholds and the latency budget.

This path — metric → trace → logs — only works if all three share ids and naming, which is
why you let Boot instrument everything through one registry.

## Sampling

Tracing every request is costly at volume. **Head sampling** (the probability above) decides
at the start of a trace, and the decision propagates so the trace is complete or absent in
every service. **Tail sampling**, done in the collector, keeps all errors and slow traces
while sampling the rest — more useful, but it requires the collector to buffer spans.

## How It Actually Works

An `Observation` has a lifecycle: `start`, optional events and errors, `stop`. The
`ObservationRegistry` holds **handlers**: the metrics handler starts a `Timer.Sample` on
start and records it on stop; the tracing handler creates a span on start, puts it *in
scope* (making it current, so child observations become child spans), and ends it on stop.
One instrumentation point, two signals.

The current span lives in a `ThreadLocal` managed by the tracer. Crossing threads (`@Async`,
executors, Reactor) requires **context propagation**: Micrometer's context-propagation
library captures thread-local values (span, MDC) when a task is submitted and restores them
on the executing thread; Boot wires this for its auto-configured executors, and Reactor
applies it automatically when enabled. Crossing *processes* uses a propagator: the outgoing
HTTP instrumentation injects `traceparent: 00-<trace-id>-<parent-span-id>-<flags>` into the
request headers; the receiving service's server instrumentation extracts it and starts its
span as a child of that parent.

## Common mistakes

- **Building your own `RestClient` without Boot's builder**, breaking propagation.
- **High-cardinality metric tags** (user or order ids).
- **Sampling at 100% in production** without checking cost, or at 0% and wondering why there
  are no traces.
- **Logging without trace ids**, so logs cannot be joined to traces.
- **Treating dashboards as observability.** Define SLOs (for example, 99% of requests under
  300 ms) and alert on them.

## Exercise

1. Add tracing to the library and orders services with sampling at 1.0 locally, run Jaeger or
   Zipkin in Docker, and view a trace spanning both services.
2. Confirm your log lines include the trace id and find a trace from a log line.
3. Add a custom observation around a pricing or search method with one low- and one
   high-cardinality key-value, and find it both in `/actuator/metrics` and in a span.
4. Make an `@Async` method log with and without context propagation and compare the trace
   ids.
