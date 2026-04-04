
# Chapter 16: Virtual Threads & Project Loom — Java 21

---

## 16.1 The Problem Virtual Threads Solve

To understand why Project Loom exists, you need to understand the fundamental limitation of platform threads.

### The Current Reality — Platform Threads

Every thread you've created in Chapters 1–15 is a **platform thread** — a thin wrapper around an OS thread:

```
Java Thread  ←→  OS Thread  ←→  CPU Core

Creating a thread:
  → OS allocates ~1MB stack
  → OS registers the thread
  → Expensive: ~100 microseconds per thread creation

Blocking a thread (DB query, HTTP call, sleep):
  → OS thread sits IDLE
  → Still consumes ~1MB memory
  → Context switch cost when unblocked
```

```
Tomcat default: 200 threads maximum
Request arrives → assigned to a thread
Thread calls DB → BLOCKS (waiting for DB response, ~5ms)
During those 5ms: thread is IDLE — wasting memory and OS resources
With 200 threads: max 200 concurrent blocking operations

Under load: 201st request → waits for a free thread → latency spike
```

This is the **thread-per-request** model's ceiling. To handle 10,000 concurrent requests, you'd need 10,000 OS threads — ~10GB of stack memory, massive context switching overhead. Impractical.

### The Workaround — Reactive Programming

The industry response was reactive/async code:

```java
// Reactive — non-blocking, but complex
return userRepository.findByIdReactive(userId)  // returns Mono<User>
    .flatMap(user -> orderRepository.findByUserReactive(user.getId()))
    .flatMap(orders -> inventoryRepository.checkStockReactive(orders))
    .map(result -> buildResponse(result))
    .onErrorResume(ex -> Mono.just(fallbackResponse()));
```

This works — but it's hard to read, hard to debug, requires reactive libraries everywhere, and completely changes how you write code. Stack traces become useless. Debugging is painful.

### Project Loom's Answer — Virtual Threads

```
What if blocking a thread was CHEAP?
What if you could have 1,000,000 threads with no problem?
What if you could write simple blocking code AND get reactive performance?

That's Project Loom.
```

---

## 16.2 What Are Virtual Threads?

Virtual threads are **lightweight threads managed by the JVM**, not the OS.

```
Platform Thread (before Loom):
  Java Thread → OS Thread → CPU Core
  1:1 mapping — one Java thread = one OS thread
  ~1MB stack, expensive to create, expensive to block

Virtual Thread (Project Loom, Java 21):
  Virtual Thread → Carrier Thread (OS Thread) → CPU Core
  M:N mapping — millions of virtual threads on a few OS threads
  ~few KB stack, cheap to create, cheap to block
```

```
Platform threads:        Virtual threads:
┌────────────────┐      ┌──────────────────────────────────────────────┐
│  OS Thread 1   │      │         JVM Scheduler                        │
│  OS Thread 2   │      │  VT1  VT2  VT3  VT4  VT5 ... VT1,000,000    │
│  OS Thread 3   │      │   ↕    ↕    ↕    ↕    ↕                      │
│  OS Thread 4   │      │  Carrier Thread 1  Carrier Thread 2          │
│  ...           │      │  (OS Thread)       (OS Thread)               │
│  OS Thread 200 │      └──────────────────────────────────────────────┘
└────────────────┘
Max ~200-1000           Max: millions
```

When a virtual thread **blocks** (waiting for I/O, DB, sleep):
- JVM **unmounts** it from the carrier thread
- Carrier thread picks up another virtual thread
- When the blocking operation completes, virtual thread is **remounted**
- This happens transparently — your code doesn't change

---

## 16.3 Creating Virtual Threads — 3 Ways

```java
// ─── Way 1: Thread.ofVirtual() ────────────────────────────────────
Thread vt = Thread.ofVirtual()
    .name("my-virtual-thread")
    .start(() -> System.out.println("Hello from virtual thread!"));

vt.join();

// ─── Way 2: Thread.startVirtualThread() — shorthand ──────────────
Thread vt2 = Thread.startVirtualThread(() -> {
    System.out.println("Running on: " + Thread.currentThread());
    System.out.println("Is virtual: " + Thread.currentThread().isVirtual());
});
vt2.join();

// ─── Way 3: Executors.newVirtualThreadPerTaskExecutor() ───────────
// ← This is the one you'll use most in production
ExecutorService executor = Executors.newVirtualThreadPerTaskExecutor();

executor.submit(() -> System.out.println("Task 1 — virtual thread"));
executor.submit(() -> System.out.println("Task 2 — virtual thread"));
executor.submit(() -> System.out.println("Task 3 — virtual thread"));

executor.shutdown();
executor.awaitTermination(10, TimeUnit.SECONDS);
```

