
# Chapter 10: Concurrent Collections — Thread-Safe Data Structures

---

## 10.1 Why Regular Collections Fail Under Concurrency

You already know `HashMap` is not thread-safe. Let's see exactly how it breaks:

```java
public class HashMapFailure {

    public static void main(String[] args) throws InterruptedException {

        Map<Integer, Integer> map = new HashMap<>();

        // 2 threads, each inserting 10,000 entries
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 10_000; i++) map.put(i, i);
        }, "writer-1");

        Thread t2 = new Thread(() -> {
            for (int i = 10_000; i < 20_000; i++) map.put(i, i);
        }, "writer-2");

        t1.start(); t2.start();
        t1.join();  t2.join();

        System.out.println("Expected: 20000");
        System.out.println("Actual:   " + map.size());
        // Possible outcomes:
        // - Wrong size (lost puts)
        // - Infinite loop (circular linked list in Java 7 HashMap)
        // - NullPointerException
        // - ConcurrentModificationException
        // All are possible — behavior is undefined
    }
}
```

The failures aren't just wrong counts. In Java 7, concurrent `HashMap` puts could create a **circular linked list** in a bucket — causing an infinite loop that hangs the JVM forever. This is a real production outage pattern.

---

## 10.2 The Three Approaches — and Why Two Are Wrong

### Approach 1: `HashMap` — Not Thread-Safe

```java
Map<String, Integer> map = new HashMap<>();
// ❌ Completely unsafe for concurrent access
```

### Approach 2: `Hashtable` — Synchronized But Obsolete

```java
Map<String, Integer> map = new Hashtable<>();
// Every method is synchronized on the ENTIRE object
// Only one thread can read OR write at any time
// ❌ Terrible performance — avoid completely
```

### Approach 3: `Collections.synchronizedMap()` — Still Too Coarse

```java
Map<String, Integer> map = Collections.synchronizedMap(new HashMap<>());
// Wraps HashMap — every method synchronized on the wrapper object
// Same problem as Hashtable — one lock for everything

// EXTRA TRAP: iteration is NOT thread-safe even with synchronizedMap
synchronized (map) {          // ← must manually lock for iteration
    for (Map.Entry<String, Integer> e : map.entrySet()) {
        System.out.println(e.getKey() + "=" + e.getValue());
    }
}
// Forget the synchronized block → ConcurrentModificationException
// ❌ Still not good enough for production
```

### Approach 4: `ConcurrentHashMap` — The Right Answer ✅

```java
Map<String, Integer> map = new ConcurrentHashMap<>();
// ✅ Designed for concurrent access
// ✅ High throughput — fine-grained locking
// ✅ Iteration is safe without external locking
// ✅ This is what you always use in production
```

---

## 10.3 `ConcurrentHashMap` — How It Works Internally

This is the most asked internals question for 2 YOE Java interviews.

### Java 7 — Segment Locking

```
ConcurrentHashMap (Java 7)

┌─────────────────────────────────────────────┐
│  Segment 0  │  Segment 1  │  ...  │  Seg 15 │
│  (lock 0)   │  (lock 1)   │       │ (lock 15)│
│  [bucket0]  │  [bucket4]  │       │          │
│  [bucket1]  │  [bucket5]  │       │          │
│  [bucket2]  │  [bucket6]  │       │          │
│  [bucket3]  │  [bucket7]  │       │          │
└─────────────────────────────────────────────┘

Default: 16 segments → 16 locks
→ Up to 16 threads can write simultaneously (to different segments)
→ Much better than 1 lock for everything
```

### Java 8+ — Bucket-Level Locking (CAS + synchronized)

Java 8 completely rewrote `ConcurrentHashMap`. No more segments:

```
ConcurrentHashMap (Java 8+)

Array of buckets (like HashMap):
[bucket0][bucket1][bucket2][bucket3]...[bucketN]

Each bucket is locked INDEPENDENTLY:
- Empty bucket    → CAS (Compare-And-Swap) to insert — NO lock at all
- Non-empty bucket → synchronized on the FIRST NODE of that bucket only

→ Threads writing to different buckets never block each other
→ Threads writing to same bucket: one waits, one proceeds
→ Maximum concurrency: number of buckets (default 16, grows with size)
```

```
Thread A puts key="user-1"   → hashes to bucket 3  → locks bucket 3
Thread B puts key="order-5"  → hashes to bucket 11 → locks bucket 11
Thread C puts key="prod-2"   → hashes to bucket 7  → locks bucket 7

All three proceed SIMULTANEOUSLY — different buckets, different locks
Zero contention
```

### Why This Matters for Performance

```
Operation       HashMap    synchronizedMap    ConcurrentHashMap
─────────────────────────────────────────────────────────────
put()           Unsafe     1 global lock      1 bucket lock (or CAS)
get()           Unsafe     1 global lock      Lock-free (volatile read)
size()          Unsafe     1 global lock      Approximate (summed counters)
iteration       Unsafe     Manual sync needed Safe (weakly consistent)
```

`get()` in `ConcurrentHashMap` is **lock-free** — it uses `volatile` reads on node values. Multiple readers never block each other and never block writers either.

---

## 10.4 `ConcurrentHashMap` — Key Methods and Atomic Operations

Regular `put`/`get` work the same as `HashMap`. The power is in the **atomic compound operations**:

```java
ConcurrentHashMap<String, Integer> stockMap = new ConcurrentHashMap<>();

// ─── putIfAbsent — atomic check-then-insert ───────────────────────
// ❌ Non-atomic version (race condition between check and put)
if (!stockMap.containsKey("PROD-001")) {
    stockMap.put("PROD-001", 100); // another thread can insert between these two lines
}

// ✅ Atomic version
stockMap.putIfAbsent("PROD-001", 100);
// Only inserts if key doesn't exist — single atomic operation

// ─── computeIfAbsent — insert if absent, return value ─────────────
// Great for lazy initialization
stockMap.computeIfAbsent("PROD-001", key -> {
    return loadStockFromDB(key); // only called if key not present
});

// ─── compute — atomic read-modify-write ───────────────────────────
// ❌ Non-atomic (race condition)
int current = stockMap.get("PROD-001");
stockMap.put("PROD-001", current - 1); // another thread can change value between get and put

// ✅ Atomic compute
stockMap.compute("PROD-001", (key, currentValue) -> {
    if (currentValue == null) return 0;
    return currentValue - 1; // decrement stock — atomic
});

// ─── merge — insert or update atomically ──────────────────────────
// Add 5 to existing value, or insert 5 if absent
stockMap.merge("PROD-001", 5, Integer::sum);
// If PROD-001 exists: new value = existing + 5
// If absent: new value = 5

// ─── getOrDefault — safe read with fallback ───────────────────────
int stock = stockMap.getOrDefault("PROD-999", 0); // 0 if not found
```

### ShopSphere — Real Usage Pattern

```java
// Cart service: tracking active item counts per user
// Multiple threads (Tomcat request threads) modify concurrently

public class CartCountService {

    // userId → item count
    private final ConcurrentHashMap<String, Integer> cartCounts
        = new ConcurrentHashMap<>();

    public void addItem(String userId, int quantity) {
        // Atomic increment — no synchronized needed
        cartCounts.merge(userId, quantity, Integer::sum);
    }

    public void removeItem(String userId, int quantity) {
        cartCounts.computeIfPresent(userId, (key, current) ->
            Math.max(0, current - quantity)
        );
    }

    public int getCount(String userId) {
        return cartCounts.getOrDefault(userId, 0);
    }

    public void clearCart(String userId) {
        cartCounts.remove(userId);
    }
}
```

---

## 10.5 `ConcurrentHashMap` — Important Caveats

### Caveat 1: `size()` is Approximate

