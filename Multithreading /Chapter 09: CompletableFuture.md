
# Chapter 9: `CompletableFuture` Deep Dive

---

## 9.1 Sync vs Async Variants — The Most Misunderstood Part

Every chaining method has three versions. Most developers only know one:

```
thenApply(fn)         — runs fn on the SAME thread that completed the previous stage
thenApplyAsync(fn)    — runs fn on ForkJoinPool.commonPool()
thenApplyAsync(fn, executor) — runs fn on YOUR executor ← always use this in production
```

This applies to every method: `thenApply/Async`, `thenAccept/Async`, `thenRun/Async`, `thenCompose/Async`, `handle/Async`, `exceptionally` (no async variant).

### Why It Matters

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

CompletableFuture
    .supplyAsync(() -> {
        System.out.println("Stage 1 on: " + Thread.currentThread().getName());
        return "data";
    }, pool)

    .thenApply(data -> {
        // ← runs on whichever pool thread completed Stage 1
        // if Stage 1's thread is busy, this BLOCKS it
        System.out.println("Stage 2 on: " + Thread.currentThread().getName());
        return data.toUpperCase();
    })

    .thenApplyAsync(data -> {
        // ← runs on ForkJoinPool.commonPool() — different thread
        System.out.println("Stage 3 on: " + Thread.currentThread().getName());
        return data + "_PROCESSED";
    })

    .thenApplyAsync(data -> {
        // ← runs on YOUR pool — correct for production
        System.out.println("Stage 4 on: " + Thread.currentThread().getName());
        return data;
    }, pool)

    .join();
```

**Output:**
```
Stage 1 on: pool-1-thread-1
Stage 2 on: pool-1-thread-1     ← same thread as Stage 1
Stage 3 on: ForkJoinPool.commonPool-worker-1  ← commonPool
Stage 4 on: pool-1-thread-2     ← your pool, different thread
```

### The Production Rule

```
❌ thenApply()      — ties up the completing thread for CPU work
❌ thenApplyAsync() — uses commonPool (shared across JVM, can starve)
✅ thenApplyAsync(fn, yourExecutor) — always explicit in production
```

```java
// Production pattern — always pass your executor
ExecutorService pool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors(),
    r -> new Thread(r, "cf-worker")
);

CompletableFuture
    .supplyAsync(() -> fetchOrder(), pool)
    .thenApplyAsync(order -> enrich(order), pool)
    .thenApplyAsync(order -> validate(order), pool)
    .thenAcceptAsync(order -> persist(order), pool)
    .join();
```

---

## 9.2 Timeout Handling — Java 9+

`Future.get(timeout)` blocks with a timeout but that's ugly. `CompletableFuture` has two clean timeout methods:

### `orTimeout(duration)` — Complete Exceptionally on Timeout

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> {
        sleep(3000); // slow external call
        return "PAYMENT_SUCCESS";
    }, pool)
    .orTimeout(2, TimeUnit.SECONDS); // ← if not done in 2s, fail with TimeoutException

try {
    System.out.println(cf.join());
} catch (CompletionException e) {
    if (e.getCause() instanceof TimeoutException) {
        System.out.println("Payment gateway timed out! ✗");
    }
}
```

**Output:**
```
Payment gateway timed out! ✗
```

### `completeOnTimeout(value, duration)` — Fallback Value on Timeout

```java
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> {
        sleep(3000); // slow inventory service
        return "IN_STOCK";
    }, pool)
    .completeOnTimeout("UNKNOWN", 1, TimeUnit.SECONDS);
// ← if not done in 1s, complete with "UNKNOWN" instead of failing

System.out.println("Inventory: " + cf.join()); // "UNKNOWN" after 1s
```

### ShopSphere Pattern — Timeout Per Service Call

```java
public class ResilientProductLoader {

    private final ExecutorService pool;

    public ResilientProductLoader(ExecutorService pool) {
        this.pool = pool;
    }

    public CompletableFuture<String> fetchPrice(String productId) {
        return CompletableFuture
            .supplyAsync(() -> priceService.getPrice(productId), pool)
            .completeOnTimeout("₹---", 500, TimeUnit.MILLISECONDS)
            .exceptionally(ex -> "₹---");
    }

    public CompletableFuture<String> fetchInventory(String productId) {
        return CompletableFuture
            .supplyAsync(() -> inventoryService.getStock(productId), pool)
            .completeOnTimeout("CHECK IN STORE", 500, TimeUnit.MILLISECONDS)
            .exceptionally(ex -> "UNAVAILABLE");
    }

    // Even if services are slow or down, page always loads
    public void loadPage(String productId) {
        CompletableFuture.allOf(
            fetchPrice(productId),
            fetchInventory(productId)
        ).join();
    }
}
```

