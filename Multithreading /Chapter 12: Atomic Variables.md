
# Chapter 12: Atomic Variables — CAS, AtomicInteger & Lock-Free Programming

---

## 12.1 The Problem Recap

From Chapter 4 you know `count++` is broken without synchronization:

```java
// ❌ Race condition
private int count = 0;
public void increment() { count++; } // READ + ADD + WRITE — not atomic

// ✅ Fixed with synchronized — but heavy
public synchronized void increment() { count++; }

// ✅ Fixed with AtomicInteger — lighter, faster
private AtomicInteger count = new AtomicInteger(0);
public void increment() { count.incrementAndGet(); } // truly atomic
```

`synchronized` works but acquires a lock — blocks other threads, causes context switching. For single-variable operations, there's a better way: **Compare-And-Swap**.

---

## 12.2 Compare-And-Swap (CAS) — The Foundation

CAS is a **single CPU instruction** — `CMPXCHG` on x86. The hardware atomically does:

```
CAS(variable, expectedValue, newValue):
  if variable == expectedValue:
      variable = newValue
      return true   ← success
  else:
      return false  ← someone else changed it first, try again
```

This is atomic at the **hardware level** — not a software lock. No thread blocking, no OS involvement, no context switching.

### CAS in Action — The Loop

```
Thread 1 wants to increment count from 5 to 6:

Step 1: Read current value  → 5
Step 2: Compute new value   → 6
Step 3: CAS(count, 5, 6)
  → Is count still 5? YES → set to 6, return true ✓

Thread 2 simultaneously wants to increment count from 5 to 6:

Step 1: Read current value  → 5
Step 2: Compute new value   → 6
Step 3: CAS(count, 5, 6)
  → Is count still 5? NO (Thread 1 already set it to 6)
  → return false — RETRY

Step 1: Read current value  → 6   ← fresh read
Step 2: Compute new value   → 7
Step 3: CAS(count, 6, 7)
  → Is count still 6? YES → set to 7, return true ✓
```

No lock. No blocking. Thread 2 simply retries with the updated value. This is called a **spin-retry loop** or **optimistic concurrency**.

---

## 12.3 `AtomicInteger` — Complete API

```java
import java.util.concurrent.atomic.AtomicInteger;

public class AtomicIntegerDemo {

    public static void main(String[] args) {

        AtomicInteger ai = new AtomicInteger(0);

        // ─── Basic operations ──────────────────────────────────────
        ai.set(10);                    // set value
        int val = ai.get();            // get current value → 10
        int old = ai.getAndSet(20);    // set 20, return OLD value → 10

        // ─── Increment / Decrement ─────────────────────────────────
        int post = ai.getAndIncrement(); // return THEN increment (i++)
        int pre  = ai.incrementAndGet(); // increment THEN return (++i)
        int postD = ai.getAndDecrement(); // return THEN decrement
        int preD  = ai.decrementAndGet(); // decrement THEN return

        // ─── Add ───────────────────────────────────────────────────
        int postAdd = ai.getAndAdd(5);   // return old, then add 5
        int preAdd  = ai.addAndGet(5);   // add 5, then return new

        // ─── CAS directly ─────────────────────────────────────────
        boolean success = ai.compareAndSet(30, 100);
        // if current == 30 → set to 100, return true
        // else → do nothing, return false

        // ─── Update with function (Java 8+) ───────────────────────
        int result = ai.updateAndGet(x -> x * 2);    // atomically double it
        int result2 = ai.accumulateAndGet(10, Integer::sum); // atomically add 10

        System.out.println("Final: " + ai.get());
    }
}
```

---

## 12.4 Race Condition Fix — AtomicInteger vs synchronized

The proof that atomics solve the original counter problem:

