
# Chapter 14: CountDownLatch, CyclicBarrier, Semaphore & Phaser

---

## 14.1 Why These Exist

Everything so far — `synchronized`, `ReentrantLock`, `wait()`/`notify()` — is about protecting shared data. Chapter 14 is different. These are **coordination primitives** — tools for orchestrating the *timing* of threads, not protecting data.

```
synchronized / locks    → PROTECTION   — only one thread accesses data at a time
wait() / notify()       → SIGNALING    — wake me when something changes

CountDownLatch          → WAITING      — wait until N events have occurred
CyclicBarrier           → MEETING POINT — wait until N threads all arrive
Semaphore               → THROTTLING   — limit N threads accessing a resource
Phaser                  → PHASED WORK  — coordinate multi-phase parallel work
```

All four are in `java.util.concurrent`. All four solve problems you cannot solve cleanly with raw locks.

---

## 14.2 `CountDownLatch` — Wait for N Events

A `CountDownLatch` starts with a count N. Threads call `countDown()` to decrement it. One or more threads call `await()` which blocks until the count reaches zero.

```
CountDownLatch(3)
  count = 3

Thread A calls countDown() → count = 2
Thread B calls countDown() → count = 1
Thread C calls countDown() → count = 0  → all await() calls unblock

Key property: ONE USE ONLY — count cannot be reset
              Once zero → stays zero → all future await() return immediately
```

### Basic Usage

```java
import java.util.concurrent.CountDownLatch;

public class CountDownLatchBasic {

    public static void main(String[] args) throws InterruptedException {

        int serviceCount = 3;
        CountDownLatch latch = new CountDownLatch(serviceCount);

        // 3 services must start before gateway opens
        String[] services = {"user-service", "product-service", "order-service"};

        for (String service : services) {
            new Thread(() -> {
                try {
                    System.out.println("[" + service + "] Starting...");
                    Thread.sleep((long)(Math.random() * 1000));
                    System.out.println("[" + service + "] Ready ✓");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    latch.countDown(); // decrement — even if startup fails
                }
            }, service).start();
        }

        System.out.println("[gateway] Waiting for all services...");
        latch.await(); // blocks until count = 0
        System.out.println("[gateway] All services ready — opening traffic ✓");
    }
}
```

**Output:**
```
[gateway] Waiting for all services...
[user-service] Starting...
[product-service] Starting...
[order-service] Starting...
[product-service] Ready ✓
[user-service] Ready ✓
[order-service] Ready ✓
[gateway] All services ready — opening traffic ✓
```

### `await()` With Timeout

```java
boolean allReady = latch.await(30, TimeUnit.SECONDS);
if (!allReady) {
    System.out.println("Services didn't start in 30s — aborting");
    System.exit(1);
}
```

---

## 14.3 `CountDownLatch` — Two Patterns

### Pattern 1: Starting Gate (1 → N)

One thread signals many threads to start simultaneously. Latch initialized to 1.

```java
public class StartingGate {

    public static void main(String[] args) throws InterruptedException {

        CountDownLatch startGate = new CountDownLatch(1); // one signal to start
        CountDownLatch endGate   = new CountDownLatch(5); // wait for all to finish

        for (int i = 1; i <= 5; i++) {
            final int workerId = i;
            new Thread(() -> {
                try {
                    startGate.await(); // all workers wait here — synchronized start
                    System.out.println("[worker-" + workerId + "] Started simultaneously");
                    Thread.sleep((long)(Math.random() * 1000)); // do work
                    System.out.println("[worker-" + workerId + "] Done");
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                } finally {
                    endGate.countDown();
                }
            }, "worker-" + i).start();
        }

        Thread.sleep(200); // let all workers reach startGate.await()
        System.out.println("[main] All workers ready — GO!");
        startGate.countDown(); // release all workers simultaneously

        endGate.await(); // wait for all workers to finish
        System.out.println("[main] All workers finished ✓");
    }
}
```

