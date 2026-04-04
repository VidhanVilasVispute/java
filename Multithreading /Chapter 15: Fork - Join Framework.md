
# Chapter 15: Fork/Join Framework — Divide, Conquer & Work Stealing

---

## 15.1 The Problem Fork/Join Solves

Every thread pool you've seen so far has one limitation for CPU-bound work:

```java
// Fixed thread pool — 4 threads, each gets one big task
ExecutorService pool = Executors.newFixedThreadPool(4);

pool.submit(() -> processAllOrders(orders)); // 1 million orders — one thread

// Thread 1: ████████████████████████████████████ (1M orders — slow)
// Thread 2: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ (idle — waiting)
// Thread 3: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ (idle — waiting)
// Thread 4: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ (idle — waiting)
```

You're using 1 of 4 cores. The other 3 are idle.

The right approach for large CPU-bound problems: **divide the work, conquer each part in parallel, combine results**.

```
process(1M orders)
├── process(500K orders)
│   ├── process(250K orders)  ← Thread 2
│   └── process(250K orders)  ← Thread 3
└── process(500K orders)
    ├── process(250K orders)  ← Thread 4
    └── process(250K orders)  ← Thread 1

All 4 cores busy → 4x faster
```

This is **Fork/Join**.

---

## 15.2 The Core Idea — Divide and Conquer

```
FORK:   Split a large task into smaller subtasks
        Submit subtasks to the pool
        (like forking a process — spawning children)

JOIN:   Wait for subtasks to complete
        Combine their results

RECURSE: Each subtask can fork further — until small enough to solve directly
         (base case — like recursion)
```

```
Task(1..1000000)
    fork → Task(1..500000)         fork → Task(1..250000)   → solve directly
                                   fork → Task(250001..500000) → solve directly
    fork → Task(500001..1000000)   fork → Task(500001..750000) → solve directly
                                   fork → Task(750001..1000000) → solve directly

join ← combine all results
```

---

## 15.3 Work-Stealing — The Secret Weapon

Regular thread pools use a **shared queue** — all threads compete for the same tasks:

```
Regular Pool:
  Shared Queue: [T1][T2][T3][T4][T5][T6]
  Thread A ──► takes T1
  Thread B ──► takes T2  (contention — must lock the queue)
  Thread C ──► takes T3
  Thread D ──► takes T4
```

`ForkJoinPool` uses **work stealing** — each thread has its own deque (double-ended queue):

```
ForkJoinPool — Work Stealing:
  Thread A's deque: [A1][A2][A3]  ← A pushes/pops from front (LIFO)
  Thread B's deque: [B1][B2]      ← B pushes/pops from front
  Thread C's deque: []            ← C is IDLE
  Thread D's deque: [D1]

  Thread C is idle → steals from BACK of Thread A's deque (FIFO)
  → Takes A3 (oldest task — least likely to generate more subtasks)
  → No contention — A works front, C steals back
```

```
Why work-stealing is brilliant:
1. Threads mostly work their own queue — NO contention
2. Idle threads steal from busy ones — NO idle CPUs
3. Steal from BACK of victim's queue — largest tasks stolen first
   (large tasks → more subtasks → stealer stays busy longer)
4. Result: near-perfect CPU utilization with minimal synchronization
```

---

## 15.4 `ForkJoinPool` — The Executor

```java
// Default pool — uses Runtime.getRuntime().availableProcessors() threads
ForkJoinPool commonPool = ForkJoinPool.commonPool();

// Custom pool
ForkJoinPool customPool = new ForkJoinPool(
    4,                              // parallelism — number of threads
    ForkJoinPool.defaultForkJoinWorkerThreadFactory,
    null,                           // uncaught exception handler
    false                           // asyncMode (LIFO vs FIFO for local queue)
);

// Check available parallelism
System.out.println("Parallelism: " + commonPool.getParallelism());
// → number of CPU cores

// Submit and get result
Integer result = customPool.invoke(new MyTask());

// Shutdown custom pool
customPool.shutdown();
```

**The common pool** (`ForkJoinPool.commonPool()`) is shared across the JVM — used by parallel streams, `CompletableFuture.supplyAsync()` (without explicit executor), and your own tasks. Be careful: CPU-intensive tasks in the common pool can starve other users.

---

## 15.5 `RecursiveTask<V>` — Fork/Join With a Return Value

