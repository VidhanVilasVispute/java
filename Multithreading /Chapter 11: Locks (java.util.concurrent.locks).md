
# Chapter 11: `ReentrantLock` & the `locks` Package

---

## 11.1 Why `synchronized` Is Sometimes Not Enough

`synchronized` is solid for most cases. But it has hard limitations that become real problems in production:

```
Limitation 1: No timeout
  synchronized (lock) { ... }
  → If lock is held, thread waits FOREVER
  → No way to say "try for 2 seconds, then give up"

Limitation 2: No interruptibility
  → Thread waiting for synchronized lock CANNOT be interrupted
  → Stuck threads can't be cleaned up gracefully

Limitation 3: No fairness control
  → Any waiting thread can acquire the lock — no ordering
  → A thread can starve (wait indefinitely while others keep winning)

Limitation 4: No read/write separation
  → synchronized treats reads and writes identically
  → 100 threads all reading? Still take turns — one at a time
  → Massive throughput loss for read-heavy scenarios

Limitation 5: No condition variables
  → One wait set per object
  → Can't have "wait for condition A" vs "wait for condition B" separately
```

`java.util.concurrent.locks` solves all five.

---

## 11.2 `ReentrantLock` — The Flexible Replacement

```java
import java.util.concurrent.locks.ReentrantLock;

public class BasicReentrantLock {

    private final ReentrantLock lock = new ReentrantLock();
    private int count = 0;

    public void increment() {
        lock.lock();         // acquire lock — blocks if held by another thread
        try {
            count++;         // critical section
        } finally {
            lock.unlock();   // ALWAYS in finally — even if exception thrown
        }
    }

    public int getCount() {
        lock.lock();
        try {
            return count;
        } finally {
            lock.unlock();
        }
    }
}
```

### The `finally` Rule — Non-Negotiable

```java
// ❌ WRONG — exception skips unlock() — lock held forever (deadlock)
lock.lock();
doWork();       // throws RuntimeException
lock.unlock();  // NEVER REACHED

// ✅ CORRECT — finally guarantees unlock even on exception
lock.lock();
try {
    doWork();
} finally {
    lock.unlock(); // ALWAYS runs
}
```

This is the most common `ReentrantLock` mistake in interviews. If you forget `finally`, you'll deadlock your application.

---

## 11.3 `tryLock()` — The Killer Feature

This is the #1 reason to choose `ReentrantLock` over `synchronized`.

### `tryLock()` — Non-Blocking Attempt

```java
public class TryLockDemo {

    private final ReentrantLock lock = new ReentrantLock();

    public void processOrder(String orderId) {

        if (lock.tryLock()) {          // attempt to acquire — returns immediately
            try {
                System.out.println("Processing: " + orderId);
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                lock.unlock();
            }
        } else {
            // Lock not available — do something else instead of blocking
            System.out.println("Order " + orderId + " queued — processor busy");
        }
    }
}
```

### `tryLock(timeout)` — Wait With a Deadline

```java
public class TryLockTimeout {

    private final ReentrantLock lock = new ReentrantLock();

    public boolean reserveInventory(String productId) {

        try {
            // Wait up to 2 seconds to acquire lock
            if (lock.tryLock(2, TimeUnit.SECONDS)) {
                try {
                    System.out.println("Reserving inventory for: " + productId);
                    Thread.sleep(500); // do the work
                    return true;
                } finally {
                    lock.unlock();
                }
            } else {
                // Couldn't get lock in 2 seconds
                System.out.println("Inventory service busy — try again later");
                return false;
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return false;
        }
    }
}
```

### ShopSphere — Deadlock Prevention with `tryLock`

The classic deadlock scenario: two resources, two threads, opposite locking order.

```java
public class DeadlockPrevention {

    private final ReentrantLock walletLock    = new ReentrantLock();
    private final ReentrantLock inventoryLock = new ReentrantLock();

    // ❌ With synchronized — can deadlock
    // Thread A: locks wallet → waits for inventory
    // Thread B: locks inventory → waits for wallet
    // Both wait forever

    // ✅ With tryLock — backs off and retries
    public boolean transferWithRetry(String userId, String productId) throws InterruptedException {

        while (true) {
            boolean gotWallet    = walletLock.tryLock(100, TimeUnit.MILLISECONDS);
            boolean gotInventory = inventoryLock.tryLock(100, TimeUnit.MILLISECONDS);

            if (gotWallet && gotInventory) {
                try {
                    System.out.println("Deducting wallet + reserving inventory for " + userId);
                    Thread.sleep(200);
                    return true;
                } finally {
                    inventoryLock.unlock();
                    walletLock.unlock();
                }
            } else {
                // Release whatever we got — back off and retry
                if (gotWallet)    walletLock.unlock();
                if (gotInventory) inventoryLock.unlock();

                System.out.println("Couldn't get both locks — backing off...");
                Thread.sleep(50 + (long)(Math.random() * 50)); // randomized backoff
            }
        }
    }
}
```

