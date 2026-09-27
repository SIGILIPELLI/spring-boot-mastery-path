---
title: "Free Spring Boot Course: REST APIs, JPA, Security, Testing & Production Services"
description: "Free Spring Boot course — 40 lessons on dependency injection, REST controllers, Spring Data JPA, security, testing, Kafka, observability, and deployment."
---

# Spring Boot Mastery Path


<div class="mp-hero-lockup">
  <div class="mp-hero-globe">
    <svg viewBox="0 0 160 160" class="mp-globe-svg mp-mark">
      <circle cx="80" cy="80" r="74" fill="var(--md-default-bg-color, #FAF7EF)" stroke="#2FA98D" stroke-width="5"/>
      <rect class="mp-bar mp-bar-1" x="36" y="90" width="17" height="24" rx="5" fill="#2FA98D"/>
      <rect class="mp-bar mp-bar-2" x="59" y="74" width="17" height="40" rx="5" fill="#7FC9B7"/>
      <rect class="mp-bar mp-bar-3" x="82" y="56" width="17" height="58" rx="5" fill="#E0B457"/>
      <rect class="mp-bar mp-bar-4" x="105" y="38" width="17" height="76" rx="5" fill="#C97B4A"/>
      <circle class="mp-dot" cx="44.5" cy="82" r="7" fill="#22261f"/>
    </svg>
  </div>
  <div class="mp-hero-divider"></div>
  <div class="mp-hero-text">
    <div class="mp-hero-title">Mastery Path</div>
    <div class="mp-hero-sub">BOOTCAMP</div>
    <div class="mp-hero-rule"></div>
    <div class="mp-hero-byline">VINOD BHANUJA</div>
  </div>
</div>

Spring Boot is how a very large share of the Java world builds backend services. Under
the hood it is the Spring Framework — an inversion-of-control container that creates your
objects, wires them together, and wraps them in proxies that add transactions, security,
and caching — plus an opinionated layer that configures all of that for you based on what
is on your classpath. Put `spring-boot-starter-webmvc` in your build and you get an
embedded Tomcat, JSON serialization, and a dispatcher servlet without writing a line of
setup. That convenience is also the main source of confusion: when something behaves
unexpectedly, the answer is usually in a bean, proxy, or auto-configuration you never saw.

