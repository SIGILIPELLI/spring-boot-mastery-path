---
description: "Detect and fix the JPA N+1 query problem: lazy proxies, join fetch, entity graphs, batch fetching, DTO projections, and pagination pitfalls."
---

# The N+1 Problem & Fetch Strategies

The N+1 problem is the single most common performance bug in JPA applications. One query
loads N rows; then touching an association on each row triggers one more query per row.
It passes every test with five rows of data and falls over in production with five
thousand.

## Seeing it

With three authors who each have two books, the Level 2 project ran:

```java
books.findAll().forEach(b -> b.getAuthor().getName());
```

and counted statements with Hibernate's statistics: **4** — one `select … from book`,
then one `select … from author where id=?` for each distinct author. (Hibernate did not
query the same author twice: the second book by the same author found the author already
in the persistence context.) The join-fetch version issued **1**. Real output:

```text
N+1 statements: naive=4 joinFetch=1
```

The number of extra queries grows with the number of *distinct* associated rows — in a
real catalog that is effectively N.

## Detecting it before users do

- **Log SQL in development:** `logging.level.org.hibernate.SQL: DEBUG`. A page of identical
  `select … where id=?` lines is the signature.
- **Count statements in tests:** enable `hibernate.generate_statistics` in the test
  profile and assert on `Statistics.getPrepareStatementCount()`, as the project did. A
  test that pins "listing books takes exactly 1 query" catches regressions forever.
- **Watch traces in production:** with tracing (Level 4), each JDBC call is a span; an
  N+1 shows up as a ladder of identical spans.

## Fix 1: join fetch in JPQL

```java
@Query("select b from Book b join fetch b.author")
List<Book> findAllWithAuthor();
```

One SQL join; the authors are loaded and attached in the same result set. Use it when a
specific use case always needs the association.

## Fix 2: entity graphs

```java
@EntityGraph(attributePaths = "author")
List<Book> findByTitleContainingIgnoreCase(String fragment);
```

The same effect declared on a derived query. You can also define reusable named graphs
with `@NamedEntityGraph` on the entity.

## Fix 3: batch fetching

When you cannot change the query — or the access pattern varies — tell Hibernate to load
lazy associations in batches:

```yaml
spring:
  jpa:
    properties:
      hibernate.default_batch_fetch_size: 50
```

Now the first time any uninitialized author proxy is touched, Hibernate loads up to 50
pending authors with one `where id in (…)` query. N+1 becomes 1 + ⌈N/50⌉. It is a good
global safety net and does not change any code; per-association control is available with
`@BatchSize`.

## Fix 4: don't load entities at all

For read-only list views, a DTO projection is usually the best answer:

```java
@Query("""
       select new com.example.library.BookSummary(b.id, b.title, a.name)
       from Book b join b.author a
       """)
List<BookSummary> summaries();
```

One query, only the needed columns, nothing added to the persistence context, nothing to
dirty-check.

## Collections and pagination

Fetching a **collection** (`join fetch a.books`) multiplies rows: an author with 10 books
appears 10 times in the SQL result. Hibernate 6+ deduplicates the root entities
automatically, but two problems remain:

1. **Two collection fetches in one query** create a Cartesian product (books × awards).
   With `List` mappings Hibernate refuses with `MultipleBagFetchException`; with `Set`s it
   silently multiplies rows. Fetch one collection per query, or use batch fetching for the
   second.
2. **Paging with a collection fetch** cannot be done in SQL (a `LIMIT 20` would cut an
   author's books in half). Hibernate then loads *everything* and pages in memory, logging a
   warning that pagination is being applied in memory. You can make it fail instead with
   `hibernate.query.fail_on_pagination_over_collection_fetch=true` — recommended. The fix
   is two queries: page the parent ids, then fetch those parents with their collection.

```java
@Query("select a.id from Author a order by a.name")
Page<Long> pageIds(Pageable pageable);

@Query("select distinct a from Author a left join fetch a.books where a.id in :ids order by a.name")
List<Author> withBooks(List<Long> ids);
```

## Worked example: fixing an endpoint

`GET /api/authors` returns each author with book titles. The naive service:

```java
@Transactional(readOnly = true)
public List<AuthorView> list() {
    return authors.findAll().stream().map(AuthorView::from).toList();   // touches a.getBooks()
}
```

issues 1 + (number of authors) queries. Steps:

1. Write a test asserting the statement count and watch it fail with the current number.
2. Add `hibernate.default_batch_fetch_size: 50` — the count drops to 1 + ⌈authors/50⌉.
3. For the hot path, switch to the two-query paged approach above; the count becomes
   exactly 2 regardless of size (plus a count query if you return a `Page`).
4. Keep the test, now asserting the fixed number.

## How It Actually Works

When Hibernate loads a `Book` whose `author` is `LAZY`, it reads `author_id` from the row
and asks the persistence context for `Author#7`. If it is not there, it creates an
**uninitialized proxy** — a runtime subclass of `Author` holding only the id and a
reference to the session — and registers it. Calling `getName()` on the proxy triggers
initialization: one `select` by primary key. That per-entity initialization is the "+1".

With batch fetching enabled, Hibernate keeps a queue of uninitialized proxies (and
collections) of each type in the persistence context. When one is initialized it takes
up to `batch_fetch_size` pending ids of the same type from that queue and loads them all
with a single `in (…)` query. With `join fetch`, the association columns are already in the
result set, so the entity is built fully initialized and no proxy is created at all.

`FetchType.EAGER` does not solve any of this: for queries (as opposed to `find` by id),
Hibernate first runs your query and *then* loads eager associations, often with exactly
the same per-row selects — N+1 that you cannot turn off per use case. That is why every
association should be `LAZY` and fetching decided per query.

## Common mistakes

- **Switching associations to EAGER** to "fix" a `LazyInitializationException`.
- **Enabling Open Session in View** to the same end, moving the N+1 into JSON serialization
  where it is harder to see.
- **Join-fetching two collections at once.**
- **Paging over a collection fetch** and not noticing the in-memory warning.
- **No statement-count tests**, so the fix regresses silently.

## Exercise

1. Seed 200 authors with 3 books each. Measure the statement count of listing authors with
   their book titles using the naive approach.
2. Apply, one at a time, batch fetching, an entity graph, and the two-query pagination
   approach. Record the statement counts and wall-clock time for each.
3. Enable `fail_on_pagination_over_collection_fetch` and write a query that trips it.
4. Add a permanent test for your hottest list endpoint that asserts its statement count.
