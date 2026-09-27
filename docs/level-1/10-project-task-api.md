---
description: "Level 1 project: build a tested task-tracker REST API with validation, ProblemDetail errors, typed configuration, logging, and an in-memory store."
---

# Project — A Task Tracker REST API

Time to put Level 1 together. You will build a small but properly structured REST API for
tasks: create, list with a filter, read, change status, delete. It validates input, returns
standard problem-detail errors, reads a limit from typed configuration, logs meaningful
events, and has controller tests.

Everything on this page was built and run against Spring Boot 4.1.1 while writing the
course; the requests and responses shown are the real ones (only the port was changed
from 8089 to the default 8080 in the commands).

## Requirements

| Endpoint | Behavior |
|----------|---------|
| `POST /api/tasks` | Create with `title` (required, ≤120 chars) and optional `description` (≤2000). 201 + `Location`. 409 when the open-task limit is reached. |
| `GET /api/tasks?status=` | List all tasks, optionally filtered by status, ordered by id. |
| `GET /api/tasks/{id}` | One task or 404. |
| `PATCH /api/tasks/{id}/status` | Body `{"status":"DONE"}`. 400 if missing/invalid. |
| `DELETE /api/tasks/{id}` | 204, or 404 if absent. |

Non-functional: errors are `application/problem+json`; the limit comes from
`tasks.max-open-tasks` and is validated at startup; each creation is logged.

## Setup

```bash
curl -s -o tasks.zip "https://start.spring.io/starter.zip?type=maven-project&language=java&javaVersion=17&groupId=com.example&artifactId=tasks&packageName=com.example.tasks&dependencies=web,validation"
unzip tasks.zip -d tasks && cd tasks
```

Package layout:

```text
com.example.tasks
├── TasksApplication.java
└── task
    ├── Task.java                    CreateTaskRequest.java   UpdateStatusRequest.java
    ├── TaskService.java             TaskController.java      ApiExceptionHandler.java
    ├── TaskProperties.java          TaskNotFoundException.java
    └── TooManyOpenTasksException.java
```

## The domain

```java
package com.example.tasks.task;

import java.time.Instant;

public record Task(long id, String title, String description, Status status, Instant createdAt) {
    public enum Status { OPEN, IN_PROGRESS, DONE }

    Task withStatus(Status next) {
        return new Task(id, title, description, next, createdAt);
    }
}
```

An immutable record: changing status produces a new value. That makes the concurrent
in-memory store below safe without locking individual tasks.

## Configuration

```java
@Validated
@ConfigurationProperties(prefix = "tasks")
public record TaskProperties(@Min(1) @Max(10_000) int maxOpenTasks) { }
```

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: tasks
tasks:
  max-open-tasks: 100
```

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class TasksApplication {
    public static void main(String[] args) {
        SpringApplication.run(TasksApplication.class, args);
    }

    @Bean
    Clock clock() {
        return Clock.systemUTC();
    }
}
```

The `Clock` bean exists so tests can control time.

## The service

```java
@Service
public class TaskService {
    private static final Logger log = LoggerFactory.getLogger(TaskService.class);

    private final Map<Long, Task> tasks = new ConcurrentHashMap<>();
    private final AtomicLong ids = new AtomicLong();
    private final TaskProperties properties;
    private final Clock clock;

    public TaskService(TaskProperties properties, Clock clock) {
        this.properties = properties;
        this.clock = clock;
    }

    public List<Task> findAll(Task.Status status) {
        return tasks.values().stream()
                .filter(t -> status == null || t.status() == status)
                .sorted(Comparator.comparingLong(Task::id))
                .toList();
    }

    public Task get(long id) {
        Task task = tasks.get(id);
        if (task == null) {
            throw new TaskNotFoundException(id);
        }
        return task;
    }

    public Task create(CreateTaskRequest request) {
        long open = tasks.values().stream().filter(t -> t.status() != Task.Status.DONE).count();
        if (open >= properties.maxOpenTasks()) {
            throw new TooManyOpenTasksException(properties.maxOpenTasks());
        }
        long id = ids.incrementAndGet();
        Task task = new Task(id, request.title().strip(), request.description(),
                Task.Status.OPEN, Instant.now(clock));
        tasks.put(id, task);
        log.info("Created task id={} title={}", id, task.title());
        return task;
    }

    public Task updateStatus(long id, Task.Status status) {
        Task updated = get(id).withStatus(status);
        tasks.put(id, updated);
        return updated;
    }

    public void delete(long id) {
        if (tasks.remove(id) == null) {
            throw new TaskNotFoundException(id);
        }
    }
}
```

This is a singleton shared by every request thread, so its mutable state is in a
`ConcurrentHashMap` and an `AtomicLong`. The limit check and insert are not atomic
together — two concurrent creates at the limit could both pass. For an in-memory demo
that is acceptable and documented here; in Level 2 the database enforces invariants
inside a transaction.

## The web layer

The request DTOs, controller, and exception handler are exactly the ones developed in
lessons 06–08:

```java
public record CreateTaskRequest(@NotBlank @Size(max = 120) String title,
                                @Size(max = 2000) String description) { }

public record UpdateStatusRequest(@NotNull Task.Status status) { }
```

