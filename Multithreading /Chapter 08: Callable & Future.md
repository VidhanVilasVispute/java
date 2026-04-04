
# Chapter 8: `Callable`, `Future` & Intro to `CompletableFuture`

---

## 8.1 The Problem With `Runnable`

Everything so far used `Runnable`. It has two limitations:

```java
@FunctionalInterface
public interface Runnable {
    void run(); // ← returns void — no result
                // ← cannot throw checked exceptions
}
```

Real tasks need to return results and report failures:

```java
// ❌ Runnable can't do this
new Thread(() -> {
    return calculateTotal(); // compile error — void can't return
    throw new SQLException(); // compile error — must catch checked exceptions
}).start();
```

`Callable` solves both problems.

---

## 8.2 `Callable<V>` — Runnable With a Return Value

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception; // returns V, can throw checked exceptions
}
```

Side by side:

```java
// Runnable — no result, no checked exceptions
Runnable r = () -> System.out.println("done");

// Callable<String> — returns String, can throw
Callable<String> c = () -> {
    Thread.sleep(1000); // checked exception — fine in Callable
    return "ORDER_CONFIRMED";
};

// Callable<Integer> — returns Integer
Callable<Integer> c2 = () -> {
    return calculateTotalItems(); // returns a value
};
```

`Callable` cannot be passed to `new Thread()` directly — it only works with `ExecutorService.submit()`.

---

## 8.3 `Future<V>` — A Promise of a Result

When you `submit()` a `Callable`, you get back a `Future<V>` immediately. The task runs on a pool thread. You collect the result later via `future.get()`.

```
submit(callable)
      │
      ▼
ExecutorService
      │
      ├── returns Future<V> immediately to caller ──► caller continues
      │
      └── runs callable on pool thread
                    │
                    ▼
             result stored in Future
                    │
      caller calls future.get() ──► blocks until result ready
                    │
                    ▼
             result returned to caller
```

---

## 8.4 `Callable` + `Future` — Complete Example

```java
import java.util.concurrent.*;

public class CallableFutureDemo {

    // ─── Callable tasks (simulate ShopSphere service calls) ───────
    static Callable<Integer> countActiveOrders() {
        return () -> {
            System.out.println("[" + Thread.currentThread().getName() + "] Counting orders...");
            Thread.sleep(1000); // simulate DB query
            return 247;
        };
    }

    static Callable<Double> calculateRevenue() {
        return () -> {
            System.out.println("[" + Thread.currentThread().getName() + "] Calculating revenue...");
            Thread.sleep(1500); // simulate analytics query
            return 1_84_329.50;
        };
    }

    static Callable<String> getTopProduct() {
        return () -> {
            System.out.println("[" + Thread.currentThread().getName() + "] Fetching top product...");
            Thread.sleep(800); // simulate cache lookup
            return "Noise Cancelling Headphones";
        };
    }

    public static void main(String[] args) throws InterruptedException {

        ExecutorService pool = Executors.newFixedThreadPool(3);

        long start = System.currentTimeMillis();

        // Submit all three — they run in parallel
        Future<Integer> ordersFuture   = pool.submit(countActiveOrders());
        Future<Double>  revenueFuture  = pool.submit(calculateRevenue());
        Future<String>  productFuture  = pool.submit(getTopProduct());

        System.out.println("[main] All tasks submitted. Building dashboard...\n");

        // Collect results — each get() blocks until that result is ready
        try {
            int    orders  = ordersFuture.get();          // blocks ~1000ms
            double revenue = revenueFuture.get();         // already done (ran in parallel)
            String product = productFuture.get();         // already done

            long elapsed = System.currentTimeMillis() - start;

            System.out.println("╔══════════════════════════════════╗");
            System.out.println("║     ShopSphere Dashboard         ║");
            System.out.println("╠══════════════════════════════════╣");
            System.out.printf( "║  Active Orders  : %-14d ║%n", orders);
            System.out.printf( "║  Revenue Today  : ₹%-13.2f ║%n", revenue);
            System.out.printf( "║  Top Product    : %-14s ║%n", product);
            System.out.println("╚══════════════════════════════════╝");
            System.out.println("\nLoaded in: " + elapsed + "ms (vs ~3300ms sequential)");

        } catch (ExecutionException e) {
            // Exception thrown inside the Callable is wrapped here
            System.err.println("Task failed: " + e.getCause().getMessage());
        }

        pool.shutdown();
    }
}
```

**Output:**
```
[main] All tasks submitted. Building dashboard...

