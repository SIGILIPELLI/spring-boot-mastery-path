---
description: "Use @Transactional correctly: where boundaries belong, propagation, rollback rules, read-only transactions, and the proxy self-invocation pitfall."
---

# Transactions with @Transactional

A transaction makes a group of database operations atomic: all of them happen, or none
do. Spring lets you declare transaction boundaries with one annotation. The annotation is
simple; understanding *where* it takes effect is what prevents production bugs.

## Where the boundary belongs

Put `@Transactional` on **service** methods that represent one business operation:

```java
@Service
public class LoanService {
    private final BookRepository books;
    private final LoanRepository loans;
    private final MemberRepository members;

    // constructor...

    @Transactional
    public Loan lend(long bookId, long memberId) {
        Book book = books.findById(bookId).orElseThrow(() -> new BookNotFoundException(bookId));
        Member member = members.findById(memberId).orElseThrow();
        if (loans.existsByBookIdAndReturnedAtIsNull(bookId)) {
            throw new BookAlreadyLentException(bookId);
        }
        if (member.activeLoans() >= member.loanLimit()) {
            throw new LoanLimitReachedException(memberId);
        }
        member.incrementActiveLoans();                       // dirty-checked
        return loans.save(new Loan(book, member));
    }
}
```

The checks and the writes happen in one transaction on one connection. If saving the
loan fails, the member's counter change is rolled back too.

Controllers should not be transactional (the transaction would span JSON serialization
and response writing), and repositories already are, per call — which is not enough when
one operation spans several calls.

## Read-only transactions

```java
@Transactional(readOnly = true)
public List<BookSummary> catalog(Pageable page) { ... }
```

`readOnly = true` tells Hibernate it can skip dirty checking and snapshots (less memory
and CPU), sets the JDBC connection read-only (some drivers and databases optimize or
route to replicas based on it), and documents intent. A common pattern is
`@Transactional(readOnly = true)` on the class and plain `@Transactional` on the
methods that write.

## Rollback rules

By default Spring rolls back on **unchecked exceptions** (`RuntimeException`, `Error`)
and **commits** on checked exceptions. That surprises people:

```java
@Transactional
public void importCatalog(Path file) throws IOException {
    books.save(...);
    Files.readAllLines(file);   // throws IOException → the save above is COMMITTED
}
```

Either make your domain exceptions unchecked (the common choice in Spring code), or say
what you want: `@Transactional(rollbackFor = IOException.class)`.

Catching an exception inside the transactional method means it never reaches the
interceptor, so no rollback happens — unless the exception came from a nested
transactional call that already marked the transaction rollback-only, in which case the
commit fails with `UnexpectedRollbackException`. If you catch, decide deliberately.

## Propagation

When a transactional method calls another transactional method, `propagation` decides
what happens:

| Propagation | Behavior |
|-------------|----------|
| `REQUIRED` (default) | Join the existing transaction, or start one |
| `REQUIRES_NEW` | Suspend the current transaction and run in a brand-new one (separate connection) |
| `MANDATORY` | Must be called inside a transaction, else error |
| `SUPPORTS` | Join if one exists, otherwise run without |
| `NOT_SUPPORTED` | Suspend any transaction and run without |
| `NESTED` | Savepoint within the current transaction; needs a transaction manager that supports savepoints, so verify it with your setup before relying on it |

`REQUIRES_NEW` is the one you will actually reach for — for an audit record that must be
saved even if the main operation rolls back. Use it sparingly: each one holds a second
connection while the first is suspended, and under load that can exhaust the pool.

## Worked example: the self-invocation trap

This class from the project has one transactional method and one plain method that calls
it:

```java
@Service
public class CatalogService {

    public boolean outerWithoutTx() {
        return innerWithTx();      // calling a method on `this`
    }

    @Transactional
    public boolean innerWithTx() {
        return TransactionSynchronizationManager.isActualTransactionActive();
    }
}
```

A test called both from outside the class. Real output (Boot 4.1):

```text
innerWithTx via proxy: true
innerWithTx via self-invocation: false
proxy class: com.example.library.CatalogService$$SpringCGLIB$$0
```

Calling `innerWithTx()` directly on the injected bean ran it in a transaction. Calling
it *through another method of the same class* did not — the `@Transactional` annotation
was silently ignored. The last line shows why: the object injected into the test is not a
`CatalogService` but a generated subclass.

## How It Actually Works

At startup, a `BeanPostProcessor` (the auto-proxy creator) inspects every bean. When it
finds `@Transactional` on a class or method, it replaces the bean with a **proxy** — a
CGLIB subclass by default in Boot — that wraps the real object. Every other bean receives
the proxy.

When a caller invokes `innerWithTx()` on the proxy, the proxy's `TransactionInterceptor`:

1. Reads the transaction attributes (propagation, isolation, readOnly, rollback rules).
2. Asks the `PlatformTransactionManager` (a `JpaTransactionManager` here) for a
   transaction — which gets a connection from the pool, sets auto-commit off, and binds
   the connection and the `EntityManager` to the current thread via
   `TransactionSynchronizationManager` (a set of `ThreadLocal`s).
3. Calls the real method on the target object.
4. On normal return, flushes the persistence context and commits; on an exception
   matching the rollback rules, rolls back; then unbinds the thread-locals and returns
   the connection.

The critical detail is step 3: *inside* the real object, `this` is the target, not the
proxy. So `outerWithoutTx()` → `this.innerWithTx()` is an ordinary Java call that never
passes through the interceptor. Fixes, in order of preference:

- Move the transactional method to a different bean and inject it.
- Make the outer method the transactional one (usually the right boundary anyway).
- Use `TransactionTemplate` for programmatic control where a boundary must sit inside a
  method.

The same proxy mechanism explains other rules: `@Transactional` has no effect on
`private` methods (a subclass cannot override them), on `final` methods, or on calls made
during the constructor. And because the transaction is bound to a `ThreadLocal`, work
handed to another thread (`@Async`, `CompletableFuture.supplyAsync`) does not run inside
it. Level 3 generalizes all of this to AOP.

## Common mistakes

- **Self-invocation**, as shown. The most common transaction bug in Spring code.
- **`@Transactional` on private methods** — ignored.
- **Checked exceptions expected to roll back** — they do not by default.
- **Long transactions** that call remote HTTP services while holding a database
  connection and row locks. Do remote calls before or after the transaction, or use the
  outbox pattern (Level 3).
- **Importing the wrong annotation.** Prefer `org.springframework.transaction.annotation.Transactional`;
  `jakarta.transaction.Transactional` is also honored but supports fewer options.

## Exercise

1. Reproduce the self-invocation test above in your project, then fix it by moving
   `innerWithTx` into a separate bean.
2. Write a service method that saves a book and then throws a checked exception. Show
   with a test that the book *was* committed. Add `rollbackFor` and show it is not.
3. Implement an `AuditService.record(...)` with `REQUIRES_NEW` and demonstrate that the
   audit row survives when the calling transaction rolls back.
4. Add `@Version` to `Book`, load the same book in two separate transactions, modify and
   save both, and show the second one fails with an optimistic locking exception. Map it
   to 409 in your exception handler.
