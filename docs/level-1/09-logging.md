---
description: "Log well in Spring Boot: SLF4J over Logback, levels and categories, parameterized messages, MDC request IDs, and built-in structured JSON logging."
---

# Logging with SLF4J & Logback

Logs are how you find out what your service did when nobody was watching. Spring Boot
ships a sensible logging setup out of the box; this lesson is about using it well and
changing it when you need to.

## The pieces

- **SLF4J** is the API your code calls (`org.slf4j.Logger`).
- **Logback** is the default implementation that actually writes lines. Boot configures
  it through `spring-boot-starter-logging`, which every starter includes.
- Bridges route other logging APIs (java.util.logging, Log4j API, Commons Logging used by
  older libraries) into SLF4J, so everything ends up in one stream.

You write against SLF4J only. Swapping Logback for Log4j2 later is a dependency change,
not a code change.

## Writing log statements

```java
@Service
public class TaskService {
    private static final Logger log = LoggerFactory.getLogger(TaskService.class);

    public Task create(CreateTaskRequest request) {
        Task task = /* ... */;
        log.info("Created task id={} title={}", task.id(), task.title());
        return task;
    }

    public void importFrom(Path file) {
        try {
            // ...
        } catch (IOException e) {
            log.error("Import failed for file={}", file, e);   // exception LAST, no placeholder for it
            throw new ImportFailedException(file, e);
        }
    }
}
```

- Use `{}` placeholders, not string concatenation. The message is only formatted if the
  level is enabled.
- Pass the exception as the final argument to get the stack trace. `log.error("…" + e)`
  logs only `e.toString()`.
- Log **events with identifiers** (`id=`, `orderId=`) rather than prose. Future you will
  search for `orderId=8812`.

The Level 1 project's `create` method produced this real line:

```text
2026-09-26T21:05:18.737+05:30  INFO 13112 --- [tasks] [nio-8089-exec-3] com.example.tasks.task.TaskService       : Created task id=1 title=Write the README
```

Reading left to right: timestamp, level, process ID, application name (from
`spring.application.name`), thread name (a Tomcat worker), logger name (abbreviated when
long), message.

## Levels and configuration

Levels, most to least verbose: `TRACE`, `DEBUG`, `INFO`, `WARN`, `ERROR`. The root
default is `INFO`. Configure per package in `application.yml`:

```yaml
logging:
  level:
    root: INFO
    com.example.tasks: DEBUG
    org.springframework.web: DEBUG          # see request mapping decisions
    org.hibernate.SQL: DEBUG                # see SQL (Level 2)
  file:
    name: logs/tasks.log                    # also write to a file (rotated by default)
```

Guidelines that hold up in production:

| Level | Use for |
|-------|---------|
| ERROR | Something failed and a human probably needs to look (unexpected exceptions, 5xx) |
| WARN | Degraded but handled (retry succeeded, fallback used, deprecated config) |
| INFO | Business-significant events and lifecycle (task created, job finished, app started) |
| DEBUG | Details useful when diagnosing (decisions, intermediate values) |
| TRACE | Very high volume internals; rarely enabled |

With Actuator you can change a level at runtime: `POST /actuator/loggers/com.example.tasks`
with `{"configuredLevel":"DEBUG"}` — invaluable during an incident, as long as the
endpoint is secured.

## Correlating a request: MDC

When 50 requests interleave, you need to know which lines belong together. SLF4J's
**Mapped Diagnostic Context** is a per-thread map whose values can be added to every line.

```java
@Component
class RequestIdFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res, FilterChain chain)
            throws ServletException, IOException {
        String id = Optional.ofNullable(req.getHeader("X-Request-Id"))
                .filter(s -> s.matches("[A-Za-z0-9-]{1,64}"))
                .orElseGet(() -> UUID.randomUUID().toString());
        MDC.put("requestId", id);
        res.setHeader("X-Request-Id", id);
        try {
            chain.doFilter(req, res);
        } finally {
            MDC.remove("requestId");   // threads are pooled: always clean up
        }
    }
}
```

Then include it in the pattern:

```yaml
logging:
  pattern:
    correlation: "[%X{requestId:-}] "
```

Boot inserts the `correlation` pattern into its default console format. In Level 4, once
Micrometer Tracing is on the classpath, trace and span IDs are put into the MDC
automatically and this hand-written filter becomes unnecessary.

## Structured (JSON) logs

Log aggregators (Elasticsearch/OpenSearch, Loki, cloud logging services) work best with
one JSON object per line. Since Boot 3.4 this is built in:

```yaml
logging:
  structured:
    format:
      console: ecs      # or: logstash, gelf
```

Every line becomes JSON with the level, logger, thread, message, MDC entries, and
exception fields. A common pattern is human-readable logs in `dev` and structured logs
in `prod` via profiles. Try it locally and look at the output before choosing a format;
the field names differ between ECS, Logstash, and GELF, and your aggregator will expect
one of them.

## How It Actually Works

Very early in startup — before the application context exists — Boot's
`LoggingApplicationListener` reacts to the first lifecycle event, detects which logging
system is on the classpath (Logback, Log4j2, or JUL), and initializes it. If there is a
`logback-spring.xml` in resources, Boot loads it (the `-spring` suffix lets Boot process
it and enables `<springProfile>` blocks); otherwise it applies its own defaults built from
the `logging.*` properties. Levels from `logging.level.*` are then applied to the
corresponding Logback loggers.

A Logback logger named `com.example.tasks.task.TaskService` inherits its level from the
nearest configured ancestor (`com.example.tasks`, then `com.example`, then root) — that
hierarchy is why setting a level on a package affects every class below it.

MDC is backed by a `ThreadLocal`. That is why it works with Tomcat's thread-per-request
model and why you must clear it in `finally`: the thread goes back to the pool and the
next request would inherit stale values. It is also why MDC values *disappear* when you
hand work to another thread (`@Async`, an executor, a reactive pipeline) unless something
copies them across — Micrometer's context-propagation library does this for tracing.

## Common mistakes

- **`System.out.println`** — no levels, no timestamps, not captured by log configuration.
- **Logging and rethrowing at every layer**, producing the same stack trace five times.
  Log where you *handle* an exception, not where you pass it on.
- **Logging secrets or personal data**: tokens, passwords, full request bodies, card
  numbers. Assume logs are read by many people and kept for a long time.
- **String concatenation in `debug` calls** on hot paths, doing formatting work even when
  DEBUG is off.
- **Leaving `org.hibernate.SQL: DEBUG` on in production** — it can dominate log volume.

## Exercise

1. Add the `RequestIdFilter` and the correlation pattern. Make two requests with
   `-H 'X-Request-Id: demo-1'` and confirm the ID appears in your service's log lines and
   the response header.
2. Add `dev` and `prod` profiles: `dev` logs your package at `DEBUG` in the default format;
   `prod` uses structured `ecs` console logging. Run with each and compare a log line.
3. Add a `logback-spring.xml` that writes WARN and above to a separate `errors.log` file,
   and confirm a deliberate 500 lands there.
4. Find one place in your service where you log and rethrow the same exception, and
   remove the duplicate.