**Output:**
```
[main] All workers ready — GO!
[worker-3] Started simultaneously
[worker-1] Started simultaneously
[worker-5] Started simultaneously
[worker-2] Started simultaneously
[worker-4] Started simultaneously
[worker-2] Done
[worker-4] Done
[worker-1] Done
[worker-5] Done
[worker-3] Done
[main] All workers finished ✓
```

### Pattern 2: Completion Gate (N → 1) — What we saw above

N threads signal one waiting thread. This is the service startup pattern.

### ShopSphere — Integration Test Setup

```java
public class IntegrationTestSetup {

    private CountDownLatch servicesReady;

    @BeforeAll
    void startAllServices() throws InterruptedException {

        servicesReady = new CountDownLatch(4);

        // Start all dependencies in parallel
        CompletableFuture.runAsync(() -> {
            startRedis();
            servicesReady.countDown();
        });
        CompletableFuture.runAsync(() -> {
            startRabbitMQ();
            servicesReady.countDown();
        });
        CompletableFuture.runAsync(() -> {
            startElasticsearch();
            servicesReady.countDown();
        });
        CompletableFuture.runAsync(() -> {
            startPostgres();
            servicesReady.countDown();
        });

        // Wait max 60 seconds for all to be ready
        boolean ready = servicesReady.await(60, TimeUnit.SECONDS);
        if (!ready) throw new RuntimeException("Infrastructure didn't start in time");
        System.out.println("All infrastructure ready for testing ✓");
    }

    void startRedis()         { sleep(1000); System.out.println("Redis ready"); }
    void startRabbitMQ()      { sleep(1500); System.out.println("RabbitMQ ready"); }
    void startElasticsearch() { sleep(2000); System.out.println("Elasticsearch ready"); }
    void startPostgres()      { sleep(800);  System.out.println("Postgres ready"); }
    void sleep(long ms)       { try { Thread.sleep(ms); } catch (InterruptedException e) {} }
}
```

---

## 14.4 `CyclicBarrier` — Meeting Point for Threads

`CyclicBarrier` makes a group of threads **wait for each other** at a barrier point. All threads must arrive before any can proceed. Unlike `CountDownLatch`, it's **reusable** — resets automatically after each cycle.

```
CyclicBarrier(3)

Thread A reaches barrier → waits
Thread B reaches barrier → waits
Thread C reaches barrier → all 3 here → all released simultaneously → next phase

Reset! Count back to 3.

Thread A reaches barrier again → waits
Thread B reaches barrier again → waits
Thread C reaches barrier again → all released → next phase
```

### Basic Usage

```java
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.BrokenBarrierException;

public class CyclicBarrierBasic {

    public static void main(String[] args) {

        int threadCount = 3;

        // Optional barrier action — runs once when all threads arrive
        CyclicBarrier barrier = new CyclicBarrier(threadCount, () -> {
            System.out.println("\n[BARRIER] All threads reached checkpoint — proceeding\n");
        });

        for (int i = 1; i <= threadCount; i++) {
            final int id = i;
            new Thread(() -> {
                try {
                    // Phase 1
                    System.out.println("[T" + id + "] Phase 1 work...");
                    Thread.sleep((long)(Math.random() * 500));
                    System.out.println("[T" + id + "] Phase 1 done — waiting at barrier");
                    barrier.await(); // ← wait for all threads

                    // Phase 2 — only starts after ALL threads finish Phase 1
                    System.out.println("[T" + id + "] Phase 2 work...");
                    Thread.sleep((long)(Math.random() * 500));
                    System.out.println("[T" + id + "] Phase 2 done — waiting at barrier");
                    barrier.await(); // ← same barrier — reused!

                    // Phase 3
                    System.out.println("[T" + id + "] Phase 3 work...");

                } catch (InterruptedException | BrokenBarrierException e) {
                    Thread.currentThread().interrupt();
                }
            }, "worker-" + id).start();
        }
    }
}
```

