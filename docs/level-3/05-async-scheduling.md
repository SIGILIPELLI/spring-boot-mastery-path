---
description: "Run work off the request thread with @Async and executors, use virtual threads, schedule jobs with @Scheduled, and keep scheduled jobs single-instance."
---

# Async Execution & Scheduling

Not everything should happen while the user waits. Sending an email, generating a report,
or calling a slow partner API can run in the background; cleaning up expired records can
run on a timer. Spring offers `@Async` and `@Scheduled` for both. They are easy to add and
easy to misuse, so this lesson spends as much time on failure modes as on syntax.

## @Async

```java
@Configuration
@EnableAsync
class AsyncConfig { }

@Service
class ReceiptMailer {
    private static final Logger log = LoggerFactory.getLogger(ReceiptMailer.class);

    @Async
    public void sendReceipt(long orderId, String email) {
        log.info("Sending receipt for order {}", orderId);
        // render + send ...
    }

    @Async
    public CompletableFuture<ReportFile> buildReport(YearMonth month) {
        ReportFile file = /* slow work */ null;
        return CompletableFuture.completedFuture(file);
    }
}
```

The caller returns immediately. A `void` method is fire-and-forget; returning a
`CompletableFuture` lets the caller combine or wait for results.

### The executor matters

Boot auto-configures an executor for `@Async` (and `@EnableAsync` uses it). With platform
threads it is a `ThreadPoolTaskExecutor` configurable through `spring.task.execution.*`:

```yaml
spring:
  task:
    execution:
      thread-name-prefix: async-
      pool:
        core-size: 8
        max-size: 16
        queue-capacity: 500
      shutdown:
        await-termination: true
        await-termination-period: 30s
```

Two properties deserve attention:

- **Queue capacity.** The pool only grows beyond `core-size` when the queue is *full*. With
  an unbounded queue, you never get more than `core-size` threads and tasks pile up in
  memory. Bound it, and decide what happens when it is full (the default rejection policy
  throws).
- **Shutdown.** Without `await-termination`, in-flight tasks are abandoned when the app
  stops — the email is never sent.

### Virtual threads