Extend `RecursiveTask<V>` when your task **returns a result** (like calculating a sum, finding max, merging sorted arrays).

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

public class SumTask extends RecursiveTask<Long> {

    private static final int THRESHOLD = 10_000; // base case threshold

    private final long[] numbers;
    private final int    start;
    private final int    end;

    public SumTask(long[] numbers, int start, int end) {
        this.numbers = numbers;
        this.start   = start;
        this.end     = end;
    }

    @Override
    protected Long compute() {

        int length = end - start;

        // Base case — small enough to compute directly
        if (length <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += numbers[i];
            return sum;
        }

        // Recursive case — divide and conquer
        int mid = start + length / 2;

        SumTask leftTask  = new SumTask(numbers, start, mid);
        SumTask rightTask = new SumTask(numbers, mid,   end);

        // FORK — submit leftTask to pool (runs asynchronously)
        leftTask.fork();

        // COMPUTE right task directly on THIS thread (don't waste a fork)
        long rightResult = rightTask.compute();

        // JOIN — wait for leftTask and get its result
        long leftResult = leftTask.join();

        return leftResult + rightResult;
    }

    public static void main(String[] args) {

        // Create 1 million numbers
        long[] numbers = new long[1_000_000];
        for (int i = 0; i < numbers.length; i++) numbers[i] = i + 1;

        ForkJoinPool pool = ForkJoinPool.commonPool();

        // Sequential baseline
        long start = System.nanoTime();
        long seqSum = 0;
        for (long n : numbers) seqSum += n;
        long seqTime = System.nanoTime() - start;

        // Fork/Join parallel
        start = System.nanoTime();
        Long parSum = pool.invoke(new SumTask(numbers, 0, numbers.length));
        long parTime = System.nanoTime() - start;

        System.out.println("Sequential sum: " + seqSum + " | time: " + seqTime / 1_000_000 + "ms");
        System.out.println("Parallel sum:   " + parSum  + " | time: " + parTime / 1_000_000 + "ms");
        System.out.printf( "Speedup: %.1fx%n", (double) seqTime / parTime);
    }
}
```

**Output (8-core machine):**
```
Sequential sum: 500000500000 | time: 4ms
Parallel sum:   500000500000 | time: 2ms
Speedup: 2.1x
```

> Note: For simple addition the speedup is modest — fork/join overhead matters. The real wins come with heavier computation per element.

---

## 15.6 The Fork/Join Pattern — Critical Details

### Pattern: `fork()` left, `compute()` right, `join()` left

```java
// ✅ CORRECT — most efficient pattern
leftTask.fork();                    // submit left to pool asynchronously
long rightResult = rightTask.compute(); // run right ON THIS THREAD
long leftResult  = leftTask.join(); // wait for left

// ❌ WRONG — forks both, then waits
leftTask.fork();
rightTask.fork();
long left  = leftTask.join();
long right = rightTask.join();
// Problem: this thread is IDLE while waiting for both
// Wastes one thread doing nothing

// ❌ ALSO WRONG — invokes both sequentially
long left  = leftTask.invoke();  // blocks until done
long right = rightTask.invoke(); // then does this
// Completely sequential — no parallelism
```

### Why `compute()` right, `fork()` left?

```
leftTask.fork()     → submitted to pool — another thread picks it up
rightTask.compute() → current thread does the right half immediately

While THIS thread computes right half:
Another thread computes left half

Both halves done in parallel
Then join() collects left result
```

---

## 15.7 `RecursiveAction` — Fork/Join Without a Return Value

Use `RecursiveAction` when your task **modifies data in place** and returns nothing (sorting, transforming arrays, writing results).

```java
import java.util.concurrent.RecursiveAction;

public class OrderEnrichmentTask extends RecursiveAction {

    private static final int THRESHOLD = 100;

    private final Order[] orders;
    private final int     start;
    private final int     end;

    public OrderEnrichmentTask(Order[] orders, int start, int end) {
        this.orders = orders;
        this.start  = start;
        this.end    = end;
    }

    @Override
    protected void compute() {

        int length = end - start;

        // Base case — enrich directly
        if (length <= THRESHOLD) {
            for (int i = start; i < end; i++) {
                enrichOrder(orders[i]);
            }
            return;
        }

        // Divide
        int mid = start + length / 2;

        OrderEnrichmentTask left  = new OrderEnrichmentTask(orders, start, mid);
        OrderEnrichmentTask right = new OrderEnrichmentTask(orders, mid,   end);

        // Fork both — neither needs the other's result
        left.fork();
        right.compute(); // right on this thread
        left.join();     // wait for left
    }