```java
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
// map.size() uses distributed counters — may not be perfectly accurate
// during concurrent modifications
// Good enough for monitoring, not for logic
```

### Caveat 2: Compound Operations Need `compute`

```java
// ❌ This looks atomic but isn't — two separate operations
if (map.containsKey(key)) {
    map.put(key, newValue); // gap here — another thread can remove key
}

// ✅ Truly atomic
map.compute(key, (k, v) -> v != null ? newValue : null);
```

### Caveat 3: Iteration is Weakly Consistent

```java
// Iteration reflects state at some point during or after iteration start
// You may see some updates made after iteration began — or not
// You will NOT get ConcurrentModificationException — safe to iterate
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + "=" + entry.getValue());
    // concurrent puts/removes won't throw — but you may or may not see them
}
```

---

## 10.6 `CopyOnWriteArrayList` — Read-Heavy Lists

Regular `ArrayList` is not thread-safe. `Collections.synchronizedList()` works but locks on every read. `CopyOnWriteArrayList` is optimized for read-heavy scenarios.

### How It Works

```
On every WRITE (add, remove, set):
  1. Copy the entire underlying array
  2. Modify the copy
  3. Replace the reference atomically (volatile)

On every READ:
  Read from the current array — NO locking at all
  Readers always see a consistent snapshot
```

```
Initial state: array = [A, B, C]

Thread 1 reads:   sees [A, B, C] — no lock
Thread 2 adds D:  copies → [A, B, C, D], replaces reference
Thread 3 reads:   sees [A, B, C, D] — no lock, no blocking
Thread 1 still iterating: still sees [A, B, C] — its snapshot, no CME
```

```java
CopyOnWriteArrayList<String> activeUsers = new CopyOnWriteArrayList<>();

// Reads — ZERO locking, extremely fast
for (String user : activeUsers) {
    sendHeartbeat(user); // concurrent adds/removes won't throw CME
}

// Writes — expensive (full array copy)
activeUsers.add("USER-vidhan");    // copies entire array
activeUsers.remove("USER-guest");  // copies entire array
```

### When to Use vs Avoid

```
✅ Use CopyOnWriteArrayList when:
   - Reads vastly outnumber writes (10:1 or more)
   - List is small (copying a 10-element array is cheap)
   - You need safe iteration without external locking
   ShopSphere example: list of active feature flags, webhook endpoints,
                        registered event listeners

❌ Avoid when:
   - Writes are frequent (every write copies full array)
   - List is large (copying 10,000 elements on every write = expensive)
   - You need real-time consistency (readers see a snapshot)
```

---

## 10.7 `BlockingQueue` — The Production Producer-Consumer Tool

Remember Chapter 6's hand-rolled producer-consumer with `wait()`/`notifyAll()`? That was for learning. In production you use `BlockingQueue` — it handles all the locking and waiting internally.

```java
public interface BlockingQueue<E> extends Queue<E> {
    void put(E e)    throws InterruptedException; // blocks if FULL
    E    take()      throws InterruptedException; // blocks if EMPTY
    boolean offer(E e, long timeout, TimeUnit unit); // wait up to timeout
    E    poll(long timeout, TimeUnit unit);           // wait up to timeout
}
```

### The Four Implementations

#### 1. `ArrayBlockingQueue` — Bounded, Array-Backed

```java
BlockingQueue<String> queue = new ArrayBlockingQueue<>(100);
// Fixed capacity: 100
// One lock shared between producer and consumer
// Fair mode available: new ArrayBlockingQueue<>(100, true)
//   → threads served in arrival order — prevents starvation

// Best for: when you need strict capacity control
// ShopSphere: payment processing queue (max 100 pending payments)
```

#### 2. `LinkedBlockingQueue` — Optionally Bounded, Linked Nodes

