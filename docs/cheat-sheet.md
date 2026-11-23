---
title: "Spring Boot Cheat Sheet: Quick Reference, Beginner to Advanced"
description: "A free Spring Boot cheat sheet: 24 quick-reference tables from every level of the Spring Boot Mastery Path, each linked to its full lesson."
---

# Spring Boot Cheat Sheet

Quick-reference tables from **24 lessons** of the free Spring Boot Mastery Path, in course order from beginner to advanced. Each section links to the full lesson with explanations and runnable examples.

## Level 1 · Entry

### [What Spring and Spring Boot Actually Are](level-1/01-what-is-spring-boot.md)

| Name | What it is | What it gives you |
|------|-----------|-------------------|
| **Spring Framework** | A set of libraries (`spring-core`, `spring-beans`, `spring-context`, `spring-webmvc`, `spring-tx`, …) | The IoC container, AOP, transaction abstraction, Spring MVC, data-access helpers |
| **Spring Boot** | An opinionated layer on top of the Framework | Starters, auto-configuration, externalized configuration, embedded servers, executable jars, Actuator |
| **The Spring portfolio** | Separate projects built on the Framework | Spring Data, Spring Security, Spring Cloud, Spring Kafka, Spring Batch, … |

### [Spring Initializr &amp; Project Structure](level-1/02-initializr-project-structure.md)

*Worked example: build and run the jar*

| Command | What it does |
|---------|-------------|
| `./mvnw spring-boot:run` | Compile and run from source |
| `./mvnw test` | Run the test suite |
| `./mvnw -Dtest=TaskControllerTest test` | Run one test class |
| `./mvnw dependency:tree` | Show every transitive dependency and where it came from |
| `./mvnw package -DskipTests` | Build the jar without tests (for quick local experiments only) |

### [Dependency Injection &amp; the IoC Container](level-1/03-dependency-injection-ioc.md)

*Three injection styles, one recommendation*

| Style | Example | Verdict |
|-------|---------|---------|
| Constructor | `TaskService(TaskRepository r)` | **Use this.** Dependencies are final, required, visible, and tests just call `new`. |
| Setter | `@Autowired void setRepository(TaskRepository r)` | For genuinely optional dependencies, rarely. |
| Field | `@Autowired private TaskRepository r;` | Avoid in production code: hides dependencies, prevents `final`, needs reflection or a container to test. |

### [Beans, Scopes &amp; Component Scanning](level-1/04-beans-component-scanning.md)

*Scopes*

| Scope | One instance per… | Typical use |
|-------|------------------|-------------|
| `singleton` (default) | container | Services, repositories, controllers, clients — almost everything |
| `prototype` | injection point / `getBean` call | Stateful helpers that must not be shared |
| `request` | HTTP request | Per-request context (web apps only) |
| `session` | HTTP session | Per-user state in server-rendered apps |

### [Building REST Controllers](level-1/06-rest-controllers.md)

*Mapping HTTP semantics properly*

| Operation | Method & path | Success status | Body |
|-----------|--------------|----------------|------|
| List | `GET /api/tasks` | 200 | array |
| Read one | `GET /api/tasks/{id}` | 200 (404 if absent) | object |
| Create | `POST /api/tasks` | **201** + `Location` header | created object |
| Replace | `PUT /api/tasks/{id}` | 200 or 204 | object or empty |
| Partial update | `PATCH /api/tasks/{id}/status` | 200 | object |
| Delete | `DELETE /api/tasks/{id}` | **204** | empty |

### [Exception Handling with @ControllerAdvice &amp; ProblemDetail](level-1/08-exception-handling.md)

*Choosing status codes*

| Situation | Status |
|-----------|--------|
| Malformed syntax, failed validation | 400 |
| Not authenticated | 401 (Level 2) |
| Authenticated but not allowed | 403 |
| Resource does not exist | 404 |
| Conflicts with current state (duplicate, version mismatch, limit reached) | 409 |
| Well-formed but semantically invalid for business rules | 422 (some teams prefer 400 for all input errors — pick one and be consistent) |
| Bug or unexpected failure | 500 |
| Dependency down / overloaded | 503 |

### [Logging with SLF4J &amp; Logback](level-1/09-logging.md)

*Levels and configuration*

| Level | Use for |
|-------|---------|
| ERROR | Something failed and a human probably needs to look (unexpected exceptions, 5xx) |
| WARN | Degraded but handled (retry succeeded, fallback used, deprecated config) |
| INFO | Business-significant events and lifecycle (task created, job finished, app started) |
| DEBUG | Details useful when diagnosing (decisions, intermediate values) |
| TRACE | Very high volume internals; rarely enabled |

### [Project — A Task Tracker REST API](level-1/10-project-task-api.md)

*Requirements*

| Endpoint | Behavior |
|----------|---------|
| `POST /api/tasks` | Create with `title` (required, ≤120 chars) and optional `description` (≤2000). 201 + `Location`. 409 when the open-task limit is reached. |
| `GET /api/tasks?status=` | List all tasks, optionally filtered by status, ordered by id. |
| `GET /api/tasks/{id}` | One task or 404. |
| `PATCH /api/tasks/{id}/status` | Body `{"status":"DONE"}`. 400 if missing/invalid. |
| `DELETE /api/tasks/{id}` | 204, or 404 if absent. |