```java
public class CounterComparison {

    // Version 1: Broken
    static int          brokenCount  = 0;

    // Version 2: synchronized
    static int          syncCount    = 0;

    // Version 3: AtomicInteger
    static AtomicInteger atomicCount = new AtomicInteger(0);

    static void runTest(String name, Runnable increment)
            throws InterruptedException {

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) increment.run();
        });
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 100_000; i++) increment.run();
        });

        t1.start(); t2.start();
        t1.join();  t2.join();
    }

    public static void main(String[] args) throws InterruptedException {

        runTest("broken",  () -> brokenCount++);
        runTest("sync",    () -> { synchronized(CounterComparison.class)
                                   { syncCount++; }});
        runTest("atomic",  () -> atomicCount.incrementAndGet());

        System.out.println("Expected:  200000");
        System.out.println("Broken:    " + brokenCount);  // ~143822 (wrong)
        System.out.println("Sync:      " + syncCount);    // 200000 ✓
        System.out.println("Atomic:    " + atomicCount);  // 200000 ✓
    }
}
```

Both `synchronized` and `AtomicInteger` give correct results. Why prefer atomic?

---

## 12.5 Performance — Atomic vs synchronized

```java
public class PerformanceComparison {

    private static final int THREADS    = 8;
    private static final int ITERATIONS = 1_000_000;

    static int              syncVal   = 0;
    static AtomicInteger    atomicVal = new AtomicInteger(0);

    public static void main(String[] args) throws InterruptedException {

        // Warm up JIT
        for (int i = 0; i < 3; i++) {
            syncVal = 0; atomicVal.set(0);
            runSynchronized();
            runAtomic();
        }

        // Measure synchronized
        syncVal = 0;
        long start = System.nanoTime();
        runSynchronized();
        long syncTime = System.nanoTime() - start;

        // Measure atomic
        atomicVal.set(0);
        start = System.nanoTime();
        runAtomic();
        long atomicTime = System.nanoTime() - start;

        System.out.printf("synchronized: %,d ms%n", syncTime  / 1_000_000);
        System.out.printf("AtomicInteger: %,d ms%n", atomicTime / 1_000_000);
        System.out.printf("Atomic is %.1fx faster%n",
            (double) syncTime / atomicTime);
    }

    static void runSynchronized() throws InterruptedException {
        Thread[] threads = new Thread[THREADS];
        for (int i = 0; i < THREADS; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITERATIONS; j++)
                    synchronized (CounterComparison.class) { syncVal++; }
            });
        }
        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();
    }

    static void runAtomic() throws InterruptedException {
        Thread[] threads = new Thread[THREADS];
        for (int i = 0; i < THREADS; i++) {
            threads[i] = new Thread(() -> {
                for (int j = 0; j < ITERATIONS; j++)
                    atomicVal.incrementAndGet();
            });
        }
        for (Thread t : threads) t.start();
        for (Thread t : threads) t.join();
    }
}
```

**Typical output:**
```
synchronized:  1,842 ms
AtomicInteger:   673 ms
Atomic is 2.7x faster
```

Why faster? `synchronized` involves OS-level mutex — context switches, kernel calls. CAS is a single CPU instruction with no OS involvement.

---

## 12.6 The Full `java.util.concurrent.atomic` Package

```
AtomicInteger          — atomic int operations
AtomicLong             — atomic long operations
AtomicBoolean          — atomic boolean operations
AtomicReference<V>     — atomic reference operations
AtomicIntegerArray     — array of atomic ints
AtomicLongArray        — array of atomic longs
AtomicReferenceArray<V>— array of atomic references
AtomicIntegerFieldUpdater — make existing int field atomic (reflection-based)
AtomicLongFieldUpdater    — same for long
AtomicReferenceFieldUpdater — same for reference
AtomicStampedReference<V>  — reference + stamp (solves ABA problem)
AtomicMarkableReference<V> — reference + boolean mark
LongAdder              — high-contention counter (Java 8+)
LongAccumulator        — general-purpose accumulator
DoubleAdder            — same for double
DoubleAccumulator      — same for double
```

