---
description: "Level 3 of Spring Boot: auto-configuration internals, AOP proxies, N+1 fixes, caching, async and scheduling, Kafka, resilient HTTP clients, and Actuator."
---

# Level 3 · Advanced <span class="level-badge">Internals & Integration</span>

Goal: stop treating Spring as a black box. This level opens the mechanisms you have been
relying on — how auto-configuration decides what exists, how a proxy turns an annotation
into behavior, why lazy loading generates hundreds of queries — and then uses that
understanding to integrate with the world outside your process: caches, background work,
message brokers, other HTTP services, and monitoring systems.

By the end you will build an event-driven order service that publishes and consumes Kafka
messages reliably, calls a flaky downstream service safely, and exposes health and metrics
for operations.

## Modules

1. [How Auto-Configuration Really Works](01-auto-configuration-internals.md) — imports files, `@Conditional` evaluation, ordering, and writing your own starter
2. [AOP & Proxies](02-aop-proxies.md) — JDK vs CGLIB proxies, advice types, writing an aspect, and the limits of proxy-based AOP
3. [The N+1 Problem & Fetch Strategies](03-n-plus-one-fetching.md) — detecting it, join fetch, entity graphs, batch fetching, and DTO queries
4. [Caching with Spring Cache](04-caching.md) — `@Cacheable`, eviction, Caffeine and Redis, keys, and stampedes
5. [Async Execution & Scheduling](05-async-scheduling.md) — `@Async`, executors, virtual threads, `@Scheduled`, and running jobs on one node
6. [Messaging with Apache Kafka](06-messaging-kafka.md) — producers, consumers, partitions, offsets, retries, dead-letter topics, and the outbox pattern
7. [HTTP Clients & Resilience](07-http-clients-resilience.md) — `RestClient`, HTTP interfaces, timeouts, retries, and circuit breakers
8. [Actuator, Health & Metrics](08-actuator-metrics.md) — endpoints, health groups and probes, Micrometer metrics, and securing it all
9. [Bean Lifecycle & Application Events](09-bean-lifecycle-events.md) — from definition to destruction, and decoupling with events
10. [Project — An Event-Driven Order Service](10-project-order-service.md) — orders, Kafka, outbox, a resilient payment client, and observability

## What you need before starting

- Levels 1 and 2, in particular `@Transactional` and the persistence context — this level
  explains *why* they behave as they do.
- Docker, for Kafka and Redis in some lessons. Lessons state clearly which examples need
  external infrastructure and which run with nothing but the JDK.
- Comfort reading stack traces: several lessons ask you to find a proxy class in one.
