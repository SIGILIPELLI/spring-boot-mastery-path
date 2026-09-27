---
description: "When microservices are worth it, and what Spring Cloud Config, Gateway, service discovery, and client-side load balancing do in today's platform landscape."
---

# Microservices & Spring Cloud

Microservices are an organizational tool first and a technical one second. They let
separate teams deploy separately. They also turn every method call between those teams
into a network call that can be slow, fail, or return stale data. This lesson helps you
decide when that trade is worth it, and then maps the Spring Cloud projects onto the
problems they solve — noting which ones modern platforms now solve for you.

## Should you split?

Split when you have **independent teams** stepping on each other in one codebase, parts
with **very different scaling or availability needs**, or parts with **different
compliance boundaries**. Do not split because the codebase feels big, or to "be modern."

The cost is real: distributed data (no joins or transactions across services), network
failure handling everywhere (Level 3's resilience lesson), versioned contracts between
services, many deployables to monitor, and harder debugging. A well-structured modular
monolith (lesson 09) gets you most of the code-organization benefits without those costs,
and makes a later split much easier.

## The problems, and the Spring Cloud answers

| Problem | Spring Cloud project | Platform alternative |
|---------|---------------------|---------------------|
| Central, versioned configuration | **Spring Cloud Config** (server backed by Git, Vault, …) | Kubernetes ConfigMaps/Secrets, cloud parameter stores |
| Finding instances of another service | **Discovery**: Eureka, Consul, Zookeeper clients | Kubernetes Services and DNS |
| Choosing an instance per call | **Spring Cloud LoadBalancer** | Kubernetes Service / service mesh |
| One entry point: routing, auth, rate limits | **Spring Cloud Gateway** (reactive or MVC variant) | Ingress controllers, cloud API gateways |
| Declarative HTTP clients | OpenFeign | Spring's own HTTP interfaces (Level 3) |
| Circuit breakers | Spring Cloud Circuit Breaker (Resilience4j) | Service mesh retries/outlier detection |
| Broadcast config changes | Spring Cloud Bus | Rolling restarts, platform reloaders |

**What is current:** Netflix OSS pieces other than Eureka (Hystrix, Ribbon, Zuul) are long
retired from Spring Cloud; their replacements are Resilience4j, Spring Cloud LoadBalancer,
and Spring Cloud Gateway. OpenFeign is in maintenance mode, and Spring's HTTP interface
clients are the recommended declarative option. On Kubernetes, most teams use the
platform's own discovery and configuration and adopt only Gateway or Config where they add
something specific. Spring Cloud releases are versioned as "release trains" (year-based
names such as 2025.1) that each target a Boot generation; use the train Initializr selects
for your Boot version.

## Worked example: a gateway in front of two services

Spring Cloud Gateway routing `/api/books/**` to the library service and `/api/orders/**` to
the orders service, with a token relay and rate limiting, configured in YAML:

```yaml
spring:
  cloud:
    gateway:
      server:
        webflux:
          routes:
            - id: library
              uri: http://library:8080
              predicates:
                - Path=/api/books/**,/api/authors/**
            - id: orders
              uri: http://orders:8080
              predicates:
                - Path=/api/orders/**
              filters:
                - name: RequestRateLimiter
                  args:
                    redis-rate-limiter.replenishRate: 20
                    redis-rate-limiter.burstCapacity: 40
```

On Kubernetes, `http://library:8080` resolves through the cluster's Service DNS — no
Eureka needed. The property prefix for Gateway routes changed in recent Gateway versions
(the `server.webflux` / `server.webmvc` segment distinguishes the two gateway flavors);
check the Gateway reference for the train you use. This example was not run for the course.

Spring Cloud Config, if you use it, is a Boot app with `@EnableConfigServer` serving
`application.yml` files from a Git repository; clients import it with
`spring.config.import: configserver:http://config:8888`. Its main advantage over ConfigMaps
is version history and review of configuration in Git, plus encrypted values.

## Data ownership

Each service owns its tables. Other services never read them directly; they ask the
owning service's API, or keep their own copy updated from events (Level 3's outbox). The
orders service stores the customer id and a snapshot of what it needs (name, shipping
address at order time) rather than joining to a customer table it does not own.
Cross-service workflows (order → payment → shipment) are sagas: a sequence of local
transactions with compensating actions (refund) when a later step fails.

## How It Actually Works

**Client-side load balancing.** Annotate a `RestClient.Builder` bean with
`@LoadBalanced` and a URL like `http://orders/api/...` is intercepted: a
`LoadBalancerInterceptor` asks a `ReactiveLoadBalancer` for an instance of service
`orders`, which gets the list from a `ServiceInstanceListSupplier` (fed by the discovery
client — Eureka, Consul, or Kubernetes), picks one (round-robin by default), and rewrites
the request URI to that instance's host and port before sending.

**Gateway.** Spring Cloud Gateway's classic flavor runs on WebFlux/Netty. Each request is
matched against route **predicates** (path, host, header, method); the matching route's
**filters** form a chain (rewrite path, add headers, rate-limit, circuit-break), ending
with a filter that proxies the request to the route's URI using a non-blocking HTTP
client and streams the response back. The MVC flavor does the same on the servlet stack.

**Config client.** `spring.config.import: configserver:` adds a config-data loader that,
during `Environment` preparation (before the context is created), fetches properties for
`{application}/{profile}/{label}` from the server and inserts them as a property source —
so they participate in normal precedence and `@ConfigurationProperties` binding.

## Common mistakes

- **Microservices for a single team**, paying distributed-systems costs for no
  organizational benefit.
- **Shared database between services**, which couples them more tightly than a monolith.
- **Chatty synchronous chains** (A calls B calls C calls D) where one slow service stalls
  all of them. Prefer events or aggregate data closer to where it is needed.
- **Running Eureka on Kubernetes** out of habit, duplicating what the platform does.
- **Copying tutorials that use Hystrix, Ribbon, or Zuul.**

## Exercise

1. List the modules of your capstone domain and, for each candidate split, write which of
   the "split when" reasons applies. Keep the ones with no reason in one service.
2. Put Spring Cloud Gateway in front of the library and orders services with Docker Compose,
   routing by path, and add a header filter that adds `X-Gateway: true`. Log it downstream.
3. Replace a hard-coded downstream URL with a `@LoadBalanced` `RestClient.Builder` using a
   static `spring.cloud.discovery.client.simple.instances` list of two instances, and show
   calls alternating.
4. Write a one-page saga design for order → payment → shipment, including the compensating
   action for each step.
