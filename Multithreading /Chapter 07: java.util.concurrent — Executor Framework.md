
# Chapter 7: The Executor Framework — Thread Pools & ExecutorService

---

## 7.1 Why `new Thread()` Is Not Production Code

Everything you've learned so far used `new Thread(() -> {...}).start()`. That's correct for learning. In production, it's a problem.

```java
// ❌ What happens in production with raw threads
public class OrderController {

    public void placeOrder(Order order) {
        // Every single HTTP request creates a brand new thread
        new Thread(() -> processOrder(order)).start();
    }
}
```

### The Problems

```
Problem 1: Unbounded thread creation
  1000 concurrent requests → 1000 threads created
  Each thread = ~1MB stack memory
  1000 threads = 1GB RAM just for stacks
  JVM crashes with OutOfMemoryError

Problem 2: No reuse
  Thread created → runs task → dies
  Next request → new thread created again
  Thread creation is expensive (~100μs each)
  Under load: you spend more time creating threads than doing work

Problem 3: No control
  No way to limit concurrency
  No way to queue excess work
  No way to track or manage threads

Problem 4: No lifecycle management
  How do you shut down all threads cleanly?
  How do you know when all tasks are done?
```

The fix: **Thread Pools** via the **Executor Framework**.

---

## 7.2 The Core Idea — Thread Pool

```
Without pool:                            With pool:
                                     ┌─────────────────────────┐
Request 1 → new Thread → dies        │     Thread Pool         │
Request 2 → new Thread → dies        │                         │
Request 3 → new Thread → dies        │  Thread 1 ─┐            │
Request 4 → new Thread → dies        │  Thread 2  ├── reused   │
Request 5 → new Thread → dies        │  Thread 3 ─┘            │
                                     │                         │
                                     │  Task Queue             │
                                     │  [task4][task5][task6]  │
                                     └─────────────────────────┘

- Fixed number of threads created ONCE at startup
- Tasks submitted to a queue
- Idle thread picks up next task from queue
- Thread completes task → goes back to pool → picks up next task
- No thread creation/destruction overhead per task
```

---

## 7.3 The Executor Framework Hierarchy

```
«interface»                             «interface»
Executor                                Future<V>
    │                                       │
    │ execute(Runnable)                     │ get(), isDone(), cancel()
    │                                       │
«interface»                            «interface»
ExecutorService                        Callable<V>
    │                                       │
    │ submit(), shutdown()                  │ call() — like Runnable but
    │ invokeAll(), invokeAny()              │ returns value + throws exception
    │
«interface»
ScheduledExecutorService
    │
    │ schedule(), scheduleAtFixedRate()
    │
«class»
ThreadPoolExecutor          ← actual implementation under the hood
    │
    │ (created via factory methods)
    │
«class»
Executors                   ← factory class — don't instantiate directly
    ├── newFixedThreadPool(n)
    ├── newCachedThreadPool()
    ├── newSingleThreadExecutor()
    └── newScheduledThreadPool(n)
```

---

## 7.4 Creating Thread Pools — 4 Factory Methods

### What is a Thread Pool?

Instead of creating a new thread every time (`new Thread()`), we use a pool of ready threads.
It’s like having a team of workers ready to do tasks instead of hiring a new worker every time.

Java gives us 4 easy factory methods in `Executors` class.

### Pool 1: `newFixedThreadPool(n)`

Fixed number of threads. Extra tasks queue up and wait.

```java
ExecutorService pool = Executors.newFixedThreadPool(4);
```

```
Threads: [T1][T2][T3][T4]   ← always exactly 4, never more, never less
Queue:   [task5][task6][task7]  ← excess tasks wait here

Best for: CPU-bound tasks where you want exactly one thread per core
          Predictable resource usage
ShopSphere use: Batch order processing, report generation
```

### Pool 2: `newCachedThreadPool()`

Creates new threads as needed, reuses idle ones. Idle threads die after 60 seconds.

```java
ExecutorService pool = Executors.newCachedThreadPool();
```

```
Request burst: 100 tasks → 100 threads created
After burst:   threads sit idle → die after 60s
Next burst:    new threads created again

Best for: Many short-lived tasks, unpredictable load
Warning:  No upper bound — can create thousands of threads under load
ShopSphere use: Sending notification emails (short tasks, variable load)
```

### Pool 3: `newSingleThreadExecutor()`

Exactly ONE thread. All tasks execute sequentially in submission order.

```java
ExecutorService pool = Executors.newSingleThreadExecutor();
```