**Output:**
```
[T1] Phase 1 work...
[T2] Phase 1 work...
[T3] Phase 1 work...
[T2] Phase 1 done — waiting at barrier
[T1] Phase 1 done — waiting at barrier
[T3] Phase 1 done — waiting at barrier

[BARRIER] All threads reached checkpoint — proceeding

[T1] Phase 2 work...
[T2] Phase 2 work...
[T3] Phase 2 work...
[T3] Phase 2 done — waiting at barrier
[T2] Phase 2 done — waiting at barrier
[T1] Phase 2 done — waiting at barrier

[BARRIER] All threads reached checkpoint — proceeding

[T1] Phase 3 work...
[T2] Phase 3 work...
[T3] Phase 3 work...
```

No thread starts Phase 2 until ALL threads complete Phase 1. This is the core guarantee.

---

## 14.5 `CountDownLatch` vs `CyclicBarrier` — The Key Distinction

```
                    CountDownLatch          CyclicBarrier
────────────────────────────────────────────────────────────
Reusable?           ❌ One use only         ✅ Resets after each cycle
Who counts down?    Any thread (or same)    Only the waiting threads
Who waits?          Separate waiter(s)      The threads themselves
Use case            N events → 1 waiter     N threads sync with each other
Direction           Many → One              Many ↔ Many
Broken state?       No                      Yes (BrokenBarrierException)
```

```
CountDownLatch: workers notify a manager
  workers call countDown() — manager calls await()
  Manager and workers are DIFFERENT threads

CyclicBarrier: workers sync with each other
  ALL workers call await() — ALL wait for each other
  Same threads both wait AND proceed
```

### ShopSphere — Parallel Report Generation

```java
public class SalesReportGenerator {

    // 4 threads each fetch one section, all must complete before merging
    private final CyclicBarrier barrier;
    private final Map<String, Object> reportSections
        = new ConcurrentHashMap<>();

    public SalesReportGenerator() {
        // Barrier action: merge all sections when all fetched
        this.barrier = new CyclicBarrier(4, this::mergeSections);
    }

    public void generateReport(String date) throws Exception {

        ExecutorService pool = Executors.newFixedThreadPool(4);

        pool.submit(() -> fetchSection("revenue",   date));
        pool.submit(() -> fetchSection("orders",    date));
        pool.submit(() -> fetchSection("inventory", date));
        pool.submit(() -> fetchSection("customers", date));

        pool.shutdown();
        pool.awaitTermination(30, TimeUnit.SECONDS);
    }

    private void fetchSection(String section, String date) {
        try {
            System.out.println("[" + section + "] Fetching data for " + date + "...");
            Thread.sleep((long)(Math.random() * 1000)); // simulate DB query
            reportSections.put(section, "Data for " + section);
            System.out.println("[" + section + "] Done — waiting for others");

            barrier.await(); // wait for all 4 sections

        } catch (InterruptedException | BrokenBarrierException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void mergeSections() {
        // Runs once when all 4 sections are ready
        System.out.println("\n[report] All sections ready — merging...");
        reportSections.forEach((k, v) ->
            System.out.println("[report]   " + k + ": " + v));
        System.out.println("[report] Report generated ✓\n");
    }

    public static void main(String[] args) throws Exception {
        new SalesReportGenerator().generateReport("2026-04-02");
    }
}
```

**Output:**
```
[revenue] Fetching data for 2026-04-02...
[orders] Fetching data for 2026-04-02...
[inventory] Fetching data for 2026-04-02...
[customers] Fetching data for 2026-04-02...
[orders] Done — waiting for others
[customers] Done — waiting for others
[revenue] Done — waiting for others
[inventory] Done — waiting for others

[report] All sections ready — merging...
[report]   revenue: Data for revenue
[report]   orders: Data for orders
[report]   inventory: Data for inventory
[report]   customers: Data for customers
[report] Report generated ✓
```

---

## 14.6 `Semaphore` — Resource Throttling

