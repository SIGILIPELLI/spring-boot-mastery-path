---
description: "Bind path variables, query parameters, headers and JSON bodies in Spring MVC, shape responses with records, and validate input with Bean Validation."
---

# Request Binding, Responses & Validation

Most bugs in an API's public surface are input bugs: a missing field treated as zero, a
negative page size, a 10 MB "title". Spring MVC gives you declarative tools for both
extracting input and rejecting bad input before it reaches your service.

## Where input comes from

```java
@GetMapping("/api/projects/{projectId}/tasks")
List<TaskView> search(
        @PathVariable long projectId,                                  // /projects/7/tasks
        @RequestParam(required = false) Task.Status status,            // ?status=OPEN
        @RequestParam(defaultValue = "0") int page,                    // ?page=2
        @RequestHeader(name = "X-Request-Id", required = false) String requestId) {
    ...
}

@PostMapping("/api/tasks")
ResponseEntity<TaskView> create(@Valid @RequestBody CreateTaskRequest body) { ... }
```

- Type conversion is automatic: `"OPEN"` becomes the enum, `"2"` an `int`. A value that
  cannot convert (`?page=abc`) produces a 400 before your method runs.
- `@RequestParam` is required by default. Missing required params are a 400.
- `@RequestBody` reads the whole body through an `HttpMessageConverter` — Jackson for
  `application/json`.
- Parameter names come from the compiled class (the Boot parent POM compiles with
  `-parameters`), so `@PathVariable long projectId` needs no explicit name.

Several query parameters can bind into one object without any annotation:

```java
record TaskFilter(Task.Status status, String q, Integer page) { }

@GetMapping("/api/tasks")
List<TaskView> list(TaskFilter filter) { ... }   // ?status=OPEN&q=readme&page=1
```

## Request and response DTOs

Use records for anything crossing the HTTP boundary:

```java
public record CreateTaskRequest(
        @NotBlank @Size(max = 120) String title,
        @Size(max = 2000) String description) { }

public record UpdateStatusRequest(@NotNull Task.Status status) { }

public record TaskView(long id, String title, Task.Status status, Instant createdAt) {
    static TaskView from(Task t) {
        return new TaskView(t.id(), t.title(), t.status(), t.createdAt());
    }
}
```

Separate request and response types mean a client cannot set `id` or `createdAt` by
sending them, and you can change internal models without breaking the contract. Records
deserialize through their canonical constructor, so they stay immutable.

## Validation

Add `spring-boot-starter-validation` (it brings Hibernate Validator, the reference
implementation of Jakarta Bean Validation). Then:

- Put constraints on fields: `@NotNull`, `@NotBlank`, `@Size`, `@Min`/`@Max`,
  `@Positive`, `@Email`, `@Pattern`, `@Past`/`@Future`, and more.
- Put `@Valid` on the `@RequestBody` parameter. Without it, the constraints are ignored.
- For `@RequestParam`/`@PathVariable` constraints (`@Min(1) int page`), Spring Framework 6.1+
  validates them automatically when the method has constraint annotations on parameters;
  failures produce a `HandlerMethodValidationException` (a 400).

Nested objects are only validated if the field is itself marked `@Valid`:

```java
record CreateProjectRequest(@NotBlank String name, @Valid @NotNull Owner owner) { }
record Owner(@Email @NotBlank String email) { }
```

### Worked example: what a client sees

With the project's controller and the exception handler from the next lesson, posting a
blank title returned:

```bash
$ curl -X POST localhost:8080/api/tasks -H 'Content-Type: application/json' -d '{"title":""}'
{"detail":"Invalid request content.","instance":"/api/tasks","status":400,"title":"Bad Request","errors":{"title":"must not be blank"}}
```

and malformed JSON returned:

```bash
$ curl -X POST localhost:8080/api/tasks -H 'Content-Type: application/json' -d '{bad'
{"detail":"Failed to read request","instance":"/api/tasks","status":400,"title":"Bad Request"}
```

Both responses are 400s in the standard problem format; neither reached `TaskService`.

### Custom constraints

When a rule spans fields or needs your own logic, write a constraint:

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = DateRangeValidator.class)
public @interface ValidDateRange {
    String message() default "start must be before end";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

public class DateRangeValidator implements ConstraintValidator<ValidDateRange, SprintRequest> {
    @Override
    public boolean isValid(SprintRequest value, ConstraintValidatorContext ctx) {
        if (value == null || value.start() == null || value.end() == null) return true; // let @NotNull handle nulls
        return value.start().isBefore(value.end());
    }
}

@ValidDateRange
public record SprintRequest(@NotNull LocalDate start, @NotNull LocalDate end) { }
```

Returning `true` for nulls is a convention: each constraint checks one thing, and
`@NotNull` is responsible for presence.

## Validation versus business rules

Bean Validation is for the *shape* of input: presence, length, format, ranges. Rules that
need data — "the title must be unique in this project," "you cannot close a task with open
subtasks" — belong in the service, which throws a domain exception mapped to 409 or 422.
Trying to do database lookups inside a `ConstraintValidator` couples validation to
persistence and makes it run at surprising times.

## How It Actually Works

**Argument resolution.** For each parameter of your handler method, Spring asks a list of
`HandlerMethodArgumentResolver`s "can you supply this?" `PathVariableMethodArgumentResolver`
reads URI template variables extracted during matching; `RequestParamMethodArgumentResolver`
reads the query string and form data; `RequestResponseBodyMethodProcessor` handles
`@RequestBody`. That last one picks an `HttpMessageConverter` whose supported media types
include the request's `Content-Type`. If none does — a client posts `text/plain` to a JSON
endpoint — you get a 415 Unsupported Media Type.

**Jackson and records.** The JSON converter delegates to Jackson (Jackson 3 in Boot 4,
package `tools.jackson`; Jackson 2 in Boot 3, package `com.fasterxml.jackson`). For a
record it uses the canonical constructor, matching JSON property names to component
names. Unknown JSON properties are ignored by Boot's default configuration; missing ones
become `null` (or `0`/`false` for primitives — a reason to prefer wrapper types plus
`@NotNull` when "absent" and "zero" differ).

**When validation runs.** After the body is deserialized, the resolver sees `@Valid`,
calls the Bean Validation `Validator` on the object, and collects violations into a
`BindingResult`. If the next parameter is not a `BindingResult`, it throws
`MethodArgumentNotValidException`. Validation happens *before* your method is invoked, so
your code can assume the constraints hold.

## Common mistakes

- **Forgetting `@Valid`.** The constraints compile and do nothing.
- **Forgetting the validation starter.** Without a Bean Validation provider on the
  classpath, `@Valid` has nothing to call.
- **Validating entities instead of request DTOs.** You end up with constraints that make
  sense for one endpoint but not another.
- **Primitive fields for optional input.** `int priority` cannot distinguish "not sent"
  from `0`. Use `Integer` and decide explicitly.
- **Echoing raw input in error messages** without limits — a 1 MB string in an error
  response or log line is its own problem.

## Exercise

1. Add `priority` (1–5, optional) and `dueDate` (must be today or later: `@FutureOrPresent`)
   to `CreateTaskRequest` and return them in `TaskView`.
2. Add a `GET /api/tasks` filter with `status`, `q` (title contains, case-insensitive), and
   `page`/`size` query parameters bound into one record. Constrain `size` to 1–100.
3. Write the `@ValidDateRange` constraint for a `SprintRequest` and test it with a plain
   `Validator` in a unit test:
   `Validation.buildDefaultValidatorFactory().getValidator().validate(request)`.
4. Send each invalid request with `curl` and note the status code and body for each.
