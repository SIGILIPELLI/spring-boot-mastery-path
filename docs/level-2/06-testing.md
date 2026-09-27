---
description: "Test Spring Boot apps at the right level: plain JUnit 5 unit tests, @WebMvcTest and @DataJpaTest slices, @SpringBootTest, and Testcontainers."
---

# Testing: JUnit 5, Slices & Testcontainers

A Spring Boot test suite has layers, and choosing the right layer for each test is most of
the skill. Too many full-context tests and the suite takes ten minutes; too many mocks and
it passes while production breaks. This lesson maps each kind of test to what it can
actually prove.

## The test pyramid, Spring edition

| Kind | Starts | Proves | Speed |
|------|--------|--------|-------|
| Plain unit test | Nothing | Business logic in one class | Milliseconds |
| `@WebMvcTest` | MVC slice: controllers, advice, converters, filters | HTTP contract: routes, status codes, JSON, validation, security rules | Fast |
| `@DataJpaTest` | JPA slice: entities, repositories, Flyway, a database | Mappings, queries, constraints | Fast–medium |
| `@SpringBootTest` | Entire application context | Wiring and end-to-end flows | Slowest |

Most tests should be in the first three rows.

## Unit tests: no Spring at all

If your services use constructor injection, test them with `new`:

```java
class LoanRulesTest {
    @Test
    void memberAtLimitCannotBorrow() {
        var member = new Member("Ada", 2);
        member.incrementActiveLoans();
        member.incrementActiveLoans();

        assertThatThrownBy(() -> LoanRules.checkCanBorrow(member))
                .isInstanceOf(LoanLimitReachedException.class);
    }
}
```

JUnit 5 (Jupiter) plus AssertJ are included by the Boot test starters. Mockito is there
too, for collaborators you cannot easily construct — but prefer real objects and small
fakes over mocks when it is cheap to do so.

## Web slice: @WebMvcTest

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
                .andExpect(jsonPath("$.title").value("Task not found"));
    }
}
```

The slice loads only web-layer beans. Services are not scanned, so you supply them with
`@MockitoBean`. This is the right place to test status codes, validation errors, JSON
field names, and (with security on the classpath) access rules. Level 1's project tests
use exactly this, and passed on Boot 4.1.

## Persistence slice: @DataJpaTest

```java
@DataJpaTest
class BookRepositoryTest {
    @Autowired BookRepository books;
    @Autowired AuthorRepository authors;

    @Test
    void findsByIsbn() {
        Author author = new Author("Octavia E. Butler");
        author.addBook(new Book("Kindred", "978-0807083697", 1979));
        authors.save(author);

        assertThat(books.findByIsbn("978-0807083697"))
                .get().extracting(Book::getTitle).isEqualTo("Kindred");
    }
}
```

`@DataJpaTest` configures JPA, runs Flyway, and wraps **each test in a transaction that
is rolled back** at the end, so tests do not leak data into each other. By default it
replaces your data source with an embedded database if one is on the classpath.

In Boot 4, import it from `org.springframework.boot.data.jpa.test.autoconfigure`
(Boot 3: `org.springframework.boot.test.autoconfigure.orm.jpa`), and add
`spring-boot-starter-data-jpa-test` to your test dependencies.

!!! warning "The rollback hides flushes"
    Because the test transaction never commits, Hibernate may never flush your inserts.
    A unique-constraint violation that would fail in production can pass silently in a
    test. Call `entityManager.flush()` (inject `TestEntityManager`) when the test is
    about constraints.

## Testcontainers: the real database

H2 is not PostgreSQL. Queries using PostgreSQL features, migrations with PostgreSQL
syntax, and subtle behaviors (case sensitivity, locking, JSON columns) need the real
engine. Testcontainers starts a throwaway Docker container per test run:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>testcontainers-postgresql</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class BookRepositoryPostgresTest {

    @Container
    @ServiceConnection
    static PostgreSQLContainer postgres = new PostgreSQLContainer("postgres:17-alpine");

    @Autowired BookRepository books;

    @Test
    void migrationsRunAndQueriesWork() { ... }
}
```

