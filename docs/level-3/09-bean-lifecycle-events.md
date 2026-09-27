---
description: "Follow a Spring bean from definition to destruction, see where post-processors create proxies, and decouple code with application and transactional events."
---

# Bean Lifecycle & Application Events

You have seen pieces of the lifecycle already: definitions registered by scanning,
constructors called with dependencies, proxies swapped in by post-processors. This lesson
assembles the full sequence — observed, not just described — and then covers application
events, Spring's built-in way to decouple components inside one process.

## The lifecycle, observed

This bean implements every common lifecycle hook and prints a line from each:

```java
@Component
class LifecycleDemo implements BeanNameAware, InitializingBean, DisposableBean, SmartLifecycle {
    private volatile boolean running;

    LifecycleDemo() { System.out.println("1 constructor"); }
    @Override public void setBeanName(String name) { System.out.println("2 setBeanName(" + name + ")"); }
    @PostConstruct void postConstruct() { System.out.println("3 @PostConstruct"); }
    @Override public void afterPropertiesSet() { System.out.println("4 afterPropertiesSet"); }
    @Override public void start() { running = true; System.out.println("5 SmartLifecycle.start"); }
    @EventListener(ApplicationReadyEvent.class) void ready() { System.out.println("6 ApplicationReadyEvent"); }
    @EventListener(ContextClosedEvent.class) void closed() { System.out.println("7 ContextClosedEvent"); }
    @Override public void stop() { running = false; System.out.println("8 SmartLifecycle.stop"); }
    @PreDestroy void preDestroy() { System.out.println("9 @PreDestroy"); }
    @Override public void destroy() { System.out.println("10 DisposableBean.destroy"); }
    @Override public boolean isRunning() { return running; }
}
```

Running the Level 1 app with it (Boot 4.1), then stopping it with `SIGTERM`, printed:

```text
1 constructor
2 setBeanName(lifecycleDemo)
3 @PostConstruct
4 afterPropertiesSet
5 SmartLifecycle.start
6 ApplicationReadyEvent
7 ContextClosedEvent
8 SmartLifecycle.stop
9 @PreDestroy
10 DisposableBean.destroy
```

Nobody implements all of these in real code. In practice:

- Use **constructor injection** for dependencies and **`@PostConstruct`** for simple
  initialization that needs them.
- Use **`SmartLifecycle`** for components that must start after the whole context is ready
  and stop in a controlled order (message listeners, schedulers, connection pools).
  `getPhase()` orders them; higher phases start later and stop earlier.
- Use **`ApplicationReadyEvent`** for "the app is fully up" work, such as warming a cache.
- Use **`@PreDestroy`** to release resources. `DisposableBean` does the same thing with a
  Spring interface in your class.

## How It Actually Works

For each singleton, `AbstractAutowireCapableBeanFactory.doCreateBean` runs:

1. **Instantiate** — choose the constructor and resolve its arguments (dependencies are
   created first, recursively).
2. **Populate** — inject fields and setters (`@Autowired`, `@Value`), via the
   `AutowiredAnnotationBeanPostProcessor`.
3. **Aware callbacks** — `BeanNameAware`, `BeanFactoryAware`, `ApplicationContextAware`, …
4. **`BeanPostProcessor.postProcessBeforeInitialization`** for every registered
   post-processor. `@PostConstruct` is actually invoked here, by
   `CommonAnnotationBeanPostProcessor` — which is why it runs before `afterPropertiesSet`.
5. **Init methods** — `InitializingBean.afterPropertiesSet`, then any custom
   `@Bean(initMethod = …)`.
6. **`BeanPostProcessor.postProcessAfterInitialization`** — the step where
   the AOP auto-proxy creator may **replace the bean with a proxy** (`@Transactional`,
   `@Async`, `@Cacheable`, aspects). From here on, the container hands out the proxy.
7. **Register for destruction** if the bean has destroy callbacks.

After *all* singletons exist, the context's `finishRefresh` starts `SmartLifecycle` beans
(phase order) — that is why step 5 in the output comes after every `@PostConstruct` in the
application — and Boot then starts the web server and publishes `ApplicationReadyEvent`.

