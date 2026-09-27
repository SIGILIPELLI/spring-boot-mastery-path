---
description: "Query with Spring Data JPA: derived query methods, JPQL with @Query, join fetch and entity graphs, DTO projections, paging, sorting, and modifying queries."
---

# Repositories, Derived Queries & @Query

Spring Data's headline feature is that you declare a method on an interface and get a
working query. Used with judgment, that removes a lot of boilerplate. Used without it,
you get method names forty words long. This lesson covers the full spectrum, from derived
queries to hand-written JPQL and native SQL.

## Derived queries

Spring Data parses the method name:

```java
public interface BookRepository extends JpaRepository<Book, Long> {
    Optional<Book> findByIsbn(String isbn);

    List<Book> findByPublishedGreaterThanEqualOrderByTitleAsc(int year);

    List<Book> findByTitleContainingIgnoreCase(String fragment);

    boolean existsByIsbn(String isbn);

    long countByAuthorId(Long authorId);   // traverses Book.author.id

    List<Book> findTop5ByOrderByPublishedDesc();
}
```

The grammar is: a subject (`find…By`, `exists…By`, `count…By`, `delete…By`, with optional
`Top5`/`First`/`Distinct`), then property expressions joined with `And`/`Or`, each with an
optional operator (`GreaterThan`, `Between`, `In`, `Like`, `Containing`, `IsNull`,
`IgnoreCase`, …), then an optional `OrderBy`.

**Rule of thumb:** derived queries are great for one or two conditions. Beyond that,
switch to `@Query` — the name has become harder to read than the query.

## @Query with JPQL

JPQL looks like SQL but talks about entities and fields, not tables and columns:

```java
@Query("""
       select b from Book b
       where b.author.name = :author and b.published between :from and :to
       order by b.published
       """)
List<Book> byAuthorBetween(String author, int from, int to);

@Query("select b from Book b join fetch b.author")
List<Book> findAllWithAuthor();
```

`join fetch` loads the author in the same query, so accessing `book.getAuthor().getName()`
afterwards costs nothing. Named parameters (`:author`) bind by method parameter name.

## Entity graphs

An alternative to writing a `join fetch` query is to annotate a derived query with which
associations to load:

```java
@EntityGraph(attributePaths = "author")
List<Book> findByTitleContainingIgnoreCase(String fragment);
```

Same SQL effect, but the query itself stays derived.

## Worked example: counting the queries

In this level's project, a test seeded three authors with two books each and counted SQL
statements with Hibernate's statistics while touching every book's author:

```java
tx.executeWithoutResult(s -> books.findAll().forEach(b -> b.getAuthor().getName()));
long naive = stats.getPrepareStatementCount();
stats.clear();
tx.executeWithoutResult(s -> books.findAllWithAuthor().forEach(b -> b.getAuthor().getName()));
long fetched = stats.getPrepareStatementCount();
System.out.println("N+1 statements: naive=" + naive + " joinFetch=" + fetched);
```

Real output:

```text
N+1 statements: naive=4 joinFetch=1
```

One query for the books plus one per distinct author (three) versus a single joined
query. With 500 authors the naive version would issue 501 statements. Level 3 treats this
N+1 problem in depth.

## Projections: select only what you need

When an endpoint needs three columns, loading full entities is wasteful and puts them in
the persistence context for dirty checking. Project straight into a record:

```java
public record BookSummary(Long id, String title, String authorName) { }

@Query("""
       select new com.example.library.BookSummary(b.id, b.title, a.name)
       from Book b join b.author a
       where a.id = :authorId
       """)
List<BookSummary> summariesForAuthor(Long authorId);
```

Spring Data also supports interface-based projections (`interface BookTitle { String getTitle(); }`
as a return type) and, for derived queries, record projections by matching constructor
parameter names.

## Paging and sorting

```java
Page<Book> findByAuthorId(Long authorId, Pageable pageable);
```

In a controller, Spring Data's web support binds `?page=0&size=20&sort=title,asc` into a
`Pageable` automatically:

```java
@GetMapping("/api/authors/{id}/books")
PagedModel<BookSummary> books(@PathVariable long id, @PageableDefault(size = 20) Pageable pageable) {
    return new PagedModel<>(books.findByAuthorId(id, pageable).map(BookSummary::from));
}
```

`PagedModel` gives a stable JSON shape (`content` plus `page` metadata); serializing a raw
`PageImpl` is discouraged because its JSON structure is not a guaranteed contract. A
`Page` runs an extra `count` query; if you only need "is there a next page," return a
`Slice` instead, which fetches `size + 1` rows and skips the count.

Always cap the page size (`spring.data.web.pageable.max-page-size`, default 2000 — set it
lower) so a client cannot request a million rows.

## Modifying queries and native SQL

```java
@Modifying(clearAutomatically = true)
@Transactional
@Query("update Book b set b.published = :year where b.published is null")
int backfillPublished(int year);

@Query(value = "select * from book where to_tsvector(title) @@ plainto_tsquery(:q)", nativeQuery = true)
List<Book> fullTextSearch(String q);   // PostgreSQL-specific
```

Bulk JPQL updates bypass the persistence context: managed entities already loaded in
the current context will still hold the old values, which is why `clearAutomatically`
exists. Native queries are fine for database-specific features; you lose portability and
compile-time checking of entity names.

## How It Actually Works

At startup, `@EnableJpaRepositories` (applied by Boot's auto-configuration) finds every
interface extending `Repository`. For each one it creates a **JDK dynamic proxy**
implementing that interface, backed by `SimpleJpaRepository` for the CRUD methods.

For every other method, a `QueryLookupStrategy` decides how to execute it: an `@Query`
annotation wins; otherwise a named query is looked up; otherwise the name is parsed by
`PartTree` into a subject, predicates, and ordering, and turned into a JPA Criteria query.
All of this happens **at startup**, so a typo like `findByTitel` fails the application
with "No property 'titel' found for type 'Book'" rather than at the first call. JPQL in
`@Query` is also validated at startup by Hibernate. Native SQL is not.

When you call a repository method, the proxy passes through Spring's interceptors —
including a transaction interceptor, which is why repository methods are transactional
by default (`readOnly = true` for finders) even when your service is not — and then into
the query execution.

## Common mistakes

- **Unreadable derived names.** Move to `@Query` after two conditions.
- **Returning entities for list endpoints** when a projection would do.
- **Unbounded `findAll()`** on tables that grow. Page everything user-facing.
- **Forgetting that bulk updates skip dirty checking, `@Version`, and entity callbacks.**
- **`join fetch` on two collections at once**, producing a Cartesian product (and in
  Hibernate, a `MultipleBagFetchException` for `List`s). Fetch one collection per query.

## Exercise

1. Add derived queries for `existsByIsbn` and `findTop5ByOrderByPublishedDesc`, with
   tests.
2. Write the `BookSummary` projection query and verify in the SQL log that only three
   columns are selected.
3. Add a paged `GET /api/authors/{id}/books` endpoint with a max page size of 50. Test
   `?size=1000` and confirm it is capped.
4. Reproduce the N+1 count above in your own test with Hibernate statistics
   (`spring.jpa.properties.hibernate.generate_statistics: true`), then fix it once with
   `join fetch` and once with `@EntityGraph`.