A `Semaphore` maintains a set of **permits**. `acquire()` takes a permit (blocks if none available). `release()` returns a permit. Only N threads can hold permits simultaneously.

```
Semaphore(3) — 3 permits

Thread 1 acquire() → permits = 2, proceeds
Thread 2 acquire() → permits = 1, proceeds
Thread 3 acquire() → permits = 0, proceeds
Thread 4 acquire() → NO PERMITS — BLOCKS
Thread 5 acquire() → NO PERMITS — BLOCKS

Thread 1 release() → permits = 1 → Thread 4 unblocks
Thread 2 release() → permits = 1 → Thread 5 unblocks
```

This is a **generalization of a lock** — a lock is a `Semaphore(1)`.

### Basic Usage

```java
import java.util.concurrent.Semaphore;

public class SemaphoreBasic {

    // Allow maximum 3 concurrent DB connections
    private static final Semaphore connectionPool = new Semaphore(3);

    static void queryDatabase(String queryId) {
        try {
            System.out.println("[" + queryId + "] Waiting for DB connection...");
            connectionPool.acquire(); // blocks if 3 already acquired
            System.out.println("[" + queryId + "] Got connection — querying..."
                + " | Available: " + connectionPool.availablePermits());

            Thread.sleep(1000); // simulate query

            System.out.println("[" + queryId + "] Done — releasing connection");
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            connectionPool.release(); // ALWAYS in finally
        }
    }

    public static void main(String[] args) {

        // 8 threads compete for 3 permits
        ExecutorService pool = Executors.newFixedThreadPool(8);
        for (int i = 1; i <= 8; i++) {
            final String qId = "Q-00" + i;
            pool.submit(() -> queryDatabase(qId));
        }
        pool.shutdown();
    }
}
```

**Output:**
```
[Q-001] Waiting for DB connection...
[Q-002] Waiting for DB connection...
[Q-003] Waiting for DB connection...
[Q-004] Waiting for DB connection...
[Q-005] Waiting for DB connection...
[Q-001] Got connection — querying... | Available: 2
[Q-002] Got connection — querying... | Available: 1
[Q-003] Got connection — querying... | Available: 0
[Q-004] Waiting...   ← blocked — no permits
[Q-005] Waiting...   ← blocked — no permits
[Q-001] Done — releasing connection
[Q-004] Got connection — querying... | Available: 0
...
```

Only 3 threads query simultaneously — exactly what connection pooling enforces.

---

## 14.7 `Semaphore` — Key Methods and Patterns

```java
Semaphore sem = new Semaphore(5);

// ─── Acquire ───────────────────────────────────────────────────────
sem.acquire();              // blocks until permit available
sem.acquire(3);             // acquire 3 permits at once
sem.acquireUninterruptibly(); // like acquire() but ignores interrupts
boolean got = sem.tryAcquire();            // non-blocking — false if none
boolean got2 = sem.tryAcquire(2, TimeUnit.SECONDS); // timeout

// ─── Release ──────────────────────────────────────────────────────
sem.release();              // return 1 permit
sem.release(3);             // return 3 permits at once

// ─── Query ────────────────────────────────────────────────────────
int available = sem.availablePermits(); // current permit count
int waiting   = sem.getQueueLength();   // threads blocked on acquire()

// ─── Fairness ─────────────────────────────────────────────────────
Semaphore fairSem = new Semaphore(3, true); // FIFO — no starvation
```

### ShopSphere — Rate Limiting External API Calls

