---
title: "Final Quiz"
description: "Test what you have learned across all four levels of Spring Boot Mastery Path with a 20-question final quiz."
---

# Final Quiz

20 questions drawn from all four levels of this course. You need **70% (14 out of 20)** to pass —
you can retake it as many times as you like. Your best result is saved in this browser.

<div id="mp-quiz-root"></div>

<script type="application/json" id="mp-quiz-data">
[
  {
    "question": "What does this model of Bean Validation print?\n```java\nimport java.util.*;\npublic class Main { public static void main(String[] a) { System.out.println(V.check(new Order(\"A1\", 2, \"x@y\"))); System.out.println(V.check(new Order(\"\", 0, \"nope\"))); } }\nrecord Order(String sku, int qty, String email) {}\nclass V { static List<String> check(Order o) { var e = new ArrayList<String>(); if (o.sku() == null || o.sku().isBlank()) e.add(\"sku required\"); if (o.qty() < 1) e.add(\"qty must be >= 1\"); if (o.email() == null || !o.email().matches(\"[^@]+@[^@]+\")) e.add(\"email invalid\"); return e; } }\n```\n",
    "options": [
      "[]\n[sku required, qty must be >= 1, email invalid]",
      "[ok]\n[email invalid]",
      "[]\n[sku required]",
      "[]\n[]"
    ],
    "correctIndex": 0,
    "explanation": "Spring runs equivalent checks from @NotBlank, @Min and @Email when a parameter is @Valid and answers 400 on violations. (Plain-Java model; Spring itself was not run here.) Verified by compiling and running it."
  },
  {
    "question": "What does this idempotent-handler model print when the same event is delivered twice?\n```java\nimport java.util.*;\npublic class Main { static Set<String> seen = new HashSet<>(); static int total = 0; static void handle(String id, int amt) { if (!seen.add(id)) { System.out.println(\"duplicate \" + id); return; } total += amt; } public static void main(String[] a) { handle(\"e1\", 10); handle(\"e2\", 5); handle(\"e1\", 10); System.out.println(total); } }\n```\n",
    "options": [
      "duplicate e1\n25",
      "15\n25",
      "duplicate e1\n10",
      "duplicate e1\n15"
    ],
    "correctIndex": 3,
    "explanation": "At-least-once delivery means handlers must tolerate duplicates, for example by recording processed IDs. (Plain-Java model; Spring itself was not run here.) Verified by compiling and running it."
  },
  {
    "question": "What does this JDK dynamic-proxy model of AOP print?\n```java\nimport java.lang.reflect.*;\npublic class Main { public static void main(String[] x) { Svc t = (a, b) -> a + b; Svc p = (Svc) Proxy.newProxyInstance(Svc.class.getClassLoader(), new Class<?>[]{Svc.class}, (pr, m, args) -> { System.out.println(\"tx begin\"); try { Object r = m.invoke(t, args); System.out.println(\"tx commit\"); return r; } catch (InvocationTargetException e) { System.out.println(\"tx rollback\"); throw e.getCause(); } }); System.out.println(p.add(2, 3)); } }\ninterface Svc { int add(int a, int b); }\n```\n",
    "options": [
      "tx begin\ntx rollback\n5",
      "tx begin\n5\ntx commit",
      "tx begin\ntx commit\n5",
      "5\ntx begin\ntx commit"
    ],
    "correctIndex": 2,
    "explanation": "Transaction advice wraps the call: begin, invoke, commit (or rollback on an exception). (Plain-Java model; Spring itself was not run here.) Verified by compiling and running it."
  },
  {
    "question": "How many queries does the N+1 problem issue for a list of 100 authors with lazy books?",
    "options": [
      "2",
      "100",
      "101: one for the list plus one per author",
      "1"
    ],
    "correctIndex": 2,
    "explanation": "Fix with a fetch join, an entity graph or batch fetching to get it down to 1 or 2."
  },
  {
    "question": "What does this cache model print?\n```java\nimport java.util.*;\npublic class Main { static Map<Integer,Integer> cache = new HashMap<>(); static int calls = 0; static int slow(int n) { calls++; return n * n; } static int cached(int n) { return cache.computeIfAbsent(n, Main::slow); } public static void main(String[] a) { System.out.println(cached(4) + cached(4) + cached(5)); System.out.println(\"calls=\" + calls); } }\n```\n",
    "options": [
      "57\ncalls=3",
      "57\ncalls=2",
      "41\ncalls=2",
      "48\ncalls=2"
    ],
    "correctIndex": 1,
    "explanation": "The second lookup of 4 is served from the cache; @Cacheable does this keyed by arguments. (Plain-Java model; Spring itself was not run here.) Verified by compiling and running it."
  },
  {
    "question": "What is the right layer for @Transactional?",
    "options": [
      "Repositories only",
      "Controllers",
      "Entities",
      "The service method that represents one business operation"
    ],
    "correctIndex": 3,
    "explanation": "Keep transactions short and at the use-case boundary."
  },
  {
    "question": "Why can a @Transactional method called from another method in the same class run without a transaction?",
    "options": [
      "It is async",
      "It is cached",
      "The call bypasses the proxy that applies the advice (self-invocation)",
      "Spring bug"
    ],
    "correctIndex": 2,
    "explanation": "Move the method to another bean."
  },
  {
    "question": "Which fetch strategy is the default for @OneToMany?",
    "options": [
      "EAGER",
      "LAZY",
      "BATCH",
      "None"
    ],
    "correctIndex": 1,
    "explanation": "@ManyToOne defaults to EAGER."
  },
  {
    "question": "Why should production use Flyway or Liquibase instead of ddl-auto=update?",
    "options": [
      "Required by JPA",
      "Faster startup",
      "Smaller jars",
      "Schema changes become versioned, reviewed and repeatable"
    ],
    "correctIndex": 3,
    "explanation": "Set ddl-auto to validate or none."
  },
  {
    "question": "Which test annotation loads only the web layer for a controller test?",
    "options": [
      "@WebMvcTest",
      "@DataJpaTest",
      "@SpringBootTest",
      "@JsonTest"
    ],
    "correctIndex": 0,
    "explanation": "Combine with @MockBean."
  },
  {
    "question": "Why prefer Testcontainers to an in-memory H2 database for repository tests?",
    "options": [
      "Speed",
      "The real engine exposes dialect, constraint and locking behaviour that H2 hides",
      "Law",
      "Colour"
    ],
    "correctIndex": 1,
    "explanation": "Requires Docker."
  },
  {
    "question": "How should passwords be stored?",
    "options": [
      "MD5",
      "Hashed with BCrypt or Argon2 through a PasswordEncoder",
      "Plain text",
      "Reversible encryption"
    ],
    "correctIndex": 1,
    "explanation": "Use per-password salts."
  },
  {
    "question": "What must a resource server check on a JWT besides the signature?",
    "options": [
      "Length",
      "Colour",
      "Expiry, issuer and audience (and the algorithm)",
      "Nothing"
    ],
    "correctIndex": 2,
    "explanation": "A valid signature on an expired token is still invalid."
  },
  {
    "question": "What does a circuit breaker do?",
    "options": [
      "Retries forever",
      "Encrypts traffic",
      "Stops calling a failing dependency for a period so it can recover and callers fail fast",
      "Caches results"
    ],
    "correctIndex": 2,
    "explanation": "Combine with timeouts and backoff."
  },
  {
    "question": "Why add jitter to retry backoff?",
    "options": [
      "So many clients do not retry in lockstep and overwhelm a recovering service",
      "Law",
      "Speed",
      "Colour"
    ],
    "correctIndex": 0,
    "explanation": "Cap the attempts."
  },
  {
    "question": "What is the outbox pattern for?",
    "options": [
      "Compression",
      "Caching",
      "Saving the state change and the event in one transaction, then publishing the event reliably",
      "Auth"
    ],
    "correctIndex": 2,
    "explanation": "Avoids lost or phantom messages."
  },
  {
    "question": "When do virtual threads help?",
    "options": [
      "Blocking-style request handling at high concurrency without reactive code",
      "CPU-bound work",
      "Only batch",
      "GPU work"
    ],
    "correctIndex": 0,
    "explanation": "Enable with spring.threads.virtual.enabled=true on Java 21+."
  },
  {
    "question": "Which Actuator exposure is safest by default?",
    "options": [
      "All endpoints",
      "env and heapdump public",
      "None needed",
      "Only health (and info), with the rest secured or disabled"
    ],
    "correctIndex": 3,
    "explanation": "Endpoints can leak configuration."
  },
  {
    "question": "What is the main trade-off of GraalVM native images?",
    "options": [
      "Very fast startup and low memory, but longer builds and limits on reflection and dynamic features",
      "No limits",
      "Slower startup",
      "Bigger memory"
    ],
    "correctIndex": 0,
    "explanation": "Needs AOT hints."
  },
  {
    "question": "What proves a Spring Boot service is production-ready?",
    "options": [
      "It compiles",
      "CI with tests and scans, load results, dashboards, alerts and a runbook",
      "It starts",
      "It looks fine"
    ],
    "correctIndex": 1,
    "explanation": "Evidence over claims."
  }
]
</script>

