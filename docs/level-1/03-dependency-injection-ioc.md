---
description: "How Spring's IoC container builds your object graph: constructor injection, programming to interfaces, resolving ambiguity, and why cycles fail."
---

# Dependency Injection & the IoC Container

Dependency injection (DI) is a simple idea: an object should *receive* the things it
depends on instead of creating them. Spring's contribution is a container that does the
receiving for you across a whole application. Once you see DI as "constructor arguments
supplied by someone else," most of Spring stops looking magical.

## Without and with DI

```java
// Hard-wired: TaskService decides its own collaborators.
public class TaskService {
    private final TaskRepository repository = new InMemoryTaskRepository();
    private final Clock clock = Clock.systemUTC();
}
```

This class cannot be tested with a fixed clock, cannot switch to a database repository
without editing it, and hides its dependencies. Compare:

```java
public class TaskService {
    private final TaskRepository repository;
    private final Clock clock;

    public TaskService(TaskRepository repository, Clock clock) {
        this.repository = repository;
        this.clock = clock;
    }
}
```

Now the dependencies are explicit, `final`, and replaceable. This version is pure Java —
no Spring annotations are needed for it to be well designed. Spring's job is only to call
that constructor with the right arguments.

## Letting Spring wire it

```java
public interface TaskRepository {
    Task save(Task task);
    Optional<Task> findById(long id);
    List<Task> findAll();
}

@Repository
class InMemoryTaskRepository implements TaskRepository {
    private final Map<Long, Task> store = new ConcurrentHashMap<>();
    // ... implementations
}

@Service
public class TaskService {
    private final TaskRepository repository;
    private final Clock clock;

    public TaskService(TaskRepository repository, Clock clock) {   // no @Autowired needed
        this.repository = repository;
        this.clock = clock;
    }
}

@Configuration
class TimeConfig {
    @Bean
    Clock clock() {
        return Clock.systemUTC();
    }
}
```

At startup, the container finds `TaskService` (a `@Service`), sees its only constructor
needs a `TaskRepository` and a `Clock`, finds exactly one bean of each type, and calls the
constructor. When a class has a single constructor, `@Autowired` is optional; you only
need it to choose between several constructors.

## Three injection styles, one recommendation

| Style | Example | Verdict |
|-------|---------|---------|
| Constructor | `TaskService(TaskRepository r)` | **Use this.** Dependencies are final, required, visible, and tests just call `new`. |
| Setter | `@Autowired void setRepository(TaskRepository r)` | For genuinely optional dependencies, rarely. |
| Field | `@Autowired private TaskRepository r;` | Avoid in production code: hides dependencies, prevents `final`, needs reflection or a container to test. |

Field injection is common in older tutorials and in test classes (where it is fine —
`@Autowired MockMvc mvc;` in a test is idiomatic).

## Resolving ambiguity

What if two beans implement `TaskRepository`?

```java
@Repository class InMemoryTaskRepository implements TaskRepository { ... }
@Repository class JdbcTaskRepository implements TaskRepository { ... }
```

Startup fails with a message like *"Parameter 0 of constructor in TaskService required a
single bean, but 2 were found"*. You have three tools:

```java
// 1. Mark one as the default.
@Primary
@Repository class JdbcTaskRepository implements TaskRepository { ... }

// 2. Ask for a specific one by qualifier (the bean name defaults to the
//    uncapitalized class name).
public TaskService(@Qualifier("inMemoryTaskRepository") TaskRepository repository, Clock clock)

// 3. Inject all of them when you actually want all of them.
public NotificationService(List<Notifier> notifiers)   // every Notifier bean, ordered by @Order
```

Option 3 is underrated. A list of strategies — notifiers, validators, pricing rules — is a
clean way to make behavior pluggable: adding a new `@Component` that implements the
interface is enough to include it.

Optional dependencies can be injected as `ObjectProvider<T>`, which lets you call
`getIfAvailable()` instead of failing at startup when no bean exists.

## Worked example: testing without the container

Because `TaskService` takes everything through its constructor, its unit test needs no
Spring at all:

```java
class TaskServiceTest {
    private final Clock fixed = Clock.fixed(Instant.parse("2026-01-01T09:00:00Z"), ZoneOffset.UTC);

    @Test
    void newTasksAreStampedWithTheClock() {
        var service = new TaskService(new InMemoryTaskRepository(), fixed);

        Task task = service.create(new CreateTaskRequest("Write docs", null));

        assertThat(task.createdAt()).isEqualTo(Instant.parse("2026-01-01T09:00:00Z"));
    }
}
```

This test starts in milliseconds and would be impossible if `TaskService` called
`Clock.systemUTC()` itself. Designing for DI and designing for testability are the same
activity.

## How It Actually Works

The container works in two phases.

**Phase 1: definitions.** Component scanning reads class files (using ASM, *without*
loading the classes into the JVM) looking for stereotype annotations, and registers a
`BeanDefinition` for each: the class, scope, whether it is lazy, and which constructor or
factory method to use. `@Bean` methods in `@Configuration` classes also become
definitions, with the method as the factory. Nothing has been instantiated yet.

**Phase 2: instantiation.** During context refresh, the `DefaultListableBeanFactory`
walks the singleton definitions. To create `TaskService` it:

1. Picks the constructor (the only one, or the `@Autowired` one).
2. For each parameter, calls `resolveDependency`, which looks up all bean *names* whose
   type matches, narrows by `@Qualifier`, then `@Primary`, then — as a last resort — by
   matching the parameter name against bean names.
3. Recursively creates any dependency not yet created (the repository, the clock),
   registering each finished singleton in a cache.
4. Invokes the constructor, then hands the new object to every `BeanPostProcessor` —
   which may return a *different* object, such as a proxy.

**Why constructor cycles fail.** If `A(B b)` and `B(A a)`, creating A requires a finished
B, which requires a finished A. There is no half-built object to hand over, so Spring
throws `BeanCurrentlyInCreationException` and Boot reports "The dependencies of some of the
beans in the application context form a cycle." Since Boot 2.6 circular references are
prohibited by default even for field injection, where older versions would silently
inject an early reference. The fix is almost always design: extract the shared logic into
a third class that both depend on, or publish an event instead of calling back.

## Common mistakes

- **Calling `new` on a Spring-managed class** inside another bean. You get an object the
  container never saw: no injected dependencies, no `@Transactional` proxy.
- **Injecting concrete classes everywhere.** It works, but you lose the ability to swap
  implementations. Inject interfaces where more than one implementation is plausible.
- **Constructors with eight parameters.** Spring will happily inject them, but the class
  is doing too much. DI makes this smell visible; do not suppress it with field injection.
- **Using `ApplicationContext.getBean(...)` in business code** (service locator). It hides
  dependencies again. Reserve it for framework-level code.

## Exercise

1. Create `TaskRepository` with an in-memory implementation and a `TaskService` that
   depends on it and on `Clock`, using constructor injection only.
2. Add a second implementation, `LoggingTaskRepository`, that wraps another repository
   and logs every save. Make the app start by using `@Qualifier` so `LoggingTaskRepository`
   receives the in-memory one and `TaskService` receives the logging one. (Hint: this is
   the decorator pattern, and you need both a `@Primary` and a `@Qualifier`.)
3. Deliberately create a constructor cycle between two services, read the startup
   failure, then remove it by extracting a third class.
4. Write the `TaskServiceTest` above with a fixed `Clock` and no Spring annotations.
