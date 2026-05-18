# Java Exceptions Deep Dive

---

## 1. The Exception Hierarchy

Everything in Java's exception system lives under `Throwable`.

```
Throwable
├── Error                          ← JVM-level, don't catch
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── AssertionError
└── Exception
    ├── RuntimeException           ← Unchecked
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   ├── IllegalStateException
    │   ├── IndexOutOfBoundsException
    │   ├── ClassCastException
    │   └── ArithmeticException
    └── (Everything else)          ← Checked
        ├── IOException
        ├── SQLException
        └── InterruptedException
```

**Checked** — compiler forces you to handle or declare them (`throws`).  
**Unchecked** — `RuntimeException` and subclasses; programmer errors, not recoverable conditions.  
**Error** — JVM failures; never catch unless you have a very specific reason (e.g., catching `Throwable` in a framework shutdown hook).

---

## 2. Checked vs Unchecked — The Real Distinction

The design intent: checked exceptions model **recoverable external failures** (file not found, network timeout). Unchecked model **programmer bugs** (null dereference, bad index).

In practice, Spring and most modern frameworks converted almost everything to unchecked (`DataAccessException`, etc.) because checked exceptions break lambda/stream pipelines and cause API pollution.

**When to use which:**

| Scenario | Use |
|---|---|
| Invalid method argument from caller | `IllegalArgumentException` (unchecked) |
| Object in wrong state | `IllegalStateException` (unchecked) |
| File/network I/O failure | `IOException` (checked) |
| DB failure in a library | Unchecked wrapper (like Spring does) |
| Business rule violation | Custom unchecked (e.g., `InsufficientFundsException`) |

---

## 3. try-catch-finally — Internals

```java
try {
    riskyOperation();
} catch (IOException e) {
    handle(e);
} catch (SQLException e) {
    handle(e);
} finally {
    cleanup(); // ALWAYS runs — even on return, even on exception
}
```

**finally quirks you must know:**

```java
// What does this return?
int foo() {
    try {
        return 1;
    } finally {
        return 2; // finally overrides the try's return — returns 2
    }
}

// What happens here?
void bar() {
    try {
        throw new RuntimeException("try");
    } finally {
        throw new RuntimeException("finally"); // "try" exception is LOST
    }
}
```

`finally` suppresses the original exception if it throws its own. This is a classic bug. Use try-with-resources instead.

---

## 4. try-with-resources (TWR)

Introduced in Java 7. Any class implementing `AutoCloseable` can be used.

```java
try (
    Connection conn = dataSource.getConnection();
    PreparedStatement ps = conn.prepareStatement(sql)
) {
    ps.executeQuery();
} catch (SQLException e) {
    throw new DataAccessException("Query failed", e);
}
// close() called on ps first, then conn — reverse declaration order
```

**Suppressed exceptions** — this is the key thing TWR solves:

```java
// TWR internally does this:
Throwable primaryException = null;
try {
    // body
} catch (Throwable t) {
    primaryException = t;
    throw t;
} finally {
    if (primaryException != null) {
        try { resource.close(); }
        catch (Throwable suppressed) {
            primaryException.addSuppressed(suppressed); // doesn't lose close() exception
        }
    } else {
        resource.close();
    }
}
```

You can retrieve suppressed exceptions:

```java
catch (Exception e) {
    for (Throwable s : e.getSuppressed()) {
        log.warn("Suppressed: {}", s.getMessage());
    }
}
```

**Java 9 enhancement** — effectively final variables work in TWR:

```java
var conn = dataSource.getConnection(); // declared outside
try (conn) { // just reference it
    // ...
}
```

---

## 5. Multi-catch and Exception Union Types

```java
catch (IOException | SQLException e) {
    // e is implicitly final here
    log.error("Failure", e);
}
```

The type of `e` in a multi-catch is the **least upper bound** (LUB) of the listed types. If `IOException` and `SQLException` both extend `Exception`, `e` is typed as `Exception`. The compiler enforces that you can't reassign `e` because that would break type safety.

---

## 6. Exception Chaining (Cause Chain)

Always preserve the original exception when wrapping:

```java
// WRONG — loses stack trace
catch (SQLException e) {
    throw new ServiceException("DB error"); // e is gone
}

// RIGHT
catch (SQLException e) {
    throw new ServiceException("DB error", e); // cause chain preserved
}
```

Traversing the chain:

```java
Throwable cause = e;
while (cause.getCause() != null) {
    cause = cause.getCause();
}
// cause is now the root exception
```

`Throwable` has four constructors — know them:

```java
new MyException(String message)
new MyException(String message, Throwable cause)
new MyException(Throwable cause)
new MyException(String message, Throwable cause, boolean enableSuppression, boolean writableStackTrace)
```

The 4th is used in performance-sensitive code to disable stack trace generation (expensive).

---

## 7. Custom Exception Design

```java
// Base for your domain
public class ShopSphereException extends RuntimeException {
    private final ErrorCode code;

    public ShopSphereException(ErrorCode code, String message) {
        super(message);
        this.code = code;
    }

    public ShopSphereException(ErrorCode code, String message, Throwable cause) {
        super(message, cause);
        this.code = code;
    }

    public ErrorCode getCode() { return code; }
}

// Specific exceptions
public class OrderNotFoundException extends ShopSphereException {
    public OrderNotFoundException(Long orderId) {
        super(ErrorCode.ORDER_NOT_FOUND, "Order not found: " + orderId);
    }
}

public class InsufficientInventoryException extends ShopSphereException {
    private final int requested;
    private final int available;

    public InsufficientInventoryException(int requested, int available) {
        super(ErrorCode.INSUFFICIENT_INVENTORY,
            String.format("Requested %d but only %d available", requested, available));
        this.requested = requested;
        this.available = available;
    }
    // getters for structured error responses
}
```

**Rules:**
- Extend `RuntimeException` for domain exceptions in Spring apps.
- Always provide a cause constructor.
- Add structured fields (error codes, IDs) — don't encode everything in the message string.
- Don't create one mega-exception with an enum type field — that's just a disguised `if-else` chain. Create distinct classes.

---

## 8. Stack Trace Internals

Stack trace generation happens in `Throwable`'s constructor via `fillInStackTrace()`, which calls a native method to walk the JVM stack. This is expensive — it's the dominant cost of exception creation, not the `throw` itself.

```java
// Reusable sentinel exception — no stack trace overhead
public class NotFoundException extends RuntimeException {
    public static final NotFoundException INSTANCE =
        new NotFoundException("Not found", null, true, false); // writableStackTrace=false
}
```

Used in frameworks like Netty and Reactor for flow control (though using exceptions for flow control is itself an antipattern in application code).

`printStackTrace()` in production is always wrong. Use a logger:

```java
log.error("Operation failed for orderId={}", orderId, e); // pass exception as last arg
```

SLF4J/Logback will format the full stack trace including cause chains.

---

## 9. Exceptions in Streams and Lambdas

Checked exceptions don't work directly in lambdas because functional interfaces don't declare `throws`:

```java
// Doesn't compile — IOException is checked
list.stream()
    .map(path -> Files.readString(path)) // compile error
    .collect(toList());
```

**Option 1 — Wrap in unchecked:**

```java
list.stream()
    .map(path -> {
        try { return Files.readString(path); }
        catch (IOException e) { throw new UncheckedIOException(e); }
    })
    .collect(toList());
```

**Option 2 — Utility wrapper:**

```java
@FunctionalInterface
public interface ThrowingFunction<T, R> {
    R apply(T t) throws Exception;

    static <T, R> Function<T, R> wrap(ThrowingFunction<T, R> f) {
        return t -> {
            try { return f.apply(t); }
            catch (Exception e) { throw new RuntimeException(e); }
        };
    }
}

// Usage
list.stream()
    .map(ThrowingFunction.wrap(Files::readString))
    .collect(toList());
```

**Option 3 — Vavr's `Try` monad** (if you use Vavr):

```java
list.stream()
    .map(path -> Try.of(() -> Files.readString(path)))
    .filter(Try::isSuccess)
    .map(Try::get)
    .collect(toList());
```

---

## 10. Exceptions in Concurrent Code

**CompletableFuture:**

```java
CompletableFuture.supplyAsync(() -> orderService.getOrder(id))
    .thenApply(order -> inventoryService.reserve(order))
    .exceptionally(ex -> {
        // ex is wrapped in CompletionException if thrown from async stage
        Throwable cause = ex.getCause() != null ? ex.getCause() : ex;
        log.error("Pipeline failed", cause);
        return fallback();
    })
    .handle((result, ex) -> {
        // handle() gets both — use this for structured response building
        if (ex != null) return ErrorResponse.of(ex);
        return SuccessResponse.of(result);
    });
```

`exceptionally` only fires on failure. `handle` always fires. Prefer `handle` when you need to map to a response type regardless of outcome.

**ExecutorService:**