Shutdown reverses it. A `SIGTERM` triggers Boot's shutdown hook, which closes the context:
publish `ContextClosedEvent`, stop `SmartLifecycle` beans (reverse phase; with
`server.shutdown: graceful`, the web server's lifecycle waits for in-flight requests
here), then destroy singletons in reverse dependency order — `@PreDestroy` (again via
`CommonAnnotationBeanPostProcessor`), then `DisposableBean.destroy`.

Two practical corollaries: code in a constructor or `@PostConstruct` runs **before**
proxies exist, so calling your own `@Transactional` method there gets no transaction; and
`BeanPostProcessor`s themselves are created very early, so a post-processor that depends
on ordinary beans forces those beans to be created early, *without* being post-processed
by later processors. Boot logs a warning ("is not eligible for getting processed by all
BeanPostProcessors") when this happens.

## Application events

Events let one component announce that something happened without knowing who cares:

```java
public record BookRetitled(long bookId, String oldTitle, String newTitle) { }

@Service
class CatalogService {
    private final ApplicationEventPublisher events;
    // ...
    @Transactional
    public BookView retitle(long id, String newTitle) {
        Book book = books.findById(id).orElseThrow();
        String old = book.getTitle();
        book.setTitle(newTitle);
        events.publishEvent(new BookRetitled(id, old, newTitle));
        return BookView.from(book);
    }
}

@Component
class SearchIndexUpdater {
    @EventListener
    void on(BookRetitled e) { /* reindex */ }
}
```

Key behaviors:

- **Synchronous by default.** `publishEvent` calls each listener on the same thread,
  before returning. An exception in a listener propagates to the publisher and — inside a
  transaction — rolls it back.
- **`@TransactionalEventListener`** defers the listener until the transaction reaches a
  phase, `AFTER_COMMIT` by default. The listener then sees committed data and cannot roll
  the publisher back. If no transaction is active, the event is dropped unless
  `fallbackExecution = true`.
- **Add `@Async`** to a listener to run it on another thread.
- Events are **in-memory only**. If the process dies after commit but before an
  `AFTER_COMMIT` listener finishes, the work is lost. For must-happen side effects, use the
  outbox (lesson 06); Spring Modulith's event publication registry (Level 4) persists
  events for you.

## Worked example: cache invalidation via events

Instead of `CatalogService` knowing about caches, search indexes, and audit logs:

```java
@Component
class CatalogCacheInvalidator {
    private final CacheManager caches;
    CatalogCacheInvalidator(CacheManager caches) { this.caches = caches; }

    @TransactionalEventListener
    void on(BookRetitled e) {
        Cache cache = caches.getCache("books");
        if (cache != null) cache.evict(e.bookId());
    }
}
```

The eviction happens only after the new title is committed — avoiding the
"cache refilled with the old value before commit" race from lesson 04 — and the catalog
service no longer depends on the cache.

## Common mistakes

- **Heavy work in constructors or `@PostConstruct`** (network calls, big loads), making
  startup slow and failures confusing. Use `ApplicationReadyEvent` or a `SmartLifecycle`.
- **Calling proxied methods during initialization.**
- **Expecting `@EventListener` to be asynchronous or transactional.** It is neither by
  default.
- **Using events to hide a required call.** If the operation must fail when the side
  effect fails, call it directly.
- **Using `@TransactionalEventListener` outside a transaction** and wondering why it never
  fires.

## Exercise

1. Add `LifecycleDemo` to your app, run it, and stop it with Ctrl+C. Compare your output
   with the listing above. Then add `@Transactional` to one of its methods and print
   `getClass()` in `@PostConstruct` and in the `ApplicationReadyEvent` listener — explain
   the difference in terms of step 6.
2. Implement `BookRetitled` with two listeners: a plain `@EventListener` that throws, and
   show that the retitle is rolled back; then change it to `@TransactionalEventListener`
   and show that it is not.
3. Implement the cache invalidator and test that a retitle evicts the cached entry.
4. Write a `SmartLifecycle` "consumer" with `getPhase()` and show, with logs, that it stops
   before your `@PreDestroy` methods run.