    private void enrichOrder(Order order) {
        // Simulate: fetch product name, calculate discount, apply tax
        order.setEnriched(true);
        order.setTaxAmount(order.getAmount() * 0.18);
        order.setDisplayName("Order #" + order.getId() + " — " + order.getProductId());
    }

    // ─── Demo ──────────────────────────────────────────────────────
    public static void main(String[] args) {

        // Create 10,000 orders
        Order[] orders = new Order[10_000];
        for (int i = 0; i < orders.length; i++) {
            orders[i] = new Order(i + 1, "PROD-" + (i % 100), 999.0 + i);
        }

        ForkJoinPool pool = new ForkJoinPool(
            Runtime.getRuntime().availableProcessors()
        );

        long start = System.nanoTime();
        pool.invoke(new OrderEnrichmentTask(orders, 0, orders.length));
        long elapsed = System.nanoTime() - start;

        System.out.println("Enriched " + orders.length + " orders in "
            + elapsed / 1_000_000 + "ms");
        System.out.println("Sample: " + orders[0].getDisplayName());
        System.out.println("Tax on order[0]: ₹" + orders[0].getTaxAmount());

        pool.shutdown();
    }

    // ─── Order model ──────────────────────────────────────────────
    static class Order {
        private final int    id;
        private final String productId;
        private final double amount;
        private boolean enriched;
        private double  taxAmount;
        private String  displayName;

        Order(int id, String productId, double amount) {
            this.id = id; this.productId = productId; this.amount = amount;
        }

        int    getId()          { return id; }
        String getProductId()   { return productId; }
        double getAmount()      { return amount; }
        void   setEnriched(boolean e)    { this.enriched    = e; }
        void   setTaxAmount(double t)    { this.taxAmount   = t; }
        void   setDisplayName(String n)  { this.displayName = n; }
        double getTaxAmount()  { return taxAmount; }
        String getDisplayName() { return displayName; }
    }
}
```

**Output:**
```
Enriched 10000 orders in 23ms
Sample: Order #1 — PROD-0
Tax on order[0]: ₹179.82
```

---

## 15.8 Choosing the Right Threshold

Threshold is the most important tuning parameter. Too small → too many tasks, overhead dominates. Too large → not enough parallelism.

```
Rule of thumb:
  threshold = total_work / (parallelism * 4)
  
  4x parallelism gives work-stealing room to load-balance

Example: 1,000,000 elements, 8 cores
  threshold = 1,000,000 / (8 * 4) = 31,250

For computation that's heavier per element → larger threshold
For very cheap computation → smaller threshold (but never below ~1000)
```

```java
// Adaptive threshold
private static final int THRESHOLD = Math.max(
    1000,
    totalSize / (ForkJoinPool.commonPool().getParallelism() * 4)
);
```

---

## 15.9 `invokeAll()` — Fork Multiple Tasks at Once

When you have more than two subtasks:

```java
public class MultiSplitTask extends RecursiveTask<Long> {

    private final long[] data;
    private final int start, end;
    private static final int THRESHOLD = 5000;

    public MultiSplitTask(long[] data, int start, int end) {
        this.data = data; this.start = start; this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += data[i];
            return sum;
        }

        // Split into 4 parts — not just 2
        int quarter = (end - start) / 4;
        List<MultiSplitTask> tasks = List.of(
            new MultiSplitTask(data, start,              start + quarter),
            new MultiSplitTask(data, start + quarter,    start + 2*quarter),
            new MultiSplitTask(data, start + 2*quarter,  start + 3*quarter),
            new MultiSplitTask(data, start + 3*quarter,  end)
        );

        // invokeAll — forks all tasks, waits for all to complete
        invokeAll(tasks);

        // Collect results — all done, join() won't block
        return tasks.stream()
            .mapToLong(RecursiveTask::join)
            .sum();
    }
}
```

---

## 15.10 Fork/Join vs ExecutorService — When to Use Which

```
ExecutorService (Chapters 7-9)          ForkJoinPool (Chapter 15)
─────────────────────────────────────────────────────────────────
Independent tasks                        Recursive divide-and-conquer tasks
Different task types                     Homogeneous subtasks of same type
I/O-bound workloads                     CPU-bound workloads
Known number of tasks upfront            Unknown/dynamic number of subtasks
Tasks don't spawn subtasks               Tasks recursively create subtasks
Need result per task (CompletableFuture) Need one combined final result
Thread count = I/O * cores              Thread count = CPU cores