```java
Future<Order> future = executor.submit(() -> orderService.process(id));
try {
    Order order = future.get();
} catch (ExecutionException e) {
    // The actual exception is wrapped
    Throwable actual = e.getCause();
    if (actual instanceof OrderNotFoundException onfe) { // pattern matching, Java 16+
        // handle
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt(); // ALWAYS restore interrupt status
    throw new RuntimeException("Interrupted", e);
}
```

**Never swallow `InterruptedException`** — it signals the thread should stop. Either re-interrupt the thread or propagate it.

---

## 11. Spring's Exception Handling

**@ExceptionHandler in a controller:**

```java
@RestController
public class OrderController {

    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(OrderNotFoundException e) {
        return new ErrorResponse(e.getCode(), e.getMessage());
    }
}
```

**@ControllerAdvice — global handler (preferred):**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(OrderNotFoundException e) {
        return ResponseEntity.status(404).body(ErrorResponse.of(e));
    }

    @ExceptionHandler(InsufficientInventoryException.class)
    public ResponseEntity<ErrorResponse> handleInventory(InsufficientInventoryException e) {
        return ResponseEntity.status(409).body(ErrorResponse.of(e));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException e) {
        List<String> errors = e.getBindingResult().getFieldErrors()
            .stream().map(FieldError::getDefaultMessage).toList();
        return ResponseEntity.status(400).body(ErrorResponse.ofValidation(errors));
    }

    @ExceptionHandler(Exception.class) // catch-all
    public ResponseEntity<ErrorResponse> handleGeneric(Exception e) {
        log.error("Unhandled exception", e);
        return ResponseEntity.status(500).body(ErrorResponse.internal());
    }
}
```

**Ordering matters** — Spring matches the most specific handler first. The `Exception.class` handler is always the last resort.

**Spring Data exceptions** — Spring wraps all JDBC/JPA exceptions into its `DataAccessException` hierarchy (unchecked). You never catch `SQLException` in a Spring app directly.

```
DataAccessException
├── TransientDataAccessException   ← retry-able (deadlock, timeout)
│   ├── QueryTimeoutException
│   └── TransientDataAccessResourceException
└── NonTransientDataAccessException ← not retry-able (constraint violation)
    ├── DataIntegrityViolationException
    └── EmptyResultDataAccessException
```

---

## 12. Exception Performance

Key points for interviews:

- **Exception creation is expensive** because of `fillInStackTrace()` — native stack walk.
- **throw/catch itself is cheap** — it's just a jump.
- Never use exceptions for normal control flow (e.g., use `containsKey` before `get`, not try-catch around `get`).
- In tight loops, pre-create exceptions or disable stack traces if you must use them as signals.
- JVM can optimize away exception overhead via escape analysis in some cases, but never rely on it.

Benchmark order of magnitude: creating an exception with a 20-frame stack trace is ~1–10 µs. In a hot path handling thousands of requests/second, this matters.

---

## 13. Java 21 Additions — Virtual Threads and Exceptions

With Project Loom, `InterruptedException` semantics shift. Virtual threads support `Thread.interrupt()` but blocking operations on virtual threads throw `InterruptedException` more readily (on I/O, locks, etc.).

Pattern:

```java
Thread.ofVirtual().start(() -> {
    try {
        SomeBlockingOp.run();
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
        // clean up
    }
});
```

Same rule applies: never swallow `InterruptedException`.

---

## 14. Interview Questions — All Levels

**Core theory**

**Q: What's the difference between `throw` and `throws`?**  
`throw` is the statement that actually throws an exception instance. `throws` is a declaration on a method signature saying it may propagate a checked exception. `throw new IOException()` vs `void read() throws IOException`.

**Q: Can you catch an Error?**  
Technically yes — `Error` extends `Throwable`. But you almost never should. The only common legitimate case is catching `OutOfMemoryError` in frameworks to trigger a graceful shutdown. Never catch it in application code to "retry" — the heap is exhausted.

**Q: What happens if both catch and finally throw an exception?**  
The finally exception propagates and the catch exception is lost. This is why you should never throw from finally.

**Q: Is `NullPointerException` checked or unchecked?**  
Unchecked — it's a `RuntimeException`. Since Java 14, the JVM gives helpful NullPointerException messages telling you exactly which variable was null.

**Q: What does `fillInStackTrace()` do and when would you override it?**  
It's called in `Throwable`'s constructor to capture the current stack. You'd override it (return `this` without calling super) to create a zero-cost exception for flow control signals — same as passing `writableStackTrace=false` to the protected constructor.

---

**Intermediate**

**Q: What are suppressed exceptions and how do they arise?**  
When try-with-resources closes a resource and `close()` throws while an exception is already in flight from the try block, the close exception is added to the primary exception as a suppressed exception via `addSuppressed()`. You retrieve them with `getSuppressed()`.

**Q: Why can't you catch checked exceptions in lambdas?**  
Because standard functional interfaces (`Function`, `Predicate`, etc.) don't declare `throws`. The compiler requires that any checked exception thrown inside a lambda must be handled or declared, and the enclosing functional interface's method doesn't declare them.

**Q: What's the difference between `exceptionally` and `handle` in CompletableFuture?**  
`exceptionally` is a recovery function that only runs on failure and returns the same type. `handle` runs always — on both success and failure — and takes a `BiFunction<T, Throwable, U>`, making it suitable for mapping to a response type regardless of outcome.

**Q: How does Spring's `DataAccessException` hierarchy help?**  
It unifies all database-specific exceptions (JDBC `SQLException`, JPA `PersistenceException`, etc.) into a common unchecked hierarchy, removing the need to catch vendor-specific exceptions. It also distinguishes transient failures (retry-able) from non-transient ones.

---

**Advanced**

**Q: What's the performance cost of exceptions and how would you handle it in a high-throughput system?**  
Stack trace generation via the native `fillInStackTrace()` call is the bottleneck — roughly 1–10 µs per exception depending on stack depth. In a system processing 100K req/s this can be significant if exceptions appear on hot paths. Mitigations: (1) avoid exceptions for expected conditions, (2) pre-create and reuse sentinel exceptions with `writableStackTrace=false`, (3) use `Optional` or result types for expected absence.

**Q: You have a microservice that wraps third-party HTTP calls. How do you design the exception hierarchy?**  

```
ServiceException (base, unchecked)
├── ExternalServiceException       ← 3rd party failure
│   ├── ServiceUnavailableException (503/network) ← retry-able
│   └── ServiceTimeoutException (408/timeout)     ← retry-able
└── DomainException                ← business rules
    ├── ResourceNotFoundException
    └── ValidationException
