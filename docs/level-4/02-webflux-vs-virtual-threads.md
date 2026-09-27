---
description: "Compare Spring WebFlux and virtual threads for high-concurrency I/O: how each avoids blocking platform threads, their costs, and a decision guide."
---

# WebFlux vs Virtual Threads

A typical request handler spends most of its time *waiting*: for the database, for another
service, for a cache. With the classic thread-per-request model, each waiting request
holds an operating-system thread, and threads are expensive (a megabyte-scale stack
reservation, kernel scheduling). At a few thousand concurrent slow requests you run out.
Spring offers two answers: the reactive stack (WebFlux) and, since Java 21, virtual threads
on the ordinary servlet stack.

## Answer 1: WebFlux (reactive)

```java
@RestController
class QuoteController {
    private final WebClient rates;
    private final QuoteRepository quotes;   // R2DBC reactive repository

    QuoteController(WebClient.Builder builder, QuoteRepository quotes) {
        this.rates = builder.baseUrl("http://rates").build();
        this.quotes = quotes;
    }

    @GetMapping("/quotes/{id}")
    Mono<QuoteView> get(@PathVariable long id) {
        return quotes.findById(id)
                .switchIfEmpty(Mono.error(new QuoteNotFoundException(id)))
                .zipWith(rates.get().uri("/latest").retrieve().bodyToMono(Rates.class))
                .map(t -> QuoteView.of(t.getT1(), t.getT2()));
    }
}
```

Nothing blocks. The method returns a `Mono` — a description of work — immediately; Netty's
small set of event-loop threads (roughly one per CPU core) runs the pipeline as data
arrives. Thousands of in-flight requests cost only memory for their pipeline objects.

The price: **everything** in the path must be non-blocking. JDBC and JPA block, so you use
R2DBC; many libraries (and most SDKs) block. One blocking call on an event-loop thread
stalls every request that loop serves. Code becomes operator chains, stack traces become
harder to read, and `ThreadLocal`-based features (MDC, security context, transactions)
need the Reactor context instead.

WebFlux shines for streaming (server-sent events, WebSockets, proxying large bodies), for
gateways (Spring Cloud Gateway is built on it), and when composing many concurrent remote
calls with backpressure.

## Answer 2: virtual threads

```yaml
spring:
  threads:
    virtual:
      enabled: true
```

That is the whole change on Java 21+. Tomcat now runs each request on a new **virtual
thread**; so do `@Async` tasks and scheduled jobs. Your controller, JPA repository, and
`RestClient` code stay exactly as written in Levels 1–3 — blocking, imperative, debuggable.

A virtual thread is a Java object scheduled by the JVM onto a small pool of platform
"carrier" threads. When it blocks on I/O, the JVM unmounts it from its carrier (saving its
stack to the heap) and runs another; when the I/O completes, it is remounted. Blocking
becomes cheap, so you can have hundreds of thousands of them.

## Worked example: the same endpoint both ways

The WebFlux version is above. The virtual-thread version:

```java
@GetMapping("/quotes/{id}")
QuoteView get(@PathVariable long id) {
    Quote quote = quotes.findById(id).orElseThrow(() -> new QuoteNotFoundException(id));
    Rates rates = ratesClient.get().uri("/latest").retrieve().body(Rates.class);
    return QuoteView.of(quote, rates);
}
```

Same behavior, sequential calls. To run the two lookups concurrently in imperative code,
submit them to an executor of virtual threads
(`Executors.newVirtualThreadPerTaskExecutor()`) and join both futures — or, where
available, use structured concurrency (still a preview API in recent JDKs; check your
JDK's status before depending on it).

This course did not run a load benchmark comparing the two, and you should be skeptical of
anyone's numbers that were not measured on your workload. What can be said without a
benchmark: both approaches remove the "one platform thread per waiting request" limit; the
next bottleneck is usually a *downstream* limit — the database connection pool, a partner
API's rate limit — which neither model removes.

## Decision guide

| Situation | Choose |
|-----------|--------|
| Typical CRUD/business API on JDBC/JPA | **Servlet stack + virtual threads** |
| Existing Spring MVC codebase needing more concurrency | Virtual threads |
| Streaming, SSE/WebSockets at scale, API gateway | WebFlux |
| Team already fluent in Reactor, fully non-blocking dependencies | WebFlux is fine |
| CPU-bound work | Neither helps; you need more cores or less work |

## How It Actually Works

**Reactive.** WebFlux runs on Netty (or other non-blocking servers). An event-loop thread
reads a request, routes it through a `DispatcherHandler`, and subscribes to the returned
`Mono`. Operators form a chain of subscribers; when a database driver or `WebClient`
completes I/O, it signals `onNext` on some thread, which runs the next operators. No thread
waits. Backpressure (`request(n)`) lets consumers limit how fast producers emit.

**Virtual threads.** The JDK implements blocking operations (socket reads, `LockSupport.park`)
so that on a virtual thread they *yield* the carrier instead of blocking it. The carriers
are a `ForkJoinPool` sized to the core count. Two historical caveats: when a virtual thread
blocked inside a `synchronized` block it **pinned** its carrier (fixed in JDK 24 by JEP 491,
so on JDK 21 prefer `ReentrantLock` in hot paths), and native calls or some file I/O still
occupy a carrier. Because virtual threads are cheap, **don't pool them** — pools exist to
limit expensive resources. Limit the resource instead, with a semaphore or
`@ConcurrencyLimit`, and size the JDBC pool to what the database can handle.

`ThreadLocal`s still work on virtual threads (each has its own), so MDC, security context,
and transactions behave as before — but caching large objects in `ThreadLocal`s is now
wasteful because threads are not reused.

## Common mistakes

- **Blocking calls inside WebFlux** (JDBC, `block()`, legacy SDKs) on the event loop.
- **Expecting virtual threads to speed up CPU-bound code.**
- **Unbounded fan-out with virtual threads**, overwhelming a database or partner.
- **Pooling virtual threads** in a fixed-size executor.
- **Rewriting a working MVC app in WebFlux** purely for concurrency, when a property would do.

## Exercise

1. Enable virtual threads in the Level 2 library API and log `Thread.currentThread()` in a
   controller. Confirm it is a `VirtualThread`.
2. Add an endpoint that makes a 1-second call to a slow stub. With virtual threads off and
   `server.tomcat.threads.max=20`, fire 200 concurrent requests (for example with `hey` or
   `wrk`) and record total time; repeat with virtual threads on. Record your measured numbers.
3. Repeat with a Hikari pool of 10 and a database query that sleeps (`pg_sleep`), and explain
   why virtual threads no longer help.
4. Write the quote endpoint in WebFlux with `WebClient`, then add a deliberate
   `Thread.sleep` in a `map` and observe the effect on other requests.