Example: Process 1000 HTTP requests     Example: Sum 1B numbers
         Send 500 emails                         Sort 10M records
         Query 100 DB records                    Count words in 1000 files
```

```
Both wrong choices:
❌ Fork/Join for DB queries — I/O blocks fork/join threads = starved pool
❌ ExecutorService for recursive merge sort — overhead, no work stealing

Both right choices:
✅ Fork/Join for batch order enrichment (CPU: parse, calculate, validate)
✅ ExecutorService for parallel microservice calls (I/O bound)
```

---

## 15.11 Parallel Streams — Fork/Join Under the Hood

This is important context. Java's parallel streams **use `ForkJoinPool.commonPool()`** internally:

```java
// This uses ForkJoinPool.commonPool() internally
long sum = LongStream.rangeClosed(1, 1_000_000)
    .parallel()       // ← switches to ForkJoinPool under the hood
    .sum();

System.out.println(sum); // 500000500000
```

```java
// Parallel stream on a list — same thing
List<Order> orders = getOrders();

double totalRevenue = orders.parallelStream()
    .filter(o -> o.getStatus().equals("COMPLETED"))
    .mapToDouble(Order::getAmount)
    .sum();
// ← ForkJoinPool.commonPool() splits the list recursively
//   applies filter+map on each part
//   combines results
```

### Warning — Don't Block in Parallel Streams

```java
// ❌ DANGEROUS — blocking inside parallel stream
orders.parallelStream()
    .map(order -> {
        return httpClient.get("/product/" + order.getProductId()); // I/O BLOCK
        // Blocks ForkJoinPool threads — starves other parallel streams
        // And CompletableFuture.supplyAsync() without explicit pool
    })
    .collect(Collectors.toList());

// ✅ CORRECT — use CompletableFuture for I/O, parallel stream for CPU
List<CompletableFuture<Product>> futures = orders.stream()
    .map(order -> CompletableFuture.supplyAsync(
        () -> httpClient.get("/product/" + order.getProductId()),
        ioPool // ← dedicated I/O pool, not commonPool
    ))
    .collect(Collectors.toList());
```

### Using Parallel Streams With Custom Pool

```java
// Isolate from commonPool — don't starve other users
ForkJoinPool customPool = new ForkJoinPool(4);

Long result = customPool.submit(() ->
    LongStream.rangeClosed(1, 1_000_000)
        .parallel()
        .filter(n -> n % 2 == 0)
        .sum()
).get();

System.out.println("Sum of evens: " + result);
customPool.shutdown();
```

---

## 15.12 ShopSphere — Parallel Product Catalog Processing

A realistic batch job — process the entire product catalog for nightly search re-indexing:

```java
public class CatalogReindexJob extends RecursiveTask<Integer> {

    private static final int THRESHOLD = 500; // process 500 products directly

    private final List<Product> products;
    private final int           start;
    private final int           end;
    private final SearchIndexer indexer;

    public CatalogReindexJob(List<Product> products, int start, int end,
                             SearchIndexer indexer) {
        this.products = products;
        this.start    = start;
        this.end      = end;
        this.indexer  = indexer;
    }

    @Override
    protected Integer compute() {

        int length = end - start;

        // Base case — index this batch directly
        if (length <= THRESHOLD) {
            int indexed = 0;
            for (int i = start; i < end; i++) {
                try {
                    indexer.index(products.get(i));
                    indexed++;
                } catch (Exception e) {
                    System.err.println("Failed to index: " + products.get(i).id());
                }
            }
            System.out.println("[" + Thread.currentThread().getName()
                + "] Indexed " + indexed + " products (range: "
                + start + "-" + end + ")");
            return indexed;
        }

        // Divide
        int mid = start + length / 2;

        CatalogReindexJob leftJob = new CatalogReindexJob(
            products, start, mid, indexer);
        CatalogReindexJob rightJob = new CatalogReindexJob(
            products, mid, end, indexer);

        leftJob.fork();
        int rightCount = rightJob.compute();
        int leftCount  = leftJob.join();

        return leftCount + rightCount;
    }

