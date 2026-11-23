---
title: "Spring Boot Interview Questions and Answers (180 Questions)"
description: "180 Spring Boot interview questions with answers and explanations, from beginner to advanced, organised by level. Free, from the Spring Boot Mastery Path."
---

# Spring Boot Interview Questions and Answers

**180 questions** with the correct answer and a short explanation, organised from beginner (Level 1) to advanced (Level 4). Every question comes from a lesson in the free Spring Boot Mastery Path, so you can open the lesson to go deeper.

!!! tip "Practise, don't just read"
    Each lesson also ends with an interactive quiz, and the [final quiz](quiz.md) tests all four levels. Try to answer each question below before reading the answer.

## Level 1 · Entry

### [What Spring and Spring Boot Actually Are](level-1/01-what-is-spring-boot.md)

**Q1.** What does Spring Boot add on top of the Spring Framework?

**Answer:** Opinionated auto-configuration, starter dependencies and an embedded server so you can run an app with minimal setup

Boot builds on Spring; it does not replace it.

**Q2.** What does the @SpringBootApplication annotation combine?

**Answer:** @Configuration, @EnableAutoConfiguration and @ComponentScan

It marks the main configuration class.

**Q3.** How do you run a Boot app during development?

**Answer:** ./mvnw spring-boot:run or ./gradlew bootRun (or run the main class)

The fat jar is run with java -jar.

**Q4.** What is a Spring Boot starter?

**Answer:** A dependency descriptor that pulls in a curated set of libraries, such as spring-boot-starter-web

It reduces dependency management.

### [Spring Initializr &amp; Project Structure](level-1/02-initializr-project-structure.md)

**Q5.** What is the main purpose of start.spring.io?

**Answer:** Generate a project skeleton with chosen build tool, Java version and dependencies

The Initializr.

**Q6.** Where does Spring Boot look for application.properties or application.yml by default?

**Answer:** On the classpath, typically src/main/resources

Externalised config can override it.

**Q7.** Where should the main class live relative to other packages?

**Answer:** In a root package so component scanning covers sub-packages

Scanning starts at its package.

**Q8.** What does the Maven/Gradle wrapper provide?

**Answer:** A pinned build tool version so everyone builds the same way

mvnw and gradlew.

### [Dependency Injection &amp; the IoC Container](level-1/03-dependency-injection-ioc.md)

**Q9.** This container-free model shows constructor injection; what does it print?

```java
public class Main { public static void main(String[] a) { System.out.println(new Service(new FixedClock(42)).stamp()); System.out.println(new Service(() -> 7).stamp()); } }
interface Clock { long now(); }
record FixedClock(long t) implements Clock { public long now() { return t; } }
class Service { private final Clock clock; Service(Clock c) { this.clock = c; } String stamp() { return "t=" + clock.now(); } }
```

**Answer:** t=42
t=7

Depending on an interface lets you inject a fake in tests; Spring does the wiring. (Plain-Java model of the idea; Spring itself was not run here, so framework behaviour such as annotations is described, not executed.) The code was compiled, run and its output checked.

**Q10.** What is Inversion of Control?

**Answer:** The framework creates and wires objects instead of your code constructing them directly

Dependencies are injected.

**Q11.** Which injection style is preferred in modern Spring?

**Answer:** Constructor injection (allows final fields and easy testing)

A single constructor needs no @Autowired.

**Q12.** What happens if two beans match an injection point?

**Answer:** Spring fails with NoUniqueBeanDefinitionException unless you qualify or mark one @Primary

Use @Qualifier.

### [Beans, Scopes &amp; Component Scanning](level-1/04-beans-component-scanning.md)

**Q13.** Which annotations make a class a candidate bean via component scanning?

**Answer:** @Component and its specialisations @Service, @Repository and @Controller

@Configuration classes with @Bean methods also define beans.

**Q14.** What is the default bean scope?

**Answer:** Singleton: one shared instance per application context

Beans must be thread-safe.

**Q15.** Why shouldn't singleton beans hold per-request mutable state?

**Answer:** The same instance serves all threads, causing race conditions

Keep them stateless.

**Q16.** What does @Bean do on a method in a @Configuration class?

**Answer:** Registers the returned object as a bean, useful for third-party classes

Method name is the bean name.

### [Configuration: application.yml, Profiles &amp; @ConfigurationProperties](level-1/05-configuration-profiles.md)

**Q17.** How do you bind configuration to a typed object?

**Answer:** @ConfigurationProperties with a prefix

Validates and groups properties.

**Q18.** How do you activate a profile?

**Answer:** spring.profiles.active=prod (or SPRING_PROFILES_ACTIVE)

application-prod.yml overrides.

**Q19.** What is the property precedence idea in Boot?

**Answer:** Command-line and environment values override files, which override defaults

Externalised configuration.

**Q20.** Where should secrets such as DB passwords be supplied in production?

**Answer:** Via environment variables or a secrets manager, not committed files

Never commit secrets.

### [Building REST Controllers](level-1/06-rest-controllers.md)

**Q21.** What does @RestController do?

**Answer:** Combines @Controller and @ResponseBody so return values are written as the response body (JSON)

Jackson converts objects.

**Q22.** Which annotation maps an HTTP GET to a method?

**Answer:** @GetMapping

@PostMapping, @PutMapping, @DeleteMapping.

**Q23.** How do you read a path variable?

**Answer:** @PathVariable on a method parameter

/tasks/{id}.

**Q24.** What status should a successful creation return?

**Answer:** 201 Created, ideally with a Location header

ResponseEntity.created(uri).

### [Request Binding, Responses &amp; Validation](level-1/07-request-response-validation.md)

**Q25.** This model of Bean Validation checks a request; what does it print?

```java
import java.util.*;
public class Main { public static void main(String[] a) { System.out.println(V.check(new Req("x", 3))); System.out.println(V.check(new Req(" ", 9))); } }
record Req(String title, int priority) {}
class V { static List<String> check(Req r) { var e = new ArrayList<String>(); if (r.title() == null || r.title().isBlank()) e.add("title must not be blank"); if (r.priority() < 1 || r.priority() > 5) e.add("priority must be 1-5"); return e; } }
```

**Answer:** []
[title must not be blank, priority must be 1-5]