---

## 12.7 `AtomicReference<V>` — Atomic Object Updates

Used when you need atomic swap of an object reference — not just a primitive.

```java
public class AtomicReferenceDemo {

    record Config(String endpoint, int timeout, boolean featureEnabled) {}

    // Current config — updated atomically on hot reload
    private static final AtomicReference<Config> currentConfig =
        new AtomicReference<>(new Config("https://api.shopsphere.com", 5000, false));

    // Simulate config hot reload — no downtime, no locks
    public static void reloadConfig() {
        Config oldConfig = currentConfig.get();
        Config newConfig = new Config(
            oldConfig.endpoint(),
            oldConfig.timeout(),
            true  // enable new feature
        );

        // Atomic swap — all threads see new config instantly
        boolean swapped = currentConfig.compareAndSet(oldConfig, newConfig);
        if (swapped) {
            System.out.println("Config reloaded successfully ✓");
        } else {
            System.out.println("Config changed by another thread — retry");
        }
    }

    // Readers always see consistent config — never half-updated
    public static void handleRequest() {
        Config config = currentConfig.get(); // atomic read — always complete object
        System.out.println("Using endpoint: " + config.endpoint()
            + " | feature: " + config.featureEnabled());
    }

    public static void main(String[] args) throws InterruptedException {

        // Multiple readers
        ExecutorService pool = Executors.newFixedThreadPool(5);

        for (int i = 0; i < 3; i++) {
            pool.submit(AtomicReferenceDemo::handleRequest);
        }

        // One reloader
        pool.submit(AtomicReferenceDemo::reloadConfig);

        // More readers after reload
        Thread.sleep(100);
        for (int i = 0; i < 3; i++) {
            pool.submit(AtomicReferenceDemo::handleRequest);
        }

        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

**Output:**
```
Using endpoint: https://api.shopsphere.com | feature: false
Using endpoint: https://api.shopsphere.com | feature: false
Using endpoint: https://api.shopsphere.com | feature: false
Config reloaded successfully ✓
Using endpoint: https://api.shopsphere.com | feature: true
Using endpoint: https://api.shopsphere.com | feature: true
Using endpoint: https://api.shopsphere.com | feature: true
```

No locks. No synchronized. Readers never see a partially-updated config.

---

## 12.8 The ABA Problem — And `AtomicStampedReference`

CAS checks "is the value still what I expect?" But this has a subtle flaw:

```
Thread 1 reads value = A
Thread 1 is paused by OS scheduler

Thread 2 changes A → B
Thread 2 changes B → A  (back to original!)

Thread 1 resumes
Thread 1 CAS(A, newValue)
  → value is A — CAS succeeds!
  → But the value WAS changed twice — Thread 1 doesn't know

For simple counters: doesn't matter
For complex structures (linked lists, stacks): can cause corruption
```

### Fix — `AtomicStampedReference`

```java
import java.util.concurrent.atomic.AtomicStampedReference;

public class ABADemo {

    // Reference + integer stamp (version counter)
    private static AtomicStampedReference<String> ref =
        new AtomicStampedReference<>("A", 0); // initial value="A", stamp=0

    public static void main(String[] args) {

        int[] stampHolder = new int[1];
        String currentValue = ref.get(stampHolder); // get value AND stamp
        int currentStamp    = stampHolder[0];

        System.out.println("Value: " + currentValue + ", Stamp: " + currentStamp);

        // Simulate ABA
        ref.compareAndSet("A", "B", 0, 1); // A→B, stamp 0→1
        ref.compareAndSet("B", "A", 1, 2); // B→A, stamp 1→2

        // Thread 1's stale stamp (0) won't match current stamp (2)
        boolean success = ref.compareAndSet("A", "C", 0, 1);
        // currentStamp=0, actual stamp=2 → FAILS correctly ✓

        System.out.println("CAS with stale stamp succeeded: " + success); // false
        System.out.println("Final value: " + ref.getReference()); // still "A"
        System.out.println("Final stamp: " + ref.getStamp());     // 2
    }
}
```

Even though the value is back to `"A"`, the stamp `2 ≠ 0` — CAS correctly detects the ABA change.

---

## 12.9 `LongAdder` — High Contention Counters

`AtomicLong` with CAS works great under low contention. Under **high contention** (many threads hammering the same counter), CAS retries accumulate and performance degrades.

```
AtomicLong under high contention:

Thread 1 CAS(5, 6) → success
Thread 2 CAS(5, 6) → FAIL (value is now 6) → retry
Thread 3 CAS(5, 6) → FAIL → retry
Thread 4 CAS(5, 6) → FAIL → retry
Thread 5 CAS(5, 6) → FAIL → retry

All threads spinning on same memory location = contention = slow
```

`LongAdder` solves this by **distributing the count across multiple cells**:

```
LongAdder internal structure:

Thread 1 → Cell[0] += 1   (no contention — own cell)
Thread 2 → Cell[1] += 1   (no contention — own cell)
Thread 3 → Cell[2] += 1   (no contention — own cell)
Thread 4 → Cell[3] += 1   (no contention — own cell)

sum() = base + Cell[0] + Cell[1] + Cell[2] + Cell[3]
```

```java
import java.util.concurrent.atomic.LongAdder;

public class LongAdderDemo {

    private static AtomicLong  atomicCounter  = new AtomicLong(0);
    private static LongAdder   adderCounter   = new LongAdder();

    public static void main(String[] args) throws InterruptedException {

        int THREADS = 16;
        int OPS_PER_THREAD = 1_000_000;

        // AtomicLong
        long start = System.nanoTime();
        runWith(THREADS, OPS_PER_THREAD, () -> atomicCounter.incrementAndGet());
        long atomicTime = System.nanoTime() - start;

        // LongAdder
        start = System.nanoTime();
        runWith(THREADS, OPS_PER_THREAD, () -> adderCounter.increment());
        long adderTime = System.nanoTime() - start;

        System.out.printf("AtomicLong:  %,d ms | result: %,d%n",
            atomicTime / 1_000_000, atomicCounter.get());
        System.out.printf("LongAdder:   %,d ms | result: %,d%n",
            adderTime  / 1_000_000, adderCounter.sum());
        System.out.printf("LongAdder is %.1fx faster%n",
            (double) atomicTime / adderTime);
    }

    static void runWith(int threads, int ops, Runnable task)
            throws InterruptedException {
        Thread[] ts = new Thread[threads];
        for (int i = 0; i < threads; i++) {
            ts[i] = new Thread(() -> {
                for (int j = 0; j < ops; j++) task.run();
            });
        }
        for (Thread t : ts) t.start();
        for (Thread t : ts) t.join();
    }
}
```

**Typical output (16 threads):**
```
AtomicLong:  2,341 ms | result: 16,000,000
LongAdder:     412 ms | result: 16,000,000
LongAdder is 5.7x faster
```

### When to Use Which

```
AtomicLong/AtomicInteger:
  ✅ Low to moderate contention
  ✅ When you need frequent reads (get() is instant)
  ✅ When CAS logic (compareAndSet) is needed
  ShopSphere: order ID generator, request counter per pod

LongAdder:
  ✅ High contention write-heavy counters
  ✅ When reads are infrequent (sum() is expensive)
  ✅ Metrics, event counters, request rate tracking
  ShopSphere: total API calls counter, total revenue accumulator,
              page view counter
```

---

## 12.10 ShopSphere — Atomic Variables in Practice

```java
public class ShopSphereAtomicDemo {

    // ─── 1. Order ID generator — atomic, no synchronized ─────────
    private final AtomicLong orderIdSequence = new AtomicLong(10_000L);