---

## 11.4 `lockInterruptibly()` — Cancellable Lock Acquisition

With `synchronized`, a thread waiting for a lock **cannot be interrupted**. With `lockInterruptibly()`, it can:

```java
public class InterruptibleLock {

    private final ReentrantLock lock = new ReentrantLock();

    public void longRunningTask(String taskId) throws InterruptedException {

        // If this thread is interrupted while waiting for the lock
        // → InterruptedException is thrown immediately
        // → thread does NOT acquire the lock
        lock.lockInterruptibly();

        try {
            System.out.println("[" + taskId + "] Acquired lock. Working...");
            Thread.sleep(3000);
            System.out.println("[" + taskId + "] Done.");
        } finally {
            lock.unlock();
        }
    }

    public static void main(String[] args) throws InterruptedException {

        InterruptibleLock demo = new InterruptibleLock();

        Thread t1 = new Thread(() -> {
            try { demo.longRunningTask("TASK-1"); }
            catch (InterruptedException e) {
                System.out.println("[TASK-1] Interrupted while waiting for lock!");
            }
        }, "task-1");

        Thread t2 = new Thread(() -> {
            try { demo.longRunningTask("TASK-2"); }
            catch (InterruptedException e) {
                System.out.println("[TASK-2] Interrupted while waiting for lock!");
            }
        }, "task-2");

        t1.start();
        Thread.sleep(100); // let t1 acquire lock first
        t2.start();        // t2 now waits for lock

        Thread.sleep(500);
        t2.interrupt();    // cancel t2 — it was waiting for lock

        t1.join();
        t2.join();
    }
}
```

**Output:**
```
[TASK-1] Acquired lock. Working...
[TASK-2] Interrupted while waiting for lock!
[TASK-1] Done.
```

`TASK-2` was cancelled cleanly while waiting — impossible with `synchronized`.

---

## 11.5 Fairness — Preventing Thread Starvation

```java
// Unfair lock (default) — any waiting thread can acquire
ReentrantLock unfairLock = new ReentrantLock();

// Fair lock — threads acquire in arrival order (FIFO)
ReentrantLock fairLock = new ReentrantLock(true);
```

```
UNFAIR (default):
Thread A waits: ──────────────────────────────────── still waiting
Thread B waits: ──────────────────── acquired! ←── jumped the queue
Thread C waits: ──────── acquired! ←── jumped the queue
Thread A:                                         ← STARVED

FAIR:
Thread A waits: ──────── acquired first (arrived first) ✓
Thread B waits: ─────────────── acquired second ✓
Thread C waits: ──────────────────────── acquired third ✓
```

### Fair vs Unfair — Performance Trade-off

```
Unfair (default): Higher throughput
  → OS can hand lock to whichever thread is scheduled next
  → Less context switching overhead
  → Risk of starvation for unlucky threads

Fair:             Lower throughput, no starvation
  → Must maintain a queue of waiting threads
  → Each acquisition requires checking the queue
  → 10-100x slower throughput in high-contention scenarios
  → Use when starvation is genuinely a concern

ShopSphere: Use fair lock for payment processing (every order must be served)
            Use unfair lock for cache updates (throughput matters more)
```

---

## 11.6 `Condition` — Multiple Wait Sets

`synchronized` has one wait set per object (`wait()`/`notify()`). `ReentrantLock` lets you create **multiple named condition variables** — separate wait sets for different conditions.

This solves the `notifyAll()` inefficiency in Chapter 6's Producer-Consumer.

```java
import java.util.concurrent.locks.*;

public class ConditionProducerConsumer {

    private final ReentrantLock lock = new ReentrantLock();

    // TWO separate wait sets — producers wait on one, consumers on other
    private final Condition notFull  = lock.newCondition(); // producers wait here
    private final Condition notEmpty = lock.newCondition(); // consumers wait here

    private final String[] buffer;
    private int head = 0, tail = 0, count = 0;

    public ConditionProducerConsumer(int capacity) {
        buffer = new String[capacity];
    }

    public void put(String item) throws InterruptedException {
        lock.lock();
        try {
            while (count == buffer.length) {
                notFull.await(); // producer waits on notFull — releases lock
            }
            buffer[tail] = item;
            tail = (tail + 1) % buffer.length;
            count++;
            System.out.println("[producer] Put: " + item + " | size: " + count);

            notEmpty.signal(); // wake ONE consumer — not all threads
            // ← precise: only wakes consumers, not other producers
        } finally {
            lock.unlock();
        }
    }

    public String take() throws InterruptedException {
        lock.lock();
        try {
            while (count == 0) {
                notEmpty.await(); // consumer waits on notEmpty — releases lock
            }
            String item = buffer[head];
            head = (head + 1) % buffer.length;
            count--;
            System.out.println("[consumer] Took: " + item + " | size: " + count);

            notFull.signal(); // wake ONE producer — not all threads
            // ← precise: only wakes producers, not other consumers
            return item;
        } finally {
            lock.unlock();
        }
    }
}
```

