---
description: "Package Spring Boot as container images with buildpacks or layered Dockerfiles, size the JVM for containers, run as non-root, and shut down gracefully."
---

# Containerizing Spring Boot

Containers are how most Spring Boot services ship today. A good image is small, rebuilds
fast when only your code changes, runs as a non-root user, uses memory the way the
container limit expects, and stops cleanly when the platform asks.

## Option 1: Cloud Native Buildpacks

No Dockerfile needed:

```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=registry.example.com/library:1.4.0
```

The Boot plugin runs the Paketo buildpacks, which pick a JRE matching your
`java.version`, create a layered image, run as a non-root user, and configure JVM memory
from the container limit via a memory calculator. This requires Docker (or a compatible
daemon) and was not run for this course.

## Option 2: a layered Dockerfile

Boot jars include a layer index so dependencies (which change rarely) and your classes
(which change every build) land in separate image layers:

```dockerfile
# syntax=docker/dockerfile:1
FROM eclipse-temurin:21-jre AS extract
WORKDIR /build
COPY target/library-*.jar app.jar
RUN java -Djarmode=tools -jar app.jar extract --layers --launcher --destination extracted

FROM eclipse-temurin:21-jre
RUN useradd --system --uid 10001 app
USER 10001
WORKDIR /app
COPY --from=extract /build/extracted/dependencies/ ./
COPY --from=extract /build/extracted/spring-boot-loader/ ./
COPY --from=extract /build/extracted/snapshot-dependencies/ ./
COPY --from=extract /build/extracted/application/ ./
ENV JAVA_TOOL_OPTIONS="-XX:MaxRAMPercentage=75 -XX:+ExitOnOutOfMemoryError"
EXPOSE 8080
ENTRYPOINT ["java", "org.springframework.boot.loader.launch.JarLauncher"]
```

You can list the layers of your own jar with
`java -Djarmode=tools -jar target/app.jar list-layers`. When only application code changes,
only the last `COPY` layer changes — pushes and pulls move kilobytes, not the 60+ MB of
dependencies. (The `tools` jar mode replaced the older `layertools` mode in Boot 3.3.)

Build the jar first (`./mvnw -q package`), or build it inside a separate Maven stage.

## JVM memory in containers

Modern JVMs read the container's memory limit (cgroups). Without flags, the default max
heap is only 25% of it — wasteful for a container that runs nothing else. Set a
percentage instead of a fixed `-Xmx`:

- `-XX:MaxRAMPercentage=75` leaves room for non-heap memory: metaspace, thread stacks, code
  cache, direct buffers (Netty, and the Kafka client use these), and GC structures.
- `-XX:+ExitOnOutOfMemoryError` makes the process die on OOM so the platform restarts it,
  rather than limping on in a broken state.
- If the container is killed with exit code 137 (OOMKilled) while heap looks fine, the
  non-heap part grew: lower the percentage or raise the limit. Native Memory Tracking
  (`-XX:NativeMemoryTracking=summary`, then `jcmd <pid> VM.native_memory`) shows where.

Set CPU requests realistically too: the JVM sizes GC threads and the common fork-join pool
from the CPUs it sees.

## Graceful shutdown

When the platform stops a container, it sends `SIGTERM`, waits a grace period, then sends
`SIGKILL`. Use the grace period:

```yaml
server:
  shutdown: graceful          # the default since Boot 3.4
spring:
  lifecycle:
    timeout-per-shutdown-phase: 20s
```

On `SIGTERM`, readiness flips to refusing traffic, the web server stops accepting new
requests and waits up to the timeout for in-flight ones, then the context closes (Level 3,
lesson 09 showed the exact order). Make sure the platform's grace period is longer than
Boot's timeout. Use the exec form of `ENTRYPOINT` (JSON array) so Java is PID 1 and
actually receives the signal — the shell form (`ENTRYPOINT java -jar …`) runs a shell that
may not forward it.

## Worked example: Docker Compose for local development

```yaml
# compose.yaml
services:
  db:
    image: postgres:17
    environment:
      POSTGRES_DB: library
      POSTGRES_USER: library
      POSTGRES_PASSWORD: devpass
    ports: ["5432:5432"]
  app:
    build: .
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/library
      SPRING_DATASOURCE_USERNAME: library
      SPRING_DATASOURCE_PASSWORD: devpass
    ports: ["8080:8080"]
    depends_on: [db]
```

Boot also has Docker Compose support (`spring-boot-docker-compose` as a development-only
dependency): when you run the app from your IDE it starts the services in `compose.yaml`
and wires connection details automatically via service connections, the same mechanism
Testcontainers' `@ServiceConnection` uses.

## How It Actually Works

**Layers.** A container image is a stack of filesystem layers identified by content hash.
Rebuilding reuses any layer whose inputs are unchanged, and registries transfer only
missing layers. Boot's `BOOT-INF/layers.idx` assigns each jar entry to a layer
(`dependencies`, `spring-boot-loader`, `snapshot-dependencies`, `application`); the `tools`
jar mode reads it to write the directories the Dockerfile copies. Running
`JarLauncher` against the exploded directory, instead of `java -jar` on a fat jar, also
avoids reading nested jars and starts slightly faster.

**Container awareness.** The JVM reads cgroup v1/v2 files (`memory.max`, `cpu.max`) at
startup to compute available memory and processors, and bases its ergonomic defaults (heap
size, GC choice, thread counts) on them instead of the host's resources.

## Common mistakes

- **Running as root** inside the container.
- **Fixed `-Xmx` equal to the container limit**, leaving no room for non-heap memory.
- **Shell-form ENTRYPOINT**, so `SIGTERM` never reaches the JVM.
- **Fat single-layer images** that re-upload all dependencies on every build.
- **`latest` tags in deployments.** Use immutable version tags or digests.

## Exercise

1. Write the layered Dockerfile for the library API, build it twice with a one-line code
   change in between, and note which layers were rebuilt.
2. Run the container with `--memory=512m` and check the max heap with
   `docker exec <id> jcmd 1 VM.flags | tr ' ' '\n' | grep MaxHeapSize`, with and without
   `MaxRAMPercentage`.
3. Add a slow endpoint (10 s), call it, and `docker stop` the container during the call.
   Confirm the request completes with graceful shutdown enabled, and is cut off with
   `server.shutdown: immediate`.
4. Build the same app with `spring-boot:build-image` and compare image size and user with
   your Dockerfile's image.
