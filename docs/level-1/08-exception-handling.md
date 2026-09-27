---
description: "Centralize API error handling with @RestControllerAdvice, return RFC 9457 ProblemDetail bodies, and map domain exceptions to the right HTTP status codes."
---

# Exception Handling with @ControllerAdvice & ProblemDetail

An API's error responses are part of its contract. Clients branch on them, dashboards
count them, and humans read them at the worst moments. The goal of this lesson is one
consistent error format, produced from one place, with no stack traces leaking out.

## The standard format: problem details

RFC 9457 (which replaced RFC 7807) defines `application/problem+json`:

```json
{
  "type": "https://example.com/problems/task-not-found",
  "title": "Task not found",
  "status": 404,
  "detail": "Task 99 does not exist",
  "instance": "/api/tasks/99"
}
```

Spring has first-class support through the `ProblemDetail` class and the
`ErrorResponse` interface. Use it instead of inventing `{"error": ..., "code": ...}` — it
is a standard your clients may already parse, and you can add your own members
(`errors`, `traceId`) as extensions.

## Domain exceptions

Services should throw exceptions that describe *what went wrong in the domain*, not HTTP:

```java
public class TaskNotFoundException extends RuntimeException {
    public TaskNotFoundException(long id) {
        super("Task " + id + " does not exist");
    }
}

public class TooManyOpenTasksException extends RuntimeException {
    public TooManyOpenTasksException(int limit) {
        super("Limit of " + limit + " open tasks reached");
    }
}
```

The service does not know it is behind HTTP; the same service could be called from a
message listener. Translating to status codes is the web layer's job.

## One handler for the whole API

```java
@RestControllerAdvice
public class ApiExceptionHandler extends ResponseEntityExceptionHandler {

    @ExceptionHandler(TaskNotFoundException.class)
    ProblemDetail notFound(TaskNotFoundException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        pd.setTitle("Task not found");
        return pd;
    }

    @ExceptionHandler(TooManyOpenTasksException.class)
    ProblemDetail tooMany(TooManyOpenTasksException ex) {
        ProblemDetail pd = ProblemDetail.forStatusAndDetail(HttpStatus.CONFLICT, ex.getMessage());
        pd.setTitle("Open task limit reached");
        return pd;
    }

    @Override
    protected ResponseEntity<Object> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex, HttpHeaders headers,
            HttpStatusCode status, WebRequest request) {
        ProblemDetail pd = ex.getBody();
        Map<String, String> errors = ex.getBindingResult().getFieldErrors().stream()
                .collect(Collectors.toMap(FieldError::getField,
                        fe -> String.valueOf(fe.getDefaultMessage()), (a, b) -> a));
        pd.setProperty("errors", errors);
        return handleExceptionInternal(ex, pd, headers, status, request);
    }
}
```

- `@RestControllerAdvice` applies to every controller (scope it with
  `basePackages`/`assignableTypes` if needed).
- Extending `ResponseEntityExceptionHandler` gives you problem-detail responses for all of
  Spring MVC's own exceptions — unsupported media type, missing parameter, unreadable body,
  method not allowed — and hooks like `handleMethodArgumentNotValid` to customize them.
- Returning a `ProblemDetail` from an `@ExceptionHandler` sets the status from its
  `status` field and the content type to `application/problem+json`. The `instance` is
  filled in from the request path.

Alternatively, set `spring.mvc.problemdetails.enabled=true` and Boot registers a
`ResponseEntityExceptionHandler` for you — useful when you have no custom handlers. Once
you write your own subclass (as above), Boot's backs off.

## Worked example: the real responses

Against the Level 1 project, the handler above produced:

```bash
$ curl localhost:8080/api/tasks/99
{"detail":"Task 99 does not exist","instance":"/api/tasks/99","status":404,"title":"Task not found"}

$ curl -X POST localhost:8080/api/tasks -H 'Content-Type: application/json' -d '{"title":""}'
{"detail":"Invalid request content.","instance":"/api/tasks","status":400,"title":"Bad Request","errors":{"title":"must not be blank"}}
```

