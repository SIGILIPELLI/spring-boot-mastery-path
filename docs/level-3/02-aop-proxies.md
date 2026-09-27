---
description: "How Spring AOP works: JDK dynamic proxies vs CGLIB subclasses, pointcuts and advice, writing a timing aspect, and the limits of proxy-based interception."
---

# AOP & Proxies

`@Transactional`, `@Cacheable`, `@Async`, `@PreAuthorize`, `@Retryable`, and Spring Data
repositories all rely on the same trick: the bean you inject is not your object but a
**proxy** that runs extra code around your method. This mechanism is called
aspect-oriented programming (AOP) in Spring's vocabulary. Understanding it explains a
whole family of "the annotation does nothing" bugs at once.

## The vocabulary, briefly

- **Aspect** — a module of cross-cutting behavior (timing, auditing, retries).
- **Join point** — a point where behavior can be inserted. In Spring AOP, always a *method
  execution* on a Spring bean.
- **Pointcut** — an expression selecting join points (`execution(* com.example..*Service.*(..))`,
  `@annotation(Timed)`).
- **Advice** — the code to run: `@Before`, `@AfterReturning`, `@AfterThrowing`, `@After`,
  or `@Around` (which wraps the call and decides whether to proceed).

## Writing an aspect

Add `spring-boot-starter-aspectj` (Boot 4; in Boot 3 it is `spring-boot-starter-aop`).
Then:

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface LogSlow {
    long thresholdMs() default 200;
}

@Aspect
@Component
class SlowCallAspect {
    private static final Logger log = LoggerFactory.getLogger(SlowCallAspect.class);

    @Around("@annotation(logSlow)")
    Object time(ProceedingJoinPoint pjp, LogSlow logSlow) throws Throwable {
        long start = System.nanoTime();
        try {
            return pjp.proceed();
        } finally {
            long ms = (System.nanoTime() - start) / 1_000_000;
            if (ms >= logSlow.thresholdMs()) {
                log.warn("Slow call {} took {} ms", pjp.getSignature().toShortString(), ms);
            }
        }
    }
}

@Service
class ReportService {
    @LogSlow(thresholdMs = 100)
    public Report monthly(YearMonth month) { ... }
}
```

Binding the annotation as a parameter (`@annotation(logSlow)` + `LogSlow logSlow`) gives
the advice access to its attributes.

Before writing an aspect for metrics, check whether Micrometer's `@Timed`/`@Observed`
already does it (Level 3, lesson 08). Custom aspects are best for genuinely
application-specific concerns.

## JDK proxies vs CGLIB

Spring can create two kinds of proxy:

| | JDK dynamic proxy | CGLIB subclass |
|---|---|---|
| How | Implements the bean's **interfaces** | **Subclasses** the bean's class |
| Injectable as | The interface only | The interface or the class |
| Cannot intercept | Methods not on an interface | `final` methods, `private` methods, `final` classes |
| Used by | Spring Data repositories (interfaces by nature) | Boot's default for everything else (`spring.aop.proxy-target-class=true`) |

Boot defaults to CGLIB so that injecting a concrete class (`ReportService` rather than an
interface) works. In the Level 2 project the injected service printed its class as
`TxDemoService$$SpringCGLIB$$0` — a CGLIB subclass generated at startup.

## Worked example: seeing the proxy

Put a breakpoint (or `new Exception().printStackTrace()`) inside a `@Transactional` method
and read the stack from the bottom up. Between your controller frame and your service
method you will find frames like:

```text
com.example.library.CatalogService.retitle(CatalogService.java:42)
... (reflection frames)
org.springframework.aop.framework.ReflectiveMethodInvocation.proceed(...)
org.springframework.transaction.interceptor.TransactionInterceptor.invoke(...)
org.springframework.aop.framework.CglibAopProxy$DynamicAdvisedInterceptor.intercept(...)
com.example.library.CatalogService$$SpringCGLIB$$0.retitle(<generated>)
com.example.library.BookController.retitle(BookController.java:31)
```

That is the whole mechanism on one screen: the controller calls the generated subclass,
which hands the call to a chain of interceptors, the last of which invokes your real
method by reflection. (Exact frame names vary a little between versions; the structure
does not.)

## How It Actually Works

**Creation.** `AnnotationAwareAspectJAutoProxyCreator` is a `BeanPostProcessor`. After each
bean is instantiated and initialized, its `postProcessAfterInitialization` asks: do any
**advisors** apply to this bean's class? Advisors come from `@Aspect` beans (one per
advice method, with its pointcut) and from Spring's own infrastructure (the transaction
advisor matches methods annotated `@Transactional`, the cache advisor matches
`@Cacheable`, and so on). If any match, it builds a `ProxyFactory` with the target object
and the matching advisors, and returns the **proxy** instead of the original bean. The
container stores and injects the proxy from then on.

**Invocation.** Calling a method on the proxy creates a `ReflectiveMethodInvocation`
holding the ordered list of interceptors that apply to *that method*. Each interceptor's
`invoke` does its work and calls `proceed()`, which moves to the next interceptor; the
last `proceed()` calls the target method. `@Around` advice is literally one of these
interceptors, and `pjp.proceed()` is that call. Order between aspects is controlled with
`@Order` — the transaction advisor, for example, has lowest precedence by default, so it
is the innermost wrapper unless configured otherwise.

**The consequences, all from "a wrapper around an object":**

1. **Self-invocation bypasses advice.** Inside the target, `this.method()` is a plain call on
   the target, not the proxy. (Level 2, lesson 04 showed this with real output.)
2. **`private` and `final` methods are never advised** with CGLIB; methods not declared on
   an interface are not advised with JDK proxies.
3. **Only Spring beans are advised.** Objects created with `new` are not proxied.
4. **Nothing is advised during construction**, because the proxy is created *after*
   initialization.

If you truly need interception of internal calls, constructors, or non-bean objects, that
is what full AspectJ weaving (compile-time or load-time) provides. It is rarely worth the
build complexity; restructuring into separate beans almost always is.

## Common mistakes

- **Expecting an aspect to fire on an internal call.** Move the method to another bean.
- **Aspects with side effects on the return value** that surprise callers. Keep advice
  transparent unless changing behavior is the whole point (retries, caching).
- **Pointcuts that are too broad** (`execution(* *(..))`) — every bean gets proxied,
  startup slows, and advice runs where nobody expects it.
- **Swallowing exceptions in `@Around`** by forgetting to rethrow.
- **Checking `obj.getClass() == ReportService.class`** in code that receives proxies. Use
  `AopUtils.getTargetClass(obj)` or, better, avoid class checks.

## Exercise

1. Implement `@LogSlow` and apply it to a service method with an artificial
   `Thread.sleep`. Confirm the warning appears.
2. Call the annotated method from another method of the same class and confirm the aspect
   does *not* fire. Fix it by moving the method.
3. Print `bean.getClass()` and `AopUtils.isCglibProxy(bean)` for a service with and without
   an advised method. Then inject a Spring Data repository and print its class — explain
   why it is a JDK proxy (`jdk.proxy…$Proxy…`).
4. Write two aspects on the same method with different `@Order` values that each log
   "enter"/"exit", and predict the log order before running it.
