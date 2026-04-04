
# Chapter 13: Thread Safety Patterns — ThreadLocal, Immutability & Design Patterns

---

## 13.1 The Three Ways to Achieve Thread Safety

Before diving into patterns, understand the three fundamental approaches:

```
Approach 1: SYNCHRONIZATION
  Control access to shared mutable state
  synchronized, ReentrantLock, Atomic variables
  → Chapters 4, 11, 12

Approach 2: IMMUTABILITY
  Don't allow state to change at all
  No mutation = no race conditions = no synchronization needed
  → This chapter

Approach 3: THREAD CONFINEMENT
  Don't share state between threads
  Each thread has its own copy
  → This chapter (ThreadLocal)
```

The best thread-safe code doesn't need locks — it either never changes or never shares.

---

## 13.2 Immutability — The Safest Thread Safety

An immutable object's state cannot change after construction. If state can't change, no thread can corrupt it. No synchronization needed — ever.

### Rules for True Immutability

```
1. All fields are final
2. All fields are private
3. No setter methods
4. Class is final (prevents subclasses from breaking immutability)
5. No mutable objects leaked (deep immutability)
```

```java
// ✅ Truly immutable — completely thread-safe with zero synchronization
public final class Money {

    private final long amountInPaisa;  // final — set once in constructor
    private final String currency;     // final

    public Money(long amountInPaisa, String currency) {
        this.amountInPaisa = amountInPaisa;
        this.currency      = currency;
    }

    // Returns NEW object — doesn't mutate this
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Currency mismatch");
        }
        return new Money(this.amountInPaisa + other.amountInPaisa, this.currency);
    }

    public Money subtract(Money other) {
        return new Money(this.amountInPaisa - other.amountInPaisa, this.currency);
    }

    public long getAmountInPaisa() { return amountInPaisa; }
    public String getCurrency()    { return currency; }

    @Override
    public String toString() {
        return currency + " " + String.format("%.2f", amountInPaisa / 100.0);
    }
}
```

```java
// 100 threads can all read and "modify" this simultaneously
// Each gets a new object — original never changes
Money price    = new Money(129900, "INR"); // ₹1299
Money discount = new Money(10000,  "INR"); // ₹100
Money final_   = price.subtract(discount); // new object ₹1199

// price is still ₹1299 — immutable, shared freely across all threads
```

### The Mutable Leak Trap

```java
// ❌ Looks immutable but isn't — mutable object leaked
public final class OrderSummary {
    private final String orderId;
    private final List<String> items; // List is MUTABLE

    public OrderSummary(String orderId, List<String> items) {
        this.orderId = orderId;
        this.items = items; // ← stores reference to caller's list
        // caller can still modify the list externally!
    }

    public List<String> getItems() {
        return items; // ← returns mutable reference — caller can modify!
    }
}

// ✅ Fixed — defensive copy
public final class OrderSummary {
    private final String orderId;
    private final List<String> items;

    public OrderSummary(String orderId, List<String> items) {
        this.orderId = orderId;
        this.items = List.copyOf(items); // ← immutable copy
    }

    public List<String> getItems() {
        return items; // List.copyOf returns unmodifiable — safe to return
    }
}
```

### ShopSphere — Immutable Value Objects

```java
// All of these are perfect candidates for immutability
public final class ProductId {
    private final String value;
    public ProductId(String value) {
        if (value == null || value.isBlank())
            throw new IllegalArgumentException("ProductId cannot be blank");
        this.value = value;
    }
    public String getValue() { return value; }
    @Override public String toString() { return value; }
}

public final class OrderStatus {
    public static final OrderStatus PENDING    = new OrderStatus("PENDING");
    public static final OrderStatus CONFIRMED  = new OrderStatus("CONFIRMED");
    public static final OrderStatus SHIPPED    = new OrderStatus("SHIPPED");
    public static final OrderStatus DELIVERED  = new OrderStatus("DELIVERED");

    private final String value;
    private OrderStatus(String value) { this.value = value; }
    public String getValue() { return value; }
}

// Java 16+ record — immutable by default
public record Address(
    String street, String city, String state, String pincode
) {
    // records are implicitly final, all fields final, no setters
    // ✅ Thread-safe with zero effort
}
```

---

## 13.3 `ThreadLocal` — Thread Confinement

`ThreadLocal<T>` gives each thread its **own independent copy** of a variable. Thread A's copy and Thread B's copy are completely separate — no sharing, no synchronization needed.