Spring does this with @NotBlank and @Min/@Max plus @Valid; failures become 400 responses. (Plain-Java model of the idea; Spring itself was not run here, so framework behaviour such as annotations is described, not executed.) The code was compiled, run and its output checked.

**Q26.** Which annotation triggers validation of a request body?

**Answer:** @Valid (or @Validated) on the parameter

Violations raise MethodArgumentNotValidException.

**Q27.** Why use DTOs instead of exposing entities in the API?

**Answer:** To control the contract, avoid leaking internals and prevent mass assignment

Map between entity and DTO.

**Q28.** Which HTTP status does a failed validation map to?

**Answer:** 400 Bad Request (or 422)

Return field-level errors.

### [Exception Handling with @ControllerAdvice &amp; ProblemDetail](level-1/08-exception-handling.md)

**Q29.** What does @RestControllerAdvice with @ExceptionHandler do?

**Answer:** Handles exceptions globally and turns them into consistent error responses

Centralise error handling.

**Q30.** What is Problem Details (RFC 9457/7807)?

**Answer:** A standard JSON format for HTTP API errors

Spring supports ProblemDetail.

**Q31.** Why not return stack traces to clients?

**Answer:** They leak internals and help attackers

Log them server-side.

**Q32.** Which exception typically maps to 404?

**Answer:** A custom not-found exception handled to return 404

ResponseStatusException is an option.

### [Logging with SLF4J &amp; Logback](level-1/09-logging.md)

**Q33.** Which logging facade does Spring Boot use by default?

**Answer:** SLF4J with Logback

Configure levels via logging.level.*.

**Q34.** Why avoid string concatenation in log calls?

**Answer:** Use parameterised messages like log.info("x {}", v) to skip work when the level is off

Cheaper and cleaner.

**Q35.** What should logs never contain?

**Answer:** Passwords, tokens and sensitive personal data

Mask them.

**Q36.** What helps trace a request across log lines?

**Answer:** A correlation ID in the MDC

Add it via a filter.

### [Project — A Task Tracker REST API](level-1/10-project-task-api.md)

**Q37.** What does the Task API project combine?

**Answer:** Controllers, validation, error handling, configuration and logging for a CRUD API

Level 1 capstone.

**Q38.** What does this in-memory repository model print for the CRUD flow?

```java
import java.util.*;
public class Main { public static void main(String[] a) { var r = new Repo(); r.add("a"); r.add("b"); r.complete(2); System.out.println(r.m.values()); System.out.println(r.find(9).isPresent()); } }
record Task(int id, String title, boolean done) {}
class Repo { Map<Integer,Task> m = new TreeMap<>(); int next = 1; Task add(String t) { var x = new Task(next++, t, false); m.put(x.id(), x); return x; } Optional<Task> find(int id) { return Optional.ofNullable(m.get(id)); } Task complete(int id) { var t = m.get(id); var d = new Task(t.id(), t.title(), true); m.put(id, d); return d; } }
```

**Answer:** [Task[id=1, title=a, done=false], Task[id=2, title=b, done=true]]
false

Optional models a missing task; a controller would map the empty case to 404. (Plain-Java model of the idea; Spring itself was not run here, so framework behaviour such as annotations is described, not executed.) The code was compiled, run and its output checked.

**Q39.** Which HTTP verb replaces a whole resource idempotently?

**Answer:** PUT

PATCH is a partial update.

**Q40.** What should the API return when a task does not exist?

**Answer:** 404 with an error body

Be consistent.

## Level 2 · Intermediate

### [Spring Data JPA &amp; Hibernate Basics](level-2/01-spring-data-jpa-basics.md)

**Q41.** What is Spring Data JPA?

**Answer:** A layer that generates repository implementations on top of JPA (Hibernate) from interfaces

extends JpaRepository.

**Q42.** What does JpaRepository&lt;T, ID&gt; give you?

**Answer:** CRUD, paging, sorting and derived query methods

Declare an interface; no implementation needed.

**Q43.** What does spring.jpa.hibernate.ddl-auto=update risk in production?

**Answer:** Unreviewed schema changes; use migrations (Flyway) instead

Use validate in production.

**Q44.** What does a DataSource configure?

**Answer:** The connection (and pool) to the database

HikariCP is the default pool.

### [Entities &amp; Relationships](level-2/02-entities-relationships.md)

**Q45.** Which annotation marks a class as a JPA entity?

**Answer:** @Entity (with an @Id field)

Needs a no-arg constructor.

**Q46.** What does @OneToMany(mappedBy = "order") express?

**Answer:** The inverse side of a bidirectional relationship owned by the 'order' field on the other entity

The owning side holds the foreign key.

**Q47.** What is the default fetch type for @ManyToOne?

**Answer:** EAGER

Collections (@OneToMany) default to LAZY.

**Q48.** Why override equals and hashCode carefully on entities?

**Answer:** Identity and proxies can break naive implementations; base them on a stable business or id key

Be careful with generated ids.

### [Repositories, Derived Queries &amp; @Query](level-2/03-repositories-queries.md)

**Q49.** What does a derived query findByStatusAndPriorityGreaterThan do?

**Answer:** Spring Data parses the method name into a query

Keep names readable.

**Q50.** When use @Query?

**Answer:** For complex or explicit JPQL/native queries

Use named parameters.

**Q51.** What does Pageable add to a repository method?

**Answer:** Pagination and sorting for large result sets

Return Page&lt;T&gt;.

**Q52.** Why prefer projections for read-heavy endpoints?

**Answer:** Fetch only needed columns instead of whole entities

Interface or record projections.

### [Transactions with @Transactional](level-2/04-transactions.md)

**Q53.** What does @Transactional do?

**Answer:** Wraps the method in a transaction that commits on success and rolls back on unchecked exceptions by default

Checked exceptions do not roll back by default.

**Q54.** Why does calling a @Transactional method from within the same class often not start a transaction?

**Answer:** The call bypasses the proxy that applies the transaction advice (self-invocation)

Call through another bean.

**Q55.** What isolation problem does optimistic locking (@Version) address?

**Answer:** Lost updates from concurrent modification, detected at write time

An OptimisticLockException is thrown.

**Q56.** Where should @Transactional normally go?

**Answer:** On service methods representing a business operation

Keep transactions short.

### [Database Migrations with Flyway](level-2/05-flyway-migrations.md)