### Why This Is Better Than Chapter 6's `notifyAll()`

```
Chapter 6 (synchronized + notifyAll):
  notifyAll() wakes ALL waiting threads — producers AND consumers
  They all race for the lock
  Only one wins — the rest go back to sleep
  Wasted context switches

Chapter 11 (Condition):
  notFull.signal()  → wakes ONLY a waiting producer
  notEmpty.signal() → wakes ONLY a waiting consumer
  Zero wasted wakeups
  Better performance under high contention
```

### `Condition` Methods Map to `Object` Methods

```
lock.newCondition().await()        ←→  object.wait()
lock.newCondition().signal()       ←→  object.notify()
lock.newCondition().signalAll()    ←→  object.notifyAll()
lock.newCondition().await(timeout) ←→  object.wait(timeout)
```

---

## 11.7 `ReadWriteLock` — Multiple Readers, One Writer

The biggest performance win from the locks package.

### The Problem With `synchronized` for Read-Heavy Data

```
Product catalog: 1000 reads/second, 1 write/second (price update)

With synchronized:
  All 1000 reads take turns — one at a time
  Even though reads don't modify anything
  No reason readers should block each other

With ReadWriteLock:
  All 1000 reads proceed SIMULTANEOUSLY
  Only when a write happens do readers wait
  Massive throughput improvement
```

### The Rules

```
Multiple readers    → allowed simultaneously (no writer active)
One writer          → exclusive — all readers and writers wait
Reader + Writer     → writer waits for readers to finish first
Two writers         → one waits for the other
```

```java
import java.util.concurrent.locks.*;

public class ProductCatalogCache {

    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock  = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    private final Map<String, Product> catalog = new HashMap<>();

    // READ — multiple threads can call this simultaneously
    public Product getProduct(String id) {
        readLock.lock();
        try {
            return catalog.get(id); // concurrent reads — no blocking between readers
        } finally {
            readLock.unlock();
        }
    }

    // READ — multiple threads simultaneously
    public List<Product> searchByCategory(String category) {
        readLock.lock();
        try {
            return catalog.values().stream()
                .filter(p -> p.category().equals(category))
                .collect(Collectors.toList());
        } finally {
            readLock.unlock();
        }
    }

    // WRITE — exclusive access
    public void updatePrice(String id, double newPrice) {
        writeLock.lock();
        try {
            // While this runs: ALL readers and writers wait
            Product existing = catalog.get(id);
            if (existing != null) {
                catalog.put(id, new Product(id, existing.name(),
                    existing.category(), newPrice));
                System.out.println("Price updated for: " + id);
            }
        } finally {
            writeLock.unlock();
        }
    }

    // WRITE — exclusive
    public void addProduct(Product product) {
        writeLock.lock();
        try {
            catalog.put(product.id(), product);
        } finally {
            writeLock.unlock();
        }
    }

    record Product(String id, String name, String category, double price) {}
}
```

### Throughput Comparison

```
Scenario: 10 threads reading simultaneously, 1 thread writing occasionally

synchronized:
  10 readers take turns → 10x slower than necessary
  Writer waits, readers wait — everyone waiting

ReadWriteLock:
  10 readers proceed simultaneously → full throughput
  Writer waits only for active readers to finish
  Readers wait only during active writes (milliseconds)

Speedup for read-heavy workloads: 5x to 50x depending on read:write ratio
```

---

## 11.8 `StampedLock` — The Most Advanced Lock (Java 8+)

`ReadWriteLock` has one weakness: **read lock starvation of writers**. If readers keep arriving, a writer may wait a very long time. `StampedLock` solves this with **optimistic reads**.

