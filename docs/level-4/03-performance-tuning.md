---
description: "Tune Spring Boot services by measuring first: connection pool sizing, JPA batching and fetching, serialization and logging costs, JVM memory, startup."
---

# Performance Tuning

Performance work has one rule that matters more than any tip: **measure, change one
thing, measure again.** Most Spring Boot services are slow for a small number of reasons —
too many queries, a starved connection pool, a slow downstream — and guessing tends to
optimize something else.

## Step 1: find where the time goes

- **Traces** (lesson 06) show a request's time broken down by span: controller, each SQL
  statement, each HTTP call. This is usually the fastest way to see an N+1 or a slow
  dependency.
- **Metrics**: `http.server.requests` percentiles per route, `hikaricp.connections.pending`
  and `hikaricp.connections.acquire` (time waiting for a connection), JVM GC pause metrics.
- **Profilers** for CPU and allocation: Java Flight Recorder is built into the JDK
  (`jcmd <pid> JFR.start duration=60s filename=rec.jfr`, open in JDK Mission Control);
  async-profiler produces flame graphs.
- **Load tests** against a production-like environment (Gatling, k6, JMeter), with realistic
  data volumes.

## The usual suspects, in order

### 1. Database access

- N+1 queries — Level 3, lesson 03. Assert statement counts in tests.
- Missing indexes — check `EXPLAIN ANALYZE` for the slow queries a trace shows.
- Loading entities for read-only views — use DTO projections.
- Writes one row at a time. Enable JDBC batching:

```yaml
spring:
  jpa:
    properties:
      hibernate.jdbc.batch_size: 50
      hibernate.order_inserts: true
      hibernate.order_updates: true
```

Batching does not work for inserts with `GenerationType.IDENTITY`, because Hibernate needs
each generated id immediately. Use a sequence with an allocation size
(`@GeneratedValue(strategy = SEQUENCE)` + `@SequenceGenerator(allocationSize = 50)`) for
tables with bulk inserts. For truly large imports, bypass JPA with `JdbcClient` batch
updates or the database's bulk-load tool.

### 2. Connection pool

HikariCP's `maximum-pool-size` defaults to 10. Bigger is not better: the database does the
work, and hundreds of connections contending for the same CPU cores and locks make each
query slower. Start small, watch `hikaricp.connections.pending`, and remember the total
across *all instances* must fit the database's limit (`max_connections` in PostgreSQL). Keep
transactions short — a transaction holding a connection while calling a remote API is the
classic cause of pool exhaustion.

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      connection-timeout: 2s     # fail fast instead of queueing forever
```

### 3. Downstream calls

Timeouts, parallelizing independent calls, and caching (Level 3). A 300 ms partner call
done three times sequentially per request is the whole latency budget.

### 4. Serialization and logging

Large JSON responses cost CPU and bandwidth: page them, and project only needed fields.
Excessive logging on hot paths (especially DEBUG SQL logging left on) and synchronous file
appenders can show up in profiles.

### 5. The JVM

- **Heap**: in containers, size by percentage (`-XX:MaxRAMPercentage=75`), not a fixed
  `-Xmx` that ignores the container limit (lesson 05).
- **GC**: G1 is the default and a good one for most services. ZGC (generational in recent
  JDKs) targets very low pause times for large heaps. Change the collector only when GC
  pause metrics show a problem.
- **JIT warm-up**: the first requests after startup are slower while code is interpreted and
  compiled. Readiness probes plus a brief warm-up avoid sending full traffic to a cold
  instance.

## Startup time

For fast scaling and short-lived jobs, startup matters:

- `spring.main.lazy-initialization=true` defers bean creation until first use (shifts cost
  to the first request and hides wiring errors until then — use with care).
- **Class Data Sharing (CDS)**: Boot can extract the jar and create a CDS archive so the
  JVM loads classes faster; newer JDKs extend this with ahead-of-time class loading and
  linking (JEP 483, Java 24). Check the Boot reference for the current "efficient
  deployments" steps.
- **GraalVM native images** (next lesson) start in a fraction of the time, with trade-offs.
- **CRaC** (Coordinated Restore at Checkpoint) restores a snapshot of a warmed-up JVM, on
  JDK builds that support it.

## Worked example: a slow list endpoint

A plausible investigation, step by step (hypothetical service, no numbers invented):

1. The p95 of `GET /api/orders` is high in `http.server.requests`.
2. A trace of a slow request shows one `select from orders` followed by 50 short
   `select from customer where id=?` spans: an N+1.
3. Add `@EntityGraph(attributePaths = "customer")`; a test pins the statement count at 1.
4. Re-measure under the same load test: the query ladder is gone from traces. Now
   `hikaricp.connections.pending` is non-zero at peak.
5. Traces show each request also holds its transaction open during a 200 ms call to a
   pricing service. Move the call outside the transaction.
6. Re-measure. Pending connections drop to zero. Stop — the goal is met; further tuning
   without a target wastes time.

## How It Actually Works

**Why pool size is small.** A database executes queries on a finite number of CPU cores
and disks. Beyond roughly that many *active* queries, extra concurrency only adds context
switching and lock contention; requests queue somewhere either way. Queuing in the
application pool (with a short `connection-timeout`) is cheaper and more visible than
queuing inside the database. HikariCP hands out connections from a lock-free structure
(`ConcurrentBag`) with thread-affinity to reduce contention, and tracks acquire time,
usage time, and pending threads as Micrometer metrics.

**Why JDBC batching helps.** Without batching, each `INSERT` is a network round-trip. With
`batch_size`, Hibernate groups statements with the same SQL (hence `order_inserts`, which
sorts statements by entity so they can be grouped) into one `executeBatch()` call. Some
drivers go further — PostgreSQL's `reWriteBatchedInserts=true` rewrites a batch into
multi-row inserts.

## Common mistakes

- **Tuning without a measurement or a target.**
- **Raising the pool size** to fix slow requests caused by long transactions.
- **Microbenchmarks in a unit test**, measuring the JIT rather than your code (use JMH if you
  really need a microbenchmark).
- **Copying JVM flags from blog posts** without knowing what each does on your JDK version.
- **Load testing with an empty database.**

## Exercise

1. Enable Hibernate statistics and JDBC batching, insert 10,000 books with IDENTITY ids and
   then with a pooled sequence, and compare statement counts and time on your machine.
2. Record a 60-second JFR during a load test of the library API and find the top three
   methods by CPU and by allocation.
3. Set `maximum-pool-size: 2`, add a slow endpoint that holds a transaction for 500 ms, and
   watch `hikaricp.connections.pending` climb under load. Fix it by shortening the
   transaction, not by raising the pool.
4. Measure your app's startup time with and without lazy initialization and with a CDS
   archive, and write down the trade-off you would choose.