## Coding challenges

Try these on your own computer, then compare your output. Nothing you write is sent anywhere.

### A tiny IoC container with constructor injection

Write a mini dependency-injection container in plain Java: map interfaces to implementations, create beans by resolving constructor parameters recursively, cache singletons and detect missing beans and circular dependencies with clear messages. This models what Spring's ApplicationContext does; it uses only the JDK, and Spring itself was not run.

**Starter code**

```java
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

**Expected output**

```text
Hello, Ana
same instance: true, shared audit: true
[run Ana]
no bean for Greeter
circular dependency: A -> B -> A
```

<details><summary>Show solution and explanation</summary>

<pre><code>import java.lang.reflect.Constructor;
import java.util.*;

public class Main {
    interface Greeter { String greet(String name); }
    static class EnglishGreeter implements Greeter { public String greet(String n) { return &quot;Hello, &quot; + n; } }
    static class Audit { final List&lt;String&gt; log = new ArrayList&lt;&gt;(); void add(String s) { log.add(s); } }
    static class Service {
        final Greeter greeter; final Audit audit;
        Service(Greeter g, Audit a) { this.greeter = g; this.audit = a; }
        String run(String n) { audit.add(&quot;run &quot; + n); return greeter.greet(n); }
    }
    static class A { A(B b) {} }
    static class B { B(A a) {} }

    /** A tiny IoC container: register implementations for types, resolve by constructor injection, singletons by default. */
    static class Container {
        private final Map&lt;Class&lt;?&gt;, Class&lt;?&gt;&gt; impls = new HashMap&lt;&gt;();
        private final Map&lt;Class&lt;?&gt;, Object&gt; singletons = new HashMap&lt;&gt;();
        private final Deque&lt;Class&lt;?&gt;&gt; resolving = new ArrayDeque&lt;&gt;();

        void register(Class&lt;?&gt; type, Class&lt;?&gt; impl) { impls.put(type, impl); }

        @SuppressWarnings(&quot;unchecked&quot;)
        &lt;T&gt; T get(Class&lt;T&gt; type) {
            if (singletons.containsKey(type)) return (T) singletons.get(type);
            if (resolving.contains(type)) throw new IllegalStateException(&quot;circular dependency: &quot; + path(type));
            resolving.push(type);
            try {
                Class&lt;?&gt; impl = impls.getOrDefault(type, type);
                if (impl.isInterface()) throw new IllegalStateException(&quot;no bean for &quot; + type.getSimpleName());
                Constructor&lt;?&gt; c = impl.getDeclaredConstructors()[0];
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

        private String path(Class&lt;?&gt; again) {
            List&lt;String&gt; names = new ArrayList&lt;&gt;();
            for (Class&lt;?&gt; c : resolving) names.add(c.getSimpleName());
            Collections.reverse(names);
            names.add(again.getSimpleName());
            return String.join(&quot; -&gt; &quot;, names);
        }
    }

    public static void main(String[] args) {
        Container ctx = new Container();
        ctx.register(Greeter.class, EnglishGreeter.class);
        Service s1 = ctx.get(Service.class);
        Service s2 = ctx.get(Service.class);
        System.out.println(s1.run(&quot;Ana&quot;));
        System.out.println(&quot;same instance: &quot; + (s1 == s2) + &quot;, shared audit: &quot; + (s1.audit == ctx.get(Audit.class)));
        System.out.println(ctx.get(Audit.class).log);
        try { new Container().get(Service.class); } catch (IllegalStateException e) { System.out.println(e.getMessage()); }
        try { new Container().get(A.class); } catch (IllegalStateException e) { System.out.println(e.getMessage()); }
    }
}
</code></pre>

<p>Reflection reads each constructor&#x27;s parameter types and get() resolves them recursively. A stack of types being resolved reveals cycles, and the singleton map gives Spring&#x27;s default scope. Verified by compiling and running with Java 20; real Spring adds scopes, lifecycle callbacks and proxies.</p>
</details>

### Retry with backoff and a circuit breaker

Combine retries (exponential backoff) with a circuit breaker that opens after repeated failures, allows one half-open probe after a cool-down and closes again on success. A fake clock makes the run deterministic. This is a plain-Java model of what Resilience4j provides in a Spring Boot service; no real service or Spring was involved.

**Starter code**

```java
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