```java
BlockingQueue<String> queue = new LinkedBlockingQueue<>(1000); // bounded
BlockingQueue<String> queue = new LinkedBlockingQueue<>();      // unbounded (Integer.MAX_VALUE)
// Separate locks for put and take — higher throughput than ArrayBlockingQueue
// Unbounded version: be careful — can grow until OutOfMemoryError

// Best for: high-throughput pipelines where you want separate read/write locks
// ShopSphere: notification queue, order event queue
// ⚠ Always use bounded version in production
```

#### 3. `PriorityBlockingQueue` — Priority Ordered

```java
BlockingQueue<Order> queue = new PriorityBlockingQueue<>(
    100,
    Comparator.comparingInt(Order::getPriority).reversed()
);
// Elements dequeued in priority order
// Unbounded — grows without limit
// No blocking on put() — only take() blocks when empty

// Best for: priority lanes
// ShopSphere: premium orders processed before regular orders
```

#### 4. `SynchronousQueue` — Zero Capacity Handoff

```java
BlockingQueue<String> queue = new SynchronousQueue<>();
// NO internal storage — zero capacity
// put() blocks until a consumer calls take() — direct handoff
// take() blocks until a producer calls put()
// Used internally by Executors.newCachedThreadPool()

// Best for: direct thread-to-thread handoff with no buffering
```

### Comparison Table

```
                    ArrayBlocking  LinkedBlocking  PriorityBlocking  Synchronous
Capacity            Fixed          Optional        Unbounded         Zero
put() blocks when   Full           Full            Never             No consumer
take() blocks when  Empty          Empty           Empty             No producer
Throughput          Medium         High            Medium            Highest
Lock strategy       1 lock         2 locks         1 lock            CAS
Use case            Strict cap     High throughput Priority lanes    Thread handoff
```

---

## 10.8 Producer-Consumer With `BlockingQueue` — Clean Version

Compare this to Chapter 6's `wait()`/`notifyAll()` version — same behavior, much less code:

```java
public class BlockingQueuePipeline {

    // ─── Shared queue — all synchronization handled internally ────
    private static final BlockingQueue<String> orderQueue
        = new LinkedBlockingQueue<>(50); // max 50 pending orders

    private static volatile boolean producingDone = false;

    // ─── Producer ─────────────────────────────────────────────────
    static class OrderProducer implements Runnable {
        private final String[] orders;

        OrderProducer(String[] orders) { this.orders = orders; }

        @Override
        public void run() {
            try {
                for (String order : orders) {
                    orderQueue.put(order); // blocks if queue full — no wait() needed
                    System.out.println("[" + Thread.currentThread().getName()
                        + "] Produced: " + order
                        + " | Queue: " + orderQueue.size());
                    Thread.sleep(200);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    // ─── Consumer ─────────────────────────────────────────────────
    static class NotificationConsumer implements Runnable {

        @Override
        public void run() {
            try {
                while (!producingDone || !orderQueue.isEmpty()) {
                    // poll with timeout — so we can check producingDone
                    String order = orderQueue.poll(500, TimeUnit.MILLISECONDS);
                    if (order != null) {
                        System.out.println("[" + Thread.currentThread().getName()
                            + "] Consumed: " + order
                            + " | Queue: " + orderQueue.size());
                        Thread.sleep(400); // simulate email sending
                    }
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println("[" + Thread.currentThread().getName() + "] Done.");
        }
    }

    public static void main(String[] args) throws InterruptedException {

        // 2 producers
        Thread p1 = new Thread(new OrderProducer(
            new String[]{"ORD-001", "ORD-002", "ORD-003"}), "order-svc-1");
        Thread p2 = new Thread(new OrderProducer(
            new String[]{"ORD-004", "ORD-005", "ORD-006"}), "order-svc-2");

        // 2 consumers
        Thread c1 = new Thread(new NotificationConsumer(), "notif-worker-1");
        Thread c2 = new Thread(new NotificationConsumer(), "notif-worker-2");

        c1.start(); c2.start();
        p1.start(); p2.start();

        p1.join(); p2.join();
        producingDone = true; // signal consumers: no more orders coming

        c1.join(); c2.join();
        System.out.println("\n✅ All orders processed.");
    }
}
```