**Output:**
```
Hello from virtual thread!
Running on: VirtualThread[#21]/runnable@ForkJoinPool-1-worker-1
Is virtual: true
Task 1 — virtual thread
Task 2 — virtual thread
Task 3 — virtual thread
```

Notice: `VirtualThread[#21]/runnable@ForkJoinPool-1-worker-1` — the virtual thread runs ON a ForkJoinPool carrier thread. The ForkJoinPool is the default scheduler for virtual threads.

---

## 16.4 The Fundamental Difference — Blocking Comparison

This single demo shows why virtual threads are revolutionary:

```java
public class BlockingComparison {

    static void simulateDbQuery(int id) {
        try {
            Thread.sleep(100); // simulate 100ms DB query
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }

    public static void main(String[] args) throws InterruptedException {

        int taskCount = 10_000; // 10,000 concurrent "requests"

        // ─── Platform threads ──────────────────────────────────────
        System.out.println("Testing platform threads...");
        long start = System.nanoTime();

        // Can't create 10,000 platform threads easily — limit to 200
        ExecutorService platformPool = Executors.newFixedThreadPool(200);

        CountDownLatch platformLatch = new CountDownLatch(taskCount);
        for (int i = 0; i < taskCount; i++) {
            final int id = i;
            platformPool.submit(() -> {
                simulateDbQuery(id);
                platformLatch.countDown();
            });
        }
        platformLatch.await();
        long platformTime = System.nanoTime() - start;
        platformPool.shutdown();

        // ─── Virtual threads ───────────────────────────────────────
        System.out.println("Testing virtual threads...");
        start = System.nanoTime();

        ExecutorService virtualPool = Executors.newVirtualThreadPerTaskExecutor();

        CountDownLatch virtualLatch = new CountDownLatch(taskCount);
        for (int i = 0; i < taskCount; i++) {
            final int id = i;
            virtualPool.submit(() -> {
                simulateDbQuery(id);
                virtualLatch.countDown();
            });
        }
        virtualLatch.await();
        long virtualTime = System.nanoTime() - start;
        virtualPool.shutdown();

        // ─── Results ───────────────────────────────────────────────
        System.out.println("\n═══════════════════════════════════════════");
        System.out.println("  10,000 tasks each blocking 100ms");
        System.out.println("═══════════════════════════════════════════");
        System.out.printf("Platform threads (pool=200): %,d ms%n",
            platformTime / 1_000_000);
        System.out.printf("Virtual threads:             %,d ms%n",
            virtualTime  / 1_000_000);
        System.out.printf("Speedup: %.1fx%n",
            (double) platformTime / virtualTime);
    }
}
```

**Output:**
```
═══════════════════════════════════════════
  10,000 tasks each blocking 100ms
═══════════════════════════════════════════
Platform threads (pool=200): 5,124 ms   ← 10,000 tasks / 200 threads = 50 batches × 100ms
Virtual threads:               124 ms   ← all 10,000 run "simultaneously"
Speedup: 41.3x
```

Platform threads: 50 batches × 100ms = 5000ms.
Virtual threads: all 10,000 unmount when blocking, carrier threads stay busy → completes in ~100ms.

---

## 16.5 How Virtual Thread Scheduling Works

```
Virtual Thread starts → mounted on Carrier Thread → runs
         │
         ├── hits blocking call (sleep, I/O, DB, HTTP)
         │
         ▼
   JVM UNMOUNTS virtual thread from carrier
         │
         ├── virtual thread saved to heap (tiny ~few KB)
         │   (continuation — the thread's stack state)
         │
         ├── carrier thread is now FREE
         │
         └── carrier thread picks up ANOTHER virtual thread → runs it
                    │
                    ▼
   blocking operation completes (OS signals completion)
         │
         ├── JVM schedules virtual thread for remounting
         │
         └── next available carrier thread REMOUNTS the virtual thread
             → continues exactly where it left off
```