## Level 2 · Intermediate

### [Spring Data JPA &amp; Hibernate Basics](level-2/01-spring-data-jpa-basics.md)

| Layer | What it is |
|-------|-----------|
| **JPA** (Jakarta Persistence) | A *specification*: annotations (`@Entity`, `@Id`, …) and interfaces (`EntityManager`, JPQL). No implementation. |
| **Hibernate ORM** | The implementation Boot uses. It turns entity operations into SQL, manages the persistence context, caches, and lazy loading. |
| **Spring Data JPA** | A layer on top that generates repository implementations from interfaces and integrates JPA with Spring transactions. |

### [Entities &amp; Relationships](level-2/02-entities-relationships.md)

*Fetch types*

| Relationship | JPA default | Recommendation |
|--------------|-------------|----------------|
| `@ManyToOne`, `@OneToOne` | EAGER | **Set `LAZY` explicitly** |
| `@OneToMany`, `@ManyToMany` | LAZY | Keep LAZY |

### [Transactions with @Transactional](level-2/04-transactions.md)

*Propagation*

| Propagation | Behavior |
|-------------|----------|
| `REQUIRED` (default) | Join the existing transaction, or start one |
| `REQUIRES_NEW` | Suspend the current transaction and run in a brand-new one (separate connection) |
| `MANDATORY` | Must be called inside a transaction, else error |
| `SUPPORTS` | Join if one exists, otherwise run without |
| `NOT_SUPPORTED` | Suspend any transaction and run without |
| `NESTED` | Savepoint within the current transaction; needs a transaction manager that supports savepoints, so verify it with your setup before relying on it |

### [Database Migrations with Flyway](level-2/05-flyway-migrations.md)

*Changing a schema without downtime*

| Release | Migration | Code |
|---------|-----------|------|
| 1 | Add `full_name` (nullable) | Write both `name` and `full_name`; read `name` |
| 2 | Backfill `full_name` from `name` | Read `full_name` |
| 3 | Make `full_name` not null | Stop writing `name` |
| 4 | Drop `name` | — |

### [Testing: JUnit 5, Slices &amp; Testcontainers](level-2/06-testing.md)

*The test pyramid, Spring edition*

| Kind | Starts | Proves | Speed |
|------|--------|--------|-------|
| Plain unit test | Nothing | Business logic in one class | Milliseconds |
| `@WebMvcTest` | MVC slice: controllers, advice, converters, filters | HTTP contract: routes, status codes, JSON, validation, security rules | Fast |
| `@DataJpaTest` | JPA slice: entities, repositories, Flyway, a database | Mappings, queries, constraints | Fast–medium |
| `@SpringBootTest` | Entire application context | Wiring and end-to-end flows | Slowest |

### [JWT &amp; the OAuth2 Resource Server](level-2/08-jwt-resource-server.md)

*The roles*

| Role | Example | Responsibility |
|------|---------|---------------|
| Resource owner | The user | Grants access |
| Client | A web or mobile app, another service | Obtains tokens, calls the API |
| Authorization server | Keycloak, Auth0, Okta, Entra ID, Cognito, Spring Authorization Server | Authenticates users, issues tokens |
| Resource server | **Your Spring Boot API** | Validates tokens, enforces access |

### [Project — A Secured Library API with a Database](level-2/10-project-secured-library-api.md)

*Requirements*

| Endpoint | Access | Behavior |
|----------|--------|----------|
| `GET /api/books` | public | All books with author names, in one query |
| `GET /api/books/{id}` | public | One book or 404 problem detail |
| `POST /api/authors` | `SCOPE_books:write` | Create an author, optionally with books |
| `POST /api/books/{id}/retitle` | `SCOPE_books:write` | Change a title; 409 on concurrent modification |

## Level 3 · Advanced

### [How Auto-Configuration Really Works](level-3/01-auto-configuration-internals.md)

*The conditions you will meet*

| Annotation | Matches when |
|-----------|--------------|
| `@ConditionalOnClass` / `OnMissingClass` | A class is / is not on the classpath |
| `@ConditionalOnBean` / `OnMissingBean` | A bean of a type (or name) is / is not already defined |
| `@ConditionalOnProperty` / `OnBooleanProperty` | A property has a value (optionally a specific one; `matchIfMissing` controls the default) |
| `@ConditionalOnWebApplication` / `OnNotWebApplication` | Servlet or reactive web app (or neither) |
| `@ConditionalOnResource` | A resource exists (e.g. a config file) |
| `@ConditionalOnSingleCandidate` | Exactly one bean of a type, or one `@Primary` |
| `@ConditionalOnThreading(Threading.VIRTUAL)` | Virtual threads are enabled |
| `@Profile` | A profile is active (a condition too, under the hood) |

### [AOP &amp; Proxies](level-3/02-aop-proxies.md)

*JDK proxies vs CGLIB*

