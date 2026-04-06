
# Chapter 4: Race Conditions & the `synchronized` Keyword

---

## 4.1 The Problem — Race Condition

A **race condition** occurs when two or more threads access and modify the same shared data at the same time, and the final result depends on which thread finishes first (the "race").

- The outcome is unpredictable.
- You get different results every time you run the program.
- It often leads to wrong data, bugs that are very hard to reproduce and debug.

### The Broken Counter

```java
public class BrokenCounter {

    private int count = 0; // This is SHARED data (lives on the HEAP)

    public void increment() {
        count++;// Looks simple, but it's NOT safe!
    }

    public int getCount() {
        return count;
    }

    public static void main(String[] args) throws InterruptedException {

        BrokenCounter counter = new BrokenCounter();

       // Two threads, each trying to increment 1000 times
        // Expected final count: 2000
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) counter.increment();
        }, "thread-1");

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) counter.increment();
        }, "thread-2");

        t1.start();
        t2.start();
        t1.join(); // Wait for thread-1 to finish
        t2.join(); // Wait for thread-2 to finish


        System.out.println("Expected: 2000");
        System.out.println("Actual:   " + counter.getCount()); // 1643? 1821? 1956? Never 2000
    }
}
```

**Output (varies every run):**
```
Expected: 2000
Actual:   1743
```

Run it 5 times — you get 5 different wrong answers. Never 2000.

---

## 4.2 Why Does This Happen?

`count++` **looks** like one operation. It is actually **three**:

```
count++  compiles to:
  1. READ  → load count from heap into CPU register (count = 5)
  2. ADD   → increment in register           (register = 6)
  3. WRITE → store back to heap              (count = 6)
```

Now put two threads doing this simultaneously:

```
Time │  Thread 1                    Thread 2
─────┼────────────────────────────────────────────────
  1  │  READ  count = 5
  2  │                              READ  count = 5   ← both read 5!
  3  │  ADD   register = 6
  4  │                              ADD   register = 6
  5  │  WRITE count = 6
  6  │                              WRITE count = 6   ← overwrites T1's write!
─────┴────────────────────────────────────────────────
Expected: count = 7  (two increments)
Actual:   count = 6  (one increment lost)
```

Thread 2 read the **stale value** (5) before Thread 1 wrote back (6). One increment was silently lost. Multiply this across 2000 operations — you lose hundreds of increments.

This is a **lost update** — the most common form of race condition.

---

## 4.3 The Solution — `synchronized`

`synchronized` ensures that only **one thread at a time** can execute a block of code. It works using a **monitor lock** (also called intrinsic lock) that every Java object has built into it.

```java
public class FixedCounter {

    private int count = 0;

    // synchronized method — only one thread can execute this at a time
    public synchronized void increment() {
        count++; // now atomic from other threads' perspective
    }

    public synchronized int getCount() {
        return count;
    }

    public static void main(String[] args) throws InterruptedException {

        FixedCounter counter = new FixedCounter();

        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) counter.increment();
        }, "thread-1");

        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 1000; i++) counter.increment();
        }, "thread-2");

        t1.start();
        t2.start();
        t1.join();
        t2.join();

        System.out.println("Expected: 2000");
        System.out.println("Actual:   " + counter.getCount()); // always 2000
    }
}
```

**Output (every single run):**
```
Expected: 2000
Actual:   2000
```

---

## 4.4 How `synchronized` Works Internally

Every Java object has an invisible **monitor lock**. `synchronized` acquires it on entry and releases it on exit — automatically, even if an exception is thrown.

```
Thread 1 calls increment()          Thread 2 calls increment()
         │                                    │
         ▼                                    ▼
  tries to acquire lock                tries to acquire lock
  on `counter` object                  on `counter` object
         │                                    │
  LOCK IS FREE                        LOCK HELD BY THREAD 1
         │                                    │
  acquires lock ✓                     goes BLOCKED ←─────────────┐
         │                                    │                   │
  READ count (5)                      waiting...                  │
  ADD  → 6                            waiting...                  │
  WRITE count (6)                     waiting...                  │
         │                                    │                   │
  releases lock ──────────────────────────────┘                   │
         │                                                        │
         ▼                            acquires lock ✓             │
  continues                                   │                   │
                                      READ count (6)              │
                                      ADD  → 7                    │
                                      WRITE count (7)             │
                                              │                   │
                                      releases lock               │
```

No two threads can be inside a `synchronized` block on the **same object** at the same time.

---

## 4.5 Two Forms of `synchronized`

### Form 1: Synchronized Method

The lock is acquired on `this` (the current object instance).

```java
public class OrderService {

    private int activeOrders = 0;

    // Lock acquired on `this` (the OrderService instance)
    public synchronized void placeOrder() {
        activeOrders++;
        System.out.println("Order placed. Active: " + activeOrders);
    }

    public synchronized void cancelOrder() {
        activeOrders--;
        System.out.println("Order cancelled. Active: " + activeOrders);
    }

    // For static methods — lock is on the CLASS object (OrderService.class)
    public static synchronized void printServiceInfo() {
        System.out.println("ShopSphere Order Service v1.0");
    }
}
```