```
Key insight: Virtual thread stack is stored on the HEAP when unmounted
  Platform thread stack: 1MB on OS stack — can't be moved
  Virtual thread stack:  few KB on JVM heap — movable, growable, cheap
```

---

## 16.6 Virtual Threads vs Platform Threads — Full Comparison

```
                        Platform Thread      Virtual Thread
────────────────────────────────────────────────────────────────
Managed by              OS                   JVM
Stack size              ~1MB fixed           ~few KB (grows as needed)
Creation cost           ~100μs               ~1μs (100x cheaper)
Context switch          OS-level (~1-10μs)   JVM-level (~100ns)
Blocking cost           OS thread idle       JVM unmounts — carrier free
Max practical count     ~1,000-10,000        ~1,000,000+
Identity (==)           Stable               Stable
Thread.currentThread()  Works                Works (returns virtual thread)
ThreadLocal             Works                Works (but avoid — memory)
synchronized            Works                Works (but causes pinning*)
ReentrantLock           Works                Works perfectly ✓
Debugging/stack trace   Normal               Normal (JVM 21 improved)
Spring Boot support     Always               Spring Boot 3.2+ (auto)
```

*Pinning — covered in 16.8

---

## 16.7 `newVirtualThreadPerTaskExecutor` — The Production API

In production, you'll use this executor. It creates **one virtual thread per submitted task**:

```java
public class VirtualThreadExecutorDemo {

    // ─── Service that makes blocking calls ────────────────────────
    static String fetchUserFromDB(String userId) throws InterruptedException {
        Thread.sleep(50); // simulate DB latency
        return "User{id=" + userId + ", name=Vidhan}";
    }

    static String fetchOrdersFromDB(String userId) throws InterruptedException {
        Thread.sleep(80); // simulate DB latency
        return "Orders[ORD-001, ORD-002] for " + userId;
    }

    static String callPaymentService(String orderId) throws InterruptedException {
        Thread.sleep(120); // simulate HTTP call latency
        return "Payment status: CONFIRMED for " + orderId;
    }

    public static void main(String[] args) throws Exception {

        // One virtual thread per task — scales to millions
        try (ExecutorService executor =
                Executors.newVirtualThreadPerTaskExecutor()) {

            long start = System.nanoTime();

            // Submit 1000 concurrent "request handlers"
            List<Future<String>> futures = new ArrayList<>();

            for (int i = 0; i < 1000; i++) {
                final String userId = "USER-" + i;
                futures.add(executor.submit(() -> {
                    // Each of these blocks — that's fine with virtual threads
                    String user   = fetchUserFromDB(userId);
                    String orders = fetchOrdersFromDB(userId);
                    String payment = callPaymentService("ORD-" + userId);
                    return user + " | " + orders + " | " + payment;
                }));
            }

            // Collect results
            int completed = 0;
            for (Future<String> f : futures) {
                f.get(); // blocks until done
                completed++;
            }

            long elapsed = System.nanoTime() - start;
            System.out.println("Completed: " + completed + " requests");
            System.out.printf("Time: %,d ms%n", elapsed / 1_000_000);
            System.out.printf("Throughput: %.0f req/sec%n",
                completed * 1000.0 / (elapsed / 1_000_000));
        } // try-with-resources auto-shuts down executor
    }
}
```

**Output:**
```
Completed: 1000 requests
Time: 253 ms
Throughput: 3,952 req/sec
```

Each request does 50+80+120=250ms of blocking. Sequential: 250,000ms. Virtual threads: ~253ms. Near-perfect parallelism at zero configuration.

---

## 16.8 Pinning — The One Gotcha

A virtual thread is **pinned** to its carrier thread (cannot unmount) when:

```
1. Inside a synchronized block or method
2. Inside a native method call
```

```java
// ❌ PINNING — virtual thread cannot unmount while blocking inside synchronized
synchronized (lock) {
    Thread.sleep(1000);   // ← virtual thread PINNED for 1 second
    // carrier thread is STUCK — can't serve other virtual threads
    // destroys the benefit of virtual threads
}

// ✅ NO PINNING — ReentrantLock allows unmounting
ReentrantLock lock = new ReentrantLock();
lock.lock();
try {
    Thread.sleep(1000);   // ← virtual thread unmounts cleanly
    // carrier thread free to run other virtual threads
} finally {
    lock.unlock();
}
```