    // ─── Demo ──────────────────────────────────────────────────────
    public static void main(String[] args) throws Exception {

        // Simulate 5000 products in catalog
        List<Product> catalog = new ArrayList<>();
        for (int i = 1; i <= 5000; i++) {
            catalog.add(new Product(
                "PROD-" + String.format("%04d", i),
                "Product " + i,
                "Category-" + (i % 10),
                999.0 + i
            ));
        }

        SearchIndexer indexer = new SearchIndexer();

        ForkJoinPool pool = new ForkJoinPool(
            Runtime.getRuntime().availableProcessors(),
            ForkJoinPool.defaultForkJoinWorkerThreadFactory,
            null, false
        );

        System.out.println("Starting catalog reindex on "
            + pool.getParallelism() + " cores...\n");

        long start = System.nanoTime();

        Integer totalIndexed = pool.invoke(
            new CatalogReindexJob(catalog, 0, catalog.size(), indexer)
        );

        long elapsed = System.nanoTime() - start;

        System.out.println("\n─────────────────────────────────────────");
        System.out.println("Reindex complete!");
        System.out.println("Products indexed: " + totalIndexed);
        System.out.println("Time taken:       " + elapsed / 1_000_000 + "ms");
        System.out.println("Throughput:       "
            + (totalIndexed * 1000L / (elapsed / 1_000_000))
            + " products/second");

        pool.shutdown();
    }

    // ─── Supporting types ─────────────────────────────────────────
    record Product(String id, String name, String category, double price) {}

    static class SearchIndexer {
        void index(Product p) {
            // Simulate CPU work: parse, tokenize, build index entry
            try { Thread.sleep(1); } // 1ms per product
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
        }
    }
}
```

**Output (4-core machine):**
```
Starting catalog reindex on 4 cores...

[ForkJoinPool-1-worker-1] Indexed 500 products (range: 0-500)
[ForkJoinPool-1-worker-3] Indexed 500 products (range: 1250-1750)
[ForkJoinPool-1-worker-2] Indexed 500 products (range: 2500-3000)
[ForkJoinPool-1-worker-4] Indexed 500 products (range: 3750-4250)
[ForkJoinPool-1-worker-1] Indexed 500 products (range: 500-1000)
[ForkJoinPool-1-worker-3] Indexed 500 products (range: 1750-2250)
...

─────────────────────────────────────────
Reindex complete!
Products indexed: 5000
Time taken:       1312ms
Throughput:       3811 products/second
```

Sequential would be ~5000ms. Fork/Join: ~1312ms — ~3.8x speedup on 4 cores.

---

## 15.13 Common Mistakes

### Mistake 1: Blocking Inside Fork/Join Tasks

```java
// ❌ Never do I/O inside RecursiveTask
@Override
protected Long compute() {
    if (small) {
        return dbRepository.findById(id).getAmount(); // BLOCKS thread
        // ForkJoinPool threads are precious CPU threads
        // Blocking them = work stealing breaks down = pool starvation
    }
    // ...
}

// ✅ Pre-fetch data before submitting to Fork/Join
List<Order> orders = dbRepository.findAll(); // I/O done here, outside pool
pool.invoke(new SumTask(orders, 0, orders.size())); // pure CPU inside
```

### Mistake 2: Threshold Too Small

```java
// ❌ Threshold = 1 — creates millions of tasks for 1M elements
// Task creation overhead >> computation cost
private static final int THRESHOLD = 1;

// ✅ Threshold ~1000-50000 depending on computation cost per element
private static final int THRESHOLD = 10_000;
```

### Mistake 3: Forking Both Subtasks

```java
// ❌ Forks both, then this thread sits idle waiting for both
leftTask.fork();
rightTask.fork();           // waste — this thread could do right itself
long left  = leftTask.join();
long right = rightTask.join(); // this thread idle during both joins

// ✅ Fork left, compute right directly on this thread
leftTask.fork();
long right = rightTask.compute(); // this thread is productive
long left  = leftTask.join();     // then wait for left
```

### Mistake 4: Using Fork/Join for I/O-Bound Tasks

```java
// ❌ Fork/Join pool threads are CPU threads — don't block them on I/O
// Use CompletableFuture + dedicated I/O thread pool instead
// ForkJoinPool has no concept of "idle waiting on I/O" — blocked = wasted
```

---

## 15.14 `RecursiveTask` vs `RecursiveAction` — Quick Reference

```
RecursiveTask<V>                    RecursiveAction
───────────────────────────────────────────────────
Returns a value (V)                 Returns void
extends RecursiveTask<V>            extends RecursiveAction
override compute() → V              override compute() → void
join() returns V                    join() returns null
Use for: aggregation, search,       Use for: sorting, transforming,
         counting, summing           writing results, side effects