```java
import java.util.concurrent.locks.StampedLock;

public class StampedLockDemo {

    private final StampedLock lock = new StampedLock();
    private double price = 0.0;

    // OPTIMISTIC READ — no lock acquired at all!
    // Fastest path — works when no writes are happening
    public double getPrice() {

        long stamp = lock.tryOptimisticRead(); // returns a stamp — no lock acquired
        double currentPrice = price;           // read the value

        // Validate: was there a write between tryOptimisticRead and now?
        if (!lock.validate(stamp)) {
            // A write happened — fall back to real read lock
            stamp = lock.readLock();
            try {
                currentPrice = price;
            } finally {
                lock.unlockRead(stamp);
            }
        }

        return currentPrice; // no lock held if validate() succeeded
    }

    // WRITE — exclusive
    public void updatePrice(double newPrice) {
        long stamp = lock.writeLock();
        try {
            price = newPrice;
        } finally {
            lock.unlockWrite(stamp);
        }
    }
}
```

```
Three lock modes in StampedLock:

1. writeLock()          — exclusive, like regular write lock
2. readLock()           — shared read lock, blocks writers
3. tryOptimisticRead()  — NO LOCK — just gets a stamp
                          validate(stamp) checks if a write occurred
                          If valid: return value (zero lock overhead)
                          If invalid: retry with real readLock()

Performance: optimistic read ≈ volatile read (virtually free)
             Only falls back to real lock on write contention
```

### When to Use Which Lock

```
Scenario                                    Use
────────────────────────────────────────────────────────────────
Simple mutual exclusion                     synchronized or ReentrantLock
Need tryLock() / timeout                    ReentrantLock
Need interruptible lock acquisition         ReentrantLock
Need fairness                               ReentrantLock(true)
Multiple condition variables                ReentrantLock + Condition
Read-heavy, infrequent writes               ReadWriteLock
Read-heavy, high performance critical path  StampedLock
```

---

## 11.9 `synchronized` vs `ReentrantLock` — Complete Comparison

```
Feature                      synchronized        ReentrantLock
────────────────────────────────────────────────────────────────
Syntax                       Built-in keyword    Explicit lock/unlock
Auto unlock on exception      ✅                  ✅ (with finally)
tryLock() / timeout           ❌                  ✅
Interruptible wait            ❌                  ✅ lockInterruptibly()
Fairness control              ❌                  ✅ new ReentrantLock(true)
Multiple conditions           ❌ (one per object) ✅ lock.newCondition()
Lock query (isLocked etc.)    ❌                  ✅
Performance (low contention)  Slightly faster     Slightly slower
Performance (high contention) Comparable          Comparable
Risk of forgetting unlock     N/A                 ❌ (must use finally)
Code verbosity                Low                 Higher
```

### The Rule

> **Default to `synchronized`.** Switch to `ReentrantLock` only when you need one of its specific features: `tryLock`, interruptibility, fairness, or multiple conditions.

---

## 11.10 Complete ShopSphere Example

Putting it all together — a realistic inventory service with read/write separation and tryLock for non-blocking reservation:

```java
public class InventoryService {

    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock   = rwLock.readLock();
    private final Lock writeLock  = rwLock.writeLock();

    private final ReentrantLock reservationLock = new ReentrantLock();

    private final Map<String, Integer> stock = new HashMap<>();

    // ─── Reads — fully concurrent ─────────────────────────────────
    public int getStock(String productId) {
        readLock.lock();
        try {
            return stock.getOrDefault(productId, 0);
        } finally {
            readLock.unlock();
        }
    }

    public Map<String, Integer> getLowStockItems(int threshold) {
        readLock.lock();
        try {
            return stock.entrySet().stream()
                .filter(e -> e.getValue() <= threshold)
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
        } finally {
            readLock.unlock();
        }
    }

    // ─── Restock — exclusive write ────────────────────────────────
    public void restock(String productId, int quantity) {
        writeLock.lock();
        try {
            stock.merge(productId, quantity, Integer::sum);
            System.out.println("Restocked " + productId + " +" + quantity);
        } finally {
            writeLock.unlock();
        }
    }

    // ─── Reserve — tryLock with timeout ───────────────────────────
    // Non-blocking with 3s timeout — won't hold up HTTP request forever
    public ReservationResult reserve(String productId, int qty) {
        try {
            if (!reservationLock.tryLock(3, TimeUnit.SECONDS)) {
                return ReservationResult.TIMEOUT;
            }
            try {
                writeLock.lock(); // upgrade to write for stock modification
                try {
                    int available = stock.getOrDefault(productId, 0);
                    if (available < qty) {
                        return ReservationResult.INSUFFICIENT_STOCK;
                    }
                    stock.put(productId, available - qty);
                    System.out.println("Reserved " + qty + "x " + productId
                        + " | Remaining: " + (available - qty));
                    return ReservationResult.SUCCESS;
                } finally {
                    writeLock.unlock();
                }
            } finally {
                reservationLock.unlock();
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            return ReservationResult.INTERRUPTED;
        }
    }

    enum ReservationResult {
        SUCCESS, INSUFFICIENT_STOCK, TIMEOUT, INTERRUPTED
    }

    // ─── Demo ──────────────────────────────────────────────────────
    public static void main(String[] args) throws InterruptedException {

        InventoryService service = new InventoryService();
        service.restock("PROD-001", 10);

        ExecutorService pool = Executors.newFixedThreadPool(5);

        // 3 readers and 2 buyers simultaneously
        for (int i = 1; i <= 3; i++) {
            final int id = i;
            pool.submit(() ->
                System.out.println("[reader-" + id + "] Stock: "
                    + service.getStock("PROD-001"))
            );
        }

        for (int i = 1; i <= 4; i++) {
            final int id = i;
            pool.submit(() -> {
                ReservationResult result = service.reserve("PROD-001", 3);
                System.out.println("[buyer-" + id + "] Reservation: " + result);
            });
        }

        pool.shutdown();
        pool.awaitTermination(10, TimeUnit.SECONDS);
    }
}
```

