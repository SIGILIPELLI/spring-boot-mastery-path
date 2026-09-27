---
description: "Build REST endpoints with @RestController: mapping annotations, resource-oriented URLs, correct status codes, Location headers, and ResponseEntity."
---

# Building REST Controllers

A controller is the thin layer that turns HTTP into method calls and return values back
into HTTP. The best Spring controllers are boring: they parse the request, call one
service method, and choose a status code. Business rules live elsewhere.

## A first controller

```java
@RestController
@RequestMapping("/api/tasks")
class TaskController {
    private final TaskService service;

    TaskController(TaskService service) {
        this.service = service;
    }

    @GetMapping
    List<Task> list() {
        return service.findAll(null);
    }

    @GetMapping("/{id}")
    Task get(@PathVariable long id) {
        return service.get(id);
    }
}
```

- `@RestController` = `@Controller` + `@ResponseBody`: return values are written to the
  response body (as JSON, by default) rather than treated as view names.
- `@RequestMapping` on the class sets a URL prefix; method-level `@GetMapping`,
  `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping` add the rest.
- The controller is package-private. Spring does not require `public`.

## Mapping HTTP semantics properly

REST is mostly about using HTTP's existing vocabulary correctly:

| Operation | Method & path | Success status | Body |
|-----------|--------------|----------------|------|
| List | `GET /api/tasks` | 200 | array |
| Read one | `GET /api/tasks/{id}` | 200 (404 if absent) | object |
| Create | `POST /api/tasks` | **201** + `Location` header | created object |
| Replace | `PUT /api/tasks/{id}` | 200 or 204 | object or empty |
| Partial update | `PATCH /api/tasks/{id}/status` | 200 | object |
| Delete | `DELETE /api/tasks/{id}` | **204** | empty |

Spring returns 200 by default. For anything else, either annotate the method or return a
`ResponseEntity`:

```java
@PostMapping
ResponseEntity<Task> create(@Valid @RequestBody CreateTaskRequest request) {
    Task task = service.create(request);
    URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(task.id())
            .toUri();
    return ResponseEntity.created(location).body(task);   // 201 + Location
}

@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT)                    // 204, no body
void delete(@PathVariable long id) {
    service.delete(id);
}
```

`ServletUriComponentsBuilder.fromCurrentRequest()` builds the URL from the incoming
request (scheme, host, port, path), so the `Location` header is correct whether the app is
behind `localhost:8080` or a load balancer — provided forwarded headers are configured,
which Level 4 covers.

`ResponseEntity` is the general tool when status, headers, or body vary at runtime:
`ResponseEntity.ok(body)`, `.notFound().build()`, `.status(HttpStatus.ACCEPTED).build()`,
`.ok().eTag("\"v3\"").body(body)`.

## Worked example: the endpoints in action

With the controller from this level's project running, real requests and the responses
the app returned:

```bash
$ curl -i -X POST localhost:8080/api/tasks \
    -H 'Content-Type: application/json' \
    -d '{"title":"Write the README","description":"cover setup"}'
HTTP/1.1 201
Location: http://localhost:8080/api/tasks/1
Content-Type: application/json

{"id":1,"title":"Write the README","description":"cover setup","status":"OPEN","createdAt":"2026-09-26T15:35:18.737961Z"}

$ curl -X PATCH localhost:8080/api/tasks/1/status \
    -H 'Content-Type: application/json' -d '{"status":"DONE"}'
{"id":1,"title":"Write the README","description":"cover setup","status":"DONE","createdAt":"2026-09-26T15:35:18.737961Z"}

$ curl 'localhost:8080/api/tasks?status=DONE'
[{"id":1,"title":"Write the README","description":"cover setup","status":"DONE","createdAt":"2026-09-26T15:35:18.737961Z"}]
```

(The run used port 8089; the port is the only thing changed above.) Notice
`createdAt` is an ISO-8601 string, not a number — Boot configures Jackson to write
`java.time` types that way by default.

## Designing URLs

- Use nouns for resources (`/tasks`, `/tasks/{id}/comments`), HTTP methods for verbs.
- State changes that are not simple field updates can be sub-resources:
  `PATCH /tasks/{id}/status` or `POST /orders/{id}/cancellation`. That is clearer than
  `POST /cancelOrder?id=5`.
- Keep a version strategy in mind from the start (`/api/v1/...`, or a header). Spring
  Framework 7 added first-class API versioning support (`@GetMapping(version = "1.1")`
  with a configurable resolver); a path prefix remains the most visible and
  cache-friendly option.
- Return collections wrapped in an object if you expect to add paging metadata later —
  changing `[...]` to `{"items": [...]}` is a breaking change. Level 2 uses Spring Data's
  paging for this.

## How It Actually Works

Every request passes through one servlet, the **`DispatcherServlet`**, which Boot
registers at `/`. For each request it:

1. Asks its `HandlerMapping`s who can handle it. `RequestMappingHandlerMapping` built a
   registry at startup by scanning every `@Controller` bean for `@RequestMapping`-style
   annotations; it matches on path pattern, HTTP method, and optionally `consumes`,
   `produces`, headers, and params. The most specific match wins (`/tasks/{id}` beats
   `/tasks/**`).
2. Finds a `HandlerAdapter` for the chosen handler method and runs any
   `HandlerInterceptor`s' `preHandle`.
3. Resolves each method argument with a chain of `HandlerMethodArgumentResolver`s — one
   handles `@PathVariable`, another `@RequestBody`, another `Principal`, and so on (next
   lesson).
4. Invokes your method.
5. Hands the return value to a `HandlerMethodReturnValueHandler`. For `@ResponseBody`, it
   uses **content negotiation**: it looks at the request's `Accept` header and the
   registered `HttpMessageConverter`s, picks one that can write your type in an
   acceptable media type (Jackson's JSON converter for a record), and writes the body.
6. If an exception escaped, passes it to the `HandlerExceptionResolver` chain
   (lesson 08).

When you see "No endpoint GET /api/task" or a 405 Method Not Allowed, step 1 is what
failed. Setting `logging.level.org.springframework.web=DEBUG` shows the mapping decision
for every request, and at startup Actuator's `/actuator/mappings` lists every route.

## Common mistakes

- **Business logic in controllers.** If a controller method has an `if` about domain
  rules, move it into the service where it can be unit-tested and reused.
- **Returning JPA entities directly** (Level 2). You leak your schema, risk lazy-loading
  exceptions during serialization, and couple API changes to database changes. Return DTOs
  (records).
- **200 for everything**, with `{"success": false}` in the body. Clients, caches, and
  monitoring all rely on status codes.
- **Using `@Controller` instead of `@RestController`** and then wondering why Spring looks
  for a template named after your return value.

## Exercise

1. Implement `TaskController` with list, get, create (201 + `Location`), status update
   (PATCH), and delete (204).
2. Add `GET /api/tasks/count` returning `{"open": n, "done": m}`. Watch out: does
   `/api/tasks/{id}` swallow `/api/tasks/count`? Try it, then explain what Spring's
   matching did. (Hint: a literal segment is more specific than a variable.)
3. Call every endpoint with `curl -i` and check each status code and header against the
   table above.
4. Enable `logging.level.org.springframework.web=DEBUG`, make one request, and identify in
   the log the handler method chosen and the message converter used.
