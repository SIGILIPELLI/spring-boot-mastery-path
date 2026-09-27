---
description: "How Spring Boot auto-configuration works: AutoConfiguration.imports, @Conditional evaluation and ordering, back-off rules, and writing your own starter."
---

# How Auto-Configuration Really Works

"Boot configured it for me" is true, but not an explanation. Auto-configuration is
ordinary Spring configuration with conditions attached, loaded from a list in each jar.
Once you can read an auto-configuration class, you can predict what Boot will do, and you
can write your own for code shared across services.

## Reading a real one

Here is the essential shape of an auto-configuration, simplified from the one Boot uses
for Jackson (Boot 4 uses Jackson 3's `JsonMapper`):

```java
@AutoConfiguration
@ConditionalOnClass(JsonMapper.class)
public class JacksonAutoConfiguration {

    @Bean
    @Primary
    @ConditionalOnMissingBean
    JsonMapper jacksonJsonMapper(JsonMapper.Builder builder) {
        return builder.build();
    }
    // ... builder bean, customizers bound to spring.jackson.* properties
}
```

Read it as a set of rules:

- **Only if Jackson is on the classpath** (`@ConditionalOnClass`).
- **Only if you have not defined your own `JsonMapper`** (`@ConditionalOnMissingBean`).
- Otherwise build one from a builder that applies `spring.jackson.*` properties and every
  customizer bean.

That is the entire contract of Boot: *sensible defaults that back off when you
intervene*. The condition report printed by `--debug` (Level 1, lesson 01) is literally
these conditions being evaluated; for this class it printed
`@ConditionalOnMissingBean (types: tools.jackson.databind.json.JsonMapper; SearchStrategy: all) did not find any beans`.

## The conditions you will meet

| Annotation | Matches when |
|-----------|--------------|
| `@ConditionalOnClass` / `OnMissingClass` | A class is / is not on the classpath |
| `@ConditionalOnBean` / `OnMissingBean` | A bean of a type (or name) is / is not already defined |
| `@ConditionalOnProperty` / `OnBooleanProperty` | A property has a value (optionally a specific one; `matchIfMissing` controls the default) |
| `@ConditionalOnWebApplication` / `OnNotWebApplication` | Servlet or reactive web app (or neither) |
| `@ConditionalOnResource` | A resource exists (e.g. a config file) |
| `@ConditionalOnSingleCandidate` | Exactly one bean of a type, or one `@Primary` |
| `@ConditionalOnThreading(Threading.VIRTUAL)` | Virtual threads are enabled |
| `@Profile` | A profile is active (a condition too, under the hood) |

## Worked example: your own starter

Suppose every service in your company needs an `AuditClient` configured the same way.
Build a small library with an auto-configuration:

```java
// audit-spring-boot-starter/src/main/java/com/acme/audit/AuditProperties.java
@ConfigurationProperties("acme.audit")
public record AuditProperties(URI endpoint, @DefaultValue("2s") Duration timeout, @DefaultValue("true") boolean enabled) { }

// .../AuditAutoConfiguration.java
@AutoConfiguration
@ConditionalOnClass(AuditClient.class)
@ConditionalOnBooleanProperty(name = "acme.audit.enabled", matchIfMissing = true)
@EnableConfigurationProperties(AuditProperties.class)
public class AuditAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    AuditClient auditClient(AuditProperties props, RestClient.Builder http) {
        return new HttpAuditClient(http.baseUrl(props.endpoint().toString()).build(), props.timeout());
    }
}
```

Register it in the starter jar's
`src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`:

```text
com.acme.audit.AuditAutoConfiguration
```

A service now adds the dependency and sets `acme.audit.endpoint`. It gets a working
`AuditClient`, can disable it with `acme.audit.enabled=false`, and can replace it by
declaring its own `AuditClient` bean.

Test auto-configurations with `ApplicationContextRunner`, which builds tiny contexts in
milliseconds:

```java
class AuditAutoConfigurationTest {
    private final ApplicationContextRunner runner = new ApplicationContextRunner()
            .withConfiguration(AutoConfigurations.of(AuditAutoConfiguration.class,
                    RestClientAutoConfiguration.class))
            .withPropertyValues("acme.audit.endpoint=http://audit.local");

    @Test
    void createsClientByDefault() {
        runner.run(ctx -> assertThat(ctx).hasSingleBean(AuditClient.class));
    }

    @Test
    void backsOffWhenUserDefinesOne() {
        runner.withBean(AuditClient.class, () -> event -> { })
              .run(ctx -> assertThat(ctx).hasSingleBean(AuditClient.class)
                                        .doesNotHaveBean(HttpAuditClient.class));
    }

    @Test
    void canBeDisabled() {
        runner.withPropertyValues("acme.audit.enabled=false")
              .run(ctx -> assertThat(ctx).doesNotHaveBean(AuditClient.class));
    }
}
```

(The import location of `RestClientAutoConfiguration` differs between Boot 3 and Boot 4's
modular layout; your IDE will find it. The pattern is what matters.)

## How It Actually Works

**Discovery.** `@EnableAutoConfiguration` imports `AutoConfigurationImportSelector`, a
*deferred* import selector. "Deferred" is the key: it runs **after** all your own
`@Configuration` classes and component-scanned beans have been registered. It reads every
`META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` file on
the classpath (older Boot versions used `spring.factories`), removes exclusions
(`spring.autoconfigure.exclude`, `@SpringBootApplication(exclude = …)`), and sorts the
candidates by `@AutoConfiguration(before/after = …)` and `@AutoConfigureOrder`.

In Boot 4 every technology ships its auto-configuration in its own module
(`spring-boot-webmvc`, `spring-boot-jdbc`, `spring-boot-flyway`, …), each with its own
imports file. Listing the one in `spring-boot-webmvc` shows entries such as
`org.springframework.boot.webmvc.autoconfigure.DispatcherServletAutoConfiguration` and
`...WebMvcAutoConfiguration`. The consequence: if the module is not on the classpath, its
auto-configurations are not even candidates.

**Cheap filtering first.** Loading hundreds of classes just to find out that their
conditions fail would slow startup. So Boot's build generates
`META-INF/spring-autoconfigure-metadata.properties`, recording each auto-configuration's
`@ConditionalOnClass` values as strings. An `AutoConfigurationImportFilter` checks those
against the classpath *before* loading the classes, discarding most candidates cheaply.

**Condition evaluation.** For the survivors, each `@Conditional` is evaluated by its
`Condition` implementation (`OnClassCondition`, `OnBeanCondition`, `OnPropertyCondition`).
Conditions are checked at two phases: when parsing the configuration class
(`PARSE_CONFIGURATION`) and when registering its beans (`REGISTER_BEAN`). `OnBeanCondition`
runs at the registration phase and looks at bean **definitions** registered so far — which
is exactly why ordering matters and why auto-configurations must come after user
configuration. It also means `@ConditionalOnMissingBean` inside *your own* regular
`@Configuration` classes is unreliable: it depends on processing order you do not control.
Use it only in auto-configurations.

Each outcome is recorded in the `ConditionEvaluationReport` — the source of the `--debug`
output and of Actuator's `/actuator/conditions` endpoint.

## Common mistakes

- **Component-scanning an auto-configuration.** If your starter's auto-config class sits in
  a package the app scans, it is registered as a normal configuration too early and its
  `@ConditionalOnMissingBean` checks become order-dependent. Keep auto-configs out of
  scanned packages and register them only via the imports file.
- **Excluding a whole auto-configuration to change one bean.** Define the bean instead, or
  use a customizer (`Jackson2ObjectMapperBuilderCustomizer` in Boot 3,
  `JsonMapperBuilderCustomizer` in Boot 4).
- **Starters with hard dependencies** on optional libraries. Mark them `<optional>true</optional>`
  and guard with `@ConditionalOnClass`.
- **Forgetting `proxyBeanMethods = false`** style: auto-configurations should inject
  dependencies as `@Bean` method parameters rather than calling other `@Bean` methods.

## Exercise

1. Run your Level 2 app with `--debug` and find, for `FlywayAutoConfiguration` and
   `HibernateJpaAutoConfiguration`, which conditions matched. Then hit
   `/actuator/conditions` (add Actuator) and find the same information as JSON.
2. List the imports file in one Boot module jar:
   `unzip -p ~/.m2/repository/org/springframework/boot/spring-boot-webmvc/*/spring-boot-webmvc-*.jar META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
3. Build the audit starter as a separate Maven module, use it from the library API, and
   write the three `ApplicationContextRunner` tests.
4. Add a `@ConditionalOnMissingBean` to a bean in one of your *application's* ordinary
   `@Configuration` classes, then explain in a comment why this is fragile.