### Detecting Pinning

```bash
# JVM flag to log pinning events
java -Djdk.tracePinnedThreads=full -jar shopsphere.jar

# Output when pinning occurs:
Thread[#21,ForkJoinPool-1-worker-1,5,CarrierThreads]
    com.shopsphere.InventoryService.reserveStock(InventoryService.java:42)
    ← pinned here
```

### ShopSphere Pinning Migration

```java
// ❌ Will pin virtual threads — common in existing codebases
public class InventoryService {
    private final Object lock = new Object();

    public boolean reserveStock(String productId, int qty) {
        synchronized (lock) {          // ← causes pinning
            Thread.sleep(10);          // ← blocks here while pinned
            return checkAndDeduct(productId, qty);
        }
    }
}

// ✅ Virtual thread friendly — use ReentrantLock
public class InventoryServiceV2 {
    private final ReentrantLock lock = new ReentrantLock();

    public boolean reserveStock(String productId, int qty)
            throws InterruptedException {
        lock.lock();
        try {
            Thread.sleep(10);          // ← virtual thread unmounts cleanly
            return checkAndDeduct(productId, qty);
        } finally {
            lock.unlock();
        }
    }
}
```

### Practical Rule

```
For new code with virtual threads:
  → Use ReentrantLock instead of synchronized for blocks containing I/O
  → synchronized is fine for pure CPU work (no blocking inside)

For existing Spring Boot code:
  → Spring Boot 3.2+ handles this — mostly transparent
  → Watch for synchronized methods in YOUR code that do I/O
```

---

## 16.9 Structured Concurrency — Java 21 Preview

Structured concurrency is a companion feature to virtual threads. It makes **multi-threaded code as structured as single-threaded code**.

```java
import java.util.concurrent.StructuredTaskScope;

public class StructuredConcurrencyDemo {

    // ─── ShopSphere: Fetch order details from 3 services ──────────
    static String fetchUser(String userId) throws InterruptedException {
        Thread.sleep(50);
        return "User{" + userId + "}";
    }

    static String fetchProduct(String productId) throws InterruptedException {
        Thread.sleep(80);
        return "Product{" + productId + "}";
    }

    static String fetchPayment(String orderId) throws InterruptedException {
        Thread.sleep(60);
        return "Payment{CONFIRMED}";
    }

    // ─── Structured: all succeed or all fail together ─────────────
    public static void buildOrderResponse(
            String userId, String productId, String orderId)
            throws Exception {

        try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {

            // Fork all three — run in parallel on virtual threads
            StructuredTaskScope.Subtask<String> userTask =
                scope.fork(() -> fetchUser(userId));

            StructuredTaskScope.Subtask<String> productTask =
                scope.fork(() -> fetchProduct(productId));

            StructuredTaskScope.Subtask<String> paymentTask =
                scope.fork(() -> fetchPayment(orderId));

            // Wait for ALL to complete (or any to fail)
            scope.join()           // wait for all
                 .throwIfFailed(); // propagate first failure

            // All succeeded — results available
            System.out.println("User:    " + userTask.get());
            System.out.println("Product: " + productTask.get());
            System.out.println("Payment: " + paymentTask.get());

        } // scope closes — all subtasks done or cancelled
    }

    // ─── ShutdownOnSuccess: first to succeed wins ─────────────────
    public static String fetchWithFallback(String productId) throws Exception {

        try (var scope = new StructuredTaskScope.ShutdownOnSuccess<String>()) {

            // Try primary and fallback simultaneously
            scope.fork(() -> {
                Thread.sleep(200); // primary — slow
                return "PRIMARY: " + productId;
            });

            scope.fork(() -> {
                Thread.sleep(50); // fallback — fast
                return "FALLBACK: " + productId;
            });

            scope.join(); // wait for first success
            return scope.result(); // returns first successful result
            // other subtask automatically cancelled when scope closes
        }
    }

    public static void main(String[] args) throws Exception {

        System.out.println("=== ShutdownOnFailure — all must succeed ===");
        long start = System.nanoTime();
        buildOrderResponse("USER-42", "PROD-001", "ORD-789");
        System.out.printf("Time: %dms (parallel, not 50+80+60=190ms)%n\n",
            (System.nanoTime() - start) / 1_000_000);

        System.out.println("=== ShutdownOnSuccess — first wins ===");
        start = System.nanoTime();
        String result = fetchWithFallback("PROD-001");
        System.out.println("Result: " + result);
        System.out.printf("Time: %dms (fallback won at 50ms)%n",
            (System.nanoTime() - start) / 1_000_000);
    }
}
```