**Q57.** What is Flyway?

**Answer:** A database migration tool that applies versioned SQL scripts in order

V1__init.sql.

**Q58.** Why never edit an applied migration?

**Answer:** Flyway's checksum would fail; add a new migration instead

Migrations are immutable history.

**Q59.** What should ddl-auto be in production with Flyway?

**Answer:** validate or none

Let migrations own the schema.

**Q60.** How do you handle risky migrations?

**Answer:** Expand/contract in small steps, tested against production-like data

Backward compatible changes.

### [Testing: JUnit 5, Slices &amp; Testcontainers](level-2/06-testing.md)

**Q61.** What does @SpringBootTest do?

**Answer:** Loads the full application context for an integration test

Slower than slices.

**Q62.** What does @WebMvcTest do?

**Answer:** Loads only the web layer for controller tests with MockMvc

Use @MockBean for collaborators.

**Q63.** What does @DataJpaTest provide?

**Answer:** A slice with JPA and an embedded or container database, rolling back after each test

Testcontainers match production better.

**Q64.** Why prefer Testcontainers over H2 for DB tests?

**Answer:** The same database engine as production avoids dialect surprises

Needs Docker.

### [Spring Security Fundamentals](level-2/07-spring-security-basics.md)

**Q65.** What does adding spring-boot-starter-security do by default?

**Answer:** Secures all endpoints with basic authentication and a generated password

Customise with a SecurityFilterChain bean.

**Q66.** What does the SecurityFilterChain define?

**Answer:** Which requests need which authentication and authorisation, plus CSRF, CORS and sessions

Replaces WebSecurityConfigurerAdapter.

**Q67.** How should passwords be stored?

**Answer:** Hashed with a strong adaptive algorithm via PasswordEncoder (BCrypt or Argon2)

DelegatingPasswordEncoder is the default.

**Q68.** Why is CSRF protection usually disabled only for stateless token APIs?

**Answer:** Browsers do not send tokens automatically, so cookie-based CSRF attacks do not apply

Keep it for session cookies.

### [JWT &amp; the OAuth2 Resource Server](level-2/08-jwt-resource-server.md)

**Q69.** This JDK-only check verifies an HS256 JWT signature (the core of what a resource server does). What does it print?

```java
import javax.crypto.*; import javax.crypto.spec.*; import java.util.*; import java.nio.charset.StandardCharsets;
public class Main {
 static String b64(byte[] b){ return Base64.getUrlEncoder().withoutPadding().encodeToString(b); }
 static String sign(String data, String key) throws Exception { Mac m = Mac.getInstance("HmacSHA256"); m.init(new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "HmacSHA256")); return b64(m.doFinal(data.getBytes(StandardCharsets.UTF_8))); }
 static boolean verify(String jwt, String key) throws Exception { var p = jwt.split("\\."); return p.length == 3 && java.security.MessageDigest.isEqual(sign(p[0] + "." + p[1], key).getBytes(), p[2].getBytes()); }
 public static void main(String[] a) throws Exception {
  String h = b64("{\"alg\":\"HS256\",\"typ\":\"JWT\"}".getBytes()), p = b64("{\"sub\":\"ana\"}".getBytes());
  String jwt = h + "." + p + "." + sign(h + "." + p, "secret-key");
  System.out.println(verify(jwt, "secret-key") + " " + verify(jwt, "other") + " " + verify(h + "." + b64("{\"sub\":\"root\"}".getBytes()) + "." + jwt.split("\\.")[2], "secret-key"));
 }
}
```

**Answer:** true false false

A valid signature verifies only with the right key and untampered payload. (Plain-Java model of the idea; Spring itself was not run here, so framework behaviour such as annotations is described, not executed.) The code was compiled, run and its output checked.

**Q70.** What does an OAuth2 resource server do with a bearer JWT?

**Answer:** Validates its signature, issuer, audience and expiry, then maps claims to authorities

spring-boot-starter-oauth2-resource-server.

**Q71.** Why is a JWT payload not secret?

**Answer:** It is only base64url-encoded and signed, not encrypted; anyone can read it

Do not put sensitive data in claims.

**Q72.** Which claims should always be validated?

**Answer:** exp, iss and aud (plus signature and algorithm)

Reject alg=none and unexpected algorithms.

### [API Documentation with springdoc-openapi](level-2/09-openapi-springdoc.md)

**Q73.** What is springdoc-openapi for?

**Answer:** Generating OpenAPI documentation and Swagger UI from your controllers

Annotate for descriptions.

**Q74.** Why publish an OpenAPI spec?

**Answer:** Clients can generate code and tests, and the contract is explicit

Keep it accurate in CI.

**Q75.** Where should the OpenAPI UI be exposed in production?

**Answer:** Restricted or disabled unless intended to be public

Reduce attack surface.

**Q76.** What keeps docs and code in sync?

**Answer:** Generate from code and verify in CI (contract tests)

Avoid drift.

### [Project — A Secured Library API with a Database](level-2/10-project-secured-library-api.md)

**Q77.** What does the secured library API project combine?

**Answer:** CRUD, JPA, Flyway, security with JWT roles, tests and OpenAPI

Level 2 capstone.

**Q78.** How do you restrict an endpoint to admins?

**Answer:** hasRole('ADMIN') in the filter chain or @PreAuthorize

Method security needs enabling.

**Q79.** What should tests verify for secured endpoints?

**Answer:** 401 without a token, 403 with insufficient role, 200 with the right role

Test all three.

**Q80.** What belongs in the migration for a library schema?

**Answer:** Tables, constraints and indexes for the access patterns

Version them.

## Level 3 · Advanced

### [How Auto-Configuration Really Works](level-3/01-auto-configuration-internals.md)

**Q81.** What does auto-configuration do?

**Answer:** Registers beans conditionally based on the classpath and properties

@ConditionalOnClass and friends.

**Q82.** How can you see what was auto-configured?

**Answer:** Run with --debug to print the conditions evaluation report (or actuator conditions)

Helps debug surprises.

**Q83.** How do you exclude an auto-configuration?

**Answer:** @SpringBootApplication(exclude = ...) or spring.autoconfigure.exclude

Prefer properties.

**Q84.** What do your own beans do to a conditional default?

**Answer:** They usually win: @ConditionalOnMissingBean backs off

