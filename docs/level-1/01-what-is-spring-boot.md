---
description: "What the Spring Framework and Spring Boot each are, how the IoC container and auto-configuration divide the work, and what happens when a Boot app starts."
---

# What Spring and Spring Boot Actually Are

"Spring" is used loosely for three different things, and untangling them is the first
step to understanding any error message you will ever get from a Spring application.

| Name | What it is | What it gives you |
|------|-----------|-------------------|
| **Spring Framework** | A set of libraries (`spring-core`, `spring-beans`, `spring-context`, `spring-webmvc`, `spring-tx`, …) | The IoC container, AOP, transaction abstraction, Spring MVC, data-access helpers |
| **Spring Boot** | An opinionated layer on top of the Framework | Starters, auto-configuration, externalized configuration, embedded servers, executable jars, Actuator |
| **The Spring portfolio** | Separate projects built on the Framework | Spring Data, Spring Security, Spring Cloud, Spring Kafka, Spring Batch, … |

Spring Boot does not replace the Framework. Every `@RestController`, `@Transactional`, or
`@Autowired` you write is a Framework feature. What Boot does is *decide how to configure
the Framework for you*, based on the libraries it finds on the classpath and the
properties you set.

## The problem Spring solves

Consider a service that needs a repository, which needs a data source, which needs
configuration. Without a container you wire this by hand:

```java
var config = Config.load("app.properties");
var dataSource = new HikariDataSource(config.toHikari());
var repository = new JdbcOrderRepository(dataSource);
var payments = new StripePaymentClient(config.paymentsKey(), HttpClient.newHttpClient());
var service = new OrderService(repository, payments, Clock.systemUTC());
var controller = new OrderController(service);
```

That is fine for six objects. It stops being fine at two hundred, when some objects must be
shared, some created per request, some wrapped so that their methods run inside a
database transaction, and tests need to swap real implementations for fakes. Spring's
answer is **inversion of control**: you describe the objects (called *beans*) and their
dependencies, and a container builds the graph, manages lifecycles, and applies
cross-cutting behavior.

## The smallest Boot application

```java
package com.example.hello;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

@SpringBootApplication
public class HelloApplication {
    public static void main(String[] args) {
        SpringApplication.run(HelloApplication.class, args);
    }
}

@RestController
class GreetingController {
    @GetMapping("/greet")
    String greet(@RequestParam(defaultValue = "world") String name) {
        return "Hello, " + name;
    }
}
```

With `spring-boot-starter-webmvc` on the classpath, `./mvnw spring-boot:run` starts an
embedded Tomcat on port 8080, and `curl 'localhost:8080/greet?name=Ada'` returns
`Hello, Ada`. You never configured Tomcat, a `DispatcherServlet`, or a JSON converter.

`@SpringBootApplication` is a shorthand for three annotations:

- `@SpringBootConfiguration` — marks this class as a configuration source (a specialized
  `@Configuration`).
- `@EnableAutoConfiguration` — turns on Boot's auto-configuration.
- `@ComponentScan` — scans this class's package **and its sub-packages** for components.

That last point matters: put the main class in the root package (`com.example.hello`), and
everything else below it. A controller in `com.example.other` will simply never be found.

## What Boot adds, concretely

1. **Starters** — dependency bundles. `spring-boot-starter-webmvc` pulls in Spring MVC,
   Jackson, validation hooks, and an embedded Tomcat, at versions tested together.
2. **Auto-configuration** — hundreds of `@Configuration` classes that each say "if this
   library is present and you have not defined your own bean of this type, I will create
   one with sensible defaults."
3. **Externalized configuration** — `application.yml`, environment variables, and
   command-line arguments bound into one `Environment`, with a defined precedence.
4. **Embedded server and executable jar** — the app *contains* its web server, so you ship
   one `java -jar app.jar` instead of deploying a WAR into an application server.
5. **Production features** — Actuator health checks, metrics, and info endpoints.

## Worked example: watching Boot decide

Run any Boot app with `--debug` (or set `debug: true`) and it prints a
**Condition Evaluation Report** at startup, split into *Positive matches* (auto-configurations
that applied, and why) and *Negative matches* (ones that did not). Running the Level 1
project app (Boot 4.1) with `--debug` printed, among a few hundred lines:

```text
Positive matches:
-----------------
   DispatcherServletAutoConfiguration matched:
      - @ConditionalOnClass found required class 'org.springframework.web.servlet.DispatcherServlet' (OnClassCondition)
      - found 'session' scope (OnWebApplicationCondition)

   JacksonAutoConfiguration#jacksonJsonMapper matched:
      - @ConditionalOnMissingBean (types: tools.jackson.databind.json.JsonMapper; SearchStrategy: all) did not find any beans (OnBeanCondition)

Negative matches:
-----------------
   AopAutoConfiguration.AspectJAutoProxyingConfiguration:
      Did not match:
         - @ConditionalOnClass did not find required class 'org.aspectj.weaver.Advice' (OnClassCondition)
```

Read those three entries carefully. The dispatcher servlet exists because Spring MVC is on
the classpath *and* the app is a servlet web app. The JSON mapper exists because Jackson is
present *and* you did not define your own. AspectJ-based proxying was skipped because
AspectJ is not on the classpath. Add a dependency that brings AspectJ, run again, and that
entry moves to *Positive matches*. Nothing in your code changed; only the classpath did.
That is the core idea of Boot in one experiment. (In Boot 4 there is also no
`DataSourceAutoConfiguration` entry at all until you add a JDBC starter, because that
auto-configuration now lives in its own module jar rather than one giant
`spring-boot-autoconfigure` jar.)

!!! tip "Where does the exact wording come from?"
    The report text changes a little between Boot versions. Read the one your own app
    prints rather than memorizing it — it is the single most useful debugging tool for
    "why does (or doesn't) this bean exist?"

## How It Actually Works

`SpringApplication.run(...)` does roughly this, in order:

1. **Decides the application type** by checking the classpath: servlet web app (Spring MVC
   present), reactive web app (only WebFlux present), or plain non-web app.
2. **Prepares the `Environment`** — loads property sources (command line, environment
   variables, `application.yml`, profile-specific files) in precedence order.
3. **Creates an `ApplicationContext`** — the container. For a servlet app it is a web
   context that knows how to start an embedded server.
4. **Loads bean definitions.** Component scanning reads your classes and registers a
   `BeanDefinition` (a recipe: class, scope, constructor arguments, init methods) for each
   `@Component`. Then auto-configuration classes listed in
   `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` files
   inside the Boot jars are evaluated. Their `@Conditional…` annotations are checked, and
   only matching ones contribute definitions. Crucially, *your* definitions are processed
   first, so `@ConditionalOnMissingBean` can back off when you have supplied your own.
5. **Refreshes the context.** Bean factory post-processors run (they may modify
   definitions), then every non-lazy singleton is instantiated: constructor called,
   dependencies injected, `BeanPostProcessor`s applied — which is where proxies for
   `@Transactional` or `@Async` get wrapped around your object.
6. **Starts the web server** and publishes `ApplicationReadyEvent`. Any
   `CommandLineRunner`/`ApplicationRunner` beans run just before that.

Everything later in this course — injection, configuration, proxies, auto-configuration
internals — is a zoom-in on one of these steps.

## Common mistakes

- **Treating Boot as a different framework.** Searching for "Spring Boot how to X" often
  leads to Framework documentation, and that is correct: MVC, transactions, and DI are
  Framework features. Boot only configures them.
- **Main class in the wrong package.** Components outside the main class's package tree
  are not scanned. The symptom is a 404 for a controller you are sure exists, or a
  "required a bean of type … that could not be found" failure.
- **Fighting auto-configuration instead of overriding it.** If you need a different
  `ObjectMapper` or `DataSource`, define your own bean or set a property; do not exclude
  whole auto-configurations unless you know what else they provide.
- **Copying Boot 2 tutorials.** Boot 3 moved from `javax.*` to `jakarta.*` packages
  (Jakarta EE 9+), and Boot 4 split auto-configuration into per-technology modules. An
  import of `javax.persistence.Entity` means the tutorial is out of date.

## Exercise

1. Generate a project at [start.spring.io](https://start.spring.io/) with only the
   **Spring Web** dependency. Add the `GreetingController` above and run it.
2. Start it with `--debug` and find the entries for `DispatcherServletAutoConfiguration`
   and `AopAutoConfiguration`. Write down, in your own words, why each did or did not
   match. Then add the **JDBC API** and **H2** dependencies and find the new
   `DataSourceAutoConfiguration` entry.
3. Move `GreetingController` into a package *outside* the main class's package. Predict
   what happens, run it, and confirm. Then fix it two ways: by moving the class back, and
   by adding `scanBasePackages` to `@SpringBootApplication`. Which fix would you prefer in
   a real codebase, and why?