[pool-thread-1] Counting orders...
[pool-thread-2] Calculating revenue...
[pool-thread-3] Fetching top product...

╔══════════════════════════════════╗
║     ShopSphere Dashboard         ║
╠══════════════════════════════════╣
║  Active Orders  : 247            ║
║  Revenue Today  : ₹184329.50     ║
║  Top Product    : Noise Cancel.. ║
╚══════════════════════════════════╝

Loaded in: 1504ms (vs ~3300ms sequential)
```

All three tasks ran in parallel. Total time = slowest task (1500ms), not sum (3300ms).

---

## 8.5 Exception Handling With `Future`

This is where `Future` gets awkward. Exceptions are wrapped and deferred:

```java
ExecutorService pool = Executors.newFixedThreadPool(2);

Future<String> future = pool.submit(() -> {
    Thread.sleep(500);
    if (true) throw new RuntimeException("Payment gateway timeout");
    return "SUCCESS";
});

try {
    String result = future.get(); // exception surfaces HERE, not when task ran

} catch (ExecutionException e) {
    // e.getCause() = the ACTUAL exception thrown inside the Callable
    Throwable actual = e.getCause();
    System.out.println("Task threw: " + actual.getClass().getSimpleName());
    System.out.println("Message:    " + actual.getMessage());

} catch (InterruptedException e) {
    // The thread calling get() was interrupted while waiting
    Thread.currentThread().interrupt();

} catch (CancellationException e) {
    // future.cancel() was called before task completed
    System.out.println("Task was cancelled");
}

pool.shutdown();
```

**Output:**
```
Task threw: RuntimeException
Message:    Payment gateway timeout
```

---

## 8.6 `Future` Limitations — Why It Becomes Painful

`Future` works for simple cases. At scale it becomes a problem:

```
Problem 1: get() BLOCKS
  future.get() blocks the calling thread.
  If you have 10 futures, you block 10 times sequentially.
  No way to say "call me when it's done."

Problem 2: No chaining
  Can't say: "when task A completes, feed result into task B"
  You must block on A's future, then manually submit B.

Problem 3: No combining
  Can't say: "when BOTH A and B complete, run C"
  Must block on A, block on B, then run C.

Problem 4: No fallback
  Can't say: "if task fails, return a default value"
  Must try/catch around every get().

Problem 5: No composition
  10 futures? You're writing 10 try/catch blocks and 10 get() calls.
  Code becomes deeply nested and hard to read.
```

```java
// ❌ Future hell — real code starts looking like this
Future<User> userFuture = pool.submit(() -> fetchUser(userId));
User user;
try {
    user = userFuture.get(); // BLOCK
} catch (ExecutionException e) { handle(e); return; }

Future<List<Order>> ordersFuture = pool.submit(() -> fetchOrders(user));
List<Order> orders;
try {
    orders = ordersFuture.get(); // BLOCK AGAIN
} catch (ExecutionException e) { handle(e); return; }

Future<Double> totalFuture = pool.submit(() -> calculateTotal(orders));
try {
    Double total = totalFuture.get(); // BLOCK AGAIN
    System.out.println("Total: " + total);
} catch (ExecutionException e) { handle(e); }
```

Three tasks, three blocking calls, three try/catch blocks. And this is a simple linear chain. Now imagine branching, combining, and fallbacks.

`CompletableFuture` solves all of this.

---

## 8.7 `CompletableFuture` — The Modern Solution

Introduced in Java 8. A `CompletableFuture<T>` is a `Future<T>` that you can:
- **Chain** — attach callbacks that run when it completes
- **Combine** — merge multiple futures without blocking
- **Handle errors** — attach fallback logic inline
- **Compose** — feed output of one into input of another

The core mental model: **instead of blocking and pulling the result, you describe what to do WHEN the result arrives.**

```
Future<T>             CompletableFuture<T>
─────────────────     ────────────────────────────────
You PULL result       You describe callbacks
get() blocks          Non-blocking pipeline
One result            Chain of transformations
Manual error handling Inline error handling
No composition        Rich composition API
```

---

## 8.8 Creating a `CompletableFuture`

```java
// ─── supplyAsync — runs Supplier, returns result ──────────────────
CompletableFuture<String> cf1 = CompletableFuture.supplyAsync(() -> {
    // runs on ForkJoinPool.commonPool() by default
    return "ORDER_CONFIRMED";
});