Override by defining the bean.

### [AOP &amp; Proxies](level-3/02-aop-proxies.md)

**Q85.** A Spring AOP-style proxy wraps a call. What does this JDK dynamic-proxy model print?

```java
import java.lang.reflect.*;
public class Main { public static void main(String[] a) { Svc target = x -> "ran " + x; Svc proxy = (Svc) Proxy.newProxyInstance(Svc.class.getClassLoader(), new Class<?>[]{Svc.class}, (p, m, args) -> { System.out.println("before"); Object r = m.invoke(target, args); System.out.println("after"); return r; }); System.out.println(proxy.run("job")); } }
interface Svc { String run(String x); }
```

**Answer:** before
after
ran job

Spring wraps beans in proxies so advice (transactions, caching) runs around calls. (Plain-Java model of the idea; Spring itself was not run here, so framework behaviour such as annotations is described, not executed.) The code was compiled, run and its output checked.

**Q86.** Why does self-invocation bypass AOP advice?

**Answer:** The call goes directly to this, not through the proxy that applies the advice

Extract to another bean.

**Q87.** Which kinds of proxies does Spring use?

**Answer:** JDK dynamic proxies (interfaces) and CGLIB subclass proxies

Boot defaults to CGLIB.

**Q88.** What is cross-cutting logic suited to AOP?

**Answer:** Transactions, security checks, logging and caching

Keep aspects small.

### [The N+1 Problem &amp; Fetch Strategies](level-3/03-n-plus-one-fetching.md)

**Q89.** This model counts queries for N+1 versus a join. What does it print?

```java
import java.util.*;
public class Main { public static void main(String[] x) { var d = new Db(); for (int a : d.authors()) d.books(a); System.out.println("n+1: " + d.queries); var e = new Db(); e.booksFor(e.authors()); System.out.println("batched: " + e.queries); } }
class Db { int queries = 0; List<Integer> authors() { queries++; return List.of(1,2,3,4); } List<String> books(int a) { queries++; return List.of("b" + a); } Map<Integer,List<String>> booksFor(List<Integer> ids) { queries++; var m = new TreeMap<Integer,List<String>>(); for (int i : ids) m.put(i, List.of("b" + i)); return m; } }
```

**Answer:** n+1: 5
batched: 2

One query for authors plus one per author versus two queries total. (Plain-Java model of the idea; Spring itself was not run here, so framework behaviour such as annotations is described, not executed.) The code was compiled, run and its output checked.

**Q90.** What is the N+1 problem?

**Answer:** One query to load a list plus one extra query per item to load its associations

Fix with fetch joins, entity graphs or batch fetching.

**Q91.** How do you spot N+1 in development?

**Answer:** Enable SQL logging or use a query counter in tests

Assert query counts.

**Q92.** What is a pitfall of eager fetching everywhere?

**Answer:** Loads data you do not need and can create cartesian products

Fetch per use case.

### [Caching with Spring Cache](level-3/04-caching.md)

**Q93.** What does @Cacheable do?

**Answer:** Stores a method's result keyed by its arguments and returns it on later calls

Needs @EnableCaching.

**Q94.** What is cache invalidation's rule of thumb?

**Answer:** Evict or update when the underlying data changes (@CacheEvict, TTLs)

Prefer TTLs as a safety net.

**Q95.** Why is caching mutable objects risky?

**Answer:** Callers can modify the cached instance; cache immutable values

DTOs or records.

**Q96.** Which provider suits a distributed cache?

**Answer:** Redis (versus in-process Caffeine for single-instance)

Shared across instances.

### [Async Execution &amp; Scheduling](level-3/05-async-scheduling.md)

**Q97.** What does @Async do?

**Answer:** Runs a method on a separate executor thread, returning immediately

Needs @EnableAsync; self-invocation bypasses it.

**Q98.** What does @Scheduled(fixedRate = 5000) do?

**Answer:** Runs the method every 5 seconds from the start of the previous run

fixedDelay waits after completion.

**Q99.** What is a problem with @Scheduled across multiple instances?

**Answer:** It runs on every instance unless you use a lock (such as ShedLock)

Avoid duplicates.

**Q100.** Why configure a bounded executor for async work?

**Answer:** Unbounded threads or queues can exhaust memory

Set pool and queue sizes.

### [Messaging with Apache Kafka](level-3/06-messaging-kafka.md)

**Q101.** What is Kafka's key guarantee per partition?

**Answer:** Records in a partition are ordered

Use keys to route related events.

**Q102.** What delivery semantics should consumers assume by default?

**Answer:** At-least-once, so handlers must be idempotent

Deduplicate by key.

**Q103.** What does @KafkaListener do?

**Answer:** Consumes records from a topic and invokes the method

Configure group id.

**Q104.** What should happen to messages that keep failing?

**Answer:** Retry with backoff, then send to a dead-letter topic

Alert on DLT.

### [HTTP Clients &amp; Resilience](level-3/07-http-clients-resilience.md)

**Q105.** What is a circuit breaker?

**Answer:** Stops calling a failing dependency for a while to protect the caller and let it recover

Resilience4j.

**Q106.** What must you set on every outgoing HTTP call?

**Answer:** Connect and read timeouts

Hung calls block threads.

**Q107.** Why combine retries with backoff and jitter?

**Answer:** To avoid retry storms hammering a recovering service

Limit attempts.

**Q108.** Which Spring client is current for blocking HTTP?

**Answer:** RestClient (or WebClient for reactive)

RestTemplate is in maintenance.

### [Actuator, Health &amp; Metrics](level-3/08-actuator-metrics.md)

**Q109.** What do Actuator endpoints provide?

**Answer:** Health, metrics, info, environment and more for monitoring

Expose selectively.

**Q110.** Why secure Actuator endpoints?

**Answer:** They can leak configuration and allow dangerous operations

Expose only health and info publicly.

**Q111.** What does Micrometer do?

**Answer:** Provides a vendor-neutral metrics facade (Prometheus, etc.)

Timers, counters, gauges.

**Q112.** What are liveness and readiness probes?

**Answer:** Whether the app is alive and whether it can serve traffic

Kubernetes uses them.

### [Bean Lifecycle &amp; Application Events](level-3/09-bean-lifecycle-events.md)