### Form 2: Synchronized Block

More precise — you specify exactly which object's lock to acquire, and exactly which lines need protection. **Prefer this over synchronized methods** because it minimizes the locked section.

```java
public class CartService {

    private final Object cartLock = new Object(); // dedicated lock object
    private int itemCount = 0;
    private String lastModifiedBy = "";

    public void addItem(String userId, int quantity) {

        // ✅ Only the critical section is locked
        // expensive operations outside the lock run concurrently
        System.out.println("[" + userId + "] Validating item availability...");
        // simulate: DB check, network call — NOT locked, runs in parallel
        try { Thread.sleep(100); } catch (InterruptedException e) { Thread.currentThread().interrupt(); }

        synchronized (cartLock) { // ← only this section is mutual exclusive
            itemCount += quantity;
            lastModifiedBy = userId;
            System.out.println("[" + userId + "] Added " + quantity + " item(s). Total: " + itemCount);
        } // lock released here automatically
    }
}
```

---

## 4.6 Synchronized Method vs Block — The Key Difference

```java
public class LockScopeDemo {

    private final Object lock = new Object();
    private int data = 0;

    // ❌ Locks the ENTIRE method
    // Even expensive non-critical operations are locked
    public synchronized void methodLock() {
        expensiveDbCall();       // locked — other threads wait for this too
        data++;                  // critical section — needs lock
        expensiveApiCall();      // locked — other threads wait for this too
    }

    // ✅ Only locks the critical section
    public void blockLock() {
        expensiveDbCall();       // NOT locked — runs in parallel across threads
        synchronized (lock) {
            data++;              // critical section only
        }
        expensiveApiCall();      // NOT locked — runs in parallel
    }

    private void expensiveDbCall()  { try { Thread.sleep(200); } catch (InterruptedException e) {} }
    private void expensiveApiCall() { try { Thread.sleep(200); } catch (InterruptedException e) {} }
}
```

**Rule:** Lock the smallest possible section. Every millisecond spent inside a `synchronized` block is time other threads are blocked waiting.

---

## 4.7 The Lock Is Per Object Instance

This is a subtle point that trips many developers:

```java
public class LockPerInstanceDemo {

    private int count = 0;

    public synchronized void increment() {
        count++;
    }

    public static void main(String[] args) throws InterruptedException {

        // Two DIFFERENT instances → two DIFFERENT locks
        LockPerInstanceDemo obj1 = new LockPerInstanceDemo();
        LockPerInstanceDemo obj2 = new LockPerInstanceDemo();

        Thread t1 = new Thread(() -> obj1.increment(), "t1"); // acquires lock on obj1
        Thread t2 = new Thread(() -> obj2.increment(), "t2"); // acquires lock on obj2
        // t1 and t2 do NOT block each other — they hold different locks

        // Same instance → same lock
        Thread t3 = new Thread(() -> obj1.increment(), "t3"); // tries lock on obj1
        Thread t4 = new Thread(() -> obj1.increment(), "t4"); // tries lock on obj1
        // t3 and t4 WILL block each other — same lock
    }
}
```

**In ShopSphere:** If you have multiple instances of `CartService` (e.g., in a cluster), `synchronized` on `this` doesn't protect across JVMs. That's where distributed locks (Redis `SETNX`, Redisson) come in — but that's an advanced topic. For now, understand that `synchronized` is JVM-local.

---

## 4.8 Reentrant Locking

Java's intrinsic locks are **reentrant** — a thread that already holds a lock can acquire it again without deadlocking itself.

```java
public class ReentrantDemo {

    public synchronized void methodA() {
        System.out.println("In methodA");
        methodB(); // ← this thread already holds the lock on 'this'
                   //   calling synchronized methodB is fine — reentrant
    }

    public synchronized void methodB() {
        System.out.println("In methodB");
        // same thread enters again — Java allows this
    }

    public static void main(String[] args) {
        new ReentrantDemo().methodA();
        // Output:
        // In methodA
        // In methodB   ← no deadlock
    }
}
```

If locks weren't reentrant, `methodA` calling `methodB` would deadlock — the thread would wait for a lock it already holds, forever.

---

## 4.9 ShopSphere — Real Race Condition Scenario

Imagine your `InventoryService` handles concurrent purchase requests:

```java
public class InventoryService {

    // Shared state — on the heap, accessed by multiple Tomcat threads
    private Map<String, Integer> stock = new HashMap<>();

    public InventoryService() {
        stock.put("PRODUCT-001", 5); // 5 units in stock
    }

    // ❌ BROKEN — race condition
    public boolean reserveStock(String productId, int quantity) {
        int available = stock.get(productId);        // READ
        if (available >= quantity) {
            // Thread 2 can slip in RIGHT HERE
            // Both threads see available=5, both pass the check
            stock.put(productId, available - quantity); // WRITE
            return true;
        }
        return false;
    }

    // ✅ FIXED — synchronized block
    public boolean reserveStockSafe(String productId, int quantity) {
        synchronized (this) {
            int available = stock.get(productId);    // READ  ─┐
            if (available >= quantity) {                        │ atomic
                stock.put(productId, available - quantity); // WRITE ─┘
                return true;
            }
            return false;
        }
    }

    public static void main(String[] args) throws InterruptedException {

        InventoryService service = new InventoryService();

        // 10 concurrent requests all trying to buy 1 unit (only 5 in stock)
        Thread[] buyers = new Thread[10];
        int[] successCount = {0}; // array trick for lambda access

        for (int i = 0; i < 10; i++) {
            final int buyerId = i + 1;
            buyers[i] = new Thread(() -> {
                boolean success = service.reserveStockSafe("PRODUCT-001", 1);
                if (success) {
                    synchronized (successCount) { successCount[0]++; }
                    System.out.println("Buyer " + buyerId + " ✓ reserved stock");
                } else {
                    System.out.println("Buyer " + buyerId + " ✗ out of stock");
                }
            }, "buyer-" + i);
        }

        for (Thread t : buyers) t.start();
        for (Thread t : buyers) t.join();

        System.out.println("\nTotal successful reservations: " + successCount[0]);
        System.out.println("Expected: 5 (only 5 in stock)");
    }
}
```

**Output (with `reserveStockSafe`):**
```
Buyer 1 ✓ reserved stock
Buyer 3 ✓ reserved stock
Buyer 7 ✓ reserved stock
Buyer 2 ✓ reserved stock
Buyer 5 ✓ reserved stock
Buyer 4 ✗ out of stock
Buyer 6 ✗ out of stock
Buyer 8 ✗ out of stock
Buyer 9 ✗ out of stock
Buyer 10 ✗ out of stock

Total successful reservations: 5
Expected: 5 (only 5 in stock)
```

Exactly 5 reservations — no overselling.

---

## 4.10 What `synchronized` Guarantees

`synchronized` gives you two guarantees:

```
1. MUTUAL EXCLUSION
   Only one thread executes the synchronized section at a time.
   → Prevents race conditions.

2. VISIBILITY
   When a thread exits a synchronized block, all writes it made
   are flushed to main memory.
   When a thread enters a synchronized block, it reads fresh
   values from main memory.
   → Prevents stale reads.
```

Both matter. Race conditions are about mutual exclusion. Visibility is about CPU caches — covered more in Chapter 5 (`volatile`).

---

## 4.11 What `synchronized` Does NOT Do

```
❌ Does not prevent deadlocks
   (Two threads each holding a lock the other needs — Chapter 17)

❌ Does not guarantee fairness
   (A thread may starve — never get the lock — if others keep winning)

❌ Does not work across multiple JVMs
   (Clusters need Redis/Zookeeper distributed locks)

❌ Does not make compound operations on separate objects atomic
   synchronized(lockA) { ... }
   synchronized(lockB) { ... }
   ← gap between these two is not protected
```

---

## 4.12 Key Takeaways

| Concept | Remember This |
|---|---|
| Race condition | Result depends on thread timing — non-deterministic bug |
| `count++` | Not atomic — it's READ + ADD + WRITE |
| `synchronized` method | Lock on `this` (or `.class` if static) |
| `synchronized` block | Lock on specified object — prefer this, minimizes scope |
| Lock per instance | Two different objects = two different locks = no mutual exclusion |
| Reentrant | A thread can re-acquire a lock it already holds |
| Guarantees | Mutual exclusion + visibility |

---

## Interview Questions from This Chapter

**Q1. What is a race condition? Give an example.**
> A race condition occurs when two threads access shared mutable data simultaneously and the result depends on execution order. Classic example: two threads doing `count++` — each does read-add-write, and one thread's write can overwrite the other's, causing lost updates.

**Q2. What does `synchronized` guarantee?**
> Two things: mutual exclusion (only one thread executes the block at a time) and visibility (changes made inside the block are visible to other threads that subsequently enter a synchronized block on the same lock).

**Q3. What is the difference between a synchronized method and a synchronized block?**
> A synchronized method locks `this` for the entire method duration. A synchronized block lets you specify exactly which object to lock and which lines to protect — giving finer granularity and better performance by minimizing the locked section.

**Q4. Is `count++` thread-safe?**
> No. It compiles to three operations: read, increment, write. Another thread can interleave between any of these steps, causing lost updates.

**Q5. Can two threads enter synchronized methods on different instances simultaneously?**
> Yes. The lock is per object instance. Two threads on two different objects hold two different locks and don't block each other.

---

Ready for **Chapter 5: `volatile` keyword** — CPU caches, visibility bugs, and when `volatile` is enough vs when you still need `synchronized`?