```java
public class PaymentGatewayRateLimiter {

    // Razorpay allows max 10 concurrent API calls
    private final Semaphore concurrencyLimit = new Semaphore(10, true);

    // Also limit total calls per second using scheduled release
    private final Semaphore rateLimit = new Semaphore(100); // 100 permits

    public PaymentResponse charge(PaymentRequest request) throws InterruptedException {

        // Acquire concurrency slot — max 10 simultaneous
        concurrencyLimit.acquire();
        try {
            System.out.println("[payment] Calling Razorpay for: "
                + request.orderId()
                + " | Concurrent: " + (10 - concurrencyLimit.availablePermits()));

            // Simulate API call
            Thread.sleep(300);
            return new PaymentResponse(request.orderId(), "SUCCESS");

        } finally {
            concurrencyLimit.release(); // ALWAYS release
        }
    }

    // Scheduled task: refill rate limit permits every second
    public void startRateLimitRefill() {
        Executors.newSingleThreadScheduledExecutor().scheduleAtFixedRate(() -> {
            int deficit = 100 - rateLimit.availablePermits();
            if (deficit > 0) rateLimit.release(deficit);
        }, 1, 1, TimeUnit.SECONDS);
    }

    record PaymentRequest(String orderId, double amount) {}
    record PaymentResponse(String orderId, String status) {}
}
```

---

## 14.8 `Semaphore(1)` as a Mutex — Binary Semaphore

A `Semaphore(1)` acts like a lock but with one crucial difference: **any thread can release it** (not just the one that acquired it).

```java
Semaphore mutex = new Semaphore(1); // binary semaphore

// Thread A acquires
mutex.acquire();
doProtectedWork();
mutex.release(); // Thread A releases

// But Thread B could also release it!
mutex.release(); // now permits = 2 — broken!

// Unlike ReentrantLock which ONLY the lock-holder can release
// Semaphore is a "permit" — not tied to a specific thread
```

Use cases for binary semaphore vs lock:
```
ReentrantLock:     owner-based — same thread must lock/unlock
Semaphore(1):      signal-based — one thread acquires, different thread releases
                   useful for: producer signals consumer
                               "parking" pattern
                               thread handoff
```

---

## 14.9 `Phaser` — Flexible Multi-Phase Coordination

`Phaser` is the most powerful of the four — it combines `CountDownLatch` and `CyclicBarrier` and adds dynamic registration.

```
CountDownLatch: fixed count, single use
CyclicBarrier:  fixed party size, reusable
Phaser:         dynamic parties, reusable, phases numbered,
                parties can join/deregister mid-flight
```

```java
import java.util.concurrent.Phaser;

public class PhaserDemo {

    public static void main(String[] args) {

        // Register main thread + 3 workers
        Phaser phaser = new Phaser(1); // 1 = main thread registered

        for (int i = 1; i <= 3; i++) {
            final int id = i;
            phaser.register(); // dynamically register each worker

            new Thread(() -> {
                try {
                    // Phase 0
                    System.out.println("[W" + id + "] Phase 0 work");
                    Thread.sleep((long)(Math.random() * 500));
                    phaser.arriveAndAwaitAdvance(); // wait for all — advance to phase 1

                    // Phase 1
                    System.out.println("[W" + id + "] Phase 1 work");
                    Thread.sleep((long)(Math.random() * 500));
                    phaser.arriveAndAwaitAdvance(); // advance to phase 2

                    // Phase 2
                    System.out.println("[W" + id + "] Phase 2 work — then deregister");
                    phaser.arriveAndDeregister(); // done — remove from phaser

                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }, "worker-" + id).start();
        }

        // Main thread participates in phases
        phaser.arriveAndAwaitAdvance(); // phase 0 complete
        System.out.println("[main] Phase 0 complete — phase: " + phaser.getPhase());

        phaser.arriveAndAwaitAdvance(); // phase 1 complete
        System.out.println("[main] Phase 1 complete — phase: " + phaser.getPhase());

        phaser.arriveAndDeregister();   // main done
        System.out.println("[main] All phases complete ✓");
    }
}
```

**Output:**
```
[W1] Phase 0 work
[W2] Phase 0 work
[W3] Phase 0 work
[main] Phase 0 complete — phase: 1
[W1] Phase 1 work
[W2] Phase 1 work
[W3] Phase 1 work
[main] Phase 1 complete — phase: 2
[W1] Phase 2 work — then deregister
[W2] Phase 2 work — then deregister
[W3] Phase 2 work — then deregister
[main] All phases complete ✓
```

