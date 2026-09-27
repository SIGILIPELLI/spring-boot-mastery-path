---
description: "Level 1 of Spring Boot: the IoC container, Initializr, beans, configuration and profiles, REST controllers, validation, ProblemDetail errors, and logging."
---

# Level 1 · Entry <span class="level-badge">Foundations</span>

Goal: understand what the Spring container does with your classes, and build a clean,
validated JSON API on top of it. By the end of this level you will have a task-tracker
REST API with typed configuration, consistent `application/problem+json` error
responses, structured logs, and controller tests.

This level is deliberately about the core, not the database. Persistence arrives in
Level 2; keeping storage in memory here means every behavior you see comes from Spring
MVC and the container, not from Hibernate.

## Modules

1. [What Spring and Spring Boot Actually Are](01-what-is-spring-boot.md) — the framework, the container, and what Boot adds on top
2. [Spring Initializr & Project Structure](02-initializr-project-structure.md) — starters, the parent POM, the Maven wrapper, and the executable jar
3. [Dependency Injection & the IoC Container](03-dependency-injection-ioc.md) — constructor injection, interfaces, and how the container resolves a graph
4. [Beans, Scopes & Component Scanning](04-beans-component-scanning.md) — stereotypes, `@Bean` methods, scopes, and qualifiers
5. [Configuration: application.yml, Profiles & @ConfigurationProperties](05-configuration-profiles.md) — property sources, precedence, profiles, and validated config records
6. [Building REST Controllers](06-rest-controllers.md) — `@RestController`, mappings, status codes, and `ResponseEntity`
7. [Request Binding, Responses & Validation](07-request-response-validation.md) — path/query/body binding, Jackson, and Jakarta Bean Validation
8. [Exception Handling with @ControllerAdvice & ProblemDetail](08-exception-handling.md) — RFC 9457 error bodies and one place for error policy
9. [Logging with SLF4J & Logback](09-logging.md) — levels, parameterized messages, MDC, and structured JSON logs
10. [Project — A Task Tracker REST API](10-project-task-api.md) — everything above in one small, tested service

## What you need before starting

- Solid Java: classes and interfaces, generics, lambdas and streams, records, exceptions.
  The [Java Mastery Path](https://sigilipelli.github.io/java-mastery-path/) covers these.
- A JDK 17 or newer (21 or 25 recommended). Check with `java -version`.
- Basic Maven familiarity is helpful but not required — every project uses the Maven
  wrapper (`./mvnw`), so you do not need Maven installed separately.
- `curl` (or any HTTP client) to call your endpoints.

An IDE with Spring support (IntelliJ IDEA, or VS Code with the Spring Boot extension pack)
makes navigating beans and properties much easier, but everything here also works from a
terminal.