**Q113.** This model of bean lifecycle callbacks prints what order?

```java
import java.util.*;
public class Main { public static void main(String[] a) { var b = new Bean(); b.inject(); b.postConstruct(); b.preDestroy(); System.out.println(b.log); } }
class Bean { List<String> log = new ArrayList<>(); Bean() { log.add("construct"); } void inject() { log.add("inject"); } void postConstruct() { log.add("init"); } void preDestroy() { log.add("destroy"); } }
```

**Answer:** [construct, inject, init, destroy]

Instantiate, inject dependencies, run @PostConstruct, later @PreDestroy. (Plain-Java model of the idea; Spring itself was not run here, so framework behaviour such as annotations is described, not executed.) The code was compiled, run and its output checked.

**Q114.** Which annotation runs initialisation after dependencies are injected?

**Answer:** @PostConstruct

Or InitializingBean.

**Q115.** What do application events enable?

**Answer:** Decoupled communication between components via ApplicationEventPublisher and @EventListener

@TransactionalEventListener runs after commit.

**Q116.** What is ApplicationReadyEvent for?

**Answer:** Work that should run once the application is fully started

Avoid heavy startup work.

### [Project — An Event-Driven Order Service](level-3/10-project-order-service.md)

**Q117.** What does the order service project combine?

**Answer:** JPA, transactions, messaging, caching, resilience and metrics in a realistic service

Level 3 capstone.

**Q118.** How do you avoid double-processing an order event?

**Answer:** Idempotency keys and unique constraints

At-least-once delivery.

**Q119.** What belongs in a transaction boundary for placing an order?

**Answer:** Persisting the order and the outbox record together

Outbox pattern.

**Q120.** What does the outbox pattern solve?

**Answer:** Atomically saving state and the event to publish, avoiding lost or phantom messages

A relay publishes later.

## Level 4 · Master

### [Microservices &amp; Spring Cloud](level-4/01-spring-cloud-microservices.md)

**Q121.** What does Spring Cloud provide?

**Answer:** Building blocks for distributed systems such as config, discovery, gateway and resilience

Many are optional with Kubernetes.

**Q122.** What does an API gateway do?

**Answer:** A single entry point handling routing, auth, rate limiting and cross-cutting concerns

Spring Cloud Gateway.

**Q123.** What is externalised config in microservices?

**Answer:** Central config (config server or Kubernetes) so services read environment-specific settings

Refreshable.

**Q124.** Why do microservices need distributed tracing?

**Answer:** To follow one request across services and find where time goes

Trace IDs propagate.

### [WebFlux vs Virtual Threads](level-4/02-webflux-vs-virtual-threads.md)

**Q125.** What do virtual threads (Java 21) simplify?

**Answer:** Blocking-style code that scales to many concurrent requests without reactive complexity

spring.threads.virtual.enabled=true.

**Q126.** When does WebFlux still make sense?

**Answer:** For streaming, backpressure and fully non-blocking pipelines end to end

Mixed blocking calls defeat it.

**Q127.** What is the danger of blocking calls on an event-loop thread?

**Answer:** It stalls other requests on that thread

Use bounded elastic schedulers.

**Q128.** What should drive the choice between the models?

**Answer:** Workload shape and team skills, measured rather than assumed

Benchmark.

### [Performance Tuning](level-4/03-performance-tuning.md)

**Q129.** What should you do first when an endpoint is slow?

**Answer:** Measure with metrics, traces and profiling to find the bottleneck

Database is the usual suspect.

**Q130.** What does connection pool exhaustion look like?

**Answer:** Requests waiting for connections and timing out under load

Size pools to the database.

**Q131.** Why enable response compression and HTTP caching carefully?

**Answer:** They trade CPU for bandwidth and need correct cache headers

Measure benefits.

**Q132.** What does JVM warm-up affect?

**Answer:** Initial request latency while the JIT compiles hot code

Test with realistic load.

### [GraalVM Native Images](level-4/04-graalvm-native.md)

**Q133.** What does GraalVM native image offer?

**Answer:** Fast startup and low memory by ahead-of-time compilation, with build-time constraints

Reflection needs hints.

**Q134.** What is the trade-off of native images?

**Answer:** Longer builds and restrictions on reflection and dynamic features

Test thoroughly.

**Q135.** What do AOT processing and runtime hints do?

**Answer:** Generate configuration so reflection and resources work in a native build

Boot 3 supports it.

**Q136.** When are native images most valuable?

**Answer:** Serverless and scale-to-zero where startup time matters

Measure.

### [Containerizing Spring Boot](level-4/05-containerization.md)

**Q137.** What do Cloud Native Buildpacks (spring-boot:build-image) produce?

**Answer:** A layered OCI image without writing a Dockerfile

Reproducible builds.

**Q138.** Why use layered jars in Dockerfiles?

**Answer:** Dependencies change rarely, so layers cache and rebuilds are fast

Better image pushes.

**Q139.** Why run as non-root in containers?

**Answer:** Limits damage if the app is compromised

Set a USER.

**Q140.** What should container memory settings account for?

**Answer:** JVM heap plus non-heap, with container-aware flags

Avoid OOM kills.

### [Observability: Logs, Metrics &amp; Tracing](level-4/06-observability-tracing.md)

**Q141.** What are the three pillars of observability?

**Answer:** Metrics, logs and traces

Micrometer Observation unifies them.

**Q142.** What does trace context propagation do?

**Answer:** Passes trace IDs between services so spans link into one trace

W3C traceparent.

**Q143.** What makes alerts actionable?

**Answer:** They signal user impact and link to a runbook

Alert on symptoms.

**Q144.** How do you keep cardinality in check?

**Answer:** Avoid unbounded values such as user IDs as metric tags

Prevents cost blow-ups.

### [Security Hardening](level-4/07-security-hardening.md)

**Q145.** Which is a hardening step for Spring Boot apps?

**Answer:** Update dependencies promptly, minimise exposed actuators and validate all inputs

Scan with tools.

**Q146.** What does the Content-Security-Policy and security headers set help with?

**Answer:** Mitigating XSS and clickjacking for browser-facing apps

Spring Security adds many defaults.

**Q147.** How do you handle CORS safely?

**Answer:** Allow only specific origins, methods and headers

Credentials with * is forbidden.

**Q148.** Why use dependency scanning?