```
Without ThreadLocal:
  HEAP: sharedVariable = X
  Thread A reads X, modifies → Y  ← race condition with Thread B
  Thread B reads X, modifies → Z  ← both see each other's changes

With ThreadLocal:
  Thread A's storage: variable = X_A  (Thread A's private copy)
  Thread B's storage: variable = X_B  (Thread B's private copy)
  Thread A modifies X_A → Y_A  ← no effect on Thread B
  Thread B modifies X_B → Z_B  ← no effect on Thread A
```

### Basic Usage

```java
public class ThreadLocalBasic {

    // Each thread gets its own Integer
    private static final ThreadLocal<Integer> requestCounter =
        ThreadLocal.withInitial(() -> 0); // initial value factory

    public static void main(String[] args) throws InterruptedException {

        Runnable task = () -> {
            String name = Thread.currentThread().getName();

            // Each thread increments ITS OWN counter
            for (int i = 0; i < 3; i++) {
                int current = requestCounter.get();
                requestCounter.set(current + 1);
                System.out.println("[" + name + "] counter = " + requestCounter.get());
            }
        };

        Thread t1 = new Thread(task, "thread-A");
        Thread t2 = new Thread(task, "thread-B");

        t1.start(); t2.start();
        t1.join();  t2.join();
    }
}
```

**Output:**
```
[thread-A] counter = 1
[thread-B] counter = 1   ← B starts from 0 too — own copy
[thread-A] counter = 2
[thread-B] counter = 2
[thread-A] counter = 3
[thread-B] counter = 3
```

Neither thread sees the other's counter. They're completely isolated.

---

## 13.4 `ThreadLocal` — Real Production Patterns

### Pattern 1: Request Context (How Spring Does It)

This is the most important real-world use of `ThreadLocal`. Spring stores the current HTTP request, security context, and transaction on the thread:

```java
public class RequestContext {

    // Holds current request's user info — per thread
    private static final ThreadLocal<UserContext> userContext =
        new ThreadLocal<>();

    public static void setUser(UserContext ctx) {
        userContext.set(ctx);
    }

    public static UserContext getUser() {
        return userContext.get();
    }

    public static void clear() {
        userContext.remove(); // CRITICAL — always clean up!
    }

    record UserContext(String userId, String role, String sessionId) {}
}
```

```java
// Simulating what Spring's DispatcherServlet does per request
public class RequestInterceptor {

    public void handleRequest(String userId, String role, Runnable handler) {
        try {
            // Set context for THIS thread (this request)
            RequestContext.setUser(
                new RequestContext.UserContext(userId, role, "SESSION-" + userId)
            );

            handler.run(); // calls your @RestController method

        } finally {
            RequestContext.clear(); // ← ALWAYS remove — prevents memory leak
        }
    }
}
```

```java
// Anywhere in the call stack for THIS request — no need to pass userId around
public class OrderService {

    public void placeOrder(String productId) {
        // Who is placing this order? Get from ThreadLocal — no parameter passing
        RequestContext.UserContext user = RequestContext.getUser();

        System.out.println("Order placed by: " + user.userId()
            + " | role: " + user.role()
            + " | product: " + productId);

        // validate permissions, audit log, etc. — all from ThreadLocal
    }
}
```

```java
// Demo — 3 concurrent requests, each with own context
public class ThreadLocalSpringDemo {

    public static void main(String[] args) throws InterruptedException {

        RequestInterceptor interceptor = new RequestInterceptor();
        OrderService       orderService = new OrderService();
        ExecutorService    pool = Executors.newFixedThreadPool(3);

        // Simulate 3 concurrent HTTP requests
        pool.submit(() -> interceptor.handleRequest("USER-vidhan", "CUSTOMER",
            () -> orderService.placeOrder("PROD-headphones")));

        pool.submit(() -> interceptor.handleRequest("USER-admin", "ADMIN",
            () -> orderService.placeOrder("PROD-monitor")));

        pool.submit(() -> interceptor.handleRequest("USER-priya", "CUSTOMER",
            () -> orderService.placeOrder("PROD-keyboard")));

        pool.shutdown();
        pool.awaitTermination(5, TimeUnit.SECONDS);
    }
}
```

**Output:**
```
Order placed by: USER-vidhan | role: CUSTOMER | product: PROD-headphones
Order placed by: USER-admin  | role: ADMIN    | product: PROD-monitor
Order placed by: USER-priya  | role: CUSTOMER | product: PROD-keyboard
```