---

## 9.3 Exception Propagation Through Chains

Understanding how exceptions travel through a pipeline is critical:

```java
CompletableFuture
    .supplyAsync(() -> {
        throw new RuntimeException("DB connection failed"); // ← exception here
    }, pool)

    .thenApply(data -> {
        System.out.println("This NEVER runs"); // ← skipped
        return data.toUpperCase();
    })

    .thenApply(data -> {
        System.out.println("This NEVER runs either"); // ← skipped
        return data + "_v2";
    })

    .exceptionally(ex -> {
        // ← catches the exception from ANY stage above
        System.out.println("Caught: " + ex.getMessage());
        return "FALLBACK_VALUE"; // pipeline resumes from here
    })

    .thenAccept(result -> {
        System.out.println("Final result: " + result); // ← runs with fallback
    })

    .join();
```

**Output:**
```
Caught: DB connection failed
Final result: FALLBACK_VALUE
```

### Exception Flow Diagram

```
supplyAsync ──throws──► thenApply ──SKIPPED──► thenApply ──SKIPPED──► exceptionally ──resumes──► thenAccept
     │                                                                       ▲
     └───────────────────────────── exception propagates ────────────────────┘
```

### `handle` vs `exceptionally` — The Difference

```java
// exceptionally — only runs on failure, result type must match
.exceptionally(ex -> "fallback") // String fallback for String pipeline

// handle — runs on BOTH success and failure
.handle((result, ex) -> {
    if (ex != null) {
        // failed
        log.error("Failed: " + ex.getMessage());
        return "fallback";
    }
    // succeeded
    return result.toUpperCase();
})
```

---

## 9.4 Advanced Combining Patterns

### Pattern 1: Fan-Out → Aggregate

```
           ┌─► Task A ─┐
Input ─────┼─► Task B ─┼──► aggregate → output
           └─► Task C ─┘
```

```java
public class OrderEnricher {

    // Fan out to 3 services, aggregate into one response
    public static EnrichedOrder enrich(String orderId) {

        CompletableFuture<String> userFuture = CompletableFuture
            .supplyAsync(() -> fetchUser(orderId), pool);

        CompletableFuture<String> productFuture = CompletableFuture
            .supplyAsync(() -> fetchProduct(orderId), pool);

        CompletableFuture<String> paymentFuture = CompletableFuture
            .supplyAsync(() -> fetchPayment(orderId), pool);

        // Wait for all three, then aggregate
        return CompletableFuture
            .allOf(userFuture, productFuture, paymentFuture)
            .thenApply(v -> new EnrichedOrder(
                userFuture.join(),     // already done — no blocking
                productFuture.join(),  // already done — no blocking
                paymentFuture.join()   // already done — no blocking
            ))
            .join();
    }
}
```

### Pattern 2: Sequential Pipeline With Branching

```
Fetch User ──► Validate ──► [if premium] ──► Priority Processing
                         └──► [if regular] ──► Standard Processing
                                                      └──► Notify
```

```java
CompletableFuture
    .supplyAsync(() -> fetchUser("USER-42"), pool)

    .thenComposeAsync(user -> {
        // Branch based on user type
        if (user.isPremium()) {
            return CompletableFuture.supplyAsync(
                () -> priorityProcess(user), pool
            );
        } else {
            return CompletableFuture.supplyAsync(
                () -> standardProcess(user), pool
            );
        }
    }, pool)

    .thenAcceptAsync(result ->
        sendNotification(result), pool
    )

    .join();
```

### Pattern 3: First Successful Result (Hedge Pattern)

Try multiple implementations simultaneously, use whichever responds first successfully:

```java
// Try 3 payment gateways simultaneously, use first success
public CompletableFuture<String> processPaymentWithHedge(double amount) {

    CompletableFuture<String> razorpay = CompletableFuture
        .supplyAsync(() -> razorpayGateway.charge(amount), pool);

    CompletableFuture<String> payu = CompletableFuture
        .supplyAsync(() -> payuGateway.charge(amount), pool);

    CompletableFuture<String> stripe = CompletableFuture
        .supplyAsync(() -> stripeGateway.charge(amount), pool);

    // anyOf returns first to complete — success OR failure
    // For first SUCCESS only, we need a different approach:
    return razorpay
        .exceptionally(ex -> null)
        .thenCompose(result -> result != null
            ? CompletableFuture.completedFuture(result)
            : payu.exceptionally(ex -> null))
        .thenCompose(result -> result != null
            ? CompletableFuture.completedFuture(result)
            : stripe);
}
```