**Output:**
```
=== ShutdownOnFailure — all must succeed ===
User:    User{USER-42}
Product: Product{PROD-001}
Payment: Payment{CONFIRMED}
Time: 82ms (parallel, not 50+80+60=190ms)

=== ShutdownOnSuccess — first wins ===
Result: FALLBACK: PROD-001
Time: 52ms (fallback won at 50ms)
```

### Why Structured Concurrency Matters

```
Problem with CompletableFuture / raw threads:
  If one of 3 subtasks fails, the other 2 keep running (wasted work)
  If the parent thread is interrupted, subtasks don't know
  Subtasks can outlive their parent — "thread leak"
  Hard to reason about lifetime of concurrent operations

Structured Concurrency guarantees:
  Subtasks CANNOT outlive their scope
  If scope exits — all subtasks complete or are cancelled
  Failure propagation is structured — not scattered try/catch
  Cancellation propagates down automatically
  Code reads like sequential code — easy to reason about
```

---

## 16.10 `ThreadLocal` with Virtual Threads — Warning

Virtual threads support `ThreadLocal` but you should be careful:

```java
// ThreadLocal works with virtual threads — but be mindful
ThreadLocal<String> context = new ThreadLocal<>();

Thread.startVirtualThread(() -> {
    context.set("USER-42");
    System.out.println(context.get()); // "USER-42" — works fine
    context.remove(); // still required — same rules as platform threads
});
```

### The Problem — Scale

```
Platform threads: 200 threads × ThreadLocal storage = manageable
Virtual threads: 1,000,000 threads × ThreadLocal storage = memory pressure

If each virtual thread holds a large ThreadLocal value
  and you have 1M virtual threads → massive memory usage

Recommendation:
  For virtual threads prefer ScopedValue (Java 21 preview)
  over ThreadLocal for large or request-scoped data
```

### `ScopedValue` — The Virtual Thread Alternative

```java
import jdk.incubator.concurrent.ScopedValue;

public class ScopedValueDemo {

    // ScopedValue — immutable, scope-bound, more efficient than ThreadLocal
    static final ScopedValue<String> CURRENT_USER = ScopedValue.newInstance();

    static void handleRequest(String userId) {
        // Bind value for the scope of this call tree
        ScopedValue.where(CURRENT_USER, userId).run(() -> {
            processOrder();   // can read CURRENT_USER
            sendNotification(); // can read CURRENT_USER
        }); // binding automatically released here
    }

    static void processOrder() {
        System.out.println("Processing order for: " + CURRENT_USER.get());
    }

    static void sendNotification() {
        System.out.println("Notifying: " + CURRENT_USER.get());
    }

    public static void main(String[] args) {
        handleRequest("USER-vidhan");
        handleRequest("USER-priya");
    }
}
```

```
ScopedValue vs ThreadLocal:
  ThreadLocal: mutable, lives as long as thread, explicit remove()
  ScopedValue: immutable, dies with scope, automatic cleanup
  ScopedValue is preferred for virtual threads — lower memory, safer
```

---

## 16.11 Spring Boot + Virtual Threads — ShopSphere Integration

Spring Boot 3.2+ makes virtual threads trivially easy. Add one property:

```yaml
# application.yml — ShopSphere
spring:
  threads:
    virtual:
      enabled: true  # ← that's it — all Tomcat threads become virtual
```

Or in Java config:

```java
@Configuration
public class VirtualThreadConfig {

    // Replace Tomcat's platform thread executor with virtual threads
    @Bean
    public TomcatProtocolHandlerCustomizer<?> virtualThreadTomcatCustomizer() {
        return protocolHandler -> protocolHandler.setExecutor(
            Executors.newVirtualThreadPerTaskExecutor()
        );
    }

    // Virtual thread executor for @Async methods
    @Bean(name = "virtualThreadExecutor")
    public Executor virtualThreadExecutor() {
        return Executors.newVirtualThreadPerTaskExecutor();
    }
}
```