Each request's context is completely isolated — this is exactly how `SecurityContextHolder` and `@Transactional` work in Spring.

### Pattern 2: Expensive Object Reuse Per Thread

```java
// DateFormat is NOT thread-safe — but creating one per call is expensive
// Solution: one per thread via ThreadLocal

public class DateFormatter {

    private static final ThreadLocal<SimpleDateFormat> dateFormat =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd HH:mm:ss"));

    public static String format(Date date) {
        return dateFormat.get().format(date); // each thread uses its own instance
        // no synchronization needed — thread-confined
    }
}
```

### Pattern 3: Database Connection Per Thread

```java
// Each thread gets its own DB connection — no connection pool contention
public class ConnectionHolder {

    private static final ThreadLocal<Connection> connectionHolder =
        new ThreadLocal<>();

    public static Connection getConnection() {
        Connection conn = connectionHolder.get();
        if (conn == null) {
            conn = dataSource.getConnection();
            connectionHolder.set(conn);
        }
        return conn;
    }

    public static void closeConnection() {
        Connection conn = connectionHolder.get();
        if (conn != null) {
            conn.close();
            connectionHolder.remove(); // ALWAYS remove after use
        }
    }
}
```

---

## 13.5 `ThreadLocal` Memory Leak — The Critical Warning

This is a real production issue that causes **`OutOfMemoryError`** in thread pool environments.

```
Thread pools REUSE threads. If you don't call remove():

Request 1 → Thread A → set(UserContext{userId="USER-1"}) → request ends
                        threadLocal NOT removed
Request 2 → Thread A (reused!) → threadLocal.get() → still has USER-1's context!
                                  WRONG DATA for USER-2's request

Worse with thread pools:
Thread lives forever (pool thread)
ThreadLocal value never GC'd
Each request adds to thread's ThreadLocalMap
Eventually: OutOfMemoryError
```

```java
// ✅ ALWAYS use try/finally to clean up
public void processRequest(Request request) {
    try {
        contextHolder.set(buildContext(request));
        doWork(request);
    } finally {
        contextHolder.remove(); // ← NEVER skip this
    }
}

// ❌ WRONG — memory leak in thread pool
public void processRequest(Request request) {
    contextHolder.set(buildContext(request));
    doWork(request);
    contextHolder.remove(); // ← skipped if doWork() throws
}
```

### The Rule
> **`ThreadLocal.remove()` in `finally`.** Same rule as `lock.unlock()`. No exceptions.

---

## 13.6 Double-Checked Locking — The Complete Pattern

You saw this briefly in Chapter 5. Now the full picture with everything you know:

```java
public class PaymentGatewayClient {

    // volatile — MANDATORY (Chapter 5 explains why)
    private static volatile PaymentGatewayClient instance = null;

    private final String apiKey;
    private final HttpClient httpClient;

    private PaymentGatewayClient() {
        System.out.println("Initializing payment gateway client...");
        this.apiKey     = System.getenv("PAYMENT_API_KEY");
        this.httpClient = HttpClient.newBuilder()
            .connectTimeout(Duration.ofSeconds(5))
            .build();
    }

    public static PaymentGatewayClient getInstance() {

        // First check — no lock — 99.99% of calls return here
        if (instance == null) {

            // Lock only when instance might be null
            synchronized (PaymentGatewayClient.class) {

                // Second check — with lock — another thread may have set it
                if (instance == null) {
                    instance = new PaymentGatewayClient(); // volatile write
                }
            }
        }
        return instance; // volatile read
    }

    public String charge(long amountInPaisa) {
        return "TXN-" + System.nanoTime();
    }
}
```

### Why Every Part Is Necessary

```
Remove volatile →
  Object partially constructed (constructor not done)
  Another thread sees non-null reference to broken object
  Crash or silent data corruption

Remove second null check (inner if) →
  Thread A and B both pass first check
  A gets lock, creates instance
  A releases lock, B gets lock
  B creates SECOND instance — singleton broken

Remove first null check (outer if) →
  Every call acquires the lock
  Defeats the whole optimization — as slow as synchronized method
  (functional but unnecessary)
```

### Modern Alternative — Initialization-on-Demand Holder

Simpler, thread-safe without `volatile`, lazy:

```java
public class RazorpayClient {

    private RazorpayClient() {
        System.out.println("Initializing Razorpay client...");
    }

    // JVM loads inner class only when getInstance() is first called
    // Class loading is inherently thread-safe — no synchronized needed
    private static class Holder {
        static final RazorpayClient INSTANCE = new RazorpayClient();
    }

    public static RazorpayClient getInstance() {
        return Holder.INSTANCE; // class loaded here on first call
    }
}
```