**Answer:** Libraries carry known CVEs

Automate in CI.

### [Deploying to Production](level-4/08-deployment.md)

**Q149.** What is a safe deployment strategy?

**Answer:** Rolling or blue-green with health checks and quick rollback

Readiness gates traffic.

**Q150.** What do liveness and readiness probes protect against?

**Answer:** Sending traffic to unready instances and keeping stuck ones running

Tune thresholds.

**Q151.** What should graceful shutdown do?

**Answer:** Finish in-flight requests before stopping (server.shutdown=graceful)

Avoid dropped requests.

**Q152.** How are schema changes coordinated with deploys?

**Answer:** Backward-compatible migrations applied before or alongside the new version

Expand then contract.

### [Architecture: Modular Monoliths &amp; Boundaries](level-4/09-modular-architecture.md)

**Q153.** What is Spring Modulith?

**Answer:** A toolkit for building well-structured modular monoliths with verified module boundaries

Tests verify dependencies.

**Q154.** Why start with a modular monolith?

**Answer:** Clear boundaries without distributed-system cost, easing later extraction

Split only when needed.

**Q155.** How should modules communicate?

**Answer:** Through public APIs or events, not by reaching into each other's internals

Keep coupling low.

**Q156.** What does ArchUnit-style testing enforce?

**Answer:** Architectural rules such as allowed package dependencies

Fail the build on violations.

### [Capstone — A Production-Ready Service](level-4/10-capstone.md)

**Q157.** What is the capstone's goal?

**Answer:** A production-ready Spring Boot service with security, persistence, tests, observability and deployment

Bring it together.

**Q158.** What proves the service is production-ready?

**Answer:** CI with tests and scans, load results, dashboards, alerts and a runbook

Evidence.

**Q159.** What belongs in the README?

**Answer:** Setup, configuration, run, test, deploy and design decisions

Handover.

**Q160.** How do you handle residual risk?

**Answer:** Document it and decide to accept, mitigate or transfer

Risk register.

## Mixed questions across all levels

**Q161.** What does this model of Bean Validation print?

```java
import java.util.*;
public class Main { public static void main(String[] a) { System.out.println(V.check(new Order("A1", 2, "x@y"))); System.out.println(V.check(new Order("", 0, "nope"))); } }
record Order(String sku, int qty, String email) {}
class V { static List<String> check(Order o) { var e = new ArrayList<String>(); if (o.sku() == null || o.sku().isBlank()) e.add("sku required"); if (o.qty() < 1) e.add("qty must be >= 1"); if (o.email() == null || !o.email().matches("[^@]+@[^@]+")) e.add("email invalid"); return e; } }
```

**Answer:** []
[sku required, qty must be &gt;= 1, email invalid]

Spring runs equivalent checks from @NotBlank, @Min and @Email when a parameter is @Valid and answers 400 on violations. (Plain-Java model; Spring itself was not used.) The code was compiled, run and its output checked.

**Q162.** What does this idempotent-handler model print when the same event is delivered twice?

```java
import java.util.*;
public class Main { static Set<String> seen = new HashSet<>(); static int total = 0; static void handle(String id, int amt) { if (!seen.add(id)) { System.out.println("duplicate " + id); return; } total += amt; } public static void main(String[] a) { handle("e1", 10); handle("e2", 5); handle("e1", 10); System.out.println(total); } }
```

**Answer:** duplicate e1
15

At-least-once delivery means handlers must tolerate duplicates, for example by recording processed IDs. (Plain-Java model; Spring itself was not used.) The code was compiled, run and its output checked.

**Q163.** What does this JDK dynamic-proxy model of AOP print?

```java
import java.lang.reflect.*;
public class Main { public static void main(String[] x) { Svc t = (a, b) -> a + b; Svc p = (Svc) Proxy.newProxyInstance(Svc.class.getClassLoader(), new Class<?>[]{Svc.class}, (pr, m, args) -> { System.out.println("tx begin"); try { Object r = m.invoke(t, args); System.out.println("tx commit"); return r; } catch (InvocationTargetException e) { System.out.println("tx rollback"); throw e.getCause(); } }); System.out.println(p.add(2, 3)); } }
interface Svc { int add(int a, int b); }
```

**Answer:** tx begin
tx commit
5

Transaction advice wraps the call: begin, invoke, commit (or rollback on an exception). (Plain-Java model; Spring itself was not used.) The code was compiled, run and its output checked.

**Q164.** How many queries does the N+1 problem issue for a list of 100 authors with lazy books?

**Answer:** 101: one for the list plus one per author

Fix with a fetch join, an entity graph or batch fetching to get it down to 1 or 2.

**Q165.** What does this cache model print?

```java
import java.util.*;
public class Main { static Map<Integer,Integer> cache = new HashMap<>(); static int calls = 0; static int slow(int n) { calls++; return n * n; } static int cached(int n) { return cache.computeIfAbsent(n, Main::slow); } public static void main(String[] a) { System.out.println(cached(4) + cached(4) + cached(5)); System.out.println("calls=" + calls); } }
```

**Answer:** 57
calls=2

The second lookup of 4 is served from the cache; @Cacheable does this keyed by arguments. (Plain-Java model; Spring itself was not used.) The code was compiled, run and its output checked.

**Q166.** What is the right layer for @Transactional?

**Answer:** The service method that represents one business operation

Keep transactions short and at the use-case boundary.

**Q167.** Why can a @Transactional method called from another method in the same class run without a transaction?

**Answer:** The call bypasses the proxy that applies the advice (self-invocation)

Move the method to another bean.

**Q168.** Which fetch strategy is the default for @OneToMany?

**Answer:** LAZY

@ManyToOne defaults to EAGER.

**Q169.** Why should production use Flyway or Liquibase instead of ddl-auto=update?

**Answer:** Schema changes become versioned, reviewed and repeatable

Set ddl-auto to validate or none.

**Q170.** Which test annotation loads only the web layer for a controller test?

**Answer:** @WebMvcTest

Combine with @MockBean.

**Q171.** Why prefer Testcontainers to an in-memory H2 database for repository tests?

**Answer:** The real engine exposes dialect, constraint and locking behaviour that H2 hides

Requires Docker.

**Q172.** How should passwords be stored?

**Answer:** Hashed with BCrypt or Argon2 through a PasswordEncoder

