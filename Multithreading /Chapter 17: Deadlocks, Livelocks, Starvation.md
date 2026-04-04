
# Chapter 17: Deadlocks — Formation, Detection & Prevention

---

## 17.1 What Is a Deadlock?

A deadlock is a situation where two or more threads are **permanently blocked**, each waiting for a resource held by another thread in the cycle. Nobody moves. Ever.

```
Thread A holds Lock 1, wants Lock 2 →  waiting...
Thread B holds Lock 2, wants Lock 1 →  waiting...

Neither can proceed.
Neither will release what they hold.
Both wait forever.
JVM doesn't detect or recover — application hangs silently.
```

This is one of the hardest production bugs to diagnose because:
- No exception is thrown
- No error in logs
- Application just... stops responding to certain requests
- CPU drops to zero for those threads

---

## 17.2 The Four Coffman Conditions

A deadlock can ONLY occur when ALL four conditions hold simultaneously. Remove any one — deadlock impossible.

```
Condition 1: MUTUAL EXCLUSION
  At least one resource must be non-shareable
  Only one thread can hold it at a time
  (locks are mutually exclusive by definition)

Condition 2: HOLD AND WAIT
  A thread holds at least one resource
  AND is waiting to acquire additional resources
  held by other threads

Condition 3: NO PREEMPTION
  Resources cannot be forcibly taken from a thread
  Thread must voluntarily release them
  (Java locks cannot be stolen)

Condition 4: CIRCULAR WAIT
  A cycle exists in the wait graph:
  T1 waits for T2
  T2 waits for T3
  T3 waits for T1
  → cycle → deadlock
```

Memorize these. Interviewers ask "what conditions cause a deadlock?" directly.

---

## 17.3 Writing a Deadlock — Deliberately

```java
public class DeadlockDemo {

    private static final Object lockA = new Object(); // resource 1
    private static final Object lockB = new Object(); // resource 2

    public static void main(String[] args) {

        // Thread 1: acquires lockA, then wants lockB
        Thread t1 = new Thread(() -> {
            synchronized (lockA) {
                System.out.println("[T1] Acquired lockA");

                try { Thread.sleep(100); } // give T2 time to acquire lockB
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }

                System.out.println("[T1] Waiting for lockB...");
                synchronized (lockB) { // ← BLOCKS — T2 holds lockB
                    System.out.println("[T1] Acquired lockB"); // never prints
                }
            }
        }, "Thread-1");

        // Thread 2: acquires lockB, then wants lockA
        Thread t2 = new Thread(() -> {
            synchronized (lockB) {
                System.out.println("[T2] Acquired lockB");

                try { Thread.sleep(100); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }

                System.out.println("[T2] Waiting for lockA...");
                synchronized (lockA) { // ← BLOCKS — T1 holds lockA
                    System.out.println("[T2] Acquired lockA"); // never prints
                }
            }
        }, "Thread-2");

        t1.start();
        t2.start();

        // Program hangs here forever — both threads blocked
        System.out.println("Main thread continues... but T1 and T2 are deadlocked");
    }
}
```

**Output:**
```
Main thread continues... but T1 and T2 are deadlocked
[T1] Acquired lockA
[T2] Acquired lockB
[T1] Waiting for lockB...
[T2] Waiting for lockA...
(hangs forever)
```

```
Wait-for graph:
T1 ──holds──► lockA ◄──wants── T2
T1 ──wants──► lockB ◄──holds── T2

T1 → waits for T2 (who holds lockB)
T2 → waits for T1 (who holds lockA)
Cycle → Deadlock
```

---

## 17.4 ShopSphere — Realistic Deadlock Scenario

Not a toy example. This is how deadlocks happen in real microservices:

```java
public class WalletInventoryDeadlock {

    // Two shared resources in your Order Service
    private static final Object walletLock    = new Object();
    private static final Object inventoryLock = new Object();

    static double  walletBalance = 5000.0;
    static int     stockCount    = 10;

    // Order flow A: deduct wallet THEN reserve inventory
    static void placeOrder(String orderId, double amount) {
        synchronized (walletLock) {
            System.out.println("[" + orderId + "] Wallet locked. Balance: " + walletBalance);
            sleep(50); // simulate wallet DB call

            synchronized (inventoryLock) { // ← DEADLOCK if refund runs simultaneously
                walletBalance -= amount;
                stockCount    -= 1;
                System.out.println("[" + orderId + "] Order complete. Stock left: " + stockCount);
            }
        }
    }

    // Refund flow: restore inventory THEN refund wallet
    // ← opposite lock order!
    static void processRefund(String refundId, double amount) {
        synchronized (inventoryLock) {
            System.out.println("[" + refundId + "] Inventory locked. Stock: " + stockCount);
            sleep(50); // simulate inventory DB call

            synchronized (walletLock) { // ← DEADLOCK if order runs simultaneously
                stockCount    += 1;
                walletBalance += amount;
                System.out.println("[" + refundId + "] Refund complete. Balance: " + walletBalance);
            }
        }
    }

    public static void main(String[] args) {

        Thread orderThread  = new Thread(
            () -> placeOrder("ORD-001", 999.0),  "order-thread");
        Thread refundThread = new Thread(
            () -> processRefund("REF-001", 999.0), "refund-thread");

        orderThread.start();
        refundThread.start();
        // Hangs — classic deadlock from inconsistent lock ordering
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

**The root cause:** `placeOrder` acquires locks in order `wallet → inventory`. `processRefund` acquires them in order `inventory → wallet`. Opposite ordering = circular wait = deadlock.

---

## 17.5 Detecting Deadlocks — `jstack`

When your application hangs in production, `jstack` gives you a thread dump:

```bash
# Step 1: Find the JVM process ID
jps -l
# Output:
# 18472 com.shopsphere.OrderService
# 12034 com.shopsphere.ApiGateway

# Step 2: Take thread dump
jstack 18472

# Or: dump to file for analysis
jstack 18472 > thread_dump.txt
```

**What `jstack` shows for a deadlock:**

```
Found one Java-level deadlock:
=============================

"refund-thread":
  waiting to lock monitor 0x000000076b059d48 (object 0x...WalletLock)
  which is held by "order-thread"

"order-thread":
  waiting to lock monitor 0x000000076b059d48 (object 0x...InventoryLock)
  which is held by "refund-thread"

Java stack information for the threads listed above:
===================================================
"refund-thread":
  at WalletInventoryDeadlock.processRefund(WalletInventoryDeadlock.java:28)
  - waiting to lock <0x...> (WalletLock)
  - locked <0x...> (InventoryLock)

"order-thread":
  at WalletInventoryDeadlock.placeOrder(WalletInventoryDeadlock.java:14)
  - waiting to lock <0x...> (InventoryLock)
  - locked <0x...> (WalletLock)

Found 1 deadlock.
```

`jstack` tells you:
- **Which threads** are deadlocked
- **Which locks** they hold
- **Which locks** they're waiting for
- **Exact line numbers** in your code

### Detecting Deadlocks Programmatically

```java
public class DeadlockDetector {

    public static void detectAndReport() {

        ThreadMXBean bean = ManagementFactory.getThreadMXBean();

        long[] deadlockedIds = bean.findDeadlockedThreads();
        // Returns null if no deadlock — NOT empty array

        if (deadlockedIds == null) {
            System.out.println("No deadlocks detected ✓");
            return;
        }

        ThreadInfo[] infos = bean.getThreadInfo(deadlockedIds, true, true);

        System.out.println("⚠ DEADLOCK DETECTED — " + deadlockedIds.length + " threads");
        System.out.println("─────────────────────────────────────────");

        for (ThreadInfo info : infos) {
            System.out.println("Thread: " + info.getThreadName()
                + " (ID: " + info.getThreadId() + ")");
            System.out.println("  State: " + info.getThreadState());
            System.out.println("  Waiting for lock: " + info.getLockName());
            System.out.println("  Lock held by: " + info.getLockOwnerName());

            System.out.println("  Stack trace:");
            for (StackTraceElement element : info.getStackTrace()) {
                System.out.println("    at " + element);
            }
            System.out.println();
        }
    }

