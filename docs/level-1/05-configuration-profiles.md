---
description: "Configure Spring Boot with application.yml, environment variables, and profiles, and bind settings to validated @ConfigurationProperties records."
---

# Configuration: application.yml, Profiles & @ConfigurationProperties

The same jar should run on your laptop, in CI, and in production. What changes between
them — database URLs, credentials, feature flags, timeouts — belongs in configuration, not
code. Spring Boot merges many configuration sources into one `Environment` and lets you
bind it to typed objects.

## application.yml

```yaml
spring:
  application:
    name: tasks
server:
  port: 8080
  shutdown: graceful

tasks:
  max-open-tasks: 100
  default-page-size: 20
  notifications:
    enabled: false
    from-address: noreply@example.com
```

Keys are hierarchical: `tasks.notifications.enabled` is one property. Boot's own
properties (`server.port`, `spring.datasource.url`, `logging.level.*`) are documented in
the "Common Application Properties" appendix of the reference docs, and IDEs autocomplete
them.

## Where values come from, and who wins

Boot reads many sources. Higher in this list overrides lower (simplified to the ones you
will actually use):

1. Command-line arguments: `java -jar app.jar --server.port=9090`
2. `SPRING_APPLICATION_JSON` environment variable
3. OS environment variables: `SERVER_PORT=9090`
4. Profile-specific files: `application-prod.yml`
5. `application.yml` packaged in the jar
6. Defaults in `@ConfigurationProperties` classes

Environment variables use **relaxed binding**. The documented rule for turning a
property name into a variable name is: replace dots with underscores, remove dashes, and
uppercase. So `tasks.max-open-tasks` becomes `TASKS_MAXOPENTASKS`, and
`spring.datasource.url` becomes `SPRING_DATASOURCE_URL`. (Underscores in an env var always
mean "a dot", so `TASKS_MAX_OPEN_TASKS` would be read as `tasks.max.open.tasks` — a
different, nested key.)

Files are also loaded from outside the jar — a `config/` directory next to where you run
it, or anywhere you point `spring.config.import` / `spring.config.additional-location` at
— and those override the packaged ones. This is how you add production settings without
rebuilding.

## Profiles

A profile is a named set of configuration that is only active when you ask for it:

```yaml
# application.yml — shared defaults
tasks:
  max-open-tasks: 100
logging:
  level:
    com.example.tasks: INFO
---
spring:
  config:
    activate:
      on-profile: dev
logging:
  level:
    com.example.tasks: DEBUG
---
spring:
  config:
    activate:
      on-profile: prod
server:
  shutdown: graceful
```

The `---` separators create multiple documents in one file; each later document
overrides the earlier ones when its condition matches. The alternative is separate
`application-dev.yml` / `application-prod.yml` files, which is easier to read once a
profile grows beyond a few lines.

Activate profiles with `--spring.profiles.active=dev`, the `SPRING_PROFILES_ACTIVE`
environment variable, or `@ActiveProfiles("test")` in tests. Beans can be profile-specific
with `@Profile("dev")`.

Use profiles for *environments or modes*, not for every tweak. If you find yourself with
`prod-eu-large-cache`, you want properties, not profiles.

## @ConfigurationProperties: typed, validated config

Injecting single values with `@Value("${tasks.max-open-tasks}")` works but scatters string
keys across the codebase and gives no validation. Bind a whole prefix to a record instead:

```java
package com.example.tasks.task;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.bind.DefaultValue;
import org.springframework.validation.annotation.Validated;

@Validated
@ConfigurationProperties(prefix = "tasks")
public record TaskProperties(
        @Min(1) @Max(10_000) int maxOpenTasks,
        @DefaultValue("20") int defaultPageSize,
        Notifications notifications) {

    public record Notifications(boolean enabled, @NotBlank String fromAddress) { }
}
```