This is cleaner than DCL for most cases. Use DCL when you need the field to be resettable or when you're not creating a singleton.

---

## 13.7 Safe Publication — How to Share Objects Between Threads

**Publication** = making an object visible to other threads. **Safe publication** = doing it so other threads see a fully constructed object.

### Unsafe Publication

```java
public class UnsafePublication {
    public static Product product; // non-volatile, not final

    public static void initialize() {
        product = new Product("PROD-001", "Headphones", 1299.0);
        // JVM may reorder — other threads could see product != null
        // but fields inside Product may still be default values (null, 0)
    }
}
// Thread 2: if (product != null) use(product.name) → NullPointerException!
```

### Four Ways to Safely Publish

```java
// ✅ Method 1: static initializer — safest, JVM guarantees it
public class SafePublication1 {
    public static final Product PRODUCT =
        new Product("PROD-001", "Headphones", 1299.0);
    // JVM initializes before any thread accesses the class
}

// ✅ Method 2: volatile field
public class SafePublication2 {
    public static volatile Product product; // volatile write = safe publish
    public static void set(Product p) { product = p; }
}

// ✅ Method 3: synchronized setter/getter
public class SafePublication3 {
    private static Product product;
    public static synchronized void set(Product p) { product = p; }
    public static synchronized Product get() { return product; }
}

// ✅ Method 4: AtomicReference
public class SafePublication4 {
    private static AtomicReference<Product> product = new AtomicReference<>();
    public static void set(Product p) { product.set(p); }
    public static Product get() { return product.get(); }
}
```

---

## 13.8 The `@GuardedBy` Pattern — Self-Documenting Thread Safety