**Compare to Chapter 6:** No `synchronized` blocks, no `wait()`, no `notifyAll()`, no `while` loop for spurious wakeups. `BlockingQueue` handles all of it. This is production code.

---

## 10.9 Other Concurrent Utilities Worth Knowing

### `ConcurrentLinkedQueue` — Non-Blocking Lock-Free Queue

```java
Queue<String> queue = new ConcurrentLinkedQueue<>();
// Lock-free — uses CAS internally
// Does NOT block on empty (poll() returns null)
// Unbounded
// Best for: multiple producers/consumers, non-blocking required
// NOT a BlockingQueue — no put()/take()
```

### `ConcurrentSkipListMap` — Sorted Concurrent Map

```java
ConcurrentNavigableMap<String, Integer> map = new ConcurrentSkipListMap<>();
// Like ConcurrentHashMap but SORTED by key
// Also thread-safe like ConcurrentHashMap
// Best for: leaderboards, time-series data, range queries
// ShopSphere: sorted product rankings, price range queries
```

### `LinkedTransferQueue` — Most Powerful BlockingQueue

```java
TransferQueue<String> queue = new LinkedTransferQueue<>();
// Like LinkedBlockingQueue but with extra transfer() method:
queue.transfer(item);
// Blocks until a consumer actually takes the item (like SynchronousQueue)
// But also has a buffer (unlike SynchronousQueue)
// Used internally by Java ForkJoinPool
```

---

## 10.10 Full Comparison — Which Collection to Use

```
Need                                 Use
────────────────────────────────────────────────────────────
Thread-safe key-value store          ConcurrentHashMap
  with atomic operations             (compute, merge, putIfAbsent)

Thread-safe list, read-heavy         CopyOnWriteArrayList

Thread-safe list, write-heavy        Collections.synchronizedList()
                                     or protect with ReentrantLock (Ch.11)

Bounded producer-consumer queue      ArrayBlockingQueue (strict cap)
                                     LinkedBlockingQueue (high throughput)

Priority-based task queue            PriorityBlockingQueue

Direct thread handoff                SynchronousQueue

Non-blocking concurrent queue        ConcurrentLinkedQueue

Sorted thread-safe map               ConcurrentSkipListMap

Never use these                      Hashtable, Vector,
                                     Collections.synchronizedMap
                                     (in new code)
```

---

## 10.11 ShopSphere — Concurrent Collections in Context

```java
public class ShopSphereCollectionsDemo {

    // ─── 1. Product cache — many reads, occasional writes ─────────
    private final ConcurrentHashMap<String, Product> productCache
        = new ConcurrentHashMap<>();

    public Product getProduct(String id) {
        return productCache.computeIfAbsent(id, this::loadFromDB);
        // If not cached: loads from DB atomically
        // If cached: returns existing — zero locking
    }

    // ─── 2. Active WebSocket connections — read-heavy ────────────
    private final CopyOnWriteArrayList<WebSocketSession> connections
        = new CopyOnWriteArrayList<>();

    public void broadcast(String message) {
        // Safe iteration — no CME even if connections change
        for (WebSocketSession session : connections) {
            session.send(message);
        }
    }

    // ─── 3. Order processing queue — bounded, backpressure ───────
    private final BlockingQueue<Order> orderQueue
        = new LinkedBlockingQueue<>(500);

    public void submitOrder(Order order) throws InterruptedException {
        orderQueue.put(order); // blocks if 500 orders already pending
        // Natural backpressure — slows down producers automatically
    }

    public Order nextOrder() throws InterruptedException {
        return orderQueue.take(); // blocks until order available
    }

    // ─── 4. Inventory counts — concurrent increments/decrements ──
    private final ConcurrentHashMap<String, Integer> inventory
        = new ConcurrentHashMap<>();

    public boolean reserveStock(String productId, int qty) {
        // Atomic check-and-decrement
        int[] success = {0};
        inventory.compute(productId, (id, current) -> {
            if (current != null && current >= qty) {
                success[0] = 1;
                return current - qty;
            }
            return current; // unchanged — not enough stock
        });
        return success[0] == 1;
    }

    // ─── Stubs ────────────────────────────────────────────────────
    Product loadFromDB(String id) { return new Product(id); }
    record Product(String id) {}
    record Order(String id) {}
    interface WebSocketSession { void send(String msg); }
}
```