// With explicit executor (always do this in production)
ExecutorService pool = Executors.newFixedThreadPool(4);
CompletableFuture<String> cf2 = CompletableFuture.supplyAsync(() -> {
    return "ORDER_CONFIRMED";
}, pool); // ← use your own pool, not common pool

// ─── runAsync — runs Runnable, no result ──────────────────────────
CompletableFuture<Void> cf3 = CompletableFuture.runAsync(() -> {
    sendEmail("user@example.com");
}, pool);

// ─── completedFuture — already-done future (useful for testing) ───
CompletableFuture<String> done = CompletableFuture.completedFuture("CACHED_RESULT");
```

---

## 8.9 Chaining — `thenApply`, `thenAccept`, `thenRun`

These are the core transformation methods:

```java
// thenApply(Function)  — transform result, returns new CompletableFuture<U>
// thenAccept(Consumer) — consume result, returns CompletableFuture<Void>
// thenRun(Runnable)    — run something after, ignore result, returns Void
```

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

CompletableFuture
    .supplyAsync(() -> {
        System.out.println("[1] Fetching order from DB...");
        sleep(500);
        return "ORD-1001"; // String
    }, pool)

    .thenApply(orderId -> {
        // transform: orderId → enriched order details
        System.out.println("[2] Enriching order: " + orderId);
        sleep(300);
        return "Order{id=" + orderId + ", product=Headphones, qty=2}"; // String
    })

    .thenApply(orderDetails -> {
        // transform again: details → invoice
        System.out.println("[3] Generating invoice...");
        sleep(200);
        return "Invoice{" + orderDetails + ", total=₹2598}"; // String
    })

    .thenAccept(invoice -> {
        // consume — no further transformation needed
        System.out.println("[4] Sending invoice to customer: " + invoice);
    })

    .thenRun(() -> {
        // cleanup/logging after everything
        System.out.println("[5] Pipeline complete. Logging audit entry.");
    })

    .join(); // block main thread until the whole chain completes

pool.shutdown();
```

**Output:**
```
[1] Fetching order from DB...
[2] Enriching order: ORD-1001
[3] Generating invoice...
[4] Sending invoice to customer: Invoice{Order{id=ORD-1001...}, total=₹2598}
[5] Pipeline complete. Logging audit entry.
```

The whole pipeline is non-blocking — each stage runs when the previous one completes. No manual `get()` calls, no `try/catch` per stage.

---

## 8.10 Error Handling — `exceptionally`, `handle`

```java
// ─── exceptionally — fallback if anything above fails ─────────────
CompletableFuture<String> result = CompletableFuture
    .supplyAsync(() -> {
        if (Math.random() > 0.5) throw new RuntimeException("Gateway timeout");
        return "RAZORPAY_SUCCESS";
    }, pool)
    .exceptionally(ex -> {
        // ex = the exception thrown upstream
        System.out.println("Primary gateway failed: " + ex.getMessage());
        return "FALLBACK_SUCCESS"; // provide a default
    });

System.out.println("Payment result: " + result.join());
// Either "RAZORPAY_SUCCESS" or "FALLBACK_SUCCESS" — never throws
```

```java
// ─── handle — runs ALWAYS (success OR failure), like try/finally ──
CompletableFuture<String> result2 = CompletableFuture
    .supplyAsync(() -> {
        if (true) throw new RuntimeException("DB connection lost");
        return "DATA";
    }, pool)
    .handle((value, ex) -> {
        // value = result if success (null if failed)
        // ex    = exception if failed (null if success)
        if (ex != null) {
            System.out.println("Handling error: " + ex.getMessage());
            return "DEFAULT_DATA";
        }
        return value.toUpperCase(); // transform on success
    });

System.out.println("Result: " + result2.join()); // never throws
```

---

## 8.11 Combining Futures — `thenCompose`, `thenCombine`

### `thenCompose` — Sequential Dependency (flatMap)

Use when task B needs the **output of task A** as its input.

```java
// Fetch user → then fetch their orders (needs userId from first call)
CompletableFuture<List<String>> userOrders = CompletableFuture
    .supplyAsync(() -> {
        System.out.println("Fetching user...");
        sleep(500);
        return "USER-42"; // userId
    }, pool)
    .thenCompose(userId -> {
        // userId available — use it to start the next async task
        return CompletableFuture.supplyAsync(() -> {
            System.out.println("Fetching orders for: " + userId);
            sleep(600);
            return List.of("ORD-1", "ORD-2", "ORD-3");
        }, pool);
    });

System.out.println("Orders: " + userOrders.join());
```