Not a Java annotation (it's in JSR-305) but the pattern matters — documenting which lock protects which field:

```java
public class OrderBook {

    private final ReentrantLock lock = new ReentrantLock();

    // @GuardedBy("lock") — only access while holding lock
    private final List<Order>  pendingOrders   = new ArrayList<>();
    private final List<Order>  completedOrders = new ArrayList<>();
    private       double       totalRevenue    = 0.0;

    // @GuardedBy("itself") — ConcurrentHashMap is self-guarding
    private final ConcurrentHashMap<String, Order> orderIndex
        = new ConcurrentHashMap<>();

    // @GuardedBy("none") — immutable, no guard needed
    private final String exchangeId = "NSE";

    public void addOrder(Order order) {
        lock.lock();
        try {
            pendingOrders.add(order);     // guarded by lock ✓
            totalRevenue += order.amount(); // guarded by lock ✓
        } finally {
            lock.unlock();
        }
        // Outside lock — ConcurrentHashMap is self-guarding
        orderIndex.put(order.id(), order);
    }
}
```

This pattern makes it immediately clear which fields need protection and which are self-safe.

---

## 13.9 Patterns Summary — Choosing the Right Approach

```
What are you dealing with?
│
├── Immutable data (value objects, configs, constants)
│   └── Make it immutable — final fields, no setters, defensive copies
│       Zero synchronization needed
│       ShopSphere: Money, Address, ProductId, OrderStatus
│
├── Thread-specific data (user context, DB connection, formatters)
│   └── ThreadLocal — each thread has its own copy
│       ALWAYS call remove() in finally
│       ShopSphere: RequestContext, SecurityContext, TransactionContext
│
├── Single shared counter / flag / reference
│   └── AtomicInteger / AtomicBoolean / AtomicReference
│       Lock-free, fast
│       ShopSphere: request counter, circuit breaker state, feature flags
│
├── Shared mutable map / collection
│   └── ConcurrentHashMap / CopyOnWriteArrayList / BlockingQueue
│       ShopSphere: product cache, active sessions, order queue
│
├── Shared mutable object (multiple fields together)
│   └── synchronized method or block / ReentrantLock
│       ShopSphere: inventory reservation, wallet deduction
│
├── Read-heavy shared object
│   └── ReadWriteLock — many readers, exclusive writer
│       ShopSphere: product catalog, config cache
│
└── Lazy initialization of expensive singleton
    └── Initialization-on-demand holder OR double-checked locking
        ShopSphere: payment gateway client, Elasticsearch client
```

---

## 13.10 Complete Thread-Safe ShopSphere Service

Tying every pattern from Chapters 4–13 together in one realistic service:

```java
public class ThreadSafeProductService {

    // ─── Immutable config ─────────────────────────────────────────
    private final String serviceId;  // final — immutable

    // ─── Self-guarding concurrent collection ──────────────────────
    private final ConcurrentHashMap<String, Product> cache
        = new ConcurrentHashMap<>();

    // ─── ReadWriteLock for product catalog ────────────────────────
    private final ReadWriteLock catalogLock = new ReentrantReadWriteLock();
    private final Map<String, Product> catalog = new HashMap<>();

    // ─── Atomic counters ──────────────────────────────────────────
    private final AtomicLong   cacheHits   = new AtomicLong(0);
    private final AtomicLong   cacheMisses = new AtomicLong(0);
    private final LongAdder    totalViews  = new LongAdder();

    // ─── ThreadLocal for request tracing ─────────────────────────
    private final ThreadLocal<String> traceId =
        ThreadLocal.withInitial(() -> "TRACE-" + Thread.currentThread().getName());

    public ThreadSafeProductService(String serviceId) {
        this.serviceId = serviceId;
    }

    // ─── Read product — cache-first, then catalog ────────────────
    public Product getProduct(String productId) {
        totalViews.increment(); // LongAdder — high throughput

        String currentTraceId = traceId.get(); // ThreadLocal — no sharing

        // Check cache first — lock-free ConcurrentHashMap read
        Product cached = cache.get(productId);
        if (cached != null) {
            cacheHits.incrementAndGet();
            System.out.println("[" + currentTraceId + "] Cache HIT: " + productId);
            return cached;
        }

        // Cache miss — check catalog with read lock
        cacheMisses.incrementAndGet();
        catalogLock.readLock().lock(); // multiple threads can read simultaneously
        try {
            Product product = catalog.get(productId);
            if (product != null) {
                cache.put(productId, product); // populate cache — self-guarding
                System.out.println("[" + currentTraceId + "] Cache MISS → loaded: " + productId);
                return product;
            }
        } finally {
            catalogLock.readLock().unlock();
        }

        System.out.println("[" + currentTraceId + "] Product not found: " + productId);
        return null;
    }

    // ─── Add/update product — exclusive write ────────────────────
    public void upsertProduct(Product product) {
        catalogLock.writeLock().lock(); // exclusive — all readers wait
        try {
            catalog.put(product.id(), product);
            cache.remove(product.id()); // invalidate stale cache
            System.out.println("Upserted product: " + product.id());
        } finally {
            catalogLock.writeLock().unlock();
        }
    }

    // ─── Metrics — atomic reads ───────────────────────────────────
    public void printMetrics() {
        long hits   = cacheHits.get();
        long misses = cacheMisses.get();
        long total  = hits + misses;
        System.out.printf(
            "Views: %d | Cache hits: %d (%.0f%%) | Misses: %d%n",
            totalViews.sum(), hits,
            total > 0 ? (hits * 100.0 / total) : 0,
            misses
        );
    }

    // ─── ThreadLocal cleanup ──────────────────────────────────────
    public void clearRequestContext() {
        traceId.remove(); // prevent memory leak in thread pool
    }

    // ─── Immutable product record ─────────────────────────────────
    public record Product(String id, String name, double price) {}

    // ─── Demo ──────────────────────────────────────────────────────
    public static void main(String[] args) throws InterruptedException {

        ThreadSafeProductService service = new ThreadSafeProductService("prod-svc-1");

        // Add products
        service.upsertProduct(new Product("PROD-001", "Headphones", 2899.0));
        service.upsertProduct(new Product("PROD-002", "Keyboard",   1499.0));
        service.upsertProduct(new Product("PROD-003", "Monitor",    12999.0));

        ExecutorService pool = Executors.newFixedThreadPool(8);

        // 20 concurrent reads
        for (int i = 0; i < 20; i++) {
            final String productId = "PROD-00" + ((i % 3) + 1);
            pool.submit(() -> {
                try {
                    service.getProduct(productId);
                } finally {
                    service.clearRequestContext(); // ThreadLocal cleanup
                }
            });
        }

        // 2 concurrent writes
        pool.submit(() -> service.upsertProduct(
            new Product("PROD-001", "Headphones v2", 3199.0)));
        pool.submit(() -> service.upsertProduct(
            new Product("PROD-004", "Webcam", 2199.0)));

        pool.shutdown();
        pool.awaitTermination(10, TimeUnit.SECONDS);

        System.out.println("\n─── Service Metrics ──────────────────");
        service.printMetrics();
    }
}
```

**Output:**
```
Upserted product: PROD-001
Upserted product: PROD-002
Upserted product: PROD-003
[TRACE-pool-thread-1] Cache MISS → loaded: PROD-001
[TRACE-pool-thread-2] Cache MISS → loaded: PROD-002
[TRACE-pool-thread-3] Cache HIT: PROD-001
[TRACE-pool-thread-4] Cache MISS → loaded: PROD-003
[TRACE-pool-thread-5] Cache HIT: PROD-002
...
Upserted product: PROD-001   ← write invalidates cache
Upserted product: PROD-004
...
─── Service Metrics ──────────────────
Views: 20 | Cache hits: 14 (70%) | Misses: 6
```

---

## 13.11 Key Takeaways

| Concept | Remember This |
|---|---|
| Immutability | `final` fields, no setters, defensive copies, `final` class. Zero sync needed. |
| Mutable leak | `final List` field still mutable — use `List.copyOf()` in constructor |
| `ThreadLocal` | Per-thread copy — no sharing, no sync. Spring uses this everywhere. |
| `ThreadLocal.remove()` | Always in `finally`. Memory leak in thread pools without it. |
| DCL | `volatile` + double null check + `synchronized` inner block |
| Initialization holder | Cleaner alternative to DCL for singletons |
| Safe publication | Use `volatile`, `static final`, `synchronized`, or `AtomicReference` |
| `@GuardedBy` pattern | Document which lock protects which field — self-documenting code |
| Default choice | Immutability > ThreadLocal > Atomics > Concurrent Collections > Locks |

---

## Interview Questions From This Chapter

**Q1. What is `ThreadLocal` and how does Spring use it?**
> `ThreadLocal` gives each thread its own isolated copy of a variable — no sharing, no synchronization needed. Spring uses it extensively: `SecurityContextHolder` stores the current authenticated user per request thread, `TransactionSynchronizationManager` stores the current transaction, and `RequestContextHolder` stores the HTTP request. Each incoming request runs on its own Tomcat thread and gets its own `ThreadLocal` storage — completely isolated from other concurrent requests.

**Q2. What is the `ThreadLocal` memory leak and how do you prevent it?**
> Thread pools reuse threads. If you set a `ThreadLocal` value and don't call `remove()`, the next request that reuses the same thread will inherit the stale value — wrong data and growing memory usage. Since pool threads live forever, the `ThreadLocalMap` entries are never garbage collected. Prevention: always call `remove()` in a `finally` block after the operation completes — same pattern as `lock.unlock()`.

**Q3. Why is `volatile` mandatory in double-checked locking?**
> Without `volatile`, the JVM can reorder instructions inside the constructor. It may write the object reference to `instance` before the constructor finishes executing — partially constructed object published. Another thread sees `instance != null` and uses a broken object. `volatile` inserts a memory barrier that prevents reordering — the constructor must fully complete before the reference is published.

**Q4. What makes an object truly immutable?**
> Five rules: all fields are `private final`; no setter methods; class is `final` (prevents subclassing); constructor performs defensive copies of any mutable inputs; getter methods return defensive copies or unmodifiable views of any mutable fields. Violating any rule — especially defensive copying — breaks immutability even if fields are `final`.

**Q5. If data is immutable, why does safe publication still matter?**
> Even an immutable object can be seen in a partially constructed state if published unsafely. The JVM can reorder writes inside the constructor and publish the reference before all field assignments complete. Another thread could see the reference but read default values (null, 0) for fields. Safe publication via `volatile`, `static final`, `synchronized`, or `AtomicReference` ensures the fully constructed object is visible before any thread can access the reference.

---

## Where You Stand Now

```
Chapter 10  ConcurrentHashMap, CopyOnWriteArrayList, BlockingQueue  ✅
Chapter 11  ReentrantLock, ReadWriteLock, Condition, StampedLock     ✅
Chapter 12  CAS, AtomicInteger, AtomicReference, LongAdder           ✅
Chapter 13  Immutability, ThreadLocal, DCL, Safe Publication         ✅
```

**Updated interview readiness: ~87%** on multithreading.

One chapter left on the critical path before you're interview-ready:

**Chapter 17: Deadlocks** — how they form, how to write one deliberately, detection with `jstack`, and the four prevention strategies. This is asked in almost every 2 YOE Java interview.

Ready for **Chapter 17: Deadlocks**?