```
Tasks submitted: [A][B][C][D]
Execution order: A → B → C → D  (guaranteed sequential)

Best for: Tasks that must run one at a time, maintaining order
          Replacing synchronized queues with a single worker
ShopSphere use: Writing audit logs (sequential, ordered)
               Updating a leaderboard/ranking
```

### Pool 4: `newScheduledThreadPool(n)`

For delayed or periodic tasks.

```java
ScheduledExecutorService pool = Executors.newScheduledThreadPool(2);
```

```
Best for: @Scheduled equivalent in plain Java
          Cron-like periodic jobs
ShopSphere use: Cleanup expired sessions every hour
               Send daily sales report at 8am
               Retry failed payments every 5 minutes
```

---

## 7.5 `execute()` vs `submit()` — The Key Difference

```
execute(Runnable)
  → fire and forget
  → returns void
  → exceptions are swallowed silently (handled by UncaughtExceptionHandler)
  → use when you don't need the result

submit(Runnable or Callable)
  → returns Future<?>
  → exceptions are stored in the Future — thrown when you call future.get()
  → use when you need result OR want to track completion OR handle exceptions
```

### 1. `execute(Runnable)` — Fire and Forget

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

pool.execute(() -> {
    System.out.println("Doing some work...");
    // If exception happens here → you will NEVER know!
});
```

**Simple Meaning:**
- You give the task to the pool.
- You don’t get anything back.
- You don’t know when it finishes.
- If something goes wrong (exception), it is **silently ignored** (swallowed).
- Like throwing a letter in the postbox and walking away.

**Use when:**
- You just want the task to run in background.
- You don’t need any result.
- You don’t care about exceptions.

**Example in ShopSphere:**
- Logging a user action
- Updating some cache (no need to wait)

---

### 2. `submit()` — Get a Future (Receipt)

`submit()` has two versions:

#### a) submit(Runnable)

```java
Future<?> future = pool.submit(() -> {
    System.out.println("Task completed");
});

future.get();        // Waits until task is done
```

#### b) submit(Callable) ← Most Useful

`Callable` is like Runnable but it **can return a value**.

```java
Future<String> future = pool.submit(() -> {
    // Do some work...
    return "Order-12345 processed successfully";
});

String result = future.get();   // This will give you the returned value
System.out.println(result);
```

**Simple Meaning:**
- You get a **Future** object (like a receipt or tracking number).
- You can use `future.get()` to:
  - Wait for the task to finish
  - Get the result
  - Know if it completed successfully

**Big Advantage:**
- If an exception happens inside the task, it is **saved inside the Future**.
- When you call `future.get()`, the exception is thrown so you can catch and handle it.

---

### Simple Comparison with Analogy

| Feature                    | execute()                          | submit()                                   |
|---------------------------|------------------------------------|--------------------------------------------|
| Like sending                | Throw paper plane                  | Send registered post with tracking         |
| Can get result?             | No                                 | Yes (using `future.get()`)                 |
| Know when finished?         | No                                 | Yes                                        |
| Handle exceptions?          | No (hidden)                        | Yes (thrown when calling `get()`)          |
| Most commonly used          | Rarely in real projects            | **Most of the time**                       |

---

### Real ShopSphere Example

```java
// Bad way - using execute()
pool.execute(() -> sendWelcomeEmail(user));   // If email fails, you never know!

// Good way - using submit()
Future<Boolean> future = pool.submit(() -> sendWelcomeEmail(user));

try {
    boolean success = future.get();     // Wait and get result
    if (success) {
        System.out.println("Email sent");
    }
} catch (Exception e) {
    System.out.println("Email failed: " + e.getMessage());
}
```

---

## 7.6 `Future` — Handling Results and Exceptions

`Future<V>` represents the result of an asynchronous computation. The result isn't available yet — it will be in the future.

```java
public class FutureDemo {

