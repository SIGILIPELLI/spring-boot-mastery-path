---
description: "Cache in Spring Boot with @Cacheable, @CachePut and @CacheEvict, choose Caffeine or Redis, design keys and TTLs, and avoid stale data and stampedes."
---

# Caching with Spring Cache

A cache trades freshness for speed: you remember an expensive answer and serve it again
instead of recomputing it. Spring's cache abstraction makes adding a cache a matter of
annotations. The hard part, as always, is deciding *what* may be stale and for *how
long*.

## When to cache

Good candidates are reads that are expensive (slow queries, remote calls), frequent, and
tolerant of slight staleness: product catalogs, reference data, exchange rates fetched
every minute, permission lookups. Poor candidates are data that must be exactly current
(account balances before a withdrawal), data that is cheap to compute, or data that is
rarely requested twice.

Measure first. A cache in front of a 2 ms indexed query mostly adds complexity.

## Enabling and using it

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-cache</artifactId>
</dependency>
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```java
@Configuration
@EnableCaching
class CacheConfig { }
```

```yaml
spring:
  cache:
    cache-names: books, authors
    caffeine:
      spec: maximumSize=10000,expireAfterWrite=10m
```

```java
@Service
@Transactional(readOnly = true)
public class CatalogService {

    @Cacheable("books")
    public BookView get(long id) {
        return books.findById(id).map(BookView::from)
                .orElseThrow(() -> new BookNotFoundException(id));
    }

    @Transactional
    @CachePut(cacheNames = "books", key = "#id")
    public BookView retitle(long id, String title) { ... }   // result replaces the cache entry

    @Transactional
    @CacheEvict(cacheNames = "books", key = "#id")
    public void delete(long id) { ... }

    @CacheEvict(cacheNames = "books", allEntries = true)
    public void reloadCatalog() { ... }
}
```

- `@Cacheable` — return the cached value if present; otherwise call the method and cache
  the result. Exceptions are not cached.
- `@CachePut` — always call the method and store the result.
- `@CacheEvict` — remove an entry (or everything) after the method runs
  (`beforeInvocation = true` to evict first).
- Keys default to the method parameters (one parameter → that value; several → a
  `SimpleKey` of all). Use SpEL (`key = "#id"`, `key = "#request.isbn()"`) to be explicit.
- `condition = "#id > 0"` and `unless = "#result == null"` control what gets cached.

**Cache DTOs, not entities.** A cached entity is detached, carries lazy proxies that will
fail later, and is mutable — someone may modify the cached instance. Records are ideal.

## Choosing a cache provider

| Provider | Where data lives | Good for | Trade-offs |
|----------|-----------------|----------|-----------|
| `ConcurrentMapCache` (default with no provider) | JVM heap, no eviction | Tests, demos | Unbounded — do not use in production |
| **Caffeine** | JVM heap, per instance | Hot reference data, very low latency | Each instance has its own copy; invalidation does not reach other instances |
| **Redis** | Shared server | Consistency across instances, larger data, survives restarts | Network hop, serialization, another system to run |

With several application instances behind a load balancer, a local Caffeine cache means
an update on instance A leaves stale data on B until it expires. That is often fine with a
short TTL. If it is not, use Redis (`spring-boot-starter-data-redis` plus
`spring.cache.type=redis`, TTL via `spring.cache.redis.time-to-live`), or a two-level
design with short local TTLs in front of Redis.

## Worked example: timing the difference

A simple way to see the cache work, without inventing benchmark numbers:

```java
@SpringBootTest
class CachingTest {
    @Autowired CatalogService catalog;
    @MockitoSpyBean BookRepository books;

    @Test
    void secondReadIsServedFromCache() {
        long id = /* seed a book */ 1L;
        catalog.get(id);
        catalog.get(id);
        verify(books, times(1)).findById(id);   // the repository was called once
    }
}
```

Counting calls to the collaborator proves the cache is used and is deterministic, unlike
wall-clock timing. Add a test that evicts and asserts a second call to the repository.

## Stampedes and consistency

- **Stampede:** when a popular entry expires, hundreds of concurrent requests miss at once
  and all hit the database. `@Cacheable(sync = true)` makes concurrent callers for the same
  key wait for one computation (supported by Caffeine and Redis providers).
- **Update ordering:** with `@CachePut` inside a transaction, the cache is updated when
  the method returns — *before* the transaction commits if the cache interceptor wraps
  inside the transaction interceptor, or after, depending on advisor order. If the
  transaction then rolls back, the cache holds a value that never existed. Evicting (rather
  than putting) on writes, or wrapping the cache in a
  `TransactionAwareCacheManagerProxy` so puts and evictions happen after commit, avoids
  this.
- **TTL is your safety net.** Even with careful eviction, some path will forget. A TTL
  bounds how long a mistake lives.

## How It Actually Works

`@EnableCaching` registers a `CacheInterceptor` and an advisor that matches methods
carrying cache annotations — the same proxy mechanism as `@Transactional` (lesson 02). For
a `@Cacheable` call, the interceptor:

1. Evaluates the key from the arguments (SpEL or the default `SimpleKeyGenerator`).
2. Asks the `CacheManager` for the named `Cache` and calls `cache.get(key)`.
3. On a hit, returns the value **without invoking your method**.
4. On a miss, invokes the method, then `cache.put(key, result)` unless `unless` says
   otherwise.

Boot's `CacheAutoConfiguration` picks the `CacheManager` by checking which provider is on
the classpath (Caffeine, Redis, JCache, …) or honoring `spring.cache.type`, and configures
it from `spring.cache.*`.

Because it is proxy-based, everything you learned about proxies applies: calling a
`@Cacheable` method from inside the same class **skips the cache**, and `private` methods
cannot be cached.

## Common mistakes

- **Self-invocation**, as above.
- **Caching mutable entities.**
- **Unbounded caches** (the default `ConcurrentMapCacheManager`), turning into memory
  leaks.
- **Keys that ignore a relevant parameter** — caching `search(query, page)` by `query`
  alone returns page 1 for every page.
- **Caching per-user data under a key that does not include the user.** This is a data
  leak, not just a bug.

## Exercise

1. Add Caffeine caching to `CatalogService.get` with a 10-minute TTL and a maximum size.
2. Write the spy-based test above, plus one for eviction after `delete`.
3. Call the cached method from another method in the same class and prove with the spy
   that the cache is bypassed.
4. If you have Docker, switch to Redis with `spring.cache.type=redis`, run two instances of
   the app on different ports, and show that an eviction on one is visible on the other.
   Then switch back to Caffeine and show that it is not.