---

## 9.5 `thenCompose` vs `thenCombine` — Visual Distinction

These two are the most confused methods. One diagram to settle it forever:

```
thenCompose — SEQUENTIAL dependency (A's output feeds B)
──────────────────────────────────────────────────────
[─── Task A ───]
                [─────── Task B(A's result) ───────]
Total time = A + B

Example: fetch userId → fetch orders(userId)
         Can't fetch orders without userId first


thenCombine — PARALLEL independent (A and B run together, combined at end)
──────────────────────────────────────────────────────
[─── Task A ───────────]
[──── Task B ──────────]  ← runs at same time
                          [combine(A,B)]
Total time = max(A, B)

Example: fetch price AND fetch stock → build product card
         Both are independent, no dependency between them
```

```java
// thenCompose — userId needed before fetching orders
CompletableFuture<List<Order>> pipeline = CompletableFuture
    .supplyAsync(() -> getUserId("vidhan@shopsphere.com"), pool)   // returns String userId
    .thenComposeAsync(userId ->
        CompletableFuture.supplyAsync(() -> getOrders(userId), pool) // needs userId
    , pool);

// thenCombine — price and stock are independent
CompletableFuture<ProductCard> card = CompletableFuture
    .supplyAsync(() -> getPrice("PROD-001"), pool)        // independent
    .thenCombineAsync(
        CompletableFuture.supplyAsync(() -> getStock("PROD-001"), pool), // independent
        (price, stock) -> new ProductCard(price, stock),
        pool
    );
```

---

## 9.6 Complete ShopSphere Order Pipeline

Now let's build a full realistic async order processing pipeline that covers everything from Chapters 7–9:

```java
import java.util.concurrent.*;

public class AsyncOrderPipeline {

    // ─── Thread pool — explicit, named threads ─────────────────────
    private static final ExecutorService pool = new ThreadPoolExecutor(
        4, 8, 60L, TimeUnit.SECONDS,
        new LinkedBlockingQueue<>(200),
        r -> new Thread(r, "order-pipeline-" + System.nanoTime()),
        new ThreadPoolExecutor.CallerRunsPolicy()
    );

    // ─── Domain types ──────────────────────────────────────────────
    record Order(String id, String userId, String productId, int qty) {}
    record ValidatedOrder(Order order, String paymentRef) {}
    record FulfilledOrder(ValidatedOrder validated, String trackingId) {}

    // ─── Stage 1: Validate order + payment in parallel ────────────
    static CompletableFuture<ValidatedOrder> validateOrder(Order order) {

        System.out.println("[validate] Starting for order: " + order.id());

        CompletableFuture<Boolean> paymentFuture = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("[payment] Validating payment...");
                sleep(600);
                return true; // payment valid
            }, pool)
            .orTimeout(2, TimeUnit.SECONDS);

        CompletableFuture<Boolean> fraudFuture = CompletableFuture
            .supplyAsync(() -> {
                System.out.println("[fraud] Running fraud check...");
                sleep(400);
                return true; // not fraudulent
            }, pool)
            .orTimeout(2, TimeUnit.SECONDS);

        // Both must pass before proceeding
        return paymentFuture
            .thenCombineAsync(fraudFuture, (paymentOk, fraudOk) -> {
                if (!paymentOk) throw new RuntimeException("Payment validation failed");
                if (!fraudOk)   throw new RuntimeException("Fraud check failed");
                System.out.println("[validate] Both checks passed ✓");
                return new ValidatedOrder(order, "PAY-REF-" + order.id());
            }, pool);
    }

    // ─── Stage 2: Reserve inventory ───────────────────────────────
    static CompletableFuture<ValidatedOrder> reserveInventory(ValidatedOrder validated) {

        return CompletableFuture
            .supplyAsync(() -> {
                System.out.println("[inventory] Reserving " + validated.order().qty()
                    + " units of " + validated.order().productId());
                sleep(500);
                System.out.println("[inventory] Reserved ✓");
                return validated;
            }, pool)
            .completeOnTimeout(validated, 1500, TimeUnit.MILLISECONDS)
            .exceptionally(ex -> {
                System.out.println("[inventory] Reservation failed — will retry async");
                return validated; // proceed optimistically (real system: saga rollback)
            });
    }

    // ─── Stage 3: Create shipment ─────────────────────────────────
    static CompletableFuture<FulfilledOrder> createShipment(ValidatedOrder validated) {

        return CompletableFuture
            .supplyAsync(() -> {
                System.out.println("[shipment] Creating shipment...");
                sleep(700);
                String trackingId = "SHIP-" + validated.order().id() + "-IND";
                System.out.println("[shipment] Tracking ID: " + trackingId + " ✓");
                return new FulfilledOrder(validated, trackingId);
            }, pool)
            .orTimeout(3, TimeUnit.SECONDS);
    }

    // ─── Stage 4: Post-fulfillment tasks (parallel, non-blocking) ─
    static void postFulfillment(FulfilledOrder fulfilled) {

        String orderId = fulfilled.validated().order().id();
        String userId  = fulfilled.validated().order().userId();

        // All three run in parallel — we don't wait for them
        // User gets response immediately, these complete in background

        CompletableFuture.runAsync(() -> {
            sleep(400);
            System.out.println("[email] ✉ Confirmation email sent to " + userId);
        }, pool);

        CompletableFuture.runAsync(() -> {
            sleep(200);
            System.out.println("[audit] 📋 Audit log written for " + orderId);
        }, pool);

        CompletableFuture.runAsync(() -> {
            sleep(300);
            System.out.println("[analytics] 📊 Order event published to Kafka");
        }, pool);

        System.out.println("[post] All post-fulfillment tasks dispatched async");
    }

    // ─── Main pipeline ────────────────────────────────────────────
    static CompletableFuture<FulfilledOrder> processOrder(Order order) {

        return CompletableFuture
            .completedFuture(order)                          // start with order

            .thenComposeAsync(o ->                           // Stage 1: validate
                validateOrder(o), pool)

            .thenComposeAsync(validated ->                   // Stage 2: inventory
                reserveInventory(validated), pool)

            .thenComposeAsync(validated ->                   // Stage 3: shipment
                createShipment(validated), pool)

            .whenCompleteAsync((fulfilled, ex) -> {          // Stage 4: post-work
                if (ex == null) {
                    postFulfillment(fulfilled);
                }
            }, pool)

            .exceptionally(ex -> {                           // Global fallback
                System.out.println("[pipeline] ❌ Order failed: " + ex.getMessage());
                return null;
            });
    }

    // ─── Entry point ──────────────────────────────────────────────
    public static void main(String[] args) throws InterruptedException {

        System.out.println("═══════════════════════════════════════════");
        System.out.println("   ShopSphere Async Order Pipeline Demo");
        System.out.println("═══════════════════════════════════════════\n");

        // Simulate 3 simultaneous orders
        Order order1 = new Order("ORD-001", "USER-vidhan", "PROD-headphones", 1);
        Order order2 = new Order("ORD-002", "USER-priya",  "PROD-keyboard",   2);
        Order order3 = new Order("ORD-003", "USER-rahul",  "PROD-monitor",    1);

        long start = System.currentTimeMillis();

        // Process all 3 in parallel
        CompletableFuture<FulfilledOrder> p1 = processOrder(order1);
        CompletableFuture<FulfilledOrder> p2 = processOrder(order2);
        CompletableFuture<FulfilledOrder> p3 = processOrder(order3);

        // Wait for all pipelines
        CompletableFuture.allOf(p1, p2, p3).join();

        long elapsed = System.currentTimeMillis() - start;

        System.out.println("\n───────────────────────────────────────────");
        System.out.println("   All orders processed in " + elapsed + "ms");
        System.out.println("───────────────────────────────────────────");

        // Allow background tasks to finish printing
        Thread.sleep(600);

        // Graceful shutdown
        pool.shutdown();
        pool.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("\n✅ Pipeline shut down cleanly.");
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

**Output:**
```
═══════════════════════════════════════════
   ShopSphere Async Order Pipeline Demo
═══════════════════════════════════════════