On Java 21+, set `spring.threads.virtual.enabled=true` and Boot switches the async
executor (and Tomcat's request handling, and the scheduler) to virtual threads. For
I/O-bound background work this removes most pool tuning: each task gets its own cheap
virtual thread. It does not make CPU-bound work faster, and unbounded concurrency can now
overwhelm the *downstream* (a database pool of 10 connections, a partner's rate limit) —
Spring Framework 7's `@ConcurrencyLimit` or a semaphore is how you put a bound back.
Level 4 compares virtual threads and WebFlux in depth.

### Errors in async methods

Exceptions from a `void` `@Async` method never reach the caller. By default they are
logged by `SimpleAsyncUncaughtExceptionHandler` and then lost. Provide your own handler
(implement `AsyncConfigurer.getAsyncUncaughtExceptionHandler`) to at least count and
alert, and design critical background work so it is **retryable and persisted** — which
usually means a queue or an outbox table rather than `@Async` (lesson 06).

## @Scheduled

```java
@Configuration
@EnableScheduling
class SchedulingConfig { }

@Component
class Housekeeping {
    private final LoanRepository loans;

    Housekeeping(LoanRepository loans) { this.loans = loans; }

    @Scheduled(cron = "0 0 3 * * *", zone = "UTC")         // 03:00 UTC daily
    @Transactional
    void closeExpiredReservations() {
        int closed = loans.closeExpired(Instant.now());
        LoggerFactory.getLogger(Housekeeping.class).info("Closed {} expired reservations", closed);
    }

    @Scheduled(fixedDelayString = "${housekeeping.poll-interval:30s}", initialDelayString = "10s")
    void pollPartnerFeed() { ... }
}
```

- `fixedRate` — start every N regardless of how long the last run took (runs can queue up).
- `fixedDelay` — wait N *after* the previous run finishes (the safer default).
- `cron` — six fields in Spring (seconds first): `second minute hour day month weekday`.
  Always set `zone`; otherwise the server's time zone decides, and daylight-saving changes
  can skip or repeat runs.
- Durations accept `30s`, `5m`, or ISO-8601 (`PT30S`), and can come from properties.

Boot's scheduler has **one thread** by default (`spring.task.scheduling.pool.size=1`), so
a slow job delays every other job. Raise the pool size or use virtual threads.

### Running a job on only one instance

With three replicas, every `@Scheduled` method runs three times. For jobs that must run
once — sending the daily digest, charging subscriptions — you need coordination:

- **ShedLock** (a community library) takes a lock row in your database before running;
  other instances skip that run.
- A **platform scheduler** (Kubernetes `CronJob`) runs a separate one-off process.
- Make the job **idempotent** regardless, so an accidental double run is harmless (for
  example, "charge where not yet charged for this period").

## Worked example: moving email out of the request

Before: `POST /api/orders` saves the order and sends a receipt synchronously — 800 ms of
which is SMTP. After:

```java
@Transactional
public OrderView place(PlaceOrder cmd) {
    Order order = orders.save(Order.from(cmd));
    events.publishEvent(new OrderPlaced(order.getId(), cmd.email()));
    return OrderView.from(order);
}

@Component
class OrderEmails {
    private final ReceiptMailer mailer;
    OrderEmails(ReceiptMailer mailer) { this.mailer = mailer; }

    @TransactionalEventListener   // runs after the transaction COMMITS
    void on(OrderPlaced e) {
        mailer.sendReceipt(e.orderId(), e.email());   // @Async: returns immediately
    }
}
```

The receipt is only sent if the order was actually committed, and the request no longer
waits for SMTP. The remaining weakness — a crash between commit and sending loses the
email — is what the outbox pattern in lesson 06 fixes.

## How It Actually Works

`@EnableAsync` registers `AsyncAnnotationBeanPostProcessor`, which wraps beans that have
`@Async` methods in a proxy whose `AsyncExecutionInterceptor` does:
`executor.submit(() -> invocation.proceed())` and returns immediately (or returns a future
wired to the task's result). Hence the familiar limits: self-invocation runs
synchronously, and `private` methods are never async.

Because the method now runs on a different thread, everything bound to the caller's
thread via `ThreadLocal` is gone: the transaction (the async method needs its own
`@Transactional`), the `SecurityContext` (unless you use a delegating executor), and MDC
values (unless a `TaskDecorator` copies them — Boot applies a configured `TaskDecorator`
bean to its executors, and Micrometer's context propagation does this for tracing).

`@EnableScheduling` registers `ScheduledAnnotationBeanPostProcessor`, which, after the
context is refreshed, finds `@Scheduled` methods and registers each with a
`TaskScheduler` — by default a `ThreadPoolTaskScheduler` backed by a
`ScheduledExecutorService`. Cron expressions are parsed by `CronExpression`, which
computes the next fire time after each run in the configured zone.

## Common mistakes

- **Self-invocation of `@Async` methods**, which silently run synchronously.
- **Unbounded queues** hiding overload until the heap is exhausted.
- **Using `@Async` for work that must not be lost.** Background threads die with the
  process.
- **Assuming a scheduled job runs once** in a multi-instance deployment.
- **Cron without a time zone**, or `fixedRate` jobs that overlap themselves.

## Exercise

1. Add `@Async` receipt sending with a `@TransactionalEventListener`, and log the thread
   name inside the async method to confirm it is not a Tomcat thread.
2. Add a `TaskDecorator` that copies the MDC request id into async tasks, and show the id in
   the async log line.
3. Write a `@Scheduled` job with `fixedDelay` that deliberately takes longer than the
   delay, and one with `fixedRate`. Log start/end times and explain the difference.
4. Turn on `spring.threads.virtual.enabled=true` and log `Thread.currentThread()` in a
   request handler, an async method, and a scheduled method. What changed?