### `Phaser` Key Methods

```java
phaser.register()                // add one party dynamically
phaser.bulkRegister(n)           // add n parties at once
phaser.arrive()                  // signal arrival, don't wait
phaser.awaitAdvance(phase)       // wait for specific phase to complete
phaser.arriveAndAwaitAdvance()   // arrive AND wait (most common)
phaser.arriveAndDeregister()     // arrive, then remove from future phases
phaser.getPhase()                // current phase number (0, 1, 2...)
phaser.getRegisteredParties()    // how many registered
phaser.getArrivedParties()       // how many arrived this phase
phaser.isTerminated()            // true if all parties deregistered
```

---

## 14.10 Choosing the Right Tool

```
Scenario                                         Tool
────────────────────────────────────────────────────────────────────
Wait for N services/tasks to complete            CountDownLatch
Release N threads simultaneously (starting gate) CountDownLatch(1)
Wait for integration test infra to start         CountDownLatch

N threads sync at a point, then continue         CyclicBarrier
Multi-phase parallel computation                 CyclicBarrier
Parallel report generation (merge at end)        CyclicBarrier

Limit concurrent access to a resource            Semaphore
Rate limiting external API calls                 Semaphore
DB connection pool (manual)                      Semaphore
Throttle concurrent file uploads                 Semaphore

Dynamic party count (threads join/leave)         Phaser
Multi-phase work with variable participants      Phaser
When you need both CountDownLatch + CyclicBarrier Phaser
```

```
Simplest → Most Powerful:
CountDownLatch < CyclicBarrier < Semaphore < Phaser

For 90% of use cases: CountDownLatch or Semaphore
CyclicBarrier: multi-phase parallel algorithms
Phaser: rare — when others aren't flexible enough
```

---

## 14.11 Complete ShopSphere — All Four in One System

