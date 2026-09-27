---
description: "Generate a Spring Boot project with Initializr, read every file it creates, understand starters and the parent POM, and build an executable jar."
---

# Spring Initializr & Project Structure

Almost every Spring Boot project begins at [start.spring.io](https://start.spring.io/),
the Spring Initializr. It is worth understanding what it produces line by line, because
the generated build file quietly decides your dependency versions, your Java level, and
how your app is packaged.

## Generating a project

On the web form choose:

- **Project:** Maven (Gradle is equally supported; see below)
- **Language:** Java
- **Spring Boot:** the latest stable release (not a SNAPSHOT or milestone)
- **Group / Artifact:** `com.example` / `tasks`
- **Packaging:** Jar
- **Java:** 21 (17 is the minimum for Boot 3 and 4)
- **Dependencies:** Spring Web, Validation

Initializr is also an HTTP API, which is handy for scripts and for reproducing a setup
exactly:

```bash
curl -s -o tasks.zip "https://start.spring.io/starter.zip?type=maven-project&language=java&javaVersion=21&groupId=com.example&artifactId=tasks&packageName=com.example.tasks&dependencies=web,validation"
unzip tasks.zip -d tasks && cd tasks
```

IntelliJ IDEA and the VS Code Spring extensions call the same service from their "new
project" wizards.

## What you get

```text
tasks/
├── .mvn/wrapper/maven-wrapper.properties   # which Maven version ./mvnw downloads
├── mvnw, mvnw.cmd                          # the Maven wrapper scripts
├── pom.xml                                 # the build
└── src/
    ├── main/
    │   ├── java/com/example/tasks/TasksApplication.java
    │   └── resources/application.properties
    └── test/
        └── java/com/example/tasks/TasksApplicationTests.java
```

Rename `application.properties` to `application.yml` if you prefer YAML — Boot reads
either. This course uses YAML because nested configuration is easier to scan.

### The pom.xml, section by section

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>4.1.1</version>
    <relativePath/>
</parent>
```

The **parent POM** is where most of Boot's build behavior comes from. It imports
`spring-boot-dependencies`, a bill of materials (BOM) that pins versions for hundreds of
libraries — Jackson, Hibernate, Tomcat, Flyway, JUnit, Mockito — to combinations the Boot
team tests together. That is why your dependencies below have no `<version>` tags. It also
configures sensible plugin defaults (compiler release level, resource filtering,
`-parameters` so Spring can read constructor parameter names).

```xml
<properties>
    <java.version>21</java.version>
</properties>
```

One property sets the compiler's `--release` level for the whole build.

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webmvc</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webmvc-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

A **starter** is an almost-empty jar whose only job is to declare a coherent group of
dependencies. `spring-boot-starter-webmvc` brings Spring MVC, the embedded Tomcat, Jackson,
and the Boot modules that auto-configure them.

!!! note "Boot 3 vs Boot 4 naming"
    In Boot 3.x the web starter is `spring-boot-starter-web` and there is one
    `spring-boot-starter-test` for all testing. Boot 4 modularized things: web is
    `spring-boot-starter-webmvc`, and each technology has a matching `-test` starter
    (`spring-boot-starter-webmvc-test`, `spring-boot-starter-data-jpa-test`, …). The old
    names still resolve in Boot 4 as deprecated aliases in many cases, but new projects
    should use what Initializr generates.

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

The **Boot Maven plugin** gives you `spring-boot:run`, repackages the normal jar into an
executable "fat" jar, and can build OCI container images (`spring-boot:build-image`).

## Worked example: build and run the jar

```bash
./mvnw -q package          # compile, run tests, produce target/tasks-0.0.1-SNAPSHOT.jar
java -jar target/tasks-0.0.1-SNAPSHOT.jar
```

The first `./mvnw` run downloads Maven itself (into `~/.m2/wrapper`) and every dependency
(into `~/.m2/repository`), so it is slow once and fast afterwards. The wrapper is the
reason teammates and CI servers all build with the same Maven version — commit it.

Useful commands you will use daily:

| Command | What it does |
|---------|-------------|
| `./mvnw spring-boot:run` | Compile and run from source |
| `./mvnw test` | Run the test suite |
| `./mvnw -Dtest=TaskControllerTest test` | Run one test class |
| `./mvnw dependency:tree` | Show every transitive dependency and where it came from |
| `./mvnw package -DskipTests` | Build the jar without tests (for quick local experiments only) |

### Gradle in one glance

The same project as a Gradle Kotlin DSL build:

```kotlin
plugins {
    java
    id("org.springframework.boot") version "4.1.1"
    id("io.spring.dependency-management") version "1.1.7"
}

java { toolchain { languageVersion = JavaLanguageVersion.of(21) } }

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-webmvc")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    testImplementation("org.springframework.boot:spring-boot-starter-webmvc-test")
}
```

Commands become `./gradlew bootRun`, `./gradlew test`, and `./gradlew bootJar`. Pick the
plugin versions Initializr generates for you (these are what it produced in September 2026) rather than copying these numbers; they move.

## Organizing packages

Initializr gives you one package. As the app grows, prefer **package by feature** over
package by layer:

```text
com.example.tasks
├── TasksApplication.java
├── task/            # TaskController, TaskService, Task, TaskRepository, DTOs
├── user/
└── shared/          # cross-cutting pieces: error handling, config
```

Feature packages let you make classes package-private (a controller need not be `public`),
keep related changes in one directory, and make it obvious later where module boundaries
could go (Level 4 returns to this).

## How It Actually Works

**The executable jar.** A normal jar cannot contain other jars on its classpath; the JVM's
class loader does not look inside nested archives. The Boot plugin rearranges the jar:

```text
BOOT-INF/classes/        # your compiled classes and resources
BOOT-INF/lib/            # every dependency jar, stored uncompressed
META-INF/MANIFEST.MF     # Main-Class: org.springframework.boot.loader.launch.JarLauncher
                         # Start-Class: com.example.tasks.TasksApplication