Use per-password salts.

**Q173.** What must a resource server check on a JWT besides the signature?

**Answer:** Expiry, issuer and audience (and the algorithm)

A valid signature on an expired token is still invalid.

**Q174.** What does a circuit breaker do?

**Answer:** Stops calling a failing dependency for a period so it can recover and callers fail fast

Combine with timeouts and backoff.

**Q175.** Why add jitter to retry backoff?

**Answer:** So many clients do not retry in lockstep and overwhelm a recovering service

Cap the attempts.

**Q176.** What is the outbox pattern for?

**Answer:** Saving the state change and the event in one transaction, then publishing the event reliably

Avoids lost or phantom messages.

**Q177.** When do virtual threads help?

**Answer:** Blocking-style request handling at high concurrency without reactive code

Enable with spring.threads.virtual.enabled=true on Java 21+.

**Q178.** Which Actuator exposure is safest by default?

**Answer:** Only health (and info), with the rest secured or disabled

Endpoints can leak configuration.

**Q179.** What is the main trade-off of GraalVM native images?

**Answer:** Very fast startup and low memory, but longer builds and limits on reflection and dynamic features

Needs AOT hints.

**Q180.** What proves a Spring Boot service is production-ready?

**Answer:** CI with tests and scans, load results, dashboards, alerts and a runbook

Evidence over claims.

## Practical coding questions

### A tiny IoC container with constructor injection

Write a mini dependency-injection container in plain Java: map interfaces to implementations, create beans by resolving constructor parameters recursively, cache singletons and detect missing beans and circular dependencies with clear messages. This models what Spring's ApplicationContext does; it uses only the JDK, and Spring itself was not run.

Starter code:

```
import java.lang.reflect.Constructor;
import java.util.*;

public class Main {
    interface Greeter { String greet(String name); }
    static class EnglishGreeter implements Greeter { public String greet(String n) { return "Hello, " + n; } }
    static class Audit { final List<String> log = new ArrayList<>(); void add(String s) { log.add(s); } }
    static class Service {
        final Greeter greeter; final Audit audit;
        Service(Greeter g, Audit a) { this.greeter = g; this.audit = a; }
        String run(String n) { audit.add("run " + n); return greeter.greet(n); }
    }
    static class A { A(B b) {} }
    static class B { B(A a) {} }

    // TODO: static class Container
    //   register(Class<?> type, Class<?> impl)            map an interface to its implementation
    //   <T> T get(Class<T> type)                           create the bean with constructor injection (resolve each constructor parameter recursively),
    //                                                      cache it as a singleton, throw IllegalStateException("no bean for X") for an unmapped interface,
    //                                                      and IllegalStateException("circular dependency: A -> B -> A") when resolution loops

    public static void main(String[] args) {
        Container ctx = new Container();
        ctx.register(Greeter.class, EnglishGreeter.class);
        Service s1 = ctx.get(Service.class);
        Service s2 = ctx.get(Service.class);
        System.out.println(s1.run("Ana"));
        System.out.println("same instance: " + (s1 == s2) + ", shared audit: " + (s1.audit == ctx.get(Audit.class)));
        System.out.println(ctx.get(Audit.class).log);
        try { new Container().get(Service.class); } catch (IllegalStateException e) { System.out.println(e.getMessage()); }
        try { new Container().get(A.class); } catch (IllegalStateException e) { System.out.println(e.getMessage()); }
    }
}
```

Expected output:

```text
Hello, Ana
same instance: true, shared audit: true
[run Ana]
no bean for Greeter
circular dependency: A -> B -> A
```

??? success "Show a solution"

    ```
    import java.lang.reflect.Constructor;
    import java.util.*;

    public class Main {
        interface Greeter { String greet(String name); }
        static class EnglishGreeter implements Greeter { public String greet(String n) { return "Hello, " + n; } }
        static class Audit { final List<String> log = new ArrayList<>(); void add(String s) { log.add(s); } }
        static class Service {
            final Greeter greeter; final Audit audit;
            Service(Greeter g, Audit a) { this.greeter = g; this.audit = a; }
            String run(String n) { audit.add("run " + n); return greeter.greet(n); }
        }
        static class A { A(B b) {} }
        static class B { B(A a) {} }

        /** A tiny IoC container: register implementations for types, resolve by constructor injection, singletons by default. */
        static class Container {
            private final Map<Class<?>, Class<?>> impls = new HashMap<>();
            private final Map<Class<?>, Object> singletons = new HashMap<>();
            private final Deque<Class<?>> resolving = new ArrayDeque<>();

            void register(Class<?> type, Class<?> impl) { impls.put(type, impl); }

            @SuppressWarnings("unchecked")
            <T> T get(Class<T> type) {
                if (singletons.containsKey(type)) return (T) singletons.get(type);
                if (resolving.contains(type)) throw new IllegalStateException("circular dependency: " + path(type));
                resolving.push(type);
                try {
                    Class<?> impl = impls.getOrDefault(type, type);
                    if (impl.isInterface()) throw new IllegalStateException("no bean for " + type.getSimpleName());
                    Constructor<?> c = impl.getDeclaredConstructors()[0];
                    c.setAccessible(true);
                    Object[] args = Arrays.stream(c.getParameterTypes()).map(this::get).toArray();
                    Object bean = c.newInstance(args);
                    singletons.put(type, bean);
                    return (T) bean;
                } catch (ReflectiveOperationException e) {
                    throw new RuntimeException(e);
                } finally {
                    resolving.pop();
                }
            }

            private String path(Class<?> again) {
                List<String> names = new ArrayList<>();
                for (Class<?> c : resolving) names.add(c.getSimpleName());
                Collections.reverse(names);
                names.add(again.getSimpleName());
                return String.join(" -> ", names);
            }
        }

        public static void main(String[] args) {
            Container ctx = new Container();
            ctx.register(Greeter.class, EnglishGreeter.class);
            Service s1 = ctx.get(Service.class);
            Service s2 = ctx.get(Service.class);
            System.out.println(s1.run("Ana"));
            System.out.println("same instance: " + (s1 == s2) + ", shared audit: " + (s1.audit == ctx.get(Audit.class)));
            System.out.println(ctx.get(Audit.class).log);
            try { new Container().get(Service.class); } catch (IllegalStateException e) { System.out.println(e.getMessage()); }
            try { new Container().get(A.class); } catch (IllegalStateException e) { System.out.println(e.getMessage()); }
        }
    }
    ```

    Reflection reads each constructor's parameter types and get() resolves them recursively. A stack of types being resolved reveals cycles, and the singleton map gives Spring's default scope. Verified by compiling and running with Java 20; real Spring adds scopes, lifecycle callbacks and proxies.