    public static void main(String[] args) throws InterruptedException {

        ExecutorService pool = Executors.newFixedThreadPool(2);

        // Submit a Callable that takes time
        Future<String> future = pool.submit(() -> {
            System.out.println("[task] Starting inventory check...");
            Thread.sleep(2000); // simulate slow DB query
            return "IN_STOCK"; // return value
        });

        // Main thread continues immediately — not blocked
        System.out.println("[main] Task submitted. Doing other work...");
        Thread.sleep(500);
        System.out.println("[main] Still doing other work...");

        // Now we need the result — this blocks until available
        try {
            String result = future.get(); // BLOCKS here until task completes
            System.out.println("[main] Inventory status: " + result);
        } catch (ExecutionException e) {
            // Exception thrown inside the task is wrapped in ExecutionException
            System.out.println("[main] Task failed: " + e.getCause().getMessage());
        }

        pool.shutdown();
    }
}
```

**Output:**
```
[main] Task submitted. Doing other work...
[task] Starting inventory check...
[main] Still doing other work...
[main] Inventory status: IN_STOCK   ← arrives after ~2s
```

### Most Important Methods of Future

| Method                              | Simple Meaning                                      | When to use                          |
|-------------------------------------|-----------------------------------------------------|--------------------------------------|
| `future.get()`                      | Wait until task is done and give me the result      | When you need the answer             |
| `future.get(5, TimeUnit.SECONDS)`   | Wait maximum 5 seconds                              | To avoid waiting forever             |
| `future.isDone()`                   | Is the task finished? (success or failed)           | To check without blocking            |
| `future.isCancelled()`              | Was the task cancelled?                             | After calling cancel()               |
| `future.cancel(true)`               | Try to stop the task                                | If you don’t need the task anymore   |
ture.cancel(true)          // attempt to cancel; true = interrupt if running
```
```
### How Exceptions are Handled

This is very important:

- If your task throws an exception, it is **not** thrown immediately.
- The exception is **wrapped** and saved inside the `Future`.
- When you call `future.get()`, the exception is thrown as **`ExecutionException`**.

```java
try {
    String result = future.get();
} catch (ExecutionException e) {
    System.out.println("Task failed because: " + e.getCause().getMessage());
} catch (InterruptedException e) {
    System.out.println("Waiting was interrupted");
}
```

**Tip**: Always use `e.getCause()` to see the original exception.

---

## 7.7 Shutdown — The Right Way

**Future** is like a **receipt** or **tracking number** you get after submitting a task to the thread pool.

- The task is running in the background.
- The result is **not ready yet** — it will come “in the future”.
- You can continue doing other work while waiting.
- When you need the result, you ask the Future for it.

```java
ExecutorService pool = Executors.newFixedThreadPool(4);

// submit tasks...

// ─── Shutdown Option 1: Graceful ─────────────────────────────────
pool.shutdown();
// - Stops accepting new tasks
// - Lets already-submitted tasks complete
// - Returns immediately (non-blocking)

// Wait for completion with timeout
try {
    if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
        // Didn't finish in 30s — force stop
        pool.shutdownNow();
    }
} catch (InterruptedException e) {
    pool.shutdownNow();
    Thread.currentThread().interrupt();
}

// ─── Shutdown Option 2: Immediate ────────────────────────────────
List<Runnable> unfinished = pool.shutdownNow();
// - Stops accepting new tasks
// - Attempts to interrupt running tasks
// - Returns list of tasks that never started
// - Running tasks must check isInterrupted() to stop cleanly
```

### The Shutdown Sequence — Best Practice

```java
public class GracefulShutdown {