And a `@WebMvcTest` pins the behavior so it cannot regress (this test passes on Boot 4.1):

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
}
```

## Unexpected exceptions

What about a `NullPointerException` from a bug? Do not return its message — it may contain
internal details. Add a catch-all that logs with full context and returns a generic body:

```java
private static final Logger log = LoggerFactory.getLogger(ApiExceptionHandler.class);

@ExceptionHandler(Exception.class)
ProblemDetail unexpected(Exception ex, HttpServletRequest request) {
    log.error("Unhandled error on {} {}", request.getMethod(), request.getRequestURI(), ex);
    return ProblemDetail.forStatusAndDetail(HttpStatus.INTERNAL_SERVER_ERROR,
            "An unexpected error occurred");
}
```

Boot's defaults already avoid leaking stack traces in the body (`server.error.include-stacktrace`
defaults to `never`); keep it that way in production.

## Choosing status codes

| Situation | Status |
|-----------|--------|
| Malformed syntax, failed validation | 400 |
| Not authenticated | 401 (Level 2) |
| Authenticated but not allowed | 403 |
| Resource does not exist | 404 |
| Conflicts with current state (duplicate, version mismatch, limit reached) | 409 |
| Well-formed but semantically invalid for business rules | 422 (some teams prefer 400 for all input errors — pick one and be consistent) |
| Bug or unexpected failure | 500 |
| Dependency down / overloaded | 503 |

## How It Actually Works

When an exception escapes a handler method, the `DispatcherServlet` asks its
`HandlerExceptionResolver`s, in order:

1. **`ExceptionHandlerExceptionResolver`** — looks for an `@ExceptionHandler` method first
   in the controller class itself, then in `@ControllerAdvice` beans (ordered by `@Order`).
   It picks the handler whose declared exception type is the *closest* match in the
   exception's class hierarchy, so a handler for `TaskNotFoundException` beats one for
   `RuntimeException`. The handler's return value then goes through the same return-value
   handling and message conversion as a normal controller method.
2. **`ResponseStatusExceptionResolver`** — handles exceptions annotated with
   `@ResponseStatus` or of type `ResponseStatusException`.
3. **`DefaultHandlerExceptionResolver`** — maps Spring's own MVC exceptions to status
   codes.

If none handles it, the exception propagates out of the servlet; the container forwards
to `/error`, where Boot's `BasicErrorController` renders its default error body. That
is the "Whitelabel" path you see when nothing else caught the error — including errors
thrown by filters, which run *before* the `DispatcherServlet` and therefore never reach
`@ControllerAdvice` (Spring Security errors are the common case, handled in Level 2).

## Common mistakes

- **try/catch in every controller method** to build error responses. One advice class is
  easier to keep consistent.
- **Throwing `ResponseStatusException` from services.** It ties domain code to HTTP.
  Fine in controllers; avoid it deeper.
- **Leaking `ex.getMessage()` from unexpected exceptions.** SQL fragments, file paths,
  and class names end up in client responses.
- **Logging expected errors at ERROR.** A 404 is not an incident. Log client errors at
  DEBUG/INFO, and alert only on 5xx.
- **Overriding `handleMethodArgumentNotValid` with a different body shape** than every
  other error. Keep the problem format and add fields as extensions.

## Exercise

1. Implement `ApiExceptionHandler` as above, plus the catch-all handler.
2. Give each problem a stable `type` URI (for example
   `URI.create("https://api.example.com/problems/task-not-found")`), and document the types
   in a short README section.
3. Write `@WebMvcTest` tests for: 404 on a missing task, 400 with an `errors.title` field,
   409 when the limit is reached, and 415 when posting `text/plain`.
4. Throw an exception from a servlet `Filter` and observe that your advice does not see
   it. Note which component produced the response instead.