**Expected output**

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

<details><summary>Show solution and explanation</summary>

<pre><code>import java.util.*;
import java.util.function.Supplier;

public class Main {
    /** Circuit breaker with retries and backoff, driven by a fake clock so output is deterministic. */
    enum State { CLOSED, OPEN, HALF_OPEN }

    static class Clock { long now = 0; void advance(long ms) { now += ms; } }

    static class Breaker {
        final int threshold; final long openMs; final Clock clock;
        State state = State.CLOSED; int failures = 0; long openedAt = 0;
        Breaker(int threshold, long openMs, Clock clock) { this.threshold = threshold; this.openMs = openMs; this.clock = clock; }

        &lt;T&gt; T call(Supplier&lt;T&gt; action) {
            if (state == State.OPEN) {
                if (clock.now - openedAt &gt;= openMs) state = State.HALF_OPEN;
                else throw new IllegalStateException(&quot;circuit open&quot;);
            }
            try {
                T result = action.get();
                failures = 0; state = State.CLOSED;
                return result;
            } catch (RuntimeException e) {
                failures++;
                if (state == State.HALF_OPEN || failures &gt;= threshold) { state = State.OPEN; openedAt = clock.now; }
                throw e;
            }
        }
    }

    static &lt;T&gt; T retry(int attempts, long baseMs, Clock clock, List&lt;String&gt; log, Supplier&lt;T&gt; action) {
        RuntimeException last = null;
        for (int i = 0; i &lt; attempts; i++) {
            try { return action.get(); }
            catch (RuntimeException e) {
                last = e;
                if (e.getMessage().equals(&quot;circuit open&quot;)) throw e;
                long wait = baseMs &lt;&lt; i;
                log.add(&quot;attempt &quot; + (i + 1) + &quot; failed (&quot; + e.getMessage() + &quot;), waiting &quot; + wait + &quot;ms&quot;);
                clock.advance(wait);
            }
        }
        throw last;
    }