---

## 10.12 Key Takeaways

| Concept | Remember This |
|---|---|
| `HashMap` | Never in concurrent code — undefined behavior |
| `Hashtable` / `synchronizedMap` | One global lock — obsolete, avoid |
| `ConcurrentHashMap` | Bucket-level locking + CAS. Lock-free reads. Production standard. |
| `compute`/`merge`/`putIfAbsent` | Atomic compound operations — use instead of get+put |
| `CopyOnWriteArrayList` | Copy on write. Lock-free reads. Expensive writes. Read-heavy only. |
| `ArrayBlockingQueue` | Bounded, one lock. Use when strict capacity needed. |
| `LinkedBlockingQueue` | Optionally bounded, two locks (higher throughput). Always bound it. |
| `PriorityBlockingQueue` | Priority ordered, unbounded. Use for priority lanes. |
| `SynchronousQueue` | Zero capacity, direct handoff. Used by `cachedThreadPool`. |
| `BlockingQueue` vs Chapter 6 | `BlockingQueue` replaces `wait()`/`notifyAll()` in production. |

---

## Interview Questions From This Chapter

**Q1. Why is `ConcurrentHashMap` faster than `Collections.synchronizedMap()`?**
> `synchronizedMap` uses a single lock for the entire map — only one thread can read or write at any time. `ConcurrentHashMap` uses bucket-level locking in Java 8+ — threads writing to different buckets proceed simultaneously with no contention. Reads are lock-free via volatile. This allows true concurrent access across all operations.

**Q2. How does `ConcurrentHashMap` handle concurrent reads and writes?**
> Reads are lock-free — node values are marked `volatile` so any thread reads the latest value directly from memory without acquiring any lock. Writes use CAS for inserting into empty buckets (no lock) or `synchronized` on the first node of a bucket for collision chains. Multiple readers and writers to different buckets never block each other.

**Q3. When would you use `CopyOnWriteArrayList` and when would you avoid it?**
> Use it when reads heavily outnumber writes — like a list of event listeners, feature flags, or webhook URLs. Every write copies the entire array, so it's expensive for frequent mutations or large lists. The benefit is completely lock-free iteration and zero `ConcurrentModificationException` risk.

**Q4. What is the difference between `put()` and `offer()` on a `BlockingQueue`?**
> `put()` blocks indefinitely if the queue is full — the thread sleeps until space is available. `offer()` with a timeout waits up to the specified duration and returns `false` if the queue is still full. `offer()` without a timeout returns immediately with `false` if full — non-blocking. Use `put()` for strict backpressure, `offer()` with timeout when you need to handle full-queue gracefully.

**Q5. What is the difference between `ArrayBlockingQueue` and `LinkedBlockingQueue`?**
> `ArrayBlockingQueue` is array-backed with a fixed capacity — uses one lock shared between producers and consumers. `LinkedBlockingQueue` is linked-node based with optional capacity — uses two separate locks (one for put, one for take), giving higher throughput under concurrent load. `LinkedBlockingQueue` is generally preferred for high-throughput pipelines; `ArrayBlockingQueue` when memory predictability and strict capacity are priorities.

---

Ready for **Chapter 11: `ReentrantLock` and the `locks` package** — `tryLock()`, `lockInterruptibly()`, `ReadWriteLock`, and exactly when `ReentrantLock` beats `synchronized`?