`@ServiceConnection` (Boot 3.1+) reads the container's host, port, and credentials and
configures the data source — no `@DynamicPropertySource` needed. `replace = NONE` stops
the slice from swapping in H2.

Artifact names and the container class package changed across Testcontainers major
versions (the 2.x line renamed modules to `testcontainers-*`); let Initializr's
"Testcontainers" option generate the right coordinates for your Boot version. The code
above needs Docker running and was not executed for this course.

## Full context: @SpringBootTest

```java
@SpringBootTest
@AutoConfigureMockMvc
class LibraryFlowTest {
    @Autowired MockMvc mvc;

    @Test
    void publicCatalogIsReadable() throws Exception {
        mvc.perform(get("/api/books")).andExpect(status().isOk());
    }
}
```

Use this for a handful of smoke tests of the fully wired app. Add
`webEnvironment = RANDOM_PORT` to start a real server and call it over HTTP with
`RestTestClient` (Boot 4) or `TestRestTemplate` (Boot 3).

## Worked example: a realistic mix for one feature

For "lend a book":

- 4 unit tests of `LoanRules` (limit reached, book already lent, happy path, returns).
- 3 `@WebMvcTest` tests (201 on success, 409 when already lent, 401 without a token).
- 2 `@DataJpaTest` tests with Testcontainers (the `existsByBookIdAndReturnedAtIsNull`
  query, and the unique partial index that prevents two active loans for one book).
- 1 `@SpringBootTest` covering the whole request with a real token.

Ten tests, most of them fast, each proving something the others cannot.

## How It Actually Works

**Slices** are built from the same auto-configuration machinery as the app. `@WebMvcTest`
is meta-annotated with `@TypeExcludeFilters(WebMvcTypeExcludeFilter.class)`, which limits
component scanning to web-related types, and `@ImportAutoConfiguration` of a fixed list of
auto-configurations (MVC, Jackson, validation, security) instead of everything.

**Context caching.** The Spring TestContext Framework caches application contexts
keyed by their configuration: the classes, active profiles, properties, and
**`@MockitoBean` definitions**. Two test classes with the same key share one context
(the default cache holds up to 32). Every unique combination of mocks or properties
creates a new context — which is why suites that sprinkle different `@MockitoBean`s over
dozens of `@SpringBootTest` classes get slow. Standardize a few base configurations.

**`@MockitoBean`** (Spring Framework 6.2+) replaces the bean definition in the test
context with a Mockito mock and resets it after each test. It superseded Boot's
`@MockBean`, removed in Boot 4.

**Test transactions** are managed by `TransactionalTestExecutionListener`: if a test
method is `@Transactional` (as every `@DataJpaTest` is), it starts a transaction before
the test and rolls it back after. This does not apply to code running in *another
thread* — including the server thread in a `RANDOM_PORT` test — so those tests must clean
up data themselves.

## Common mistakes

- **`@SpringBootTest` for everything**, making the suite slow and failures vague.
- **Mocking the repository in a repository test**, which proves nothing about the query.
- **Trusting H2 for PostgreSQL-specific SQL.**
- **Asserting on log output or `toString`** instead of behavior.
- **Shared mutable state between tests** (static lists, a database without rollback),
  causing order-dependent failures.

## Exercise

1. Write unit tests for your loan (or task) rules with no Spring.
2. Write `@WebMvcTest` tests for every error status your controller can return.
3. Write a `@DataJpaTest` for `findByIsbn` and one that proves the `isbn` unique
   constraint using `TestEntityManager.flush()`.
4. If you have Docker, convert one repository test to Testcontainers with
   `@ServiceConnection` and run your Flyway migrations against PostgreSQL.
5. Count how many application contexts your suite creates (enable
   `logging.level.org.springframework.test.context.cache=DEBUG`) and reduce the number by
   unifying mock configurations.