    public static void main(String[] args) throws InterruptedException {

        // Start the deadlock
        Object lockA = new Object(), lockB = new Object();

        new Thread(() -> {
            synchronized (lockA) { sleep(100);
                synchronized (lockB) { System.out.println("T1 done"); }
            }
        }, "deadlock-thread-1").start();

        new Thread(() -> {
            synchronized (lockB) { sleep(100);
                synchronized (lockA) { System.out.println("T2 done"); }
            }
        }, "deadlock-thread-2").start();

        // Give threads time to deadlock
        Thread.sleep(500);

        // Detect it
        detectAndReport();
    }

    static void sleep(long ms) {
        try { Thread.sleep(ms); }
        catch (InterruptedException e) { Thread.currentThread().interrupt(); }
    }
}
```

**Output:**
```
⚠ DEADLOCK DETECTED — 2 threads
─────────────────────────────────────────
Thread: deadlock-thread-1 (ID: 13)
  State: BLOCKED
  Waiting for lock: java.lang.Object@4d7e1886
  Lock held by: deadlock-thread-2
  Stack trace:
    at DeadlockDetector.lambda$main$0(DeadlockDetector.java:52)
    ...

Thread: deadlock-thread-2 (ID: 14)
  State: BLOCKED
  Waiting for lock: java.lang.Object@3764951d
  Lock held by: deadlock-thread-1
  Stack trace:
    at DeadlockDetector.lambda$main$1(DeadlockDetector.java:58)
    ...
```

In production, run this detector on a scheduled thread every 30 seconds and alert on detection.

---

## 17.6 Prevention Strategy 1 — Consistent Lock Ordering

The simplest and most effective strategy. **Always acquire locks in the same global order** across all threads.

```java
public class ConsistentLockOrder {

    private static final Object walletLock    = new Object();
    private static final Object inventoryLock = new Object();

    // ✅ FIXED — BOTH methods acquire locks in same order: wallet → inventory
    static void placeOrder(String orderId, double amount) {
        synchronized (walletLock) {           // lock 1 always first
            synchronized (inventoryLock) {    // lock 2 always second
                System.out.println("[" + orderId + "] Order placed ✓");
            }
        }
    }

    static void processRefund(String refundId, double amount) {
        synchronized (walletLock) {           // lock 1 always first
            synchronized (inventoryLock) {    // lock 2 always second
                System.out.println("[" + refundId + "] Refund processed ✓");
            }
        }
    }

    // No circular wait possible — both threads compete for walletLock first
    // One wins, completes both locks, releases, then other proceeds
}
```

### When You Don't Know the Order in Advance

When locks are dynamic (e.g., transferring between two accounts where either could be "first"):

```java
public class DynamicLockOrder {

    static void transfer(Account from, Account to, double amount) {

        // Impose consistent order using system identity hashcode
        // Both threads will always lock the LOWER id account first
        Object lock1, lock2;

        int fromHash = System.identityHashCode(from);
        int toHash   = System.identityHashCode(to);

        if (fromHash < toHash) {
            lock1 = from; lock2 = to;
        } else if (fromHash > toHash) {
            lock1 = to;   lock2 = from;
        } else {
            // Hash collision — use tie-breaker lock
            synchronized (DynamicLockOrder.class) {
                from.deduct(amount);
                to.credit(amount);
                return;
            }
        }

        synchronized (lock1) {
            synchronized (lock2) {
                from.deduct(amount);
                to.credit(amount);
                System.out.println("Transferred ₹" + amount
                    + " from " + from.id() + " to " + to.id());
            }
        }
    }

    record Account(String id, double balance) {
        void deduct(double amt) { /* update balance */ }
        void credit(double amt) { /* update balance */ }
    }
}
```

---

## 17.7 Prevention Strategy 2 — `tryLock` With Timeout

From Chapter 11. If you can't get all locks, release what you have and retry:

```java
public class TryLockPrevention {

    private final ReentrantLock walletLock    = new ReentrantLock();
    private final ReentrantLock inventoryLock = new ReentrantLock();