    public String generateOrderId() {
        return "ORD-" + orderIdSequence.incrementAndGet();
        // Thread-safe, lock-free, sequential IDs
    }

    // ─── 2. Active request counter — for rate limiting ────────────
    private final AtomicInteger activeRequests = new AtomicInteger(0);
    private static final int MAX_CONCURRENT = 100;

    public boolean tryHandleRequest() {
        // Atomic increment — if over limit, decrement and reject
        int current = activeRequests.incrementAndGet();
        if (current > MAX_CONCURRENT) {
            activeRequests.decrementAndGet(); // back off
            return false; // 429 Too Many Requests
        }
        return true;
    }

    public void requestComplete() {
        activeRequests.decrementAndGet();
    }

    // ─── 3. Metrics counters — LongAdder for high throughput ──────
    private final LongAdder totalRequestsServed  = new LongAdder();
    private final LongAdder totalRevenueInPaisa  = new LongAdder();
    private final LongAdder failedPayments        = new LongAdder();

    public void recordRequest()              { totalRequestsServed.increment(); }
    public void recordRevenue(long paisa)    { totalRevenueInPaisa.add(paisa); }
    public void recordFailedPayment()        { failedPayments.increment(); }

    public void printMetrics() {
        System.out.println("Total requests:  " + totalRequestsServed.sum());
        System.out.printf( "Total revenue:   ₹%.2f%n",
            totalRevenueInPaisa.sum() / 100.0);
        System.out.println("Failed payments: " + failedPayments.sum());
    }

    // ─── 4. Circuit breaker state — AtomicReference ───────────────
    enum CircuitState { CLOSED, OPEN, HALF_OPEN }

    private final AtomicReference<CircuitState> circuitState =
        new AtomicReference<>(CircuitState.CLOSED);

    public boolean isCircuitOpen() {
        return circuitState.get() == CircuitState.OPEN;
    }

    public void tripCircuit() {
        // Only transition CLOSED → OPEN (not HALF_OPEN → OPEN via this method)
        circuitState.compareAndSet(CircuitState.CLOSED, CircuitState.OPEN);
    }

    public void attemptReset() {
        circuitState.compareAndSet(CircuitState.OPEN, CircuitState.HALF_OPEN);
    }

    // ─── Demo ──────────────────────────────────────────────────────
    public static void main(String[] args) throws InterruptedException {

        ShopSphereAtomicDemo service = new ShopSphereAtomicDemo();
        ExecutorService pool = Executors.newFixedThreadPool(10);

        // Simulate 50 concurrent requests
        for (int i = 0; i < 50; i++) {
            pool.submit(() -> {
                if (service.tryHandleRequest()) {
                    try {
                        service.recordRequest();
                        service.recordRevenue(129900); // ₹1299
                        System.out.println("Handled: "
                            + service.generateOrderId()
                            + " | Active: " + service.activeRequests.get());
                        Thread.sleep(50);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                    } finally {
                        service.requestComplete();
                    }
                } else {
                    System.out.println("Rejected — too many requests");
                }
            });
        }

        pool.shutdown();
        pool.awaitTermination(10, TimeUnit.SECONDS);

        System.out.println("\n─── Metrics ───────────────────────────────");
        service.printMetrics();
        System.out.println("Circuit state: " + service.circuitState.get());
    }
}
```

**Output:**
```
Handled: ORD-10001 | Active: 3
Handled: ORD-10002 | Active: 5
Handled: ORD-10003 | Active: 7
...
Handled: ORD-10050 | Active: 1

─── Metrics ───────────────────────────────
Total requests:  50
Total revenue:   ₹64,950.00
Failed payments: 0
Circuit state: CLOSED
```

---

## 12.11 CAS Limitations — When NOT to Use Atomics

```
❌ Multiple variables that must be updated together
   AtomicInteger balance, AtomicInteger points;
   // Can't atomically update BOTH — use synchronized or a record + AtomicReference