This course teaches you to build real services with Spring Boot *and* to see through the
magic. It assumes you already know Java — classes, interfaces, generics, lambdas, records,
exceptions, and a build tool at a basic level. If you do not, start with the
[Java Mastery Path](https://sigilipelli.github.io/java-mastery-path/), which covers the
language itself; here we spend no time on Java syntax.

Level 1 builds a REST API with validation, error responses in the standard
`application/problem+json` format, typed configuration, and logging. Level 2 adds a real
database through Spring Data JPA and Hibernate, Flyway migrations, transactions, a proper
test suite, and Spring Security with JWTs. Level 3 opens the machinery — how
auto-configuration decides what to create, how AOP proxies implement `@Transactional` and
`@Cacheable`, why lazy loading causes N+1 queries — and adds caching, async work, Kafka
messaging, resilient HTTP clients, and Actuator. Level 4 is about running services for
real: microservices and Spring Cloud, WebFlux versus virtual threads, performance, native
images, containers, tracing, hardening, deployment, and a capstone.

Every lesson ends with a **How It Actually Works** section that explains the mechanism
rather than just the annotation: the proxy behind `@Transactional` and why calling a
transactional method from the same class silently skips the transaction, the
`@Conditional` checks that decide which auto-configurations apply, the bean lifecycle
from definition to destruction, and Hibernate's persistence context with its dirty
checking.

!!! note "About versions and outputs"
    Code in this course targets **Spring Boot 4.x** (the line Spring Initializr generates
    today, built on Spring Framework 7) with **Java 17 or newer** — Java 21 or 25 is
    recommended so you can use virtual threads. Boot 3.x is still widespread in existing
    codebases and nearly everything here applies to it too; where Boot 4 changed a name
    or package (for example the `spring-boot-starter-webmvc` starter or the test-slice
    packages), the lesson points out the 3.x equivalent. Builds use Maven; Gradle
    equivalents are shown where they differ in an interesting way. Console output shown in
    lessons was produced by actually running the code; where something needs an external
    service (PostgreSQL, Kafka, an identity provider, a cloud platform) and it was not run
    for this course, the lesson says so rather than inventing output.

## How the program is organized

| Level | Focus | Modules |
|-------|-------|---------|
| [Level 1 · Entry](level-1/index.md) | What Spring is, Initializr & project layout, dependency injection, beans & scanning, configuration & profiles, REST controllers, validation, error handling, logging | 9 topics + 1 project (task tracker API) |
| [Level 2 · Intermediate](level-2/index.md) | Spring Data JPA & Hibernate, entities & relationships, repositories & queries, transactions, Flyway, testing, Spring Security, JWT resource server, OpenAPI docs | 9 topics + 1 project (secured library API) |
| [Level 3 · Advanced](level-3/index.md) | Auto-configuration internals, AOP & proxies, N+1 & fetch strategies, caching, async & scheduling, Kafka, HTTP clients & resilience, Actuator, bean lifecycle & events | 9 topics + 1 project (event-driven order service) |
| [Level 4 · Master](level-4/index.md) | Spring Cloud & microservices, WebFlux vs virtual threads, performance, GraalVM native, containers, observability, security hardening, deployment, modular architecture | 9 topics + 1 capstone (production-ready service) |

## How to use this site

- Work through the levels in order. Level 3 explains *why* the annotations you used in
  Levels 1 and 2 behave the way they do, and it lands much better once you have been
  surprised by them at least once.
- **Build the code.** Generate a project from [start.spring.io](https://start.spring.io/),
  paste the classes in, and run `./mvnw spring-boot:run` or `./mvnw test`. Break things on
  purpose — remove an annotation, rename a property — and read the startup failure.
- Do the **Exercise** at the end of each lesson. Each project (module 10) grows out of the
  exercises before it.
- Keep one repository per level. By the end you will have a validated REST API, a secured
  database-backed service with tests, an event-driven service using Kafka, and a
  containerized, observable capstone.
- For the language itself, see the
  [Java Mastery Path](https://sigilipelli.github.io/java-mastery-path/); for testing in
  depth, [Java Testing](https://sigilipelli.github.io/java-testing-mastery-path/); for
  the architecture side, the
  [System Design Mastery Path](https://sigilipelli.github.io/system-design-mastery-path/).

Start here → [Level 1 · Entry](level-1/index.md)

🎥 **Prefer video?** Watch the [Mastery Path video series](https://youtube.com/@sigilipelli) on YouTube — Shorts and full walkthroughs of lessons across the series.

## More from the Mastery Path series

Free, structured, module-wise training across 68 other languages, platforms and disciplines:

<div class="mastery-grid-wrap">
<p class="mastery-grid-category">Languages</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/python-mastery-path/">🐍 Python</a>
  <a href="https://sigilipelli.github.io/java-mastery-path/">☕ Java</a>
  <a href="https://sigilipelli.github.io/javascript-mastery-path/">🟨 JavaScript</a>
  <a href="https://sigilipelli.github.io/typescript-mastery-path/">🔷 TypeScript</a>
  <a href="https://sigilipelli.github.io/csharp-mastery-path/">🔵 C#</a>
  <a href="https://sigilipelli.github.io/shell-mastery-path/">🐚 Shell/Bash</a>
  <a href="https://sigilipelli.github.io/powershell-mastery-path/">💻 PowerShell</a>
  <a href="https://sigilipelli.github.io/c-mastery-path/">🇨 C</a>
  <a href="https://sigilipelli.github.io/cpp-mastery-path/">➕ C++</a>
  <a href="https://sigilipelli.github.io/go-mastery-path/">🐹 Go</a>
  <a href="https://sigilipelli.github.io/rust-mastery-path/">🦀 Rust</a>
  <a href="https://sigilipelli.github.io/sql-mastery-path/">🗄️ SQL</a>
  <a href="https://sigilipelli.github.io/ruby-mastery-path/">💎 Ruby</a>
  <a href="https://sigilipelli.github.io/php-mastery-path/">🐘 PHP</a>
  <a href="https://sigilipelli.github.io/kotlin-mastery-path/">🟣 Kotlin</a>
  <a href="https://sigilipelli.github.io/swift-mastery-path/">🐦 Swift</a>
  <a href="https://sigilipelli.github.io/dart-mastery-path/">🎯 Dart</a>
  <a href="https://sigilipelli.github.io/scala-mastery-path/">🔴 Scala</a>
  <a href="https://sigilipelli.github.io/r-mastery-path/">📊 R</a>
  <a href="https://sigilipelli.github.io/matlab-mastery-path/">🟧 MATLAB</a>
</div>
<p class="mastery-grid-category">Web Frameworks</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/react-mastery-path/">⚛️ React</a>
  <a href="https://sigilipelli.github.io/nodejs-mastery-path/">🟩 Node.js</a>
  <a href="https://sigilipelli.github.io/spring-boot-mastery-path/">🍃 Spring Boot</a>
</div>
<p class="mastery-grid-category">Testing & QA</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/java-testing-mastery-path/">🧪 Java Testing</a>
  <a href="https://sigilipelli.github.io/cpp-testing-mastery-path/">🧪 C/C++ Testing</a>
  <a href="https://sigilipelli.github.io/python-testing-mastery-path/">🧪 Python Testing</a>
  <a href="https://sigilipelli.github.io/automotive-testing-mastery-path/">🚗 Automotive Testing</a>
</div>
<p class="mastery-grid-category">Security</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/cybersecurity-mastery-path/">🛡️ Cybersecurity</a>
</div>
<p class="mastery-grid-category">Cloud Platforms</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/aws-mastery-path/">☁️ AWS</a>
  <a href="https://sigilipelli.github.io/azure-mastery-path/">☁️ Azure</a>
  <a href="https://sigilipelli.github.io/gcp-mastery-path/">☁️ GCP</a>
  <a href="https://sigilipelli.github.io/ibm-cloud-mastery-path/">☁️ IBM Cloud</a>
  <a href="https://sigilipelli.github.io/adobe-mastery-path/">🎨 Adobe</a>
</div>
<p class="mastery-grid-category">Data & Analytics</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/data-engineering-mastery-path/">🛠️ Data Engineering</a>
  <a href="https://sigilipelli.github.io/data-science-mastery-path/">📈 Data Science</a>
  <a href="https://sigilipelli.github.io/tableau-mastery-path/">📊 Tableau</a>
  <a href="https://sigilipelli.github.io/excel-mastery-path/">📗 Excel</a>
  <a href="https://sigilipelli.github.io/pyspark-mastery-path/">⚡ PySpark</a>
  <a href="https://sigilipelli.github.io/etl-datalake-mastery-path/">🏞️ ETL & Data Lake</a>
</div>
<p class="mastery-grid-category">AI / ML / LLM</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/ai-ml-mastery-path/">🤖 AI/ML</a>
  <a href="https://sigilipelli.github.io/llm-dev-mastery-path/">🧠 LLM Dev</a>
  <a href="https://sigilipelli.github.io/rag-mastery-path/">📚 RAG</a>
  <a href="https://sigilipelli.github.io/edge-ai-mastery-path/">📱 Edge AI</a>
  <a href="https://sigilipelli.github.io/claude-training-mastery-path/">🔶 Claude Training</a>
  <a href="https://sigilipelli.github.io/ai-tools-mastery-path/">🧰 AI Tools</a>
  <a href="https://sigilipelli.github.io/ml-math-mastery-path/">➗ ML Math Foundations</a>
</div>
<p class="mastery-grid-category">Embedded Systems</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/embedded-mastery-path/">🔌 Embedded</a>
  <a href="https://sigilipelli.github.io/embedded-linux-mastery-path/">🐧 Embedded Linux</a>
  <a href="https://sigilipelli.github.io/embedded-python-mastery-path/">🐍 Embedded Python</a>
  <a href="https://sigilipelli.github.io/freertos-mastery-path/">⏱️ FreeRTOS</a>
  <a href="https://sigilipelli.github.io/s32k-mastery-path/">🔧 S32K</a>
</div>
<p class="mastery-grid-category">Leadership & Management</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/product-manager-mastery-path/">📋 Product Manager</a>
  <a href="https://sigilipelli.github.io/product-lead-mastery-path/">🧭 Product Lead</a>
  <a href="https://sigilipelli.github.io/project-manager-mastery-path/">📅 Project Manager</a>
  <a href="https://sigilipelli.github.io/ai-manager-mastery-path/">🤖 AI Manager</a>
  <a href="https://sigilipelli.github.io/servant-leadership-mastery-path/">🤝 Servant Leadership</a>
</div>
<p class="mastery-grid-category">Professional Skills</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/english-fluency-mastery-path/">🗣️ English Fluency & IELTS</a>
  <a href="https://sigilipelli.github.io/workday-mastery-path/">🧑‍💼 Workday</a>
</div>
<p class="mastery-grid-category">Careers & Interviews</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/dsa-interview-mastery-path/">🧩 DSA & Coding Interviews</a>
  <a href="https://sigilipelli.github.io/system-design-mastery-path/">🏗️ System Design</a>
</div>
<p class="mastery-grid-category">Process & APIs</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/agile-mastery-path/">🔄 Agile/Scrum/Kanban</a>
  <a href="https://sigilipelli.github.io/rest-api-mastery-path/">🔗 REST API</a>
  <a href="https://sigilipelli.github.io/playwright-mastery-path/">🎭 Playwright</a>
</div>
<p class="mastery-grid-category">Infrastructure & Ops</p>
<div class="mastery-grid">
  <a href="https://sigilipelli.github.io/server-ops-mastery-path/">🖥️ Server Ops</a>
  <a href="https://sigilipelli.github.io/nodemcu-mastery-path/">📶 NodeMCU/IoT</a>
  <a href="https://sigilipelli.github.io/terraform-mastery-path/">🌍 Terraform</a>
  <a href="https://sigilipelli.github.io/kubernetes-mastery-path/">☸️ Kubernetes</a>
  <a href="https://sigilipelli.github.io/docker-mastery-path/">🐳 Docker</a>
  <a href="https://sigilipelli.github.io/github-mastery-path/">🐙 GitHub & Git</a>
</div>
</div>