### Retry with backoff and a circuit breaker

Combine retries (exponential backoff) with a circuit breaker that opens after repeated failures, allows one half-open probe after a cool-down and closes again on success. A fake clock makes the run deterministic. This is a plain-Java model of what Resilience4j provides in a Spring Boot service; no real service or Spring was involved.

Starter code:

```
import java.util.*;
import java.util.function.Supplier;

public class Main {
    // TODO: enum State { CLOSED, OPEN, HALF_OPEN } and a Clock with now/advance(ms)
    // TODO: Breaker(threshold, openMs, clock).call(Supplier<T>)
    //   OPEN: throw IllegalStateException("circuit open") until openMs has passed, then allow one HALF_OPEN probe
    //   success: reset failures and CLOSE; failure: count it, and OPEN (recording openedAt) when the threshold is reached or the probe failed
    // TODO: retry(attempts, baseMs, clock, log, action): on failure log "attempt N failed (msg), waiting Wms", advance the clock by baseMs << (N-1),
    //   but rethrow immediately when the breaker is open

    public static void main(String[] args) {
        Clock clock = new Clock();
        Breaker breaker = new Breaker(3, 1000, clock);
        List<String> log = new ArrayList<>();
        int[] calls = {0};
        Supplier<String> flaky = () -> { calls[0]++; if (calls[0] <= 4) throw new RuntimeException("boom" + calls[0]); return "ok"; };

        try { System.out.println(retry(5, 100, clock, log, () -> breaker.call(flaky))); }
        catch (RuntimeException e) { System.out.println("gave up: " + e.getMessage()); }
        log.forEach(System.out::println);
        System.out.println("state=" + breaker.state + " calls=" + calls[0] + " t=" + clock.now);

        clock.advance(1000);
        try { System.out.println(retry(2, 100, clock, log, () -> breaker.call(flaky))); }
        catch (RuntimeException e) { System.out.println("half-open probe failed: " + e.getMessage() + ", state=" + breaker.state); }
        clock.advance(1000);
        System.out.println(retry(2, 100, clock, log, () -> breaker.call(flaky)));
        System.out.println("state=" + breaker.state + " calls=" + calls[0] + " t=" + clock.now);
    }
}
```

Expected output:

```text
gave up: circuit open
attempt 1 failed (boom1), waiting 100ms
attempt 2 failed (boom2), waiting 200ms
attempt 3 failed (boom3), waiting 400ms
state=OPEN calls=3 t=700
half-open probe failed: circuit open, state=OPEN
ok
state=CLOSED calls=5 t=2800
```

??? success "Show a solution"

    ```
    import java.util.*;
    import java.util.function.Supplier;

    public class Main {
        /** Circuit breaker with retries and backoff, driven by a fake clock so output is deterministic. */
        enum State { CLOSED, OPEN, HALF_OPEN }

        static class Clock { long now = 0; void advance(long ms) { now += ms; } }

        static class Breaker {
            final int threshold; final long openMs; final Clock clock;
            State state = State.CLOSED; int failures = 0; long openedAt = 0;
            Breaker(int threshold, long openMs, Clock clock) { this.threshold = threshold; this.openMs = openMs; this.clock = clock; }

            <T> T call(Supplier<T> action) {
                if (state == State.OPEN) {
                    if (clock.now - openedAt >= openMs) state = State.HALF_OPEN;
                    else throw new IllegalStateException("circuit open");
                }
                try {
                    T result = action.get();
                    failures = 0; state = State.CLOSED;
                    return result;
                } catch (RuntimeException e) {
                    failures++;
                    if (state == State.HALF_OPEN || failures >= threshold) { state = State.OPEN; openedAt = clock.now; }
                    throw e;
                }
            }
        }

        static <T> T retry(int attempts, long baseMs, Clock clock, List<String> log, Supplier<T> action) {
            RuntimeException last = null;
            for (int i = 0; i < attempts; i++) {
                try { return action.get(); }
                catch (RuntimeException e) {
                    last = e;
                    if (e.getMessage().equals("circuit open")) throw e;
                    long wait = baseMs << i;
                    log.add("attempt " + (i + 1) + " failed (" + e.getMessage() + "), waiting " + wait + "ms");
                    clock.advance(wait);
                }
            }
            throw last;
        }

        public static void main(String[] args) {
            Clock clock = new Clock();
            Breaker breaker = new Breaker(3, 1000, clock);
            List<String> log = new ArrayList<>();
            int[] calls = {0};
            Supplier<String> flaky = () -> { calls[0]++; if (calls[0] <= 4) throw new RuntimeException("boom" + calls[0]); return "ok"; };

            try { System.out.println(retry(5, 100, clock, log, () -> breaker.call(flaky))); }
            catch (RuntimeException e) { System.out.println("gave up: " + e.getMessage()); }
            log.forEach(System.out::println);
            System.out.println("state=" + breaker.state + " calls=" + calls[0] + " t=" + clock.now);

            clock.advance(1000);
            try { System.out.println(retry(2, 100, clock, log, () -> breaker.call(flaky))); }
            catch (RuntimeException e) { System.out.println("half-open probe failed: " + e.getMessage() + ", state=" + breaker.state); }
            clock.advance(1000);
            System.out.println(retry(2, 100, clock, log, () -> breaker.call(flaky)));
            System.out.println("state=" + breaker.state + " calls=" + calls[0] + " t=" + clock.now);
        }
    }
    ```

    The breaker counts consecutive failures, opens, and after openMs lets a single probe through; a failed probe re-opens it while a success closes it. retry() stops immediately when the circuit is open instead of waiting. Verified by compiling and running with Java 20.

---

More free practice: the [Spring Boot Mastery Path](index.md) curriculum, the [Interview Prep Pack](https://learnmasterypath.com/interview-pack/) and the [free resume ATS check](https://learnmasterypath.com/resume/).