    public static void shutdownGracefully(ExecutorService pool, String poolName) {
        System.out.println("[" + poolName + "] Initiating shutdown...");

        pool.shutdown(); // stop accepting new tasks

        try {
            // Wait up to 30 seconds for running tasks to complete
            if (pool.awaitTermination(30, TimeUnit.SECONDS)) {
                System.out.println("[" + poolName + "] Shutdown complete ✓");
            } else {
                System.out.println("[" + poolName + "] Timeout! Forcing shutdown...");
                List<Runnable> pending = pool.shutdownNow();
                System.out.println("[" + poolName + "] " + pending.size() + " tasks cancelled");

                // Wait a bit more for interrupted tasks to respond
                if (!pool.awaitTermination(5, TimeUnit.SECONDS)) {
                    System.err.println("[" + poolName + "] Pool did not terminate!");
                }
            }
        } catch (InterruptedException e) {
            pool.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

---

## 7.8 `invokeAll()` and `invokeAny()`

These are powerful methods for submitting multiple tasks at once.

### `invokeAll()` — Run All, Wait for All

"Start all tasks at the same time. I will wait until every single task is completed."

```java
ExecutorService pool = Executors.newFixedThreadPool(3);

// ShopSphere: Enrich a product with data from 3 different services
List<Callable<String>> enrichmentTasks = List.of(
    () -> { Thread.sleep(500); return "price:₹1299"; },      // Price Service
    () -> { Thread.sleep(300); return "stock:IN_STOCK"; },    // Inventory Service
    () -> { Thread.sleep(700); return "rating:4.3★"; }        // Review Service
);

// Submits all, blocks until ALL complete
List<Future<String>> results = pool.invokeAll(enrichmentTasks);

// All tasks done by this point
for (Future<String> result : results) {
    System.out.println("Enrichment data: " + result.get()); // won't block, already done
}

// Total time: ~700ms (slowest task) — not 500+300+700=1500ms
pool.shutdown();
```

### Important Benefit:

- All tasks run in parallel.
- Total time taken = time of the slowest task.
- In the example above → Total time ≈ 700ms (not 1500ms).

### ShopSphere Use Case:

- Enriching a product page with data from multiple services:
   - Get Price
   - Get Stock
   - Get Customer Ratings

You need all three results before showing the product page.

### `invokeAny()` — Run All, Return Fastest

"Start all tasks at the same time. As soon as any one task finishes successfully, give me its result and cancel the others."

```java
ExecutorService pool = Executors.newFixedThreadPool(3);

// ShopSphere: Try 3 payment gateways, use whichever responds first
List<Callable<String>> paymentAttempts = List.of(
    () -> { Thread.sleep(800); return "Razorpay: SUCCESS"; },
    () -> { Thread.sleep(200); return "PayU: SUCCESS"; },     // ← fastest
    () -> { Thread.sleep(600); return "Stripe: SUCCESS"; }
);

// Returns result of FIRST successful task, cancels the rest
String winner = pool.invokeAny(paymentAttempts);
System.out.println("Payment processed via: " + winner);
// Output: Payment processed via: PayU: SUCCESS   (~200ms)

pool.shutdown();
```
### What happens internally?

- All 3 tasks start together.
- As soon as PayU returns success, invokeAny() immediately returns.
- The other two slower tasks (Razorpay & Stripe) are cancelled automatically.

### ShopSphere Use Case:

- Trying multiple payment gateways (Razorpay, PayU, Stripe, PhonePe).
- Use whichever responds first → improves user experience.
- No need to wait for slow gateways.
---
Key Points to Remember

- Both methods block the calling thread until they return.
- Both work with Callable (not plain Runnable).
- invokeAny() is great for improving speed and user experience.
- Always call pool.shutdown() when you're done.
---

## 7.9 Scheduled Execution

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// ─── Run once after a delay ──────────────────────────────────────
scheduler.schedule(
    () -> System.out.println("Retry payment after 5s delay"),
    5, TimeUnit.SECONDS
);

// ─── Run repeatedly at fixed rate ───────────────────────────────
// Starts 1s after start, then every 10s regardless of task duration
scheduler.scheduleAtFixedRate(
    () -> System.out.println("Cleanup expired sessions: " + LocalTime.now()),
    1,   // initial delay
    10,  // period
    TimeUnit.SECONDS
);

// ─── Run repeatedly with fixed delay ────────────────────────────
// Waits 10s AFTER each task completes before starting next
// Safer when task duration is variable
scheduler.scheduleWithFixedDelay(
    () -> {
        System.out.println("Poll payment status: " + LocalTime.now());
        // if this takes 3s, next run starts 10s after it finishes (13s total gap)
    },
    1,   // initial delay
    10,  // delay between end of one task and start of next
    TimeUnit.SECONDS
);
```

```
scheduleAtFixedRate:   |--task--|..........|--task--|..........|--task--|
                                ←  10s   →          ← 10s   →
                       (10s from START of each task)

scheduleWithFixedDelay:|--task--|←10s→|----task----|←10s→|--task--|
                       (10s from END of each task)

Use scheduleAtFixedRate for: heartbeats, metrics, fixed-interval polling
Use scheduleWithFixedDelay for: retries, work that takes variable time
```

---

## 7.10 `ThreadPoolExecutor` — The Real Implementation

Every `Executors` factory method creates a `ThreadPoolExecutor` under the hood. Understanding it unlocks production tuning:

```java
ThreadPoolExecutor pool = new ThreadPoolExecutor(
    4,                          // corePoolSize    — always-alive threads
    8,                          // maximumPoolSize — max threads ever
    60L,                        // keepAliveTime   — idle non-core thread dies after 60s
    TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100), // workQueue  — holds waiting tasks
    new ThreadFactory() {          // how to create threads
        private int count = 0;
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r, "shopsphere-worker-" + ++count);
            t.setDaemon(false);
            return t;
        }
    },
    new ThreadPoolExecutor.CallerRunsPolicy() // rejection policy — explained below
);
```

### How the Pool Grows

```
Task submitted:
  1. If active threads < corePoolSize      → create new thread immediately
  2. If active threads == corePoolSize     → add to queue
  3. If queue is FULL                      → create new thread (up to maxPoolSize)
  4. If threads == maxPoolSize & queue full → REJECTION POLICY kicks in
