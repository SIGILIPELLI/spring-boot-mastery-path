---
description: "Compile Spring Boot apps to GraalVM native images: AOT processing, the closed-world assumption, reachability hints, testing native builds, and trade-offs."
---

# GraalVM Native Images

A native image is your application compiled ahead of time into a standalone executable:
no JVM warm-up, startup typically measured in tens of milliseconds, and a smaller memory
footprint. The price is a long build, some dynamic behavior that must be declared up
front, and usually lower peak throughput than a warmed-up JIT. Spring Boot has first-class
support for building them.

## When it is worth it

- Scale-to-zero and serverless deployments, where every cold start is user-visible.
- Command-line tools and short jobs.
- Many small instances where per-instance memory dominates cost.

For long-running, high-throughput services on steady traffic, the regular JVM (optionally
with CDS or CRaC for startup) is often the better choice. Decide by measuring your own app.

## Building one

Generate a project with the **GraalVM Native Support** dependency (it configures the
`native-maven-plugin` in the Boot parent's `native` profile). With a GraalVM JDK installed:

```bash
./mvnw -Pnative native:compile          # produces target/<artifact> executable
./target/library
```

Or, without a local GraalVM, build a container image with buildpacks, which run the native
compilation inside a builder image:

```bash
./mvnw -Pnative spring-boot:build-image
```

Native compilation is memory- and CPU-hungry (several GB of RAM, minutes of build time).
Native builds were **not** run for this course, so no startup figures are quoted here —
measure your own with `time` and the "Started … in" log line.

## Worked example: fixing a reflection failure

A service reads a JSON file into a class only referenced by name at runtime:

```java
Class<?> type = Class.forName(props.handlerClass());   // name from configuration
Object handler = type.getDeclaredConstructor().newInstance();
```

On the JVM this works. In a native image it fails at runtime (`ClassNotFoundException` or
missing constructor), because static analysis could not see which class would be named.
Tell the compiler with a runtime hint:

```java
@Configuration
@ImportRuntimeHints(HandlerHints.class)
class HandlerConfig { }

class HandlerHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        hints.reflection().registerType(CsvImportHandler.class,
                MemberCategory.INVOKE_DECLARED_CONSTRUCTORS);
        hints.resources().registerPattern("imports/*.json");
    }
}
```

`@RegisterReflectionForBinding(SomeDto.class)` is a shortcut for types serialized with
Jackson via reflection. Better still, avoid runtime class-name lookups: a `Map` of beans
by key needs no hints at all.

## Testing native behavior

Run your test suite in AOT mode on the JVM first (`./mvnw -PnativeTest test` compiles tests
natively; running with `-Dspring.aot.enabled=true` exercises the AOT-generated code paths
on the JVM, which is much faster and catches most problems). Keep a native smoke test in CI
that builds the image and calls a few endpoints.

## How It Actually Works

GraalVM `native-image` makes a **closed-world assumption**: it starts from `main`,
statically follows every reachable method, and compiles only those, together with a
pre-initialized heap snapshot, into the binary. Anything reached only dynamically —
reflection, resources loaded by name, JDK proxies, serialization — is invisible unless
declared in **reachability metadata** (JSON files under `META-INF/native-image/`).

Spring's contribution is **AOT processing** at build time (`process-aot` goal). Boot starts
the application context *in the build*, far enough to compute the final bean definitions —
evaluating `@Conditional`s and `@Profile`s with the build-time classpath and properties — and
then generates Java source code that registers those beans directly (functional
registration, no classpath scanning or annotation parsing at runtime). It also emits the
reachability metadata for what it knows the framework will use reflectively (proxies for
`@Transactional`, `@ConfigurationProperties` binding, controllers' parameters). Libraries
contribute metadata through the shared GraalVM Reachability Metadata Repository.

Consequences you must accept:

- **Conditions are frozen at build time.** A `@ConditionalOnProperty` bean cannot be switched
  on by a runtime property; `@Profile`-specific beans must be decided at build time
  (profile-specific *properties* still work at runtime).
- **The classpath is fixed.** No adding jars at runtime.
- **CGLIB proxies are generated at build time**, so proxied beans work, but arbitrary runtime
  bytecode generation does not.

## Common mistakes

- **Adopting native for a service whose startup time never mattered.**
- **Only testing on the JVM**, then discovering missing hints in production.
- **Runtime-only feature flags implemented with `@ConditionalOnProperty`.** Use a regular
  `if` on a property for toggles that must change without rebuilding.
- **Libraries with heavy reflection and no metadata**, which need many manual hints.
- **Comparing a native image to a cold JVM on throughput** — compare to a warmed-up JVM.

## Exercise

1. Add GraalVM Native Support to the Level 1 task API and build a native container image
   with buildpacks. Record startup time and resident memory versus the JVM jar on your
   machine.
2. Add the `Class.forName` handler above, observe the native failure, and fix it with a
   `RuntimeHintsRegistrar`. Then refactor to a `Map<String, Handler>` of beans and remove the
   hint.
3. Run your tests with AOT enabled on the JVM and fix anything that fails.
4. Write down, for your capstone, whether you would ship native or JVM, and the measurement
   that justifies it.