```java
// Your existing Spring Boot code needs ZERO changes
// Virtual threads are transparent to your business logic

@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @Autowired
    private OrderService orderService;

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable String id) {
        // This method now runs on a virtual thread — not a platform thread
        // Blocking calls (JPA, RestTemplate) — handled efficiently
        return orderService.findById(id); // JPA call — virtual thread unmounts
    }
}
```

### What Changes in ShopSphere

```
Before (platform threads):
  Tomcat: 200 platform threads
  200 concurrent requests max
  Each blocking DB call wastes a platform thread
  Under load: request queue builds up

After (virtual threads):
  Tomcat: virtual threads (effectively unlimited)
  100,000+ concurrent requests
  Blocking DB calls → virtual thread unmounts → carrier thread free
  Under load: requests handled immediately

Code changes required: ZERO (for basic adoption)
Watch out for:       synchronized blocks with I/O inside (pinning)
                     ThreadLocal with large values (memory)
```

---

## 16.12 Complete ShopSphere Virtual Thread Demo

```java
public class ShopSphereVirtualThreadDemo {

    // ─── Simulated blocking service calls ────────────────────────
    static String queryUserDB(String userId) throws InterruptedException {
        Thread.sleep(30); // DB latency
        return "User{" + userId + ", premium=true}";
    }

    static String queryInventoryDB(String productId) throws InterruptedException {
        Thread.sleep(25);
        return "Stock{" + productId + ", qty=47}";
    }

    static String callPaymentGateway(String orderId) throws InterruptedException {
        Thread.sleep(80); // external HTTP call
        return "Payment{" + orderId + ", status=CONFIRMED}";
    }

    static void writeAuditLog(String entry) throws InterruptedException {
        Thread.sleep(10); // async log write
    }

    // ─── Order processing — simple blocking code ──────────────────
    static String processOrder(String userId, String productId) {
        String threadType = Thread.currentThread().isVirtual()
            ? "virtual" : "platform";
        try {
            // All of these block — on virtual threads that's free
            String user      = queryUserDB(userId);
            String inventory = queryInventoryDB(productId);

            String orderId = "ORD-" + userId + "-" + System.nanoTime() % 10000;
            String payment = callPaymentGateway(orderId);
            writeAuditLog(orderId + " processed");

            return "[" + threadType + "] " + orderId
                + " | " + user
                + " | " + inventory
                + " | " + payment;

        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return "INTERRUPTED";
        }
    }

    public static void main(String[] args) throws InterruptedException {

        int requestCount = 500;

        System.out.println("════════════════════════════════════════════");
        System.out.println("  ShopSphere — Platform vs Virtual Threads  ");
        System.out.println("════════════════════════════════════════════\n");

        // ─── Platform thread pool ─────────────────────────────────
        System.out.println("[ Platform Thread Pool — 50 threads ]");
        ExecutorService platformPool = Executors.newFixedThreadPool(50);
        CountDownLatch  platformLatch = new CountDownLatch(requestCount);
        long start = System.nanoTime();

        for (int i = 0; i < requestCount; i++) {
            final String userId    = "USER-" + i;
            final String productId = "PROD-" + (i % 20);
            platformPool.submit(() -> {
                processOrder(userId, productId);
                platformLatch.countDown();
            });
        }
        platformLatch.await();
        long platformMs = (System.nanoTime() - start) / 1_000_000;
        platformPool.shutdown();

        System.out.printf("  %d requests completed in %,dms%n%n",
            requestCount, platformMs);

        // ─── Virtual thread executor ──────────────────────────────
        System.out.println("[ Virtual Thread Executor ]");
        ExecutorService virtualExec  = Executors.newVirtualThreadPerTaskExecutor();
        CountDownLatch  virtualLatch = new CountDownLatch(requestCount);
        start = System.nanoTime();

        for (int i = 0; i < requestCount; i++) {
            final String userId    = "USER-" + i;
            final String productId = "PROD-" + (i % 20);
            virtualExec.submit(() -> {
                processOrder(userId, productId);
                virtualLatch.countDown();
            });
        }
        virtualLatch.await();
        long virtualMs = (System.nanoTime() - start) / 1_000_000;
        virtualExec.shutdown();

        System.out.printf("  %d requests completed in %,dms%n%n",
            requestCount, virtualMs);

        // ─── Summary ──────────────────────────────────────────────
        System.out.println("════════════════════════════════════════════");
        System.out.printf("  Platform (50 threads):  %,dms%n", platformMs);
        System.out.printf("  Virtual threads:         %,dms%n", virtualMs);
        System.out.printf("  Speedup:                 %.1fx%n",
            (double) platformMs / virtualMs);
        System.out.printf("  Each request: ~145ms blocking "
            + "(30+25+80+10)%n");
        System.out.println("════════════════════════════════════════════");
    }
}
```