```java
public class ShopSphereCoordination {

    // ─── Semaphore: limit concurrent Elasticsearch queries ────────
    private final Semaphore searchConcurrency = new Semaphore(5);

    // ─── CountDownLatch: wait for all caches to warm up ──────────
    private final CountDownLatch cacheWarmup = new CountDownLatch(3);

    // ─── CyclicBarrier: sync daily report sections ────────────────
    private final Map<String, Object> reportData = new ConcurrentHashMap<>();
    private final CyclicBarrier reportBarrier;

    public ShopSphereCoordination() {
        this.reportBarrier = new CyclicBarrier(3, () -> {
            System.out.println("[report] All sections ready — generating PDF...");
            generatePDF(reportData);
        });
    }

    // ─── 1. Semaphore: throttled search ──────────────────────────
    public List<String> searchProducts(String query) throws InterruptedException {
        searchConcurrency.acquire();
        try {
            System.out.println("[search] Query: '" + query + "'"
                + " | Active searches: " + (5 - searchConcurrency.availablePermits()));
            Thread.sleep(200); // simulate ES query
            return List.of("Product A", "Product B");
        } finally {
            searchConcurrency.release();
        }
    }

    // ─── 2. CountDownLatch: cache warmup before serving ──────────
    public void warmCaches() {
        ExecutorService pool = Executors.newFixedThreadPool(3);

        pool.submit(() -> {
            System.out.println("[cache] Warming product cache...");
            sleep(1200);
            System.out.println("[cache] Product cache warm ✓");
            cacheWarmup.countDown();
        });

        pool.submit(() -> {
            System.out.println("[cache] Warming user cache...");
            sleep(800);
            System.out.println("[cache] User cache warm ✓");
            cacheWarmup.countDown();
        });

        pool.submit(() -> {
            System.out.println("[cache] Warming price cache...");
            sleep(600);
            System.out.println("[cache] Price cache warm ✓");
            cacheWarmup.countDown();
        });

        pool.shutdown();
    }

    public void waitForCaches() throws InterruptedException {
        System.out.println("[startup] Waiting for cache warmup...");
        boolean ready = cacheWarmup.await(30, TimeUnit.SECONDS);
        if (ready) System.out.println("[startup] All caches ready — serving traffic ✓");
        else        System.out.println("[startup] Cache warmup timeout! Proceeding anyway.");
    }

    // ─── 3. CyclicBarrier: parallel daily report ─────────────────
    public void generateDailyReport(String date) {
        ExecutorService pool = Executors.newFixedThreadPool(3);

        pool.submit(() -> fetchReportSection("revenue",   date));
        pool.submit(() -> fetchReportSection("orders",    date));
        pool.submit(() -> fetchReportSection("customers", date));

        pool.shutdown();
        try { pool.awaitTermination(30, TimeUnit.SECONDS); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }

    private void fetchReportSection(String section, String date) {
        try {
            System.out.println("[" + section + "] Fetching " + date + " data...");
            sleep((long)(Math.random() * 1000));
            reportData.put(section, section + " data for " + date);
            System.out.println("[" + section + "] Done — at barrier");
            reportBarrier.await();
        } catch (InterruptedException | BrokenBarrierException e) {
            Thread.currentThread().interrupt();
        }
    }

    private void generatePDF(Map<String, Object> data) {
        System.out.println("[report] PDF generated with: " + data.keySet() + " ✓");
    }

    // ─── Main ──────────────────────────────────────────────────────
    public static void main(String[] args) throws InterruptedException {

        ShopSphereCoordination system = new ShopSphereCoordination();

        System.out.println("════════════════════════════════════════════");
        System.out.println("          ShopSphere Startup");
        System.out.println("════════════════════════════════════════════\n");

        // Phase 1: warm caches (CountDownLatch)
        system.warmCaches();
        system.waitForCaches();

        System.out.println("\n════════════════════════════════════════════");
        System.out.println("          Serving Traffic");
        System.out.println("════════════════════════════════════════════\n");

        // Phase 2: serve searches (Semaphore throttling)
        ExecutorService searchPool = Executors.newFixedThreadPool(10);
        for (int i = 1; i <= 8; i++) {
            final String q = "query-" + i;
            searchPool.submit(() -> {
                try { system.searchProducts(q); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        }
        searchPool.shutdown();
        searchPool.awaitTermination(10, TimeUnit.SECONDS);

        System.out.println("\n════════════════════════════════════════════");
        System.out.println("          Daily Report (CyclicBarrier)");
        System.out.println("════════════════════════════════════════════\n");

        // Phase 3: daily report (CyclicBarrier)
        system.generateDailyReport("2026-04-02");
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

**Output:**
```
════════════════════════════════════════════
          ShopSphere Startup
════════════════════════════════════════════

[cache] Warming product cache...
[cache] Warming user cache...
[cache] Warming price cache...
[startup] Waiting for cache warmup...
[cache] Price cache warm ✓
[cache] User cache warm ✓
[cache] Product cache warm ✓
[startup] All caches ready — serving traffic ✓

════════════════════════════════════════════
          Serving Traffic
════════════════════════════════════════════

[search] Query: 'query-1' | Active searches: 1
[search] Query: 'query-2' | Active searches: 2
[search] Query: 'query-3' | Active searches: 3
[search] Query: 'query-4' | Active searches: 4
[search] Query: 'query-5' | Active searches: 5
[search] Query: 'query-6' | Active searches: 5  ← blocked until slot free
[search] Query: 'query-7' | Active searches: 5
[search] Query: 'query-8' | Active searches: 5

════════════════════════════════════════════
          Daily Report (CyclicBarrier)
════════════════════════════════════════════