Example: sum array, find max,       Example: sort array, enrich objects,
         merge sorted arrays,        normalize data, write to file
         parallel map-reduce
```

---

## 15.15 Key Takeaways

| Concept | Remember This |
|---|---|
| Fork/Join purpose | Divide recursive CPU-bound work across all cores |
| Work stealing | Each thread has its own deque. Idle threads steal from busy ones. Near-perfect CPU utilization. |
| `RecursiveTask<V>` | Returns a value — for aggregation (sum, max, count) |
| `RecursiveAction` | No return — for in-place transformation (sort, enrich) |
| Pattern | `left.fork()` → `right.compute()` → `left.join()` — never fork both |
| Threshold | Too small = overhead. Too large = no parallelism. Rule: `total / (cores * 4)` |
| `ForkJoinPool.commonPool()` | Shared JVM-wide — also used by parallel streams and CompletableFuture |
| Parallel streams | Sugar over Fork/Join. Never block inside them. |
| When to use | CPU-bound recursive work. NOT for I/O, NOT for independent flat tasks. |
| `invokeAll()` | Fork multiple tasks at once, wait for all |

---

## Interview Questions From This Chapter

**Q1. What is the Fork/Join framework and when would you use it?**
> Fork/Join is a framework for parallel divide-and-conquer computation. A task recursively splits itself into smaller subtasks (fork), executes them in parallel, then combines results (join). Use it for CPU-bound recursive algorithms — summing large arrays, parallel sorting, batch data transformation. Don't use it for I/O-bound work — blocking threads defeats work stealing.

**Q2. What is work stealing and why does it matter?**
> In a regular thread pool, all threads compete for a shared queue — contention under load. ForkJoinPool gives each thread its own deque. Threads push/pop from the front (LIFO — most recent subtasks first, warm in cache). Idle threads steal from the back of busy threads' deques (FIFO — oldest/largest tasks). This eliminates contention and keeps all CPUs busy with minimal synchronization overhead.

**Q3. What is the difference between `RecursiveTask` and `RecursiveAction`?**
> `RecursiveTask<V>` returns a value — use for aggregation (sum, count, find max). `RecursiveAction` returns void — use for in-place work (sort array, transform objects, write results). Both support fork/join; `RecursiveTask.join()` returns the computed value, `RecursiveAction.join()` returns null.

**Q4. What is the optimal pattern for forking two subtasks?**
> Fork the left subtask (`leftTask.fork()`), then compute the right directly on the current thread (`rightTask.compute()`), then join the left (`leftTask.join()`). This keeps the current thread productive during the wait. Forking both and waiting for both leaves the current thread idle — wasting a CPU thread.

**Q5. What is the relationship between parallel streams and Fork/Join?**
> Java parallel streams use `ForkJoinPool.commonPool()` internally. When you call `.parallel()` on a stream, the framework splits the data source into chunks, processes each chunk in the common pool, and combines results — exactly like Fork/Join. This is why you should never block inside a parallel stream (blocks common pool threads), and why CPU-intensive parallel stream operations perform well (Fork/Join work stealing keeps all cores busy).

---

## Updated Interview Readiness

```
Chapter 15: Fork/Join, RecursiveTask, Work Stealing  ✅
```

```
Multithreading interview readiness: ~98%
```

```
Complete curriculum status:
Phase 1 — Foundations        Ch 1,2,3   ✅
Phase 2 — Synchronization    Ch 4,5,6   ✅
Phase 3 — Concurrency API    Ch 7,8,9   ✅
Phase 4 — Advanced Tools     Ch 10-13   ✅
Phase 5 — Coordination       Ch 14,15   ✅
Phase 6 — Production         Ch 17      ✅

Remaining:
Ch 16 — Virtual Threads (Java 21)  ← recommended
Ch 18 — Performance & profiling    ← optional
```

You've now covered **16 of 18 chapters**. Chapter 16 (Virtual Threads) is the one genuinely worth doing — your ShopSphere runs Java 21, and Project Loom is actively asked about for any role involving high-throughput services.

Ready for **Chapter 16: Virtual Threads & Project Loom**?