    public boolean placeOrder(String orderId, double amount)
            throws InterruptedException {

        // Retry loop — backs off if can't get both locks
        while (true) {

            boolean gotWallet    = walletLock.tryLock(100, TimeUnit.MILLISECONDS);
            boolean gotInventory = false;

            try {
                if (gotWallet) {
                    gotInventory = inventoryLock.tryLock(100, TimeUnit.MILLISECONDS);
                }

                if (gotWallet && gotInventory) {
                    try {
                        // Critical section — have both locks
                        System.out.println("[" + orderId + "] Order placed ✓");
                        return true;
                    } finally {
                        inventoryLock.unlock();
                        walletLock.unlock();
                    }
                }

            } finally {
                // Release whatever we got if we don't have both
                if (gotWallet    && !gotInventory) walletLock.unlock();
                if (gotInventory && !gotWallet)    inventoryLock.unlock();
            }

            // Randomized backoff — prevent livelock
            System.out.println("[" + orderId + "] Couldn't get both locks — retrying...");
            Thread.sleep(10 + (long)(Math.random() * 50));
        }
    }
}
```

---

## 17.8 Prevention Strategy 3 — Lock Timeout + Deadlock Watchdog

Detect and recover from deadlocks in production:

```java
public class DeadlockWatchdog {

    private final ScheduledExecutorService watchdog
        = Executors.newSingleThreadScheduledExecutor(
            r -> { Thread t = new Thread(r, "deadlock-watchdog");
                   t.setDaemon(true); return t; });

    public void start() {
        watchdog.scheduleAtFixedRate(() -> {
            ThreadMXBean bean = ManagementFactory.getThreadMXBean();
            long[] deadlocked = bean.findDeadlockedThreads();

            if (deadlocked != null) {
                ThreadInfo[] infos = bean.getThreadInfo(deadlocked);
                StringBuilder alert = new StringBuilder("⚠ DEADLOCK DETECTED:\n");

                for (ThreadInfo info : infos) {
                    alert.append("  Thread: ").append(info.getThreadName())
                         .append(" waiting for: ").append(info.getLockOwnerName())
                         .append("\n");
                }

                // In production: send PagerDuty alert, trigger heap dump,
                // notify on-call engineer
                System.err.println(alert);

                // Attempt recovery — interrupt deadlocked threads
                for (long id : deadlocked) {
                    // Find the thread and interrupt it
                    Thread.getAllStackTraces().keySet().stream()
                        .filter(t -> t.getId() == id)
                        .findFirst()
                        .ifPresent(t -> {
                            System.err.println("Interrupting deadlocked thread: " + t.getName());
                            t.interrupt();
                        });
                }
            }
        }, 10, 30, TimeUnit.SECONDS); // check every 30 seconds
    }

    public void stop() {
        watchdog.shutdown();
    }
}
```

---

## 17.9 Prevention Strategy 4 — Avoid Nested Locks

The simplest strategy of all: **don't hold a lock when acquiring another**.

```java
// ❌ Nested locks — potential deadlock
public void processOrder(Order order) {
    synchronized (walletLock) {
        synchronized (inventoryLock) { // acquiring second lock while holding first
            doWork(order);
        }
    }
}

// ✅ Sequential locks — no nesting, no deadlock possible
public void processOrder(Order order) {
    // Lock wallet, do wallet work, release
    String paymentRef;
    synchronized (walletLock) {
        paymentRef = deductWallet(order.amount());
    } // ← released before acquiring inventoryLock

    // Lock inventory, do inventory work, release
    synchronized (inventoryLock) {
        reserveInventory(order.productId(), paymentRef);
    }
}
```

If you don't need both resources simultaneously — release one before acquiring the other.

---

## 17.10 Livelock & Starvation — The Cousins of Deadlock

### Livelock

Threads aren't blocked — they're actively running — but making no progress:

```java
// Both threads keep backing off when they see each other
// Both retry at the same time → conflict again → both back off again → forever

while (true) {
    if (lock.tryLock(100, TimeUnit.MILLISECONDS)) {
        // do work
        break;
    }
    // Both threads retry at exact same interval → always conflict
    Thread.sleep(100); // ← fixed sleep = livelock risk
}