```

```
corePoolSize=4, maxPoolSize=8, queue=100:

0-4 tasks   → 4 threads created
5-104 tasks → 4 threads + queue fills up
105-108     → 5th, 6th, 7th, 8th thread created
109+ tasks  → rejection policy!
```

### Rejection Policies

```java
// AbortPolicy (default) — throws RejectedExecutionException
new ThreadPoolExecutor.AbortPolicy()

// CallerRunsPolicy — caller's thread runs the task itself (natural backpressure)
new ThreadPoolExecutor.CallerRunsPolicy()

// DiscardPolicy — silently drops the task (dangerous — data loss)
new ThreadPoolExecutor.DiscardPolicy()

// DiscardOldestPolicy — drops oldest queued task, retries new one
new ThreadPoolExecutor.DiscardOldestPolicy()
```

**ShopSphere recommendation:** Use `CallerRunsPolicy` — when the pool is overwhelmed, the HTTP request thread itself runs the task, which slows down incoming requests and creates natural backpressure. No data lost, no crashes.

---

## 7.11 ShopSphere — Complete Real-World Example

```java
public class ShopSphereExecutorExample {

    // ─── Service-level thread pools (initialized at startup) ──────
    private static final ExecutorService orderPool =
        Executors.newFixedThreadPool(
            Runtime.getRuntime().availableProcessors(), // one thread per CPU core
            r -> new Thread(r, "order-processor-" + System.nanoTime())
        );

    private static final ExecutorService notificationPool =
        Executors.newCachedThreadPool(); // variable, short-lived email tasks

    private static final ScheduledExecutorService scheduler =
        Executors.newScheduledThreadPool(2);

    // ─── Start background jobs ────────────────────────────────────
    public static void startBackgroundJobs() {

        // Retry failed payments every 5 minutes
        scheduler.scheduleWithFixedDelay(
            () -> retryFailedPayments(),
            1, 5, TimeUnit.MINUTES
        );

        // Clean up expired cart sessions every hour
        scheduler.scheduleAtFixedRate(
            () -> cleanupExpiredCarts(),
            0, 1, TimeUnit.HOURS
        );
    }

    // ─── Process order with parallel enrichment ───────────────────
    public static OrderResponse processOrder(String orderId) throws Exception {

        // Submit 3 tasks in parallel
        Future<Boolean> paymentFuture = orderPool.submit(
            () -> validatePayment(orderId)
        );
        Future<Boolean> inventoryFuture = orderPool.submit(
            () -> reserveInventory(orderId)
        );

        // Both run in parallel — wait for both
        boolean paymentOk  = paymentFuture.get(5, TimeUnit.SECONDS);
        boolean inventoryOk = inventoryFuture.get(5, TimeUnit.SECONDS);

        if (paymentOk && inventoryOk) {
            // Send notification asynchronously — don't make user wait
            notificationPool.execute(() -> sendConfirmationEmail(orderId));
            return new OrderResponse(orderId, "CONFIRMED");
        }

        return new OrderResponse(orderId, "FAILED");
    }

    // ─── Shutdown all pools cleanly on app shutdown ───────────────
    public static void shutdown() {
        Stream.of(orderPool, notificationPool, scheduler).forEach(pool -> {
            pool.shutdown();
            try {
                if (!pool.awaitTermination(30, TimeUnit.SECONDS)) {
                    pool.shutdownNow();
                }
            } catch (InterruptedException e) {
                pool.shutdownNow();
                Thread.currentThread().interrupt();
            }
        });
        System.out.println("All pools shut down cleanly ✓");
    }

    // ─── Stubs ────────────────────────────────────────────────────
    static boolean validatePayment(String id)   { sleep(800); return true; }
    static boolean reserveInventory(String id)  { sleep(600); return true; }
    static void sendConfirmationEmail(String id){ sleep(400); System.out.println("Email sent ✓"); }
    static void retryFailedPayments()           { System.out.println("Retrying payments..."); }
    static void cleanupExpiredCarts()           { System.out.println("Cleaning carts..."); }
    static void sleep(long ms) { try { Thread.sleep(ms); } catch (InterruptedException e) {} }