**Output:**
```
Restocked PROD-001 +10
[reader-1] Stock: 10
[reader-2] Stock: 10
[reader-3] Stock: 10         ← 3 readers simultaneously
[buyer-1] Reserved 3x PROD-001 | Remaining: 7
[buyer-2] Reserved 3x PROD-001 | Remaining: 4
[buyer-3] Reserved 3x PROD-001 | Remaining: 1
[buyer-4] Reservation: INSUFFICIENT_STOCK  ← only 1 left, needs 3
```

---

## 11.11 Key Takeaways

| Concept | Remember This |
|---|---|
| `ReentrantLock` | Always `lock()` before try, `unlock()` in finally |
| `tryLock()` | Non-blocking — returns false immediately if locked |
| `tryLock(timeout)` | Waits up to timeout — returns false if still locked |
| `lockInterruptibly()` | Waiting thread can be interrupted — impossible with synchronized |
| Fairness | `new ReentrantLock(true)` — FIFO ordering, lower throughput |
| `Condition` | Multiple wait sets per lock — precise signal vs notifyAll |
| `ReadWriteLock` | Many readers simultaneously, one writer exclusively |
| `StampedLock` | Optimistic read — zero overhead on no-contention reads |
| Default choice | `synchronized` — switch to `ReentrantLock` only when you need its features |

---

## Interview Questions From This Chapter

**Q1. What can `ReentrantLock` do that `synchronized` cannot?**
> Four things: `tryLock()` for non-blocking/timeout acquisition; `lockInterruptibly()` for cancellable lock waiting; fairness mode via constructor for FIFO ordering; multiple `Condition` objects for separate wait sets. If none of these are needed, prefer `synchronized`.

**Q2. What is `tryLock()` and when would you use it?**
> `tryLock()` attempts to acquire the lock and returns immediately — `true` if acquired, `false` if not. `tryLock(timeout)` waits up to the specified duration. Use it when you want to avoid blocking indefinitely — for example, in HTTP request handlers where you'd rather return a "busy" response than hang the user's request.

**Q3. What is the difference between `ReadWriteLock` and `ReentrantLock`?**
> `ReentrantLock` is a single exclusive lock — one thread at a time regardless of read or write. `ReadWriteLock` has two locks: a read lock (shared — multiple threads simultaneously) and a write lock (exclusive). For read-heavy workloads, `ReadWriteLock` dramatically improves throughput since readers don't block each other.

**Q4. What is a `Condition` and how is it better than `wait()`/`notifyAll()`?**
> A `Condition` is a named wait set tied to a `ReentrantLock`. You can create multiple conditions per lock — one for producers (`notFull`) and one for consumers (`notEmpty`). `notFull.signal()` wakes only a waiting producer, not all threads. With `synchronized` and `notifyAll()`, every wakeup wakes all waiting threads regardless of which condition they're waiting on — causing wasted context switches.

**Q5. Why must `unlock()` always be in a `finally` block?**
> If an exception is thrown inside the `try` block and `unlock()` is not in `finally`, the lock is never released. Every other thread waiting for that lock will wait forever — a permanent deadlock. `synchronized` handles this automatically (releases lock when block exits for any reason). With `ReentrantLock` you're responsible — `finally` is the only way to guarantee release.

---

Ready for **Chapter 12: Atomic Variables** — `AtomicInteger`, `AtomicReference`, Compare-And-Swap internals, and why atomics outperform `synchronized` for single-variable operations?