❌ Long critical sections
   if (conditionA && conditionB) { update(); }
   // Multiple reads + conditional — use synchronized

❌ Complex data structure mutations
   // Updating a linked list node, tree rotation, etc.
   // Use synchronized or concurrent collections

✅ Single variable counter/flag/reference
✅ Non-blocking algorithms where retry cost is low
✅ Lock-free data structures (advanced)
✅ Metrics, ID generation, rate limiting
```

---

## 12.12 Key Takeaways

| Concept | Remember This |
|---|---|
| CAS | Hardware-level atomic instruction — no lock, no blocking, retry on failure |
| `AtomicInteger` | Lock-free int operations — faster than `synchronized` for single variable |
| `getAndIncrement()` | Returns OLD value, then increments (post-increment) |
| `incrementAndGet()` | Increments, then returns NEW value (pre-increment) |
| `compareAndSet()` | CAS directly — true if updated, false if value changed |
| `AtomicReference` | Atomic object reference swap — config hot reload, circuit breaker state |
| ABA Problem | Value changed A→B→A — CAS can't detect it. Fix: `AtomicStampedReference` |
| `LongAdder` | Distributed cells — 5-10x faster than `AtomicLong` under high contention |
| `AtomicLong` vs `LongAdder` | `AtomicLong` for CAS logic + frequent reads. `LongAdder` for write-heavy counters. |
| When NOT to use | Multiple variables, complex conditions — use `synchronized` |

---

## Interview Questions From This Chapter

**Q1. How does `AtomicInteger` achieve thread safety without `synchronized`?**
> It uses Compare-And-Swap — a single hardware instruction (`CMPXCHG` on x86). The operation atomically checks if the current value matches the expected value and only updates if it does. If another thread changed the value first, CAS fails and retries with the new value. No locks, no OS involvement, no context switching — just a hardware-level atomic operation.

**Q2. What is the ABA problem and how do you solve it?**
> Thread 1 reads value A and is paused. Thread 2 changes A→B→A. Thread 1 resumes and CAS succeeds because the value is still A — but the value was changed twice and Thread 1 doesn't know. For simple counters this doesn't matter, but for complex structures like linked lists it can cause corruption. Solution: `AtomicStampedReference` — attaches a version stamp that increments on every change. CAS checks both value AND stamp, so A→B→A is detected because the stamp changed.

**Q3. When would you use `LongAdder` instead of `AtomicLong`?**
> Under high contention — many threads incrementing the same counter simultaneously. `AtomicLong` uses CAS on a single value — under high contention, threads keep retrying. `LongAdder` maintains separate cells per thread, eliminating contention. At read time, cells are summed. It's 5-10x faster under high contention. Trade-off: `sum()` is not instantaneously consistent. Use `AtomicLong` when you need `compareAndSet` logic or frequent consistent reads; `LongAdder` for write-heavy metrics and counters.

**Q4. What is the difference between `getAndIncrement()` and `incrementAndGet()`?**
> `getAndIncrement()` returns the value BEFORE incrementing — like `i++` in regular code. `incrementAndGet()` increments first and returns the NEW value — like `++i`. For order ID generation where you want the new ID, use `incrementAndGet()`. For zero-based indexing where you want the old value as an index, use `getAndIncrement()`.

**Q5. Can `AtomicInteger` replace `synchronized` everywhere?**
> No. Atomics work for single-variable operations. When you need to update multiple variables together atomically, or when you have compound conditions (read-check-write involving multiple fields), `synchronized` is required. Atomics are lock-free but only atomic for their own single operation — they cannot provide atomicity across multiple variables or multiple operations as a unit.

---

Ready for **Chapter 13: Thread Safety Patterns** — `ThreadLocal`, immutability as thread safety, double-checked locking, and the patterns that tie everything together before we move to the final critical chapter on deadlocks?