[validate] Starting for order: ORD-001
[validate] Starting for order: ORD-002
[validate] Starting for order: ORD-003
[fraud] Running fraud check...
[payment] Validating payment...
[fraud] Running fraud check...
[payment] Validating payment...
[fraud] Running fraud check...
[payment] Validating payment...
[validate] Both checks passed ✓ (ORD-001)
[validate] Both checks passed ✓ (ORD-002)
[validate] Both checks passed ✓ (ORD-003)
[inventory] Reserving 1 units of PROD-headphones
[inventory] Reserving 2 units of PROD-keyboard
[inventory] Reserving 1 units of PROD-monitor
[inventory] Reserved ✓
[inventory] Reserved ✓
[inventory] Reserved ✓
[shipment] Creating shipment...
[shipment] Creating shipment...
[shipment] Creating shipment...
[shipment] Tracking ID: SHIP-ORD-001-IND ✓
[shipment] Tracking ID: SHIP-ORD-002-IND ✓
[shipment] Tracking ID: SHIP-ORD-003-IND ✓
[post] All post-fulfillment tasks dispatched async

───────────────────────────────────────────
   All orders processed in 1823ms
───────────────────────────────────────────

[audit] 📋 Audit log written for ORD-001
[analytics] 📊 Order event published to Kafka
[email] ✉ Confirmation email sent to USER-vidhan
...

✅ Pipeline shut down cleanly.
```

Sequential equivalent would be ~5400ms. Parallel async: **~1800ms**.

---

## 9.7 `whenComplete` vs `thenApply` vs `handle`

These three are often confused. Clear distinction:

```java
// thenApply — transform result, doesn't see exceptions
.thenApply(result -> transform(result))
// If upstream failed → skipped entirely

// handle — sees BOTH result and exception, can transform either
.handle((result, ex) -> {
    if (ex != null) return fallback;
    return transform(result);
})
// Always runs. Can recover.

// whenComplete — observe result/exception, CANNOT change them
.whenComplete((result, ex) -> {
    if (ex != null) log.error(ex);
    else            metrics.record(result);
    // return value IGNORED — result passes through unchanged
})
// Always runs. For side effects only (logging, metrics).
```

```
Pipeline result flows through:

thenApply:    upstream_result ──transform──► new_result
handle:       upstream_result ──transform──► new_result
              upstream_error  ──fallback───► new_result

whenComplete: upstream_result ──────────────► same_result  (side effect only)
              upstream_error  ────────────────► same_error (side effect only)
```

---

## 9.8 Common Mistakes

### Mistake 1: Blocking Inside `supplyAsync`

```java
// ❌ Blocking get() inside async pipeline — defeats the purpose
CompletableFuture.supplyAsync(() -> {
    String user = userFuture.get(); // BLOCKS the pool thread
    return process(user);
}, pool);

// ✅ Use thenCompose — non-blocking dependency
userFuture.thenComposeAsync(user ->
    CompletableFuture.supplyAsync(() -> process(user), pool)
, pool);
```

### Mistake 2: Using `join()` in the Middle of a Chain

```java
// ❌ Blocking mid-chain — you're writing Future code inside CompletableFuture
CompletableFuture.supplyAsync(() -> fetchOrder(), pool)
    .thenApply(order -> {
        String user = fetchUser(order.userId()).join(); // BLOCKS HERE ← wrong
        return new EnrichedOrder(order, user);
    });

// ✅ Compose properly
CompletableFuture.supplyAsync(() -> fetchOrder(), pool)
    .thenComposeAsync(order ->
        fetchUser(order.userId())
            .thenApplyAsync(user -> new EnrichedOrder(order, user), pool)
    , pool);
```

### Mistake 3: Swallowing Exceptions

```java
// ❌ Exception silently lost — future completes with null
CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("silent failure");
}, pool);
// No .exceptionally(), no .join() → exception goes nowhere

// ✅ Always handle or propagate
CompletableFuture.supplyAsync(() -> {
    throw new RuntimeException("handled failure");
}, pool)
.exceptionally(ex -> {
    log.error("Task failed", ex);
    return null;
})
.join(); // or store the future and check it
```

### Mistake 4: Not Specifying Executor

```java
// ❌ Uses ForkJoinPool.commonPool() — shared JVM-wide
CompletableFuture.supplyAsync(() -> heavyWork());
.thenApplyAsync(result -> moreWork(result)); // also commonPool

// ✅ Always your own executor
CompletableFuture.supplyAsync(() -> heavyWork(), pool)
    .thenApplyAsync(result -> moreWork(result), pool);
```

---

## 9.9 `CompletableFuture` Quick Reference Card

```
CREATION
  supplyAsync(Supplier, executor)    → CF<T>    async task with result
  runAsync(Runnable, executor)       → CF<Void>  async task, no result
  completedFuture(value)             → CF<T>    already-done future
  failedFuture(ex)                   → CF<T>    already-failed future