```

Transient exceptions get caught by Resilience4j/retry mechanisms. Non-transient bubble to the global handler. Never expose internal exception types across service boundaries — translate at the anti-corruption layer.

**Q: How do virtual threads change exception handling patterns?**  
Virtual threads make blocking acceptable again, so `InterruptedException` from blocking I/O is more common. The same rule applies — restore interrupt status or propagate. The main change is that thread-pool-based `ExecutionException` unwrapping patterns are less common; with virtual threads you often just call the blocking method directly in a structured concurrency scope, and exceptions propagate naturally.

**Q: In a Saga pattern (which you've implemented in ShopSphere), how do you handle compensation when a compensating transaction itself fails?**  
You need a persistent failure log. The orchestrator marks the saga as "compensation-failed" and a background job retries with exponential backoff. If compensation is truly un-retryable, it becomes a manual intervention case tracked in a dead-letter queue. The key design decision: compensating actions must be idempotent so retry is safe.

---

## 15. Common Antipatterns to Call Out in Interviews

```java
// 1. Swallowing exceptions
catch (Exception e) { } // silent failure — worst possible thing

// 2. Catching Exception too broadly in business logic
catch (Exception e) { return null; } // hides bugs

// 3. Exception as flow control
try { Integer.parseInt(s); return true; }
catch (NumberFormatException e) { return false; } // use regex or Apache Commons

// 4. Losing the cause
catch (SQLException e) { throw new ServiceException("DB error"); } // cause gone

// 5. Logging and rethrowing (double logging)
catch (Exception e) {
    log.error("Error", e);    // logged here
    throw e;                  // and again at the caller — duplicates in logs
}

// 6. Using getMessage() as the only info in a rethrow
throw new ServiceException(e.getMessage()); // loses type, stack trace, and cause

// 7. Not restoring interrupt status
catch (InterruptedException e) { /* do nothing */ } // breaks thread cancellation
```

---

## Summary

| Topic | Key Point |
|---|---|
| Hierarchy | `Throwable → Error / Exception → RuntimeException` |
| Checked vs Unchecked | Checked = recoverable external; Unchecked = programmer bugs |
| finally | Always runs; never throw from it; suppresses try's exception |
| TWR | Solves suppressed exceptions; closes in reverse order |
| Chaining | Always pass cause; never drop it |
| Custom exceptions | Unchecked in Spring; add error codes; separate classes per condition |
| Performance | Creation expensive (stack walk); throw/catch cheap |
| Streams | Wrap checked in unchecked or use `ThrowingFunction` wrapper |
| Concurrency | Always restore interrupt status; unwrap `ExecutionException` |
| Spring | `@RestControllerAdvice`; `DataAccessException` hierarchy; transient vs non-transient |
