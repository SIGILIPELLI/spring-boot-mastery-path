---
description: "Level 4 of Spring Boot: Spring Cloud, WebFlux vs virtual threads, tuning, native images, containers, tracing, hardening, deployment, and a capstone."
---

# Level 4 · Master <span class="level-badge">Production</span>

Goal: run Spring Boot services in production with confidence. The earlier levels taught
you to build a correct service; this level is about the decisions around it — when to
split into microservices (and when not to), which concurrency model to use, how to find
where time and memory go, how to package and ship, how to see inside a running system,
and how to harden it.

Many topics here involve infrastructure — Kubernetes, cloud platforms, tracing backends —
that cannot be run inside a lesson page. Lessons say plainly what was run for the course
and what was not, and focus on the Spring side you control.

## Modules

1. [Microservices & Spring Cloud](01-spring-cloud-microservices.md) — when to split, and what Config, Gateway, discovery, and load balancing do today
2. [WebFlux vs Virtual Threads](02-webflux-vs-virtual-threads.md) — two answers to blocking I/O, and how to choose
3. [Performance Tuning](03-performance-tuning.md) — measuring first, connection pools, JPA batching, JVM memory, and startup
4. [GraalVM Native Images](04-graalvm-native.md) — AOT processing, reachability metadata, and the real trade-offs
5. [Containerizing Spring Boot](05-containerization.md) — buildpacks, layered Dockerfiles, JVM memory in containers, and graceful shutdown
6. [Observability: Logs, Metrics & Tracing](06-observability-tracing.md) — Micrometer Observation, OpenTelemetry, and correlated signals
7. [Security Hardening](07-security-hardening.md) — secrets, headers, CORS, rate limiting, dependency scanning, and least privilege
8. [Deploying to Production](08-deployment.md) — configuration per environment, rolling deploys, probes, and migrations in the pipeline
9. [Architecture: Modular Monoliths & Boundaries](09-modular-architecture.md) — package structure, Spring Modulith, and enforcing boundaries
10. [Capstone — A Production-Ready Service](10-capstone.md) — everything together, with a launch checklist

## What you need before starting

- Levels 1–3. The capstone reuses the library and order services.
- Docker. Access to a Kubernetes cluster (kind or minikube locally is enough) helps for
  lessons 05 and 08 but is not required to follow them.
- Java 21 or newer for the virtual-thread material.