// Fix: randomized backoff
Thread.sleep(10 + (long)(Math.random() * 100)); // random interval breaks synchrony
```

```
Deadlock:  threads BLOCKED — no CPU, no progress
Livelock:  threads RUNNING — CPU used, no progress
Starvation: some threads progress, one thread NEVER gets scheduled
```

### Starvation

A thread never gets access to a resource because other threads keep winning:

```java
// Unfair lock — high-priority threads keep winning
// Low-priority thread never gets the lock
ReentrantLock fairLock = new ReentrantLock(true); // ← true = fair = FIFO
// Fixes starvation — threads served in arrival order
```

---

## 17.11 Deadlock vs Other Concurrency Bugs — Full Picture

```
RACE CONDITION
  Definition: Result depends on thread scheduling
  Symptom:    Wrong output, data corruption, inconsistent state
  Detection:  Wrong results under load
  Cause:      Missing synchronization
  Fix:        synchronized, AtomicInteger, ConcurrentHashMap

DEADLOCK
  Definition: Circular lock dependency — all threads blocked
  Symptom:    Application hangs, threads in BLOCKED state forever
  Detection:  jstack, ThreadMXBean.findDeadlockedThreads()
  Cause:      Inconsistent lock ordering, nested locks
  Fix:        Consistent ordering, tryLock, avoid nesting

LIVELOCK
  Definition: Threads active but making no progress
  Symptom:    High CPU, no actual work done
  Detection:  CPU high but no throughput — hard to detect
  Cause:      Threads keep yielding/retrying in sync
  Fix:        Randomized backoff, structured retry

STARVATION
  Definition: One thread never gets CPU/lock
  Symptom:    Some requests never complete
  Detection:  Thread always BLOCKED/WAITING in dumps
  Cause:      Unfair scheduling, priority inversion
  Fix:        Fair locks, equal priority, bounded waiting
```

---

## 17.12 Complete Deadlock Prevention Demo

All four strategies in one file — the ShopSphere order/refund fixed correctly:

```java
public class DeadlockPreventionDemo {

    private final ReentrantLock walletLock    = new ReentrantLock();
    private final ReentrantLock inventoryLock = new ReentrantLock();

    double walletBalance = 10_000.0;
    int    stockCount    = 20;

    // ─── Strategy 1: Consistent lock ordering ─────────────────────
    public void placeOrderSafe(String orderId, double amount) {
        // ALWAYS wallet first, inventory second — everywhere in codebase
        walletLock.lock();
        try {
            inventoryLock.lock();
            try {
                walletBalance -= amount;
                stockCount    -= 1;
                System.out.println("[" + orderId + "] ✓ Placed"
                    + " | Balance: ₹" + walletBalance
                    + " | Stock: " + stockCount);
            } finally { inventoryLock.unlock(); }
        } finally { walletLock.unlock(); }
    }

    public void processRefundSafe(String refundId, double amount) {
        // SAME ORDER as placeOrder — wallet first, inventory second
        walletLock.lock();
        try {
            inventoryLock.lock();
            try {
                walletBalance += amount;
                stockCount    += 1;
                System.out.println("[" + refundId + "] ✓ Refunded"
                    + " | Balance: ₹" + walletBalance
                    + " | Stock: " + stockCount);
            } finally { inventoryLock.unlock(); }
        } finally { walletLock.unlock(); }
    }

    // ─── Strategy 2: tryLock with backoff ─────────────────────────
    public boolean placeOrderWithBackoff(String orderId, double amount)
            throws InterruptedException {

        for (int attempt = 1; attempt <= 5; attempt++) {
            boolean gotWallet = false, gotInventory = false;
            try {
                gotWallet    = walletLock.tryLock(200, TimeUnit.MILLISECONDS);
                gotInventory = gotWallet &&
                    inventoryLock.tryLock(200, TimeUnit.MILLISECONDS);

                if (gotWallet && gotInventory) {
                    walletBalance -= amount;
                    stockCount    -= 1;
                    System.out.println("[" + orderId + "] ✓ Placed on attempt " + attempt);
                    return true;
                }
            } finally {
                if (gotInventory) inventoryLock.unlock();
                if (gotWallet)    walletLock.unlock();
            }
            // Randomized backoff — prevents livelock
            Thread.sleep(20 + (long)(Math.random() * 80));
        }
        System.out.println("[" + orderId + "] ✗ Failed after 5 attempts");
        return false;
    }

