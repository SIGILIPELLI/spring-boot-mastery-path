---
description: "Declare Spring beans with stereotypes and @Bean methods, choose scopes, control scanning, and understand why @Configuration classes are proxied."
---

# Beans, Scopes & Component Scanning

A *bean* is simply an object whose lifecycle the Spring container manages. There are two
ways to tell the container about one, and choosing between them is mostly about who owns
the class.

## Way 1: stereotype annotations on your own classes

```java
@Component   // generic
@Service     // business logic
@Repository  // data access (also enables exception translation)
@Controller / @RestController   // web layer
@Configuration                  // a source of @Bean definitions
```

All of these are meta-annotated with `@Component`, so scanning finds them the same way.
The specific stereotypes are mostly documentation, with two real effects:
`@Repository` beans get persistence-exception translation (vendor exceptions become
Spring's `DataAccessException` hierarchy), and `@Controller`/`@RestController` beans are
picked up by Spring MVC's handler mapping.

## Way 2: `@Bean` methods for classes you do not own

You cannot annotate `java.time.Clock`, an `HttpClient`, or a third-party SDK client. Write a
factory method instead:

```java
@Configuration
class InfrastructureConfig {

    @Bean
    Clock clock() {
        return Clock.systemUTC();
    }

    @Bean
    HttpClient httpClient() {
        return HttpClient.newBuilder()
                .connectTimeout(Duration.ofSeconds(2))
                .build();
    }

    @Bean
    PricingEngine pricingEngine(Clock clock, List<PricingRule> rules) {   // parameters are injected
        return new PricingEngine(clock, rules);
    }
}
```

The bean's name is the method name (`clock`, `httpClient`). Parameters of `@Bean`
methods are resolved exactly like constructor parameters.

## Scopes

| Scope | One instance per… | Typical use |
|-------|------------------|-------------|
| `singleton` (default) | container | Services, repositories, controllers, clients — almost everything |
| `prototype` | injection point / `getBean` call | Stateful helpers that must not be shared |
| `request` | HTTP request | Per-request context (web apps only) |
| `session` | HTTP session | Per-user state in server-rendered apps |

Singletons are shared by every thread handling requests. That is the single most
important consequence of scopes: **a singleton must be thread-safe**, which in practice
means *stateless* — only `final` references to other beans and immutable configuration.

```java
@Service
class ReportService {
    private final List<String> lines = new ArrayList<>();   // BUG: shared by all requests

    String build(Order order) {
        lines.clear();
        lines.add("Order " + order.id());
        return String.join("\n", lines);
    }
}
```

Under concurrent load, two requests interleave `clear()` and `add()` and produce each
other's reports. Move `lines` into the method as a local variable.

## Controlling component scanning

`@SpringBootApplication` scans its own package and below. To register beans only in
some situations, use conditions and profiles rather than moving packages:

```java
@Component
@Profile("dev")
class FakeEmailSender implements EmailSender { ... }   // only when the dev profile is active

@Component
@ConditionalOnProperty(name = "features.audit.enabled", havingValue = "true")
class AuditListener { ... }                           // only when a property is set
```

`@Profile` is covered with configuration in the next lesson. `@ConditionalOn…` annotations
are the same mechanism auto-configuration uses (Level 3, lesson 1).

## Worked example: plug-in rules

A pricing engine that discovers its rules as beans:

```java
public interface PricingRule {
    BigDecimal apply(Order order, BigDecimal current);
}

@Component
@Order(1)
class BulkDiscount implements PricingRule {
    public BigDecimal apply(Order order, BigDecimal current) {
        return order.quantity() >= 10 ? current.multiply(new BigDecimal("0.90")) : current;
    }
}

@Component
@Order(2)
class RoundToCents implements PricingRule {
    public BigDecimal apply(Order order, BigDecimal current) {
        return current.setScale(2, RoundingMode.HALF_UP);
    }
}

@Service
class PricingEngine {
    private final List<PricingRule> rules;   // injected in @Order order

    PricingEngine(List<PricingRule> rules) { this.rules = rules; }

    BigDecimal price(Order order) {
        BigDecimal price = order.unitPrice().multiply(BigDecimal.valueOf(order.quantity()));
        for (PricingRule rule : rules) price = rule.apply(order, price);
        return price;
    }
}
```

Adding a seasonal rule is now a new class, nothing else. The ordering matters here (you
must round *after* discounting), so it is stated explicitly with `@Order`.

## How It Actually Works

**Why `@Configuration` classes are special.** Look at this:

```java
@Configuration
class AppConfig {
    @Bean DataSource dataSource() { return new HikariDataSource(); }
    @Bean JdbcTemplate jdbcTemplate() { return new JdbcTemplate(dataSource()); }
    @Bean TransactionManager txManager() { return new DataSourceTransactionManager(dataSource()); }
}
```

In plain Java, `dataSource()` is called twice and creates two pools. In Spring it
creates one. At startup the container generates a CGLIB **subclass** of `AppConfig` and
overrides every `@Bean` method: the override first checks the singleton cache and returns
the existing bean if present, and only calls your original method the first time. That is
why `@Configuration` classes and `@Bean` methods must not be `final`.

If you write `@Configuration(proxyBeanMethods = false)` (Boot's own auto-configurations
do), no subclass is generated and calling `dataSource()` directly really *would* create a
second instance. In that "lite" mode you should receive dependencies as `@Bean` method
parameters instead — which is the clearer style anyway, and it starts slightly faster.

**Scoped beans inside singletons.** A singleton is created once, so if it injects a
`request`-scoped bean directly, which request's instance should it hold? Spring solves
this by injecting a *scoped proxy*: a stand-in object that, on every method call, looks up
the real instance for the current request and delegates. Web scopes are proxied by
default for this reason. A `prototype` bean injected into a singleton, however, is *not*
proxied — you get exactly one instance forever. If you really want a fresh one each time,
inject `ObjectProvider<MyPrototype>` and call `getObject()` when needed.

**Bean names.** A scanned class `TaskService` gets the name `taskService`. Two classes with
the same simple name in different packages collide and fail startup with a
`ConflictingBeanDefinitionException`; give one an explicit name
(`@Service("legacyTaskService")`).

## Common mistakes

- **Mutable fields in singletons** (caches in a `HashMap`, "current user" fields). Use
  local variables, `ConcurrentHashMap`, or a proper cache abstraction.
- **Making `@Configuration` classes or `@Bean` methods `final`** — the CGLIB subclass
  cannot override them.
- **Expecting a prototype bean to be new on every call** when it is injected into a
  singleton. It is created once, at injection time.
- **Scanning too widely** with `@ComponentScan("com")`, which can pick up test
  configurations or library classes. Keep the default.

## Exercise

1. Implement the pricing engine above with two rules and an `@SpringBootTest` (or a plain
   unit test that builds the list by hand) proving the discount is applied before rounding.
2. Add a third rule, `WeekendSurcharge`, that uses an injected `Clock`. Register the
   `Clock` via a `@Bean` method and test the rule with a fixed clock.
3. In a `@Configuration` class, call one `@Bean` method from another and log
   `System.identityHashCode` of the result in both places. Then switch to
   `proxyBeanMethods = false` and observe the difference.
4. Write the thread-unsafe `ReportService` above and a test that calls `build` from 50
   threads with an `ExecutorService`. Show that it produces wrong output, then fix it.