org/springframework/boot/loader/...   # the launcher code
```

`java -jar` runs `JarLauncher`, not your class. The launcher creates a class loader that
can read the nested jars in `BOOT-INF/lib` directly (that is why they are stored
uncompressed — they can be read by offset without extracting), then calls your
`Start-Class`'s `main` method with that class loader. You can inspect all of this with
`jar tf target/tasks-0.0.1-SNAPSHOT.jar | head -30`.

**Dependency management.** The BOM in the parent is a `<dependencyManagement>` section:
it does not add dependencies, it only fixes the version *if* you declare one. When two
libraries pull in different versions of the same transitive dependency, Maven's "nearest
wins" rule would normally pick one somewhat arbitrarily; the BOM makes the choice explicit
and tested. To override a single managed version, set its property — for example
`<jackson-bom.version>…</jackson-bom.version>` — rather than adding a `<version>` to one
dependency, so every related artifact moves together.

## Common mistakes

- **Choosing a SNAPSHOT or milestone Boot version** from the Initializr dropdown for a
  real project. Use the latest GA release.
- **Mixing Boot starters with hand-picked versions** of Spring Framework or Hibernate.
  Let the BOM manage them; version skew between Spring modules causes
  `NoSuchMethodError`s at startup.
- **Not committing the wrapper** (`mvnw`, `.mvn/`). Then "works on my machine" becomes a
  Maven-version problem.
- **Putting everything in one package with `public` on every class.** It works, but it
  throws away the only encapsulation boundary Java gives you by default.

## Exercise

1. Generate the `tasks` project with the curl command above. Build it with
   `./mvnw package` and run the jar.
2. Run `jar tf target/*.jar | grep -E "MANIFEST|BOOT-INF/lib" | head` and open the
   manifest with `unzip -p target/*.jar META-INF/MANIFEST.MF`. Identify `Main-Class` and
   `Start-Class`.
3. Run `./mvnw dependency:tree` and find which starter brought in Tomcat and which
   brought in Hibernate Validator.
4. Create the `task` feature package and move nothing into it yet — you will fill it over
   the next lessons.