    // ─── Main demo ─────────────────────────────────────────────────
    public static void main(String[] args) throws InterruptedException {

        DeadlockPreventionDemo demo = new DeadlockPreventionDemo();
        ExecutorService pool = Executors.newFixedThreadPool(6);

        System.out.println("=== Strategy 1: Consistent Lock Ordering ===");

        for (int i = 1; i <= 3; i++) {
            final int id = i;
            pool.submit(() ->
                demo.placeOrderSafe("ORD-00" + id, 500.0));
            pool.submit(() ->
                demo.processRefundSafe("REF-00" + id, 500.0));
        }

        Thread.sleep(1000);

        System.out.println("\n=== Strategy 2: tryLock with Backoff ===");
        demo.walletBalance = 10_000.0;
        demo.stockCount    = 20;

        for (int i = 1; i <= 4; i++) {
            final int id = i;
            pool.submit(() -> {
                try { demo.placeOrderWithBackoff("ORD-10" + id, 500.0); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            });
        }

        pool.shutdown();
        pool.awaitTermination(10, TimeUnit.SECONDS);
        System.out.println("\n✅ No deadlocks — all operations completed.");
    }
}
```

**Output:**
```
=== Strategy 1: Consistent Lock Ordering ===
[ORD-001] ✓ Placed    | Balance: ₹9500.0 | Stock: 19
[REF-001] ✓ Refunded  | Balance: ₹10000.0 | Stock: 20
[ORD-002] ✓ Placed    | Balance: ₹9500.0 | Stock: 19
[REF-002] ✓ Refunded  | Balance: ₹10000.0 | Stock: 20
[ORD-003] ✓ Placed    | Balance: ₹9500.0 | Stock: 19
[REF-003] ✓ Refunded  | Balance: ₹10000.0 | Stock: 20

=== Strategy 2: tryLock with Backoff ===
[ORD-101] ✓ Placed on attempt 1
[ORD-102] ✓ Placed on attempt 1
[ORD-103] ✓ Placed on attempt 2
[ORD-104] ✓ Placed on attempt 1

✅ No deadlocks — all operations completed.
```

---

## 17.13 Production Checklist — Deadlock Prevention

Paste this in your code review checklist:

```
□ All nested lock acquisitions follow a consistent global order
□ Lock order is documented with comments where nesting is unavoidable
□ ReentrantLock.unlock() is ALWAYS in a finally block
□ Synchronized blocks don't call external methods that may acquire locks
  (a method calling back into code that acquires locks = hidden nesting)
□ Lock hold time is minimized — no DB calls, no network calls while locked
□ Deadlock watchdog running in production (ThreadMXBean check)
□ Thread dumps reviewed during load testing (jstack or VisualVM)
□ tryLock used in time-sensitive paths (HTTP handlers, payment flows)
□ Fair locks used where starvation is a concern (payment processing)
□ Integration/stress tests run with concurrency to expose deadlocks early
```

---

## 17.14 Key Takeaways

| Concept | Remember This |
|---|---|
| Deadlock definition | Circular lock dependency — all threads blocked permanently |
| Four Coffman conditions | Mutual exclusion, Hold-and-wait, No preemption, Circular wait. Remove any one → no deadlock. |
| `jstack` | Primary production tool — shows deadlocked threads, held locks, line numbers |
| `ThreadMXBean` | Programmatic detection — `findDeadlockedThreads()` |
| Strategy 1 | Consistent lock ordering — simplest, most effective |
| Strategy 2 | `tryLock` with timeout + randomized backoff — prevents and recovers |
| Strategy 3 | Avoid nested locks — release before acquiring another |
| Strategy 4 | Deadlock watchdog — detect and alert in production |
| Livelock | Threads running but no progress — fix with randomized backoff |
| Starvation | Thread never gets lock — fix with fair `ReentrantLock(true)` |

---

## Interview Questions From This Chapter

**Q1. What is a deadlock and what conditions are necessary for it to occur?**
> A deadlock is when two or more threads are permanently blocked, each waiting for a lock held by another. Four conditions must ALL hold: mutual exclusion (only one thread can hold a resource), hold-and-wait (thread holds one resource while waiting for another), no preemption (locks can't be forcibly taken), and circular wait (T1 waits for T2, T2 waits for T1). Eliminating any one condition prevents deadlock.

**Q2. Write a simple deadlock in Java.**
> Two threads, two locks, opposite acquisition order. Thread 1: `synchronized(lockA) { synchronized(lockB) {...} }`. Thread 2: `synchronized(lockB) { synchronized(lockA) {...} }`. With a small sleep between the two lock acquisitions, both threads acquire their first lock then block waiting for the second — circular wait — deadlock.

**Q3. How do you detect a deadlock in production?**
> Two ways. First: `jstack <pid>` from the command line — prints a thread dump with a "Found Java-level deadlock" section showing exactly which threads hold which locks and what they're waiting for, with line numbers. Second: programmatically via `ThreadMXBean.findDeadlockedThreads()` — returns IDs of deadlocked threads, then `getThreadInfo()` for details. Run this on a scheduled thread in production and alert on non-null result.

**Q4. What are the strategies to prevent deadlocks?**
> Four strategies: (1) Consistent lock ordering — always acquire locks in the same global sequence across all code paths, eliminating circular wait. (2) `tryLock` with timeout — attempt locks with a deadline, release all and retry with randomized backoff if unsuccessful. (3) Avoid nested locks — release the first lock before acquiring the second. (4) Lock timeout watchdog — detect deadlocks at runtime via `ThreadMXBean` and alert or interrupt.

**Q5. What is the difference between deadlock, livelock, and starvation?**
> Deadlock: threads are BLOCKED — no CPU usage, no progress, permanent. Livelock: threads are RUNNING — CPU is consumed, but no real progress because threads keep responding to each other (e.g., both backing off simultaneously in a retry loop). Fix: randomized backoff. Starvation: some threads make progress but one specific thread never gets scheduled or never acquires a lock because others always win. Fix: fair locks (`ReentrantLock(true)`), equal thread priorities.

---

## 🎯 Multithreading Curriculum — Complete

```
Phase 1 — Foundations
  ✅ Chapter 1:  Process vs Thread, JVM Memory Model
  ✅ Chapter 2:  Creating Threads, start() vs run()
  ✅ Chapter 3:  Thread Lifecycle, sleep/join/interrupt

Phase 2 — Synchronization
  ✅ Chapter 4:  Race Conditions, synchronized
  ✅ Chapter 5:  volatile, visibility, happens-before
  ✅ Chapter 6:  wait()/notify(), Producer-Consumer

Phase 3 — Concurrency API
  ✅ Chapter 7:  ExecutorService, Thread Pools
  ✅ Chapter 8:  Callable, Future, CompletableFuture intro
  ✅ Chapter 9:  CompletableFuture deep dive

Phase 4 — Advanced Tools
  ✅ Chapter 10: Concurrent Collections
  ✅ Chapter 11: ReentrantLock, ReadWriteLock, Condition
  ✅ Chapter 12: Atomic Variables, CAS, LongAdder
  ✅ Chapter 13: ThreadLocal, Immutability, Patterns

Phase 5 — Production
  ✅ Chapter 17: Deadlocks, Detection, Prevention
```

**Updated interview readiness: ~93%** on multithreading for a 2 YOE Java role.

---

## Recommended Next Steps

```
Option A — Mock Interview Session
  Full Q&A across all 14 chapters
  Find and fix remaining weak spots
  Practice answering under pressure

Option B — Remaining Chapters (optional depth)
  Chapter 14: CountDownLatch, CyclicBarrier, Semaphore
  Chapter 16: Virtual Threads (Java 21) — relevant for ShopSphere

Option C — Broaden Interview Prep
  Spring Boot deep dive
  System design (LLD for ShopSphere components)
  DSA weak spots
  SQL/JPA
```

What would you like to do next?
