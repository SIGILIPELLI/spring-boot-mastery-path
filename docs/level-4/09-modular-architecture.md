---
description: "Structure large Spring Boot apps as modular monoliths: feature packages, internal APIs, events between modules, and boundary checks with Spring Modulith."
---

# Architecture: Modular Monoliths & Boundaries

Most Spring Boot applications do not need to be microservices. They need to not become a
ball of mud — where every class can call every other class and a change to orders breaks
invoicing. A **modular monolith** gives you explicit boundaries inside one deployable, and
Spring Modulith lets you verify them in a test.

## Package by feature, with a public face

```text
com.example.shop
├── ShopApplication.java
├── catalog/                     ← module
│   ├── CatalogApi.java          public: what other modules may call
│   ├── ProductView.java         public DTO
│   └── internal/                everything else
│       ├── Product.java  ProductRepository.java  CatalogService.java
├── orders/
│   ├── OrderApi.java  OrderPlaced.java (public event)
│   └── internal/ ...
└── invoicing/
    └── internal/ InvoiceListener.java ...
```

Rules:

1. A module is a direct sub-package of the application's root package.
2. Other modules may use only the module's top-level package (its API types and events), never
   `internal`.
3. No cycles between modules.
4. Modules own their tables; no module queries another's tables.

Java's package-private visibility helps (make internal classes package-private where you
can), but it cannot express "visible to my sub-packages, not to others." That is what the
verification test is for.

## Communicating between modules

Direct calls through the API are fine for queries:

```java
// orders module
ProductView product = catalog.findProduct(sku).orElseThrow();
```

For "something happened" — prefer events, so the publisher does not depend on every
consumer:

```java
// orders/internal/OrderService.java
@Transactional
public void place(PlaceOrder cmd) {
    Order order = orders.save(Order.from(cmd));
    events.publishEvent(new OrderPlaced(order.getId(), order.total()));
}

// invoicing/internal/InvoiceListener.java
@ApplicationModuleListener
void on(OrderPlaced event) {
    invoices.createFor(event.orderId(), event.total());
}
```

`@ApplicationModuleListener` (Spring Modulith) is shorthand for an async,
transactional-after-commit listener running in its own transaction — the combination you
usually want between modules.

## Spring Modulith

Add Spring Modulith (Initializr: "Spring Modulith"). Then one test verifies the structure:

```java
class ModularityTest {
    ApplicationModules modules = ApplicationModules.of(ShopApplication.class);

    @Test
    void verifiesModularStructure() {
        modules.verify();   // fails on cycles and on access to another module's internals
    }

    @Test
    void writesDocumentation() {
        new Documenter(modules).writeModulesAsPlantUml().writeIndividualModulesAsPlantUml();
    }
}
```

If `invoicing` imports `orders.internal.Order`, `verify()` fails with a message naming the
offending type and dependency. Modulith also offers:

- **`@ApplicationModuleTest`** — bootstraps only one module (and optionally its dependencies),
  like a slice test for your own architecture.
- **Event publication registry** — persists events published to module listeners in a
  database table and marks them complete when listeners succeed, so events survive crashes
  and can be resubmitted. This removes most of the need for a hand-written outbox within a
  monolith.
- **Event externalization** — publish selected events to Kafka or other brokers
  (`@Externalized`), backed by the same registry.

The Modulith examples here were not run for this course; check the Spring Modulith
documentation for the version matching your Boot release.

## Worked example: extracting a module later

Because `invoicing` only depends on the `OrderPlaced` event, extracting it into its own
service is mechanical:

1. Externalize `OrderPlaced` to a Kafka topic.
2. Move the `invoicing` package into a new Boot app with its own database (its tables were
   already private to it).
3. Replace `@ApplicationModuleListener` with `@KafkaListener` on the same method; make it
   idempotent (Level 3).
4. Delete the package from the monolith.

Compare that with extracting invoicing from a codebase where it joins `orders` tables and
calls `OrderRepository` directly — that is a rewrite, not an extraction.

## How It Actually Works

`ApplicationModules.of(...)` uses **ArchUnit** to import the compiled classes of your
application and builds a model: each direct sub-package of the main class's package is a
module; its base package is the API (or you can declare named interfaces with
`@NamedInterface`); everything in sub-packages is internal. `verify()` then walks every type
dependency — field types, method signatures, constructor parameters, annotations — and
checks that each cross-module dependency targets an exposed type and that the module
dependency graph has no cycles.

The event publication registry hooks into Spring's event multicaster: when an event is
published inside a transaction, Modulith writes one publication row per target listener in
the *same* transaction. After commit, listeners run; each successful completion marks its
row completed. Rows that never complete (the app crashed, the listener threw) remain
incomplete and can be republished on restart or by a scheduled job — the outbox pattern,
provided by the framework.

## Common mistakes

- **Package by layer** (`controllers`, `services`, `repositories`), which makes every module
  boundary invisible.
- **Shared "common" module** that grows to contain half the domain.
- **Modules reading each other's tables** "just for a report."
- **Synchronous calls for everything**, recreating tight coupling inside one process.
- **Splitting into microservices before the module boundaries are stable.**

## Exercise

1. Restructure the capstone code into `catalog`, `orders`, and `invoicing` modules with
   `internal` sub-packages.
2. Add Spring Modulith and the `verify()` test. Introduce a forbidden dependency on purpose and
   read the failure.
3. Replace a direct call from orders to invoicing with an `OrderPlaced` event and an
   `@ApplicationModuleListener`.
4. Enable the JPA event publication registry, make the listener throw, restart the app, and
   observe the incomplete publication being retried (per the Modulith docs for your version).