Register it by adding `@ConfigurationPropertiesScan` to the main class (or
`@EnableConfigurationProperties(TaskProperties.class)` on a configuration class), then
inject it like any bean:

```java
@Service
class TaskService {
    private final TaskProperties props;
    TaskService(TaskProperties props) { this.props = props; }
}
```

## Worked example: failing fast on bad config

With the record above, start the app with an invalid value:

```bash
java -jar target/tasks-0.0.1-SNAPSHOT.jar --tasks.max-open-tasks=0
```

Startup stops before the web server opens. Running exactly this against the Level 1
project (Boot 4.1) printed:

```text
***************************
APPLICATION FAILED TO START
***************************

Description:

Binding to target com.example.tasks.task.TaskProperties failed:

    Property: tasks.maxOpenTasks
    Value: "0"
    Origin: "tasks.max-open-tasks" from property source "commandLineArgs"
    Reason: must be greater than or equal to 1
```

Note the `Origin` line: Boot tracks which property source supplied each value, which is
invaluable when you cannot tell whether a setting came from a file, an environment
variable, or the command line.

That is precisely what you want: a misconfigured deployment fails immediately and loudly,
instead of misbehaving at 3 a.m. when the first user hits the limit. Add
`spring-boot-configuration-processor` as an optional dependency and your IDE also gets
autocomplete and documentation for `tasks.*`.

## How It Actually Works

The `Environment` is an ordered list of `PropertySource`s — command-line args, system
properties, env vars, each loaded config file — and a lookup walks them in order,
returning the first hit. Profile-specific documents are inserted ahead of the default ones
when their profile is active, which is how they override.

Binding is done by Boot's `Binder`. For `@ConfigurationProperties` it walks the target type:
for a record it finds the canonical constructor and, for each parameter, computes the
property name (`maxOpenTasks` → `tasks.max-open-tasks`), looks it up through all property
sources using relaxed-name matching, and converts the string with Spring's
`ConversionService` — which is how `"30s"` becomes a `Duration`, `"10MB"` a `DataSize`,
and `"a,b,c"` a `List<String>`. Nested records recurse. Afterwards, because the class is
`@Validated`, the bound object is passed to the Bean Validation `Validator`; any violation
aborts context startup with the report above.

`@Value` is resolved differently: a `BeanPostProcessor` evaluates `${…}` placeholders (and
`#{…}` SpEL expressions) when it injects each field or parameter. It does not support
relaxed binding in the same way and has no validation, which is why
`@ConfigurationProperties` is preferred for anything beyond a single flag.

## Common mistakes

- **Committing secrets to `application.yml`.** Passwords and API keys come from
  environment variables or a secret store (Level 4). The file can contain
  `${DB_PASSWORD}` placeholders.
- **Profile explosion.** A profile per customer or per region quickly becomes unreadable.
- **Forgetting that tests pick up `application.yml` too.** Use `src/test/resources/application.yml`
  or `@TestPropertySource`/`@DynamicPropertySource` for test overrides.
- **Mutable config classes with setters** that other beans modify at runtime. Records
  make configuration immutable by construction.
- **Typos that bind to nothing.** An unknown key such as `tasks.max-opentasks` is silently
  ignored. The configuration processor warns in the IDE; `@Validated` with `@Min` would
  catch the resulting default of `0`.

## Exercise

1. Create `TaskProperties` with `maxOpenTasks`, `defaultPageSize` (default 20), and a
   nested `Notifications` record. Validate them.
2. Add `dev` and `prod` profiles: `dev` logs your package at `DEBUG`; `prod` sets
   `server.shutdown: graceful`. Start with each profile and verify the difference with
   `/actuator/env` (add Actuator) or by logging the injected properties at startup.
3. Override `tasks.max-open-tasks` three ways — command line, environment variable, and an
   external `config/application.yml` — and confirm which wins when all three are set.
4. Start with an invalid value and paste the startup failure into a comment in your
   properties class, so the next developer knows what to expect.