```java
@RestController
@RequestMapping("/api/tasks")
public class TaskController {
    private final TaskService service;

    public TaskController(TaskService service) { this.service = service; }

    @GetMapping
    public List<Task> list(@RequestParam(required = false) Task.Status status) {
        return service.findAll(status);
    }

    @GetMapping("/{id}")
    public Task get(@PathVariable long id) { return service.get(id); }

    @PostMapping
    public ResponseEntity<Task> create(@Valid @RequestBody CreateTaskRequest request) {
        Task task = service.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
                .path("/{id}").buildAndExpand(task.id()).toUri();
        return ResponseEntity.created(location).body(task);
    }

    @PatchMapping("/{id}/status")
    public Task updateStatus(@PathVariable long id, @Valid @RequestBody UpdateStatusRequest request) {
        return service.updateStatus(id, request.status());
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable long id) { service.delete(id); }
}
```

`ApiExceptionHandler` maps `TaskNotFoundException` → 404, `TooManyOpenTasksException` →
409, and adds an `errors` map to validation failures (see lesson 08 for the full class).

This project returns the `Task` record directly because it *is* the API model; there is no
separate persistence model yet. Once JPA entities arrive in Level 2 you will introduce
response DTOs.

## Tests

```java
@WebMvcTest(TaskController.class)
class TaskControllerTest {

    @Autowired MockMvc mvc;
    @MockitoBean TaskService service;

    @Test
    void missingTaskIsA404ProblemDetail() throws Exception {
        given(service.get(42)).willThrow(new TaskNotFoundException(42));

        mvc.perform(get("/api/tasks/42"))
                .andExpect(status().isNotFound())
                .andExpect(jsonPath("$.title").value("Task not found"))
                .andExpect(jsonPath("$.detail").value("Task 42 does not exist"));
    }

    @Test
    void blankTitleIsRejected() throws Exception {
        mvc.perform(post("/api/tasks").contentType(MediaType.APPLICATION_JSON)
                        .content("{\"title\":\"  \"}"))
                .andExpect(status().isBadRequest())
                .andExpect(jsonPath("$.errors.title").exists());
    }
}
```

In Boot 4, `@WebMvcTest` is imported from `org.springframework.boot.webmvc.test.autoconfigure`
(in Boot 3 it was `org.springframework.boot.test.autoconfigure.web.servlet`), and
`@MockitoBean` comes from `org.springframework.test.context.bean.override.mockito` — it
replaced Boot's older `@MockBean`, which was deprecated in 3.4 and removed in 4.0.

`./mvnw test` ran these two tests plus the generated context-load test: `Tests run: 3,
Failures: 0, Errors: 0, Skipped: 0`.

## Running it

```bash
./mvnw -q package && java -jar target/tasks-0.0.1-SNAPSHOT.jar
```

```bash
$ curl -i -X POST localhost:8080/api/tasks -H 'Content-Type: application/json' \
       -d '{"title":"Write the README","description":"cover setup"}'
HTTP/1.1 201
Location: http://localhost:8080/api/tasks/1
Content-Type: application/json

{"id":1,"title":"Write the README","description":"cover setup","status":"OPEN","createdAt":"2026-09-26T15:35:18.737961Z"}

$ curl localhost:8080/api/tasks/99
{"detail":"Task 99 does not exist","instance":"/api/tasks/99","status":404,"title":"Task not found"}

$ curl -X PATCH localhost:8080/api/tasks/1/status -H 'Content-Type: application/json' -d '{"status":"DONE"}'
{"id":1,"title":"Write the README","description":"cover setup","status":"DONE","createdAt":"2026-09-26T15:35:18.737961Z"}
```

The app started in about a second (`Started TasksApplication in 1.014 seconds` on the
machine used for this course; yours will differ).

## How It Actually Works

Trace one `POST /api/tasks` through everything you have learned:

1. Tomcat accepts the connection and assigns a worker thread (`nio-8080-exec-N`).
2. The filter chain runs (character encoding, and your `RequestIdFilter` if you added it).
3. `DispatcherServlet` asks `RequestMappingHandlerMapping`, which matches
   `POST /api/tasks` to `TaskController.create`.
4. The `@RequestBody` resolver reads JSON with Jackson into `CreateTaskRequest`, then
   `@Valid` runs Hibernate Validator. A blank title throws
   `MethodArgumentNotValidException` here — the controller is never called.
5. `create` calls the singleton `TaskService` — the same instance the container injected
   at startup, now shared with every other request thread.
6. The service checks the limit from `TaskProperties`, which was bound and validated once
   at startup.
7. The returned `ResponseEntity` becomes a 201 with a `Location` header; Jackson writes the
   `Task` record, with `Instant` as an ISO string.
8. Had any step thrown, `ExceptionHandlerExceptionResolver` would have routed the
   exception to `ApiExceptionHandler`, producing a problem-detail body.

## Common mistakes to check for in your version

- Controllers doing filtering or limit checks that belong in the service.
- Missing `@Valid`, so blank titles are accepted.
- Mutable `ArrayList`/`HashMap` state in the singleton service.
- Tests that use `@SpringBootTest` for everything; the web slice here starts much faster
  and tests exactly the HTTP contract.

## Exercise (stretch goals)

1. Add `priority` (1–5) and `dueDate` (`@FutureOrPresent`), and a
   `GET /api/tasks/overdue` endpoint that uses the injected `Clock`. Unit-test the
   overdue logic with `Clock.fixed(...)`.
2. Add a `TaskServiceTest` (no Spring) covering the limit, including the boundary: with
   `maxOpenTasks = 2`, two open tasks succeed, the third fails, and completing one frees a
   slot.
3. Add the `RequestIdFilter` from lesson 09 and assert in a `@WebMvcTest` that the
   `X-Request-Id` header is echoed back.
4. Make the limit check and insert atomic (for example with `synchronized` or
   `ConcurrentHashMap.compute`-style logic), and write a concurrent test that proves the
   limit can no longer be exceeded.