**Output:**
```
════════════════════════════════════════════
  ShopSphere — Platform vs Virtual Threads
════════════════════════════════════════════

[ Platform Thread Pool — 50 threads ]
  500 requests completed in 1,463ms

[ Virtual Thread Executor ]
  500 requests completed in 152ms

════════════════════════════════════════════
  Platform (50 threads):  1,463ms
  Virtual threads:          152ms
  Speedup:                  9.6x
  Each request: ~145ms blocking (30+25+80+10)
════════════════════════════════════════════
```

Platform: 500 requests / 50 threads = 10 batches × 145ms = ~1450ms.
Virtual: all 500 "simultaneously" → ~145ms (slowest single call).

---

## 16.13 When Virtual Threads Help — And When They Don't

```
✅ Virtual threads shine for:
   I/O-bound workloads (DB queries, HTTP calls, file I/O)
   High concurrency with many blocking operations
   Thread-per-request servers (Tomcat, Jetty)
   Anything that currently benefits from reactive programming
   ShopSphere: REST handlers, service calls, DB queries

❌ Virtual threads don't help for:
   CPU-bound work (heavy computation)
   → Virtual thread still needs CPU — can't run faster than cores
   → Fork/Join (Chapter 15) is still right for CPU-bound

❌ Virtual threads make things WORSE when:
   Using synchronized with blocking inside (pinning)
   Large ThreadLocal values × millions of threads (memory pressure)
   CPU saturation — more threads don't help

Rule of thumb:
   Blocked on I/O?    → Virtual threads
   Burning CPU?       → Fork/Join or fixed thread pool = cores count
```

---

## 16.14 Virtual Threads — Interview Cheat Sheet

```
What:     Lightweight JVM-managed threads. Millions possible.
          M:N mapping to OS threads. Cheap to create and block.

How:      Thread.ofVirtual().start(...)
          Executors.newVirtualThreadPerTaskExecutor()
          Spring Boot: spring.threads.virtual.enabled=true

Blocking: Virtual thread UNMOUNTS from carrier. Carrier stays free.
          Carrier picks up another virtual thread.
          When I/O completes: virtual thread REMOUNTS and continues.

Pinning:  synchronized blocks with blocking inside = thread PINNED
          Fix: use ReentrantLock instead of synchronized

ThreadLocal: Works but use carefully at scale (memory)
             Prefer ScopedValue for virtual thread-heavy apps

When to use: I/O-bound, high concurrency, thread-per-request
When NOT:    CPU-bound (use Fork/Join), synchronized-heavy legacy code

Java 21:  Virtual threads — stable (JEP 444)
          Structured concurrency — preview (JEP 453)
          ScopedValue — preview (JEP 446)
```

---

## 16.15 Key Takeaways

| Concept | Remember This |
|---|---|
| Virtual thread | JVM-managed, ~KB stack, millions possible, cheap to block |
| Platform thread | OS-managed, ~MB stack, thousands max, blocking is expensive |
| Unmounting | When virtual thread blocks → JVM unmounts from carrier → carrier serves others |
| Carrier thread | OS thread running virtual threads — default: ForkJoinPool |
| Pinning | `synchronized` + blocking = virtual thread stuck to carrier. Fix: `ReentrantLock` |
| `newVirtualThreadPerTaskExecutor` | Production API — one virtual thread per task |
| Spring Boot | `spring.threads.virtual.enabled=true` — instant adoption |
| Structured concurrency | Subtasks can't outlive scope. Automatic cancellation. Cleaner than CompletableFuture. |
| `ScopedValue` | Preferred over `ThreadLocal` for virtual thread apps |
| CPU-bound | Virtual threads don't help. Use Fork/Join (Chapter 15). |

---

## Interview Questions From This Chapter