```
Timeline:
[──── fetch user 500ms ────][──── fetch orders 600ms ────]
Total: ~1100ms (sequential — B needs A's output)
```

### `thenCombine` — Parallel Independent Tasks

Use when task A and B are **independent** — run them in parallel, combine results.

```java
// Fetch price and stock simultaneously — independent queries
CompletableFuture<String> priceFuture = CompletableFuture
    .supplyAsync(() -> {
        System.out.println("Fetching price...");
        sleep(800);
        return "₹1299";
    }, pool);

CompletableFuture<String> stockFuture = CompletableFuture
    .supplyAsync(() -> {
        System.out.println("Fetching stock...");
        sleep(500);
        return "IN_STOCK";
    }, pool);

// Combine when BOTH complete — runs in ~800ms not 800+500=1300ms
CompletableFuture<String> productCard = priceFuture
    .thenCombine(stockFuture, (price, stock) -> {
        return "ProductCard{price=" + price + ", status=" + stock + "}";
    });

System.out.println(productCard.join());
```

```
Timeline:
[──── fetch price 800ms ────────────]
[──── fetch stock 500ms ─────]
                              combined instantly when both done
Total: ~800ms (parallel — independent)
```

---

## 8.12 `allOf` and `anyOf`

```java
// ─── allOf — wait for ALL to complete ─────────────────────────────
CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> { sleep(300); return "UserService: UP"; }, pool);
CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> { sleep(500); return "OrderService: UP"; }, pool);
CompletableFuture<String> f3 = CompletableFuture.supplyAsync(() -> { sleep(200); return "PaymentService: UP"; }, pool);

// Returns CompletableFuture<Void> — no result, just completion signal
CompletableFuture<Void> allDone = CompletableFuture.allOf(f1, f2, f3);

allDone.thenRun(() -> {
    // All three completed — collect results (already done, no blocking)
    System.out.println(f1.join());
    System.out.println(f2.join());
    System.out.println(f3.join());
    System.out.println("All health checks passed ✓");
}).join();

// ─── anyOf — complete when FIRST one finishes ─────────────────────
CompletableFuture<Object> first = CompletableFuture.anyOf(f1, f2, f3);
System.out.println("First to respond: " + first.join()); // PaymentService (~200ms)
```

---

## 8.13 ShopSphere — Product Page Enrichment

The most realistic use case — load a product page by fetching data from 4 services in parallel:

```java
public class ProductPageLoader {

    private static final ExecutorService pool =
        Executors.newFixedThreadPool(8, r -> new Thread(r, "enrichment-worker"));

    public static void loadProductPage(String productId) {

        long start = System.currentTimeMillis();

        // ─── 4 independent calls — run in parallel ──────────────
        CompletableFuture<String> detailsFuture = CompletableFuture
            .supplyAsync(() -> fetchProductDetails(productId), pool)
            .exceptionally(ex -> "Details unavailable");

        CompletableFuture<String> priceFuture = CompletableFuture
            .supplyAsync(() -> fetchPrice(productId), pool)
            .exceptionally(ex -> "₹---");

        CompletableFuture<String> reviewsFuture = CompletableFuture
            .supplyAsync(() -> fetchReviews(productId), pool)
            .exceptionally(ex -> "No reviews yet");

        CompletableFuture<String> inventoryFuture = CompletableFuture
            .supplyAsync(() -> fetchInventory(productId), pool)
            .exceptionally(ex -> "UNKNOWN");

        // ─── Wait for all 4, then render page ───────────────────
        CompletableFuture.allOf(
                detailsFuture, priceFuture, reviewsFuture, inventoryFuture
        ).thenRun(() -> {
            // All done — collect results
            String details   = detailsFuture.join();
            String price     = priceFuture.join();
            String reviews   = reviewsFuture.join();
            String inventory = inventoryFuture.join();

            long elapsed = System.currentTimeMillis() - start;

            System.out.println("\n╔═══════════════════════════════════════╗");
            System.out.println("║         Product Page: " + productId + "          ║");
            System.out.println("╠═══════════════════════════════════════╣");
            System.out.println("║ Details   : " + details);
            System.out.println("║ Price     : " + price);
            System.out.println("║ Reviews   : " + reviews);
            System.out.println("║ Inventory : " + inventory);
            System.out.println("╚═══════════════════════════════════════╝");
            System.out.printf("Loaded in %dms (slowest service wins)%n", elapsed);

        }).join();
    }

    // ─── Simulated service calls ───────────────────────────────────
    static String fetchProductDetails(String id) { sleep(400); return "Sony WH-1000XM5 Headphones"; }
    static String fetchPrice(String id)          { sleep(600); return "₹28,990"; }
    static String fetchReviews(String id)        { sleep(350); return "4.6★ (2,847 reviews)"; }
    static String fetchInventory(String id)      { sleep(250); return "IN STOCK — ships in 24h"; }
    static void sleep(long ms)                   { try { Thread.sleep(ms); } catch (InterruptedException e) {} }

    public static void main(String[] args) {
        loadProductPage("PROD-WH1000XM5");
        pool.shutdown();
    }
}
```

