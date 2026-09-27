---
description: "Generate an OpenAPI 3 spec and Swagger UI from a Spring Boot API with springdoc-openapi, document security and errors, and keep the spec honest in CI."
---

# API Documentation with springdoc-openapi

An API without a machine-readable description forces every client team to reverse-engineer
it from examples. The OpenAPI Specification is the industry-standard format for that
description, and **springdoc-openapi** generates it from your Spring MVC code at runtime,
so the docs cannot drift far from the implementation.

springdoc is a community project, not part of Spring itself. Its major versions track
Spring Boot's: springdoc 2.x targets Boot 3, and 3.x targets Boot 4. Check its README for
the version matching your Boot release.

## Setup

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>${springdoc.version}</version>   <!-- not managed by the Boot BOM -->
</dependency>
```

Start the app and you get:

- `/v3/api-docs` — the OpenAPI document as JSON (`/v3/api-docs.yaml` for YAML)
- `/swagger-ui.html` — an interactive UI that reads it

If your API is secured, remember to `permitAll()` those paths — ideally only outside
production, or behind authentication.

## What it infers for free

From a controller like Level 1's `TaskController`, springdoc derives paths, HTTP
methods, path and query parameters (and whether they are required), request and response
schemas from your records, enum values for `Task.Status`, and — because it reads Bean
Validation annotations — constraints such as `minLength`, `maxLength`, and `required` for
`@NotBlank @Size(max = 120) String title`.

## Adding what code cannot express

```java
@Tag(name = "Books", description = "Catalog of books and their authors")
@RestController
@RequestMapping("/api/books")
class BookController {

    @Operation(summary = "List books with their author names")
    @GetMapping
    List<BookView> list() { ... }

    @Operation(summary = "Rename a book",
               description = "Requires the books:write scope.")
    @ApiResponse(responseCode = "200", description = "Renamed")
    @ApiResponse(responseCode = "404", description = "No such book",
                 content = @Content(mediaType = "application/problem+json",
                                    schema = @Schema(implementation = ProblemDetail.class)))
    @PostMapping("/{id}/retitle")
    BookView retitle(@PathVariable long id, @RequestBody @Valid RetitleRequest request) { ... }
}

record RetitleRequest(
        @Schema(example = "The Left Hand of Darkness") @NotBlank @Size(max = 300) String title) { }
```

Global metadata and security schemes go in a bean:

```java
@Bean
OpenAPI libraryOpenApi() {
    return new OpenAPI()
            .info(new Info().title("Library API").version("v1")
                    .description("Authors, books, and loans."))
            .components(new Components().addSecuritySchemes("bearer",
                    new SecurityScheme().type(SecurityScheme.Type.HTTP)
                            .scheme("bearer").bearerFormat("JWT")))
            .addSecurityItem(new SecurityRequirement().addList("bearer"));
}
```

Swagger UI then shows an **Authorize** button where you paste a token and try secured
endpoints.

## Worked example: a docs contract check in CI

Generated docs are only useful if someone notices when they change. A simple pattern:

1. Add a test that starts the app and fetches the spec:

    ```java
    @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
    class OpenApiSnapshotTest {
        @LocalServerPort int port;

        @Test
        void specMatchesCommittedSnapshot() throws Exception {
            String spec = HttpClient.newHttpClient().send(
                    HttpRequest.newBuilder(URI.create("http://localhost:" + port + "/v3/api-docs.yaml")).build(),
                    HttpResponse.BodyHandlers.ofString()).body();
            Path snapshot = Path.of("docs/openapi.yaml");
            if (Boolean.getBoolean("updateSnapshot")) Files.writeString(snapshot, spec);
            assertThat(spec).isEqualTo(Files.readString(snapshot));
        }
    }
    ```

2. Commit `docs/openapi.yaml`. Any change to the API now fails the test until the snapshot
   is updated (`-DupdateSnapshot=true`), so API changes show up in code review as a diff
   of the spec.
3. Optionally run a breaking-change detector (such as `oasdiff`) in CI against the
   previous release's spec.

Exact byte-for-byte comparison can be brittle if generation order changes between
springdoc versions; if it bothers you, compare parsed YAML instead.

## Code-first or spec-first?

This lesson is **code-first**: the implementation is the source of truth and the spec is
generated. The alternative, **spec-first**, writes the OpenAPI document by hand and
generates server interfaces and client SDKs from it (for example with OpenAPI Generator's
`spring` generator). Spec-first suits public APIs with many consumers and a design review
process; code-first suits internal services that move quickly. Both are legitimate; mixing
them without a clear owner is not.

## How It Actually Works

At first request to `/v3/api-docs`, springdoc asks Spring MVC's
`RequestMappingHandlerMapping` for every registered handler method — the same registry the
`DispatcherServlet` uses to route requests. For each one it reads the mapping
(path, methods, consumes/produces), then inspects the method's parameters and return type
by reflection. Types are converted to JSON Schema using the **swagger-core** model
resolver, which walks record components and bean properties, reads Jackson annotations
(for names and inclusion rules), and reads Bean Validation annotations for constraints.
`@Operation`, `@ApiResponse`, and `@Schema` annotations are merged on top, and
`OpenAPI`/`OpenApiCustomizer` beans are applied last. The result is cached.

Because it reads the real handler registry, a path you deleted disappears from the spec
and a new field on a response record appears automatically. What it cannot know are
things only your code's behavior decides — which exceptions map to which statuses, or
which scope an endpoint needs — hence the annotations.

## Common mistakes

- **Exposing Swagger UI publicly in production** for an internal API.
- **Returning `ResponseEntity<?>` or `Object`**, which gives the generator nothing to
  document. Use concrete types.
- **Documenting only the happy path.** Clients need your error format and statuses too.
- **Letting annotations drown the controller.** If most of a controller is `@Operation`
  text, consider spec-first, or move descriptions into the `OpenAPI` bean.

## Exercise

1. Add springdoc to the library API and open Swagger UI. Check that `@NotBlank` and
   `@Size` show up as constraints on the request schema.
2. Add the bearer security scheme and try a secured endpoint from the UI with a token from
   your test utility.
3. Document the 404 and 409 responses with `ProblemDetail` schemas.
4. Add the snapshot test, commit the spec, rename a field in a response record, and see
   the test fail with a readable diff.