**Q1. What is a virtual thread and how is it different from a platform thread?**
> A virtual thread is a lightweight thread managed by the JVM, not the OS. Platform threads map 1:1 to OS threads — each costs ~1MB stack and is expensive to block. Virtual threads use M:N mapping — millions can exist, each costing ~few KB. When a virtual thread blocks (DB, I/O), the JVM unmounts it from its carrier OS thread, which then picks up another virtual thread. The blocker is parked on the heap — no OS thread sits idle.

**Q2. What is pinning in the context of virtual threads?**
> Pinning occurs when a virtual thread cannot unmount from its carrier thread — usually inside a `synchronized` block or native method that contains a blocking call. The carrier thread is stuck for the duration, defeating virtual thread benefits. Fix: replace `synchronized` containing blocking operations with `ReentrantLock`, which allows virtual threads to unmount cleanly. Pure CPU work inside `synchronized` (no blocking) doesn't cause problematic pinning.

**Q3. When would virtual threads NOT improve performance?**
> Virtual threads don't help CPU-bound work — computation needs CPU time regardless of how many threads exist. They also don't help when pinning is prevalent (synchronized + blocking = carrier stuck). And if you have a true CPU bottleneck (all cores at 100%), adding more virtual threads won't increase throughput. Virtual threads specifically solve the I/O blocking problem — threads sitting idle waiting for DB, network, or file responses.

**Q4. How do you enable virtual threads in Spring Boot?**
> In Spring Boot 3.2+, set `spring.threads.virtual.enabled=true` in `application.yml`. This configures Tomcat to use a virtual-thread-per-task executor instead of its platform thread pool. Existing code requires no changes — controllers, services, and repositories work identically. Watch out for `synchronized` blocks containing blocking calls in your own code — they cause pinning.

**Q5. What is structured concurrency and how does it differ from `CompletableFuture`?**
> Structured concurrency (Java 21 preview via `StructuredTaskScope`) guarantees subtasks cannot outlive their enclosing scope — when the scope closes, all subtasks are done or cancelled. `ShutdownOnFailure` cancels remaining tasks if any fail. `ShutdownOnSuccess` cancels remaining tasks when first succeeds. With `CompletableFuture`, if a parent is cancelled subtasks keep running (thread leak), failures scatter across `.exceptionally()` handlers, and cancellation doesn't propagate automatically. Structured concurrency makes multi-threaded code reason like sequential code.

---

## 🎯 Complete Multithreading Curriculum — DONE

```
Phase 1 — Foundations
  ✅ Ch 1:  Process vs Thread, JVM Memory Model
  ✅ Ch 2:  Creating Threads, start() vs run()
  ✅ Ch 3:  Thread Lifecycle, sleep/join/interrupt

Phase 2 — Synchronization
  ✅ Ch 4:  Race Conditions, synchronized
  ✅ Ch 5:  volatile, visibility, happens-before
  ✅ Ch 6:  wait()/notify(), Producer-Consumer

Phase 3 — Concurrency API
  ✅ Ch 7:  ExecutorService, Thread Pools
  ✅ Ch 8:  Callable, Future, CompletableFuture intro
  ✅ Ch 9:  CompletableFuture deep dive

Phase 4 — Advanced Tools
  ✅ Ch 10: Concurrent Collections
  ✅ Ch 11: ReentrantLock, ReadWriteLock, Condition
  ✅ Ch 12: Atomic Variables, CAS, LongAdder
  ✅ Ch 13: ThreadLocal, Immutability, Patterns

Phase 5 — Coordination & Algorithms
  ✅ Ch 14: CountDownLatch, CyclicBarrier, Semaphore, Phaser
  ✅ Ch 15: Fork/Join, RecursiveTask, Work Stealing

Phase 6 — Production & Modern Java
  ✅ Ch 16: Virtual Threads, Project Loom, Structured Concurrency
  ✅ Ch 17: Deadlocks, Detection, Prevention
```

**Multithreading interview readiness: ~99%**

---

## Recommended Next Steps

```
Option A — Full Mock Interview (Recommended now)
  Simulate a real 2 YOE Java interview round
  Q&A across all 17 chapters under time pressure
  Find any remaining weak spots before applying

Option B — Chapter 18 (Optional — diminishing returns)
  Performance tuning, false sharing, Amdahl's Law
  Good to know but rarely asked at 2 YOE level

Option C — Broaden Prep
  Spring Boot internals deep dive
  System Design / LLD (ShopSphere components)
  DSA gaps
  SQL/JPA internals
```

What would you like to do next?
