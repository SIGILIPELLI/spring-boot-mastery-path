---
description: "Spring Boot capstone: build and ship a production-ready library-lending service with security, migrations, events, observability, and a launch checklist."
---

# Capstone — A Production-Ready Service

The capstone is where you prove to yourself that you can take a Spring Boot service from
an empty directory to something you would be comfortable putting in front of real users.
You will build **Lend**, a library-lending service, reusing pieces from every level —
and this time, you are responsible for every decision.

This page specifies the system, the milestones, and how to verify each one. It deliberately
does not give you finished code: the earlier projects did that. Each milestone points back
to the lesson that covers the technique.

## The product

Members browse a catalog, borrow and return books, and join waitlists. Librarians manage
the catalog and see overdue loans. When a book is returned, the next member on its waitlist
is notified.

| Capability | Endpoints / behavior |
|-----------|---------------------|
| Catalog | `GET /api/books?q=&page=` (public, paged), `POST/PATCH /api/books` (librarian) |
| Loans | `POST /api/loans` (member, max 5 active, one active loan per copy), `POST /api/loans/{id}/return` |
| Waitlist | `POST /api/books/{id}/waitlist`, notification on return |
| Overdue | Nightly job marks overdue loans; `GET /api/loans/overdue` (librarian) |

## Architecture requirements

- One Boot application, **modular monolith** with modules `catalog`, `lending`, `waitlist`,
  `notifications`, verified with Spring Modulith (lesson 09).
- PostgreSQL with **Flyway**; `ddl-auto: validate`; every schema change backward compatible.
- **JWT resource server** with scopes/roles from an identity provider (Keycloak in Docker
  Compose locally) — no passwords stored in the app (Level 2, lessons 07–08).
- `BookReturned` event from `lending` consumed by `waitlist` via
  `@ApplicationModuleListener` with the event publication registry, so notifications survive
  crashes (Level 3, lesson 09; Level 4, lesson 09).
- Notifications go through an HTTP email provider stub with timeouts, retries with an
  idempotency key, and a circuit breaker (Level 3, lesson 07).
- **Virtual threads** enabled; a concurrency limit on the email client (lesson 02).
- Caffeine cache for book detail with event-based eviction (Level 3, lessons 04 and 09).

## Milestones

### M1 — Skeleton and catalog

Initializr project, feature packages, `GET/POST /api/books` with validation and
problem-detail errors, Flyway V1, `@WebMvcTest` and `@DataJpaTest` tests.
**Done when:** `./mvnw verify` passes and malformed input yields 400 problem details.

### M2 — Lending rules

Loans with the invariants enforced twice: in the service (clear error messages) and in the
database (a partial unique index for one active loan per copy, a check on counts where
possible). Optimistic locking on members.
**Done when:** a concurrency test with 20 threads borrowing the same copy produces exactly one
loan and 19 × 409.

### M3 — Security

Resource server, scopes (`books:write`, `loans:write`), object-level checks (a member can
only return their own loans).
**Done when:** a test matrix covers anonymous, member, other member, and librarian for every
endpoint.

### M4 — Events and notifications

`BookReturned` → waitlist → notification with the resilient client.
**Done when:** killing the app between return and notification, then restarting, still sends
exactly one notification (the registry republishes; the provider deduplicates by key).

### M5 — Observability

Actuator on a management port with probes; tracing to a local collector; structured JSON logs
with trace ids; custom metrics `lend.loans.active` (gauge) and `lend.notifications.failed`
(counter).
**Done when:** you can start from a slow-request alert and reach the log line of the slow
span, following only ids.

### M6 — Performance

Statement-count tests for every list endpoint; JDBC batching for the nightly overdue job;
pool sized from measurement.
**Done when:** a load test at your chosen target shows no connection-pool waiting and p99
within your SLO, and you have written the numbers down.

### M7 — Packaging and deployment

Layered image as non-root, `MaxRAMPercentage`, graceful shutdown; Kubernetes manifests with
startup/readiness/liveness probes, preStop, and resources; Flyway as a separate Job.
**Done when:** a rolling update under load produces zero failed requests.

### M8 — Hardening

The lesson 07 checklist: secrets from the platform, CORS for the frontend origin, rate limits
on loan creation, dependency and image scanning in CI, SBOM.
**Done when:** the scans are clean or every finding has a written decision.

## Worked example: how to verify M2's concurrency requirement

```java
@SpringBootTest
@Testcontainers
class ConcurrentBorrowTest {
    @Container @ServiceConnection
    static PostgreSQLContainer postgres = new PostgreSQLContainer("postgres:17-alpine");

    @Autowired LoanService loans;

    @Test
    void onlyOneLoanPerCopy() throws Exception {
        long copyId = seedCopy();
        List<Long> members = seedMembers(20);
        try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
            List<Future<Boolean>> results = members.stream()
                    .map(m -> executor.submit(() -> {
                        try { loans.borrow(copyId, m); return true; }
                        catch (CopyUnavailableException e) { return false; }
                    }))
                    .toList();
            long successes = 0;
            for (Future<Boolean> f : results) if (f.get()) successes++;
            assertThat(successes).isEqualTo(1);
        }
    }
}
```

`borrow` should translate the unique-index violation (a `DataIntegrityViolationException`)
into `CopyUnavailableException`. Run it against PostgreSQL, not H2 — partial indexes and
concurrency behavior are exactly where engines differ. (Adjust the Testcontainers class and
package to your Testcontainers version.)

## Launch checklist

- [ ] All tests green in CI, including Testcontainers and the Modulith `verify()`.
- [ ] Image scanned, SBOM attached, tagged with the git SHA.
- [ ] Config per environment reviewed; no secrets in the repository.
- [ ] Migrations reviewed for backward compatibility and locking impact.
- [ ] Dashboards: request rate, error rate, p99 per route; pool usage; GC; outbox/registry
      backlog.
- [ ] Alerts on SLO burn, not on every error.
- [ ] Runbook: how to roll back, how to change a log level at runtime, how to replay
      incomplete event publications.
- [ ] Load test results recorded with the date and version.

## How It Actually Works

The capstone is a tour of mechanisms you now understand from the inside: **proxies** give
you transactions, security checks, caching, retries, and async listeners — and you know to
keep those calls crossing bean boundaries. The **persistence context** decides when SQL runs,
and your statement-count tests keep N+1 away. **Auto-configuration** builds your pools,
mappers, and instrumentation, and backs off wherever you define your own bean.
**Lifecycle** ordering makes graceful shutdown drain requests before the context closes. The
**event publication registry** turns in-memory events into a durable outbox. Every
guarantee in the checklist maps to one of those mechanisms — which is the point of this
course.

## Exercise

Complete milestones M1–M8. Then write a one-page design document for Lend covering: module
boundaries and why, the consistency guarantees and where each is enforced, the failure modes
of the notification path and how each is handled, and what you would change first if traffic
grew tenfold. That document is a strong portfolio piece — and the best test of whether you
can explain Spring Boot, not just use it.