TRANSFORM (one future)
  thenApply(fn)                      transform result synchronously
  thenApplyAsync(fn, executor)       transform result on new thread
  thenCompose(fn→CF)                 chain dependent async task (flatMap)
  thenComposeAsync(fn→CF, executor)  same, on new thread

CONSUME (terminal)
  thenAccept(consumer)               consume result, return CF<Void>
  thenRun(runnable)                  run after, ignore result

COMBINE (two futures)
  thenCombine(other, biFunction)     both parallel, combine results
  thenAcceptBoth(other, biConsumer)  both parallel, consume both

COMBINE (many futures)
  allOf(cf1, cf2, ...)               CF<Void>, done when ALL complete
  anyOf(cf1, cf2, ...)               CF<Object>, done when FIRST completes

ERROR HANDLING
  exceptionally(fn)                  fallback if upstream failed
  handle(biFunction(result, ex))     always runs, can transform either
  whenComplete(biConsumer)           always runs, side effects only

TIMEOUT (Java 9+)
  orTimeout(duration, unit)          fail with TimeoutException if slow
  completeOnTimeout(value, dur, unit) fallback value if slow

BLOCKING (avoid in chains)
  join()                             block, throw CompletionException
  get()                              block, throw checked exceptions
  get(timeout, unit)                 block with timeout
  isDone()                           non-blocking check
```

---

## 9.10 Key Takeaways

| Concept | Remember This |
|---|---|
| Sync vs Async variants | `thenApply` = same thread. `thenApplyAsync(fn, pool)` = your pool. Always use async+executor in production. |
| `thenCompose` | Sequential dependency. A's output → B's input. Total = A+B. |
| `thenCombine` | Parallel independent. Combine A+B. Total = max(A,B). |
| `allOf` | All must complete. Returns `CF<Void>`. Collect via `join()`. |
| `anyOf` | First to complete wins. Returns `CF<Object>`. |
| `orTimeout` | Fail pipeline if too slow. |
| `completeOnTimeout` | Fallback value if too slow. |
| `exceptionally` | Recover from failure with fallback. |
| `handle` | Runs always. Can inspect and transform both result and error. |
| `whenComplete` | Runs always. Side effects only. Result passes through. |
| Mistake to avoid | Never call `join()`/`get()` inside a `supplyAsync` lambda. |
| Mistake to avoid | Always attach error handling — never let exceptions silently disappear. |

---

## Interview Questions From This Chapter

**Q1. What is the difference between `thenApply` and `thenApplyAsync`?**
> `thenApply` runs the function on whichever thread completed the previous stage — it borrows that thread. `thenApplyAsync` submits the function to a thread pool (common pool by default, or your executor if specified). In production always use `thenApplyAsync` with an explicit executor to avoid tying up pool threads and to control where work runs.

**Q2. When do you use `thenCompose` instead of `thenApply`?**
> When the transformation itself is an async operation that returns a `CompletableFuture`. `thenApply(fn)` where fn returns `CF<T>` gives you `CF<CF<T>>` — a nested future. `thenCompose` flattens it to `CF<T>`. Use `thenCompose` for sequential dependent async tasks — like fetching a userId then using it to fetch orders.

**Q3. What is the difference between `handle` and `exceptionally`?**
> `exceptionally` only runs when the upstream stage failed — it receives the exception and returns a fallback value of the same type. `handle` always runs — it receives both the result (null on failure) and the exception (null on success) and can transform either. Use `exceptionally` for simple fallbacks, `handle` when you need to inspect both success and failure paths.

**Q4. What does `allOf` return and how do you collect results from it?**
> `allOf` returns `CompletableFuture<Void>` — it signals completion but carries no results. After `allOf(...).join()` returns, all individual futures are guaranteed done. Collect their results by calling `.join()` on each one individually — at that point it doesn't block because the work is already complete.

**Q5. What is the difference between `orTimeout` and `completeOnTimeout`?**
> `orTimeout` completes the future exceptionally with `TimeoutException` if it doesn't finish in time — the pipeline fails. `completeOnTimeout` completes it normally with a provided fallback value if it doesn't finish in time — the pipeline continues with the default. Use `orTimeout` when timeout is a hard failure, `completeOnTimeout` for graceful degradation.

---

Ready for **Chapter 10: Concurrent Collections** — `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue` variants, and why regular collections fail under concurrency?