[revenue] Fetching 2026-04-02 data...
[orders] Fetching 2026-04-02 data...
[customers] Fetching 2026-04-02 data...
[orders] Done — at barrier
[customers] Done — at barrier
[revenue] Done — at barrier
[report] All sections ready — generating PDF...
[report] PDF generated with: [revenue, orders, customers] ✓
```

---

## 14.12 Key Takeaways

| Tool | Initialize | Key Method | Reusable | Use Case |
|---|---|---|---|---|
| `CountDownLatch` | `new CountDownLatch(N)` | `countDown()` / `await()` | ❌ | Wait for N events |
| `CyclicBarrier` | `new CyclicBarrier(N)` | `await()` | ✅ | N threads sync |
| `Semaphore` | `new Semaphore(N)` | `acquire()` / `release()` | ✅ | Limit concurrency |
| `Phaser` | `new Phaser(N)` | `arriveAndAwaitAdvance()` | ✅ | Multi-phase dynamic |

```
CountDownLatch  → events signal a waiter
CyclicBarrier   → threads wait for each other
Semaphore       → throttle access to a resource
Phaser          → dynamic multi-phase coordination

Always release() in finally — Semaphore
Always countDown() in finally — CountDownLatch
Handle BrokenBarrierException — CyclicBarrier
```

---

## Interview Questions From This Chapter

**Q1. What is the difference between `CountDownLatch` and `CyclicBarrier`?**
> `CountDownLatch` is a one-shot gate — initialized with N, decremented by `countDown()` from any thread, and one or more threads `await()` zero. Not reusable. `CyclicBarrier` is a meeting point — N threads all call `await()` and ALL block until all N arrive, then all proceed. Resets automatically. Use `CountDownLatch` when events signal a waiter; use `CyclicBarrier` when threads must synchronize with each other.

**Q2. What is a `Semaphore` and how is it different from a lock?**
> A `Semaphore` maintains N permits — `acquire()` takes one (blocks if none available), `release()` returns one. A lock is a `Semaphore(1)`. Key difference: a lock is owner-based — only the acquiring thread can release it. A `Semaphore` is permit-based — any thread can release it, even one that didn't acquire. Use `Semaphore` for throttling (max N concurrent threads), not mutual exclusion.

**Q3. How would you limit 10 concurrent DB connections using a `Semaphore`?**
> `Semaphore sem = new Semaphore(10)`. In the DB access method: `sem.acquire()` before getting connection, `sem.release()` in a `finally` block after returning it. The 11th thread blocks until one of the 10 releases. Use `new Semaphore(10, true)` for fairness — threads served in arrival order, preventing starvation.

**Q4. When would you use `Phaser` over `CyclicBarrier`?**
> When the number of participating threads is dynamic — threads can join or leave mid-execution via `register()` and `arriveAndDeregister()`. `CyclicBarrier` requires a fixed party count at construction. `Phaser` also gives you the phase number and lets you terminate early. Use it when you need the flexibility of both `CountDownLatch` and `CyclicBarrier` combined.

**Q5. What happens if one thread throws an exception before reaching a `CyclicBarrier`?**
> The barrier is broken — `BrokenBarrierException` is thrown for all other threads currently waiting at `await()`. The barrier enters a broken state and all subsequent `await()` calls throw immediately. This is why you must handle `BrokenBarrierException` and have a recovery strategy — typically logging the failure and abandoning the current cycle.

---

## Updated Interview Readiness

```
Chapter 14: CountDownLatch, CyclicBarrier, Semaphore, Phaser  ✅
```

```
Multithreading interview readiness: ~96%
```

Remaining optional chapters:
```
Chapter 15: Fork/Join Framework        (lower priority for 2 YOE)
Chapter 16: Virtual Threads Java 21    (relevant — ShopSphere uses Java 21)
Chapter 18: Performance & false sharing (good to know)
```

**Recommendation:** Chapter 16 (Virtual Threads) is worth doing — your ShopSphere already uses Java 21 (`eclipse-temurin:21`), and interviewers increasingly ask about Project Loom. It's also a genuine differentiator for a 2 YOE candidate.

Ready for **Chapter 16: Virtual Threads & Project Loom**?