    public static void main(String[] args) {
        Clock clock = new Clock();
        Breaker breaker = new Breaker(3, 1000, clock);
        List&lt;String&gt; log = new ArrayList&lt;&gt;();
        int[] calls = {0};
        Supplier&lt;String&gt; flaky = () -&gt; { calls[0]++; if (calls[0] &lt;= 4) throw new RuntimeException(&quot;boom&quot; + calls[0]); return &quot;ok&quot;; };

        try { System.out.println(retry(5, 100, clock, log, () -&gt; breaker.call(flaky))); }
        catch (RuntimeException e) { System.out.println(&quot;gave up: &quot; + e.getMessage()); }
        log.forEach(System.out::println);
        System.out.println(&quot;state=&quot; + breaker.state + &quot; calls=&quot; + calls[0] + &quot; t=&quot; + clock.now);

        clock.advance(1000);
        try { System.out.println(retry(2, 100, clock, log, () -&gt; breaker.call(flaky))); }
        catch (RuntimeException e) { System.out.println(&quot;half-open probe failed: &quot; + e.getMessage() + &quot;, state=&quot; + breaker.state); }
        clock.advance(1000);
        System.out.println(retry(2, 100, clock, log, () -&gt; breaker.call(flaky)));
        System.out.println(&quot;state=&quot; + breaker.state + &quot; calls=&quot; + calls[0] + &quot; t=&quot; + clock.now);
    }
}
</code></pre>

<p>The breaker counts consecutive failures, opens, and after openMs lets a single probe through; a failed probe re-opens it while a success closes it. retry() stops immediately when the circuit is open instead of waiting. Verified by compiling and running with Java 20.</p>
</details>