**Output:**
```
╔═══════════════════════════════════════╗
║         Product Page: PROD-WH1000XM5 ║
╠═══════════════════════════════════════╣
║ Details   : Sony WH-1000XM5 Headphones
║ Price     : ₹28,990
║ Reviews   : 4.6★ (2,847 reviews)
║ Inventory : IN STOCK — ships in 24h
╚═══════════════════════════════════════╝
Loaded in 608ms (slowest service wins)
```

Sequential would be 400+600+350+250 = **1600ms**. Parallel: **608ms**. Same result, 2.6x faster.

---

## 8.14 `get()` vs `join()`

```java
// future.get()
// - throws checked exceptions: InterruptedException, ExecutionException
// - must be in try/catch
// - use inside Callable or method that declares throws

// future.join()
// - throws unchecked CompletionException (wraps cause)
// - no try/catch required
// - cleaner in lambda chains
// - use inside CompletableFuture chains

// In practice:
cf.thenAccept(result -> {
    String val = otherFuture.join(); // ✅ clean in lambda
    String val2 = otherFuture.get(); // ❌ must handle checked exception in lambda
});
```

---

## 8.15 Key Takeaways

| Concept | Remember This |
|---|---|
| `Callable<V>` | Like `Runnable` but returns `V` and can throw checked exceptions |
| `Future<V>` | Handle to async result. `get()` blocks. |
| `ExecutionException` | Wraps exception thrown inside `Callable` — unwrap with `getCause()` |
| `CompletableFuture` | Non-blocking pipeline. Chain, combine, handle errors inline. |
| `supplyAsync` | Start async task that returns value |
| `runAsync` | Start async task with no return value |
| `thenApply` | Transform result (like `map`) |
| `thenAccept` | Consume result (terminal, no further chaining) |
| `thenCompose` | Chain dependent async tasks (like `flatMap`) |
| `thenCombine` | Combine two independent parallel tasks |
| `allOf` | Wait for all futures to complete |
| `anyOf` | Complete when first future finishes |
| `exceptionally` | Fallback on failure |
| `handle` | Runs on success AND failure — like `finally` |
| `join()` vs `get()` | `join()` throws unchecked — cleaner in chains |

---

## Interview Questions from This Chapter

**Q1. What is the difference between `Callable` and `Runnable`?**
> `Runnable.run()` returns void and cannot throw checked exceptions. `Callable.call()` returns a value of type `V` and can throw checked exceptions. `Callable` is used with `ExecutorService.submit()` to get a `Future<V>` back.

**Q2. What is `ExecutionException` and when is it thrown?**
> When a `Callable` throws an exception during execution, it's wrapped in `ExecutionException` and stored in the `Future`. It surfaces when you call `future.get()`. Use `e.getCause()` to get the original exception.

**Q3. What is the difference between `thenApply` and `thenCompose`?**
> `thenApply` transforms the result synchronously — it takes the value and returns a new value (like `map`). `thenCompose` is used when the transformation itself returns a `CompletableFuture` — it flattens the nested future (like `flatMap`). Use `thenCompose` when task B is an async operation that depends on A's result.

**Q4. What is the difference between `thenCombine` and `allOf`?**
> `thenCombine` combines exactly two futures — both run in parallel and their results are merged by a `BiFunction`. `allOf` accepts any number of futures and returns `CompletableFuture<Void>` — signals when all complete but doesn't collect results directly. You collect them manually via `join()` after `allOf` completes.

**Q5. Why use `CompletableFuture` over `Future`?**
> `Future.get()` blocks — you can't attach callbacks or chain operations without blocking. `CompletableFuture` lets you describe a pipeline declaratively: transform results, combine futures, handle errors — all without blocking. It enables truly async, non-blocking code that's also readable.

---

Ready for **Chapter 9: `CompletableFuture` Deep Dive** — advanced chaining, async variants, timeout handling, and building a complete async ShopSphere order pipeline from scratch?