    record OrderResponse(String orderId, String status) {}

    public static void main(String[] args) throws Exception {
        startBackgroundJobs();

        System.out.println("Processing order...");
        long start = System.currentTimeMillis();

        OrderResponse response = processOrder("ORD-789");

        long elapsed = System.currentTimeMillis() - start;
        System.out.println("Order status: " + response.status());
        System.out.println("Time taken: " + elapsed + "ms"); // ~800ms not 800+600=1400ms

        shutdown();
    }
}
```

**Output:**
```
Processing order...
Order status: CONFIRMED
Time taken: 812ms         ← parallel execution (not 1400ms sequential)
Email sent ✓
All pools shut down cleanly ✓
```

---

## 7.12 Choosing the Right Pool — Decision Guide

```
What kind of tasks?
│
├── CPU-bound (heavy computation, no I/O blocking)
│   └── newFixedThreadPool(Runtime.getRuntime().availableProcessors())
│       Reason: More threads than cores = context switching overhead
│
├── I/O-bound (DB calls, HTTP calls, file reads — thread sits waiting)
│   └── newFixedThreadPool(N * cores) where N = 2 to 10
│       Reason: While one thread waits on I/O, others can use the CPU
│       Or: newCachedThreadPool() if load is unpredictable
│
├── Sequential tasks (audit log, ordered processing)
│   └── newSingleThreadExecutor()
│
├── Scheduled / periodic tasks
│   └── newScheduledThreadPool(n)
│
└── Mixed workloads in production
    └── ThreadPoolExecutor with tuned parameters
        + monitoring via pool.getActiveCount(), getQueue().size()
```

---

## 7.13 Key Takeaways

| Concept | Remember This |
|---|---|
| `new Thread()` in production | Never — unbounded, no reuse, no control |
| Thread pool | Fixed threads, shared task queue, reuse across tasks |
| `execute()` | Fire and forget, void return, exceptions silently lost |
| `submit()` | Returns `Future`, exceptions stored, result retrievable |
| `future.get()` | Blocks until result available, throws `ExecutionException` on failure |
| `invokeAll()` | Submit all, wait for all, returns list of Futures |
| `invokeAny()` | Submit all, return first success, cancel rest |
| `shutdown()` | Stop accepting tasks, let running ones finish |
| `shutdownNow()` | Interrupt running tasks, return unstarted ones |
| `CallerRunsPolicy` | Best rejection policy — natural backpressure |
| Pool sizing (CPU-bound) | `availableProcessors()` threads |
| Pool sizing (I/O-bound) | `N * availableProcessors()` threads |

---

## Interview Questions from This Chapter

**Q1. Why should you use a thread pool instead of creating threads directly?**
> Thread creation is expensive and unbounded. A thread pool reuses a fixed number of threads, queues excess work, provides lifecycle management, and prevents resource exhaustion. Raw `new Thread()` in production can cause `OutOfMemoryError` under load.

**Q2. What is the difference between `execute()` and `submit()`?**
> `execute()` takes a `Runnable`, returns void, and silently discards exceptions. `submit()` takes a `Runnable` or `Callable`, returns a `Future`, and stores exceptions — which are thrown as `ExecutionException` when `future.get()` is called. Use `submit()` when you need results or exception handling.

**Q3. What happens when you call `future.get()`?**
> The calling thread blocks until the task completes. If the task threw an exception, `get()` throws `ExecutionException` wrapping the original. If the task was cancelled, it throws `CancellationException`. You can also pass a timeout to avoid blocking indefinitely.

**Q4. What is the difference between `shutdown()` and `shutdownNow()`?**
> `shutdown()` stops accepting new tasks but lets submitted tasks complete — it returns immediately (non-blocking). `shutdownNow()` attempts to interrupt all running tasks and returns the list of tasks that never started. Always follow `shutdown()` with `awaitTermination()` for truly graceful shutdown.

**Q5. How does `ThreadPoolExecutor` decide when to create new threads?**
> First it fills up to `corePoolSize` threads. Then it queues tasks until the queue is full. Only when the queue is full does it create threads beyond `corePoolSize`, up to `maximumPoolSize`. If both the pool and queue are full, the rejection policy kicks in.

---

Ready for **Chapter 8: `Callable`, `Future`, and intro to `CompletableFuture`** — return values from threads, exception handling, and why `Future` alone becomes painful at scale?
