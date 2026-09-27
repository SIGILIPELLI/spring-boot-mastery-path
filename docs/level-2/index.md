---
description: "Level 2 of Spring Boot: Spring Data JPA, relationships, queries, transactions, Flyway, testing slices, Spring Security, JWTs, and OpenAPI docs."
---

# Level 2 · Intermediate <span class="level-badge">Data & Security</span>

Goal: turn the in-memory API from Level 1 into a real service — backed by a relational
database with versioned migrations, covered by fast and realistic tests, protected by
Spring Security with JWT bearer tokens, and documented with OpenAPI.

This is the level where most day-to-day Spring Boot work lives. It is also where the
framework's abstractions start to leak: JPA's persistence context, transaction
boundaries, and the security filter chain all behave in ways that surprise people who
only learned the annotations. Each lesson's **How It Actually Works** section is there so
those surprises become predictable.

## Modules

1. [Spring Data JPA & Hibernate Basics](01-spring-data-jpa-basics.md) — JPA vs Hibernate vs Spring Data, data sources, and the persistence context
2. [Entities & Relationships](02-entities-relationships.md) — mapping tables, `@ManyToOne`/`@OneToMany`, owning sides, cascades, and equality
3. [Repositories, Derived Queries & @Query](03-repositories-queries.md) — query methods, JPQL, projections, paging, and sorting
4. [Transactions with @Transactional](04-transactions.md) — boundaries, propagation, rollback rules, and read-only transactions
5. [Database Migrations with Flyway](05-flyway-migrations.md) — versioned SQL, `ddl-auto: validate`, and safe schema evolution
6. [Testing: JUnit 5, Slices & Testcontainers](06-testing.md) — unit tests, `@WebMvcTest`, `@DataJpaTest`, and real databases in tests
7. [Spring Security Fundamentals](07-spring-security-basics.md) — the filter chain, authentication vs authorization, and password storage
8. [JWT & the OAuth2 Resource Server](08-jwt-resource-server.md) — validating bearer tokens, scopes, and identity providers
9. [API Documentation with springdoc-openapi](09-openapi-springdoc.md) — generating an OpenAPI spec and Swagger UI from your code
10. [Project — A Secured Library API with a Database](10-project-secured-library-api.md) — authors and books with JPA, Flyway, JWT security, and tests

## What you need before starting

- Level 1 completed, especially configuration, validation, and exception handling.
- Basic SQL: `SELECT` with joins, primary and foreign keys, indexes. The
  [SQL Mastery Path](https://sigilipelli.github.io/sql-mastery-path/) covers this.
- Optional but recommended: Docker, for running PostgreSQL locally and for Testcontainers.
  Lessons use the in-memory H2 database where possible so everything also runs without
  Docker, and say so explicitly when something needs a real database.