| | JDK dynamic proxy | CGLIB subclass |
|---|---|---|
| How | Implements the bean's **interfaces** | **Subclasses** the bean's class |
| Injectable as | The interface only | The interface or the class |
| Cannot intercept | Methods not on an interface | `final` methods, `private` methods, `final` classes |
| Used by | Spring Data repositories (interfaces by nature) | Boot's default for everything else (`spring.aop.proxy-target-class=true`) |

### [Caching with Spring Cache](level-3/04-caching.md)

*Choosing a cache provider*

| Provider | Where data lives | Good for | Trade-offs |
|----------|-----------------|----------|-----------|
| `ConcurrentMapCache` (default with no provider) | JVM heap, no eviction | Tests, demos | Unbounded — do not use in production |
| **Caffeine** | JVM heap, per instance | Hot reference data, very low latency | Each instance has its own copy; invalidation does not reach other instances |
| **Redis** | Shared server | Consistency across instances, larger data, survives restarts | Network hop, serialization, another system to run |

### [HTTP Clients &amp; Resilience](level-3/07-http-clients-resilience.md)

*Choosing a client*

| Client | Style | Use it for |
|--------|-------|-----------|
| **`RestClient`** (Framework 6.1+) | Synchronous, fluent | The default choice in Spring MVC apps, especially with virtual threads |
| **`WebClient`** | Reactive (`Mono`/`Flux`) | WebFlux apps, or streaming responses |
| **HTTP interfaces** | Declarative Java interface | Most service-to-service calls; backed by either client |
| `RestTemplate` | Synchronous, template methods | Existing code; in maintenance mode, not for new code |

## Level 4 · Master

### [Microservices &amp; Spring Cloud](level-4/01-spring-cloud-microservices.md)

*The problems, and the Spring Cloud answers*

| Problem | Spring Cloud project | Platform alternative |
|---------|---------------------|---------------------|
| Central, versioned configuration | **Spring Cloud Config** (server backed by Git, Vault, …) | Kubernetes ConfigMaps/Secrets, cloud parameter stores |
| Finding instances of another service | **Discovery**: Eureka, Consul, Zookeeper clients | Kubernetes Services and DNS |
| Choosing an instance per call | **Spring Cloud LoadBalancer** | Kubernetes Service / service mesh |
| One entry point: routing, auth, rate limits | **Spring Cloud Gateway** (reactive or MVC variant) | Ingress controllers, cloud API gateways |
| Declarative HTTP clients | OpenFeign | Spring's own HTTP interfaces (Level 3) |
| Circuit breakers | Spring Cloud Circuit Breaker (Resilience4j) | Service mesh retries/outlier detection |
| Broadcast config changes | Spring Cloud Bus | Rolling restarts, platform reloaders |

### [WebFlux vs Virtual Threads](level-4/02-webflux-vs-virtual-threads.md)

*Decision guide*

| Situation | Choose |
|-----------|--------|
| Typical CRUD/business API on JDBC/JPA | **Servlet stack + virtual threads** |
| Existing Spring MVC codebase needing more concurrency | Virtual threads |
| Streaming, SSE/WebSockets at scale, API gateway | WebFlux |
| Team already fluent in Reactor, fully non-blocking dependencies | WebFlux is fine |
| CPU-bound work | Neither helps; you need more cores or less work |

### [Observability: Logs, Metrics &amp; Tracing](level-4/06-observability-tracing.md)

*The three signals*

| Signal | Answers | In Spring Boot |
|--------|---------|---------------|
| Metrics | *Is something wrong, and how much?* | Micrometer → Prometheus/OTLP |
| Traces | *Where in the request did the time go?* | Micrometer Tracing → OpenTelemetry/Zipkin |
| Logs | *What exactly happened here?* | SLF4J/Logback, with trace ids in MDC |

### [Security Hardening](level-4/07-security-hardening.md)

*Worked example: a hardening review of the library API*

| Finding | Fix |
|---------|-----|
| HMAC JWT secret in `application.yml` | Move to `issuer-uri` with the identity provider's keys; no shared secret in prod |
| `GET /api/books` unbounded | Page it with a max size |
| No CORS config | Allow only the frontend's origin |
| App connects as schema owner | Separate Flyway user and app user |
| Actuator on the main port | `management.server.port: 8081`, not routed publicly |
| No dependency scanning | Dependabot + `trivy` in CI |

### [Capstone — A Production-Ready Service](level-4/10-capstone.md)

*The product*

| Capability | Endpoints / behavior |
|-----------|---------------------|
| Catalog | `GET /api/books?q=&page=` (public, paged), `POST/PATCH /api/books` (librarian) |
| Loans | `POST /api/loans` (member, max 5 active, one active loan per copy), `POST /api/loans/{id}/return` |
| Waitlist | `POST /api/books/{id}/waitlist`, notification on return |
| Overdue | Nightly job marks overdue loans; `GET /api/loans/overdue` (librarian) |

---

Go deeper with the full [Spring Boot Mastery Path](index.md) curriculum or test yourself with the [interview questions](interview-questions.md).
