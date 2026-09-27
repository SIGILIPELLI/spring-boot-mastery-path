---
description: "What JPA, Hibernate, and Spring Data JPA each do, how Boot configures a data source, and how the persistence context and dirty checking really behave."
---

# Spring Data JPA & Hibernate Basics

Three names get used interchangeably and should not be:

| Layer | What it is |
|-------|-----------|
| **JPA** (Jakarta Persistence) | A *specification*: annotations (`@Entity`, `@Id`, …) and interfaces (`EntityManager`, JPQL). No implementation. |
| **Hibernate ORM** | The implementation Boot uses. It turns entity operations into SQL, manages the persistence context, caches, and lazy loading. |
| **Spring Data JPA** | A layer on top that generates repository implementations from interfaces and integrates JPA with Spring transactions. |

When something goes wrong — a `LazyInitializationException`, an unexpected `UPDATE`, too
many queries — the cause is almost always in the Hibernate layer, even though you only
touched Spring Data.

## Setting up

Add `spring-boot-starter-data-jpa` and a JDBC driver. For development, H2 runs in memory;
for anything real, use the database you deploy (PostgreSQL in this course):

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
</dependency>
```

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/library
    username: library
    password: ${DB_PASSWORD}
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
```

- With only H2 on the classpath and no URL, Boot creates an embedded in-memory database
  automatically.
- Boot's default connection pool is **HikariCP**. Tune it with `spring.datasource.hikari.*`
  (Level 4 covers pool sizing).
- `ddl-auto: validate` makes Hibernate check that entities match the schema at startup
  and fail if not. The schema itself comes from Flyway migrations (lesson 05). Do not use
  `update` or `create` outside throwaway experiments.
- `open-in-view: false` is explained below; set it from day one.

A local PostgreSQL for development:

```bash
docker run -d --name library-db -p 5432:5432 \
  -e POSTGRES_DB=library -e POSTGRES_USER=library -e POSTGRES_PASSWORD=devpass postgres:17
```

## An entity and a repository

```java
@Entity
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false, unique = true)
    private String isbn;

    private Integer published;

    protected Book() { }                 // JPA needs a no-arg constructor

    public Book(String title, String isbn, Integer published) {
        this.title = title;
        this.isbn = isbn;
        this.published = published;
    }

    public Long getId() { return id; }
    public String getTitle() { return title; }
    public void setTitle(String title) { this.title = title; }
    public String getIsbn() { return isbn; }
}

public interface BookRepository extends JpaRepository<Book, Long> {
    Optional<Book> findByIsbn(String isbn);
}
```

`JpaRepository` gives you `save`, `findById`, `findAll` (with paging and sorting),
`deleteById`, `count`, and more. You never write the implementation class.

Entities are **not** records: JPA needs to instantiate them empty and set fields, and it
subclasses them for lazy-loading proxies. Records remain perfect for DTOs and
projections.

## Worked example: dirty checking

Here is a service method from this level's project. Notice what is missing:

```java
@Service
public class CatalogService {
    private final BookRepository books;

    public CatalogService(BookRepository books) { this.books = books; }

    @Transactional
    public void retitle(String isbn, String newTitle) {
        Book book = books.findByIsbn(isbn).orElseThrow();
        book.setTitle(newTitle);   // no save() call
    }
}
```

There is no `books.save(book)`, yet the title changes in the database. A test run against
Boot 4.1 / Hibernate 7 confirmed it: after `retitle("isbn-11", "Renamed")`, a fresh
`findByIsbn("isbn-11")` returned `"Renamed"`. How that works is the most important idea in
this lesson.

## How It Actually Works

**The persistence context.** Every `EntityManager` owns a *persistence context*: a map
from (entity type, id) to the managed entity instance, plus a snapshot of each entity's
state as loaded. In a Spring app, one persistence context is bound to the current
transaction; all repositories used inside that transaction share it.

This has three consequences:

1. **Identity.** Within one transaction, loading book 7 twice returns the *same Java
   object*, and the second load does not hit the database.
2. **Dirty checking.** At flush time (before commit, or before a query that could be
   affected), Hibernate compares each managed entity's current field values with its
   snapshot. Anything changed produces an `UPDATE`. That is why `setTitle` alone was
   enough — the entity was *managed*.
3. **Write-behind.** `save()` on a new entity does not necessarily execute `INSERT`
   immediately; statements are queued and flushed together. (With `IDENTITY` id
   generation Hibernate must insert immediately to learn the id, which is one reason
   sequence-based ids batch better — Level 4.)

**Entity states.** An object is *transient* (`new Book(...)`, unknown to Hibernate),
*managed* (loaded or persisted in the current context), *detached* (was managed, but the
context closed), or *removed*. Only managed entities are dirty-checked. A detached entity
modified after the transaction ends changes nothing in the database unless you `merge` it
back — which is what `save()` does for an entity with an id.

**What `save()` actually does.** `SimpleJpaRepository.save` calls `em.persist` if the
entity is new (id is null, or a `@Version` field is null) and `em.merge` otherwise.
`merge` copies state onto a managed instance and returns *that* instance — so always use
the return value of `save`.

**Open Session in View.** By default Boot keeps the persistence context open for the
whole web request (`spring.jpa.open-in-view=true`, with a startup warning). That lets lazy
associations load during JSON serialization in the controller — convenient, and a
reliable source of hidden queries and connections held for the entire request. Turning it
off forces you to load what you need inside the transactional service method, which is
where that decision belongs.

## Common mistakes

- **`ddl-auto: update` in real environments.** It never drops columns, cannot rename, and
  guesses. Use migrations.
- **Calling `save()` on already-managed entities** "to be safe." Harmless but it hides the
  model; worse, code that relies on `save` often breaks when the entity is detached.
- **Modifying an entity outside a transaction** and expecting it to persist.
- **Returning entities from controllers**, which triggers lazy loading during
  serialization (or `LazyInitializationException` with OSIV off). Map to DTOs in the
  service.
- **Using `@Data` (Lombok) on entities**, generating `equals`/`hashCode`/`toString` that
  touch lazy associations. Lesson 02 covers entity equality.

## Exercise

1. Create a project with Web, Data JPA, H2, and Validation. Add the `Book` entity and
   repository, set `ddl-auto: create-drop` *for this exercise only*, and
   `logging.level.org.hibernate.SQL: DEBUG`.
2. Write a `@SpringBootTest` that saves a book, then in a `@Transactional` service method
   loads it and changes the title without calling `save`. Find the `update book …` line in
   the log.
3. In one transaction, call `findById(id)` twice and assert both results are the same
   instance (`isSameAs`). Count the `select` statements in the log.
4. Load a book in one transaction, change its title *after* the transaction ends, and
   show that the database is unchanged. Then fix it with `save` and explain why that
   worked.
