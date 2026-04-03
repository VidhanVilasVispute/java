# 📚 Java Collections — Chapter 3: `HashMap`
### Full Stack: Theory → Internals → Code → Interview

---

## 🧠 1. THEORY — What is HashMap?

`HashMap` is a **key-value store** based on **hashing**. It gives you O(1) average time for `put()`, `get()`, and `remove()` — the fastest general-purpose map in Java.

It lives in `java.util` and implements:
```
Map → HashMap
```

Key characteristics:
- **Key-value pairs** — each key maps to exactly one value
- **Keys are unique** — duplicate key = value overwritten
- **One `null` key allowed**, multiple `null` values allowed
- **No guaranteed order** of iteration
- **NOT thread-safe**
- Default initial capacity: **16**, load factor: **0.75**

---

## ⚙️ 2. INTERNALS — The Deep Dive

This is the most asked internal in all of Java interviews. Know every layer.

---

### The Backing Structure

```java
// Inside java.util.HashMap source
transient Node<K,V>[] table;      // the hash table — array of buckets
transient int size;               // number of key-value pairs
int threshold;                    // size at which next resize triggers
final float loadFactor;           // default 0.75
transient int modCount;           // fail-fast support
```

```
┌─────────────────────────────────────────────────────────────┐
│              HASHMAP INTERNAL STRUCTURE                      │
│                                                             │
│   table[] (array of 16 buckets by default)                  │
│                                                             │
│   [0]  → null                                               │
│   [1]  → Node{key="Cat", val=1} → Node{key="Rat", val=2}   │
│   [2]  → null                                               │
│   [3]  → Node{key="Dog", val=5}                             │
│   [4]  → null                                               │
│   ...                                                       │
│   [15] → null                                               │
│                                                             │
│   Each slot = a "bucket"                                    │
│   Multiple nodes in a bucket = "collision chain"            │
└─────────────────────────────────────────────────────────────┘
```

---

### The Node — What's Actually Stored

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;   // cached hash of key — avoids recomputing
    final K key;
    V value;
    Node<K,V> next;   // pointer to next node (for chaining)
}
```

---

### Step-by-Step: What Happens on `put("name", "Vidhan")`

This is the single most important flow to know.

```
Step 1: Compute hash of key
        hash = HashMap.hash("name")
             = key.hashCode() XOR (key.hashCode() >>> 16)

Step 2: Find bucket index
        index = hash & (capacity - 1)
              = hash & 15   (for default capacity 16)

Step 3: Check if bucket is empty
        → Yes: create Node, place it   [DONE]
        → No:  collision! handle it

Step 4: On collision — walk the chain
        For each existing node:
          if (node.hash == hash && node.key.equals(key))
              → SAME KEY: overwrite value   [DONE]
          else
              → DIFFERENT KEY: add to chain (or tree)
```

---

### The Hash Function — Why XOR with upper bits?

```java
// Actual source code
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

```
hashCode() = 1111 1111 1111 1111  1010 1010 1010 1010
                                  (lower 16 bits used for index)

Without spreading:
index = hashCode & 15  → only lower 4 bits matter
→ upper bits are WASTED → more collisions

With XOR spread:
h ^ (h >>> 16) mixes upper bits INTO lower bits
→ all 32 bits influence the bucket index
→ fewer collisions
```

This is why you should **never override `hashCode()` poorly** — bad hash = all keys land in same bucket = O(n) performance.

---

### Collision Resolution — Chaining + Treeification (Java 8+)

```
┌──────────────────────────────────────────────────────────────┐
│              COLLISION HANDLING EVOLUTION                     │
│                                                              │
│  Java 7 and before:                                          │
│  Bucket always used LinkedList chain                         │
│  Worst case: all keys in same bucket → O(n) get()           │
│                                                              │
│  Java 8+:                                                    │
│  Chain length ≤ 7  → LinkedList (Node)                       │
│  Chain length = 8  → convert to Red-Black Tree (TreeNode)    │
│  Tree size  ≤ 6    → convert back to LinkedList              │
│                                                              │
│  Tree bucket: O(log n) worst case instead of O(n)            │
│                                                              │
│  TREEIFY_THRESHOLD = 8                                       │
│  UNTREEIFY_THRESHOLD = 6                                     │
│  MIN_TREEIFY_CAPACITY = 64  ← table must be ≥ 64 first      │
└──────────────────────────────────────────────────────────────┘
```

```
Before treeify (chain of 8+):
bucket[3] → [A] → [B] → [C] → [D] → [E] → [F] → [G] → [H]
             O(n) traversal to find any key

After treeify:
bucket[3] →       [D]
                 /   \
               [B]   [F]
              / \   / \
            [A][C][E] [G]
                        \
                        [H]
             O(log n) search
```

---

### Resizing — When and How

```
threshold = capacity × loadFactor
          = 16 × 0.75 = 12

When size > 12 → RESIZE triggered
  newCapacity = oldCapacity × 2   (always power of 2)
  16 → 32 → 64 → 128 → ...

Why power of 2?
  index = hash & (capacity - 1)
  capacity - 1 = 15 = 0000 1111  ← perfect bitmask
  This is faster than modulo (hash % capacity)
```

```
REHASHING during resize:
Every existing node must be re-bucketed into new table.

Key insight (Java 8 optimization):
  newIndex is either:
  → same as old index           (if bit at position oldCapacity is 0)
  → oldIndex + oldCapacity      (if bit at position oldCapacity is 1)

No need to recompute full hash — just check one bit!
```

---

### Full Visual Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   put("city", "Pune") FLOW                      │
│                                                                 │
│  "city".hashCode() = 3053931                                    │
│  hash = 3053931 ^ (3053931 >>> 16) = 3053884                   │
│  index = 3053884 & 15 = 12                                      │
│                                                                 │
│  table[12] == null?                                             │
│     YES → table[12] = new Node(hash, "city", "Pune", null)     │
│     NO  → walk chain, check hash+equals                         │
│              match found → update value                         │
│              no match    → append new Node to chain             │
│                                                                 │
│  size++                                                         │
│  size > threshold (12)? → resize()                             │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⏱️ 3. TIME COMPLEXITY

| Operation | Average | Worst Case | Worst Case Cause |
|---|---|---|---|
| `put(k,v)` | O(1) | O(log n) | All keys in one tree bucket |
| `get(k)` | O(1) | O(log n) | All keys in one tree bucket |
| `remove(k)` | O(1) | O(log n) | All keys in one tree bucket |
| `containsKey(k)` | O(1) | O(log n) | Same |
| `containsValue(v)` | O(n) | O(n) | Full table scan |
| Resize | O(n) | O(n) | Rehash all entries |

> Before Java 8: worst case was O(n). After Java 8 treeification: O(log n).

---

## 💻 4. CODE — Real World, Production-Grade

### Basic Operations

```java
import java.util.HashMap;
import java.util.Map;

Map<String, Integer> inventory = new HashMap<>();

// put — O(1) avg
inventory.put("apple", 100);
inventory.put("banana", 50);
inventory.put("apple", 200); // duplicate key → overwrites, returns OLD value (100)

// get — O(1) avg
int count = inventory.get("apple");        // 200
Integer val = inventory.get("mango");      // null — key not found

// getOrDefault — production must-have
int safe = inventory.getOrDefault("mango", 0); // 0 instead of null

// containsKey / containsValue
inventory.containsKey("apple");    // true  — O(1)
inventory.containsValue(200);      // true  — O(n) full scan

// remove
inventory.remove("banana");        // removes entry, returns value (50)
inventory.remove("apple", 100);    // conditional remove — only if value matches
```

---

### Iteration — The Right Ways

```java
Map<String, Integer> map = new HashMap<>(Map.of("a", 1, "b", 2, "c", 3));

// ✅ Best — entrySet() gives key+value in one shot
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

// ✅ Java 8+ — forEach with lambda
map.forEach((k, v) -> System.out.println(k + " = " + v));

// ❌ Avoid — keySet() then get() = two lookups per entry
for (String key : map.keySet()) {
    System.out.println(map.get(key)); // unnecessary second hash lookup
}
```

---

### Java 8+ Power Methods — Production Essential

```java
Map<String, Integer> wordCount = new HashMap<>();

// putIfAbsent — only inserts if key not present
wordCount.putIfAbsent("hello", 0);

// computeIfAbsent — compute value only if key absent (great for grouping)
Map<String, List<String>> grouped = new HashMap<>();
grouped.computeIfAbsent("fruits", k -> new ArrayList<>()).add("apple");
grouped.computeIfAbsent("fruits", k -> new ArrayList<>()).add("banana");
// {"fruits": ["apple", "banana"]}

// compute — always recomputes value
wordCount.compute("hello", (k, v) -> v == null ? 1 : v + 1);

// merge — most elegant for frequency counting
String[] words = {"hi", "hello", "hi", "world", "hello", "hi"};
Map<String, Integer> freq = new HashMap<>();
for (String word : words) {
    freq.merge(word, 1, Integer::sum);
    // if absent → put(word, 1)
    // if present → put(word, oldValue + 1)
}
// {hi=3, hello=2, world=1}

// replaceAll — transform all values in-place
freq.replaceAll((k, v) -> v * 10);
```

---

### Pre-sizing — A Production Habit

```java
// ❌ BAD — if you know size, don't let it resize
Map<String, Order> orders = new HashMap<>();

// ✅ GOOD — pre-size to avoid resize
// Formula: expectedSize / loadFactor + 1
int expectedSize = 1000;
Map<String, Order> orders = new HashMap<>(
    (int)(expectedSize / 0.75) + 1  // = 1334
);
// This guarantees no resize up to 1000 entries
```

> In Java 19+, you can use `HashMap.newHashMap(expectedSize)` which does this math for you.

---

### Real Service Pattern — Grouping Orders by Status

```java
// Real ShopSphere pattern — grouping orders
List<Order> orders = orderRepository.findAll();

Map<OrderStatus, List<Order>> grouped = new HashMap<>();
for (Order o : orders) {
    grouped.computeIfAbsent(o.getStatus(), k -> new ArrayList<>())
           .add(o);
}

// Or Java 8 Streams
Map<OrderStatus, List<Order>> grouped = orders.stream()
    .collect(Collectors.groupingBy(Order::getStatus));
```

---

### The `equals()` + `hashCode()` Contract — Critical

```java
public class ProductId {
    private String sku;
    private String warehouse;

    // ❌ WITHOUT these — HashMap won't work correctly
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof ProductId)) return false;
        ProductId that = (ProductId) o;
        return Objects.equals(sku, that.sku) &&
               Objects.equals(warehouse, that.warehouse);
    }

    @Override
    public int hashCode() {
        return Objects.hash(sku, warehouse);
    }
}

// Without proper equals+hashCode:
Map<ProductId, Integer> map = new HashMap<>();
map.put(new ProductId("SKU1", "WH1"), 100);
map.get(new ProductId("SKU1", "WH1")); // returns NULL — different object!
// get() uses hashCode to find bucket, equals to find node
// if either is wrong → key never found
```

---

## 🎯 5. INTERVIEW TRAPS & EDGE CASES

### Trap 1 — Mutable key in HashMap

```java
// ❌ NEVER use mutable objects as HashMap keys
List<String> key = new ArrayList<>(List.of("a", "b"));
Map<List<String>, Integer> map = new HashMap<>();
map.put(key, 1);

key.add("c");        // mutate the key!
map.get(key);        // returns NULL — hashCode changed, bucket changed
map.get(List.of("a","b")); // also NULL — old hash, empty bucket now

// ✅ Use immutable keys — String, Integer, records, enums
```

---

### Trap 2 — `null` key behavior

```java
Map<String, String> map = new HashMap<>();
map.put(null, "nullValue");   // ✅ allowed — stored at index 0 always
map.get(null);                // "nullValue"

// Contrast with Hashtable / ConcurrentHashMap:
// Both throw NullPointerException for null keys or values
```

---

### Trap 3 — `HashMap` vs `Hashtable` vs `ConcurrentHashMap`

| | HashMap | Hashtable | ConcurrentHashMap |
|---|---|---|---|
| Thread-safe | ❌ | ✅ (full sync) | ✅ (segment/bucket lock) |
| Null key | ✅ (1) | ❌ | ❌ |
| Null value | ✅ | ❌ | ❌ |
| Performance | Fast | Slow | Fast under concurrency |
| Use today | Single thread | Never | Multi-thread |

> `Hashtable` is legacy — **never use it** in new code.

---

### Trap 4 — Iteration and structural modification

```java
Map<String, Integer> map = new HashMap<>(Map.of("a", 1, "b", 2));

// ❌ ConcurrentModificationException
for (String key : map.keySet()) {
    if (key.equals("a")) map.remove(key);
}

// ✅ Use entrySet iterator
Iterator<Map.Entry<String,Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    if (it.next().getKey().equals("a")) it.remove();
}

// ✅ Or Java 8+
map.entrySet().removeIf(e -> e.getKey().equals("a"));
```

---

### Trap 5 — `containsValue()` is O(n)

```java
// ❌ Never call this in a loop — full table scan every time
for (Order o : orders) {
    if (map.containsValue(o.getId())) { ... } // O(n) per call → O(n²) total
}

// ✅ If you need reverse lookup — build an inverse map
Map<Integer, String> inverseMap = new HashMap<>();
originalMap.forEach((k, v) -> inverseMap.put(v, k));
```

---

## 🔥 6. INTERVIEW QUESTIONS — 2-3 Yr Exp Level

**Theory Level:**
1. How does `HashMap` resolve collisions?
2. What is the default capacity and load factor of HashMap? Why 0.75?
3. What happens when two keys have the same `hashCode()` but different `equals()`?
4. What changed in HashMap internals between Java 7 and Java 8?

**Internals Level:**
5. Explain the `hash()` function in HashMap. Why does it XOR with `hashCode >>> 16`?
6. Why is HashMap capacity always a power of 2?
7. What is `TREEIFY_THRESHOLD`? When does a bucket convert to a Red-Black Tree?
8. What is `MIN_TREEIFY_CAPACITY` and why does it exist?
9. Explain what happens step-by-step during a HashMap resize.

**Code/Trap Level:**
10. What happens if you use a mutable object as a HashMap key and then mutate it?
11. What is the difference between `putIfAbsent()` and `computeIfAbsent()`?
12. Why should you never use `containsValue()` in a performance-critical loop?
13. How do you safely remove entries from a HashMap during iteration?
14. What happens if `hashCode()` always returns the same value (e.g., `return 42`)?

**Design Level:**
15. Why is `String` the most common HashMap key in Java?
16. How would you implement a frequency counter for words using HashMap? What's the most elegant way?
17. How does `ConcurrentHashMap` differ from `Collections.synchronizedMap()`?
18. In your ShopSphere order service, if you're grouping 100k orders by status — how do you optimize the HashMap usage?

---

> 💡 **Answer to Q14 — `hashCode()` always returns 42:**
> All keys hash to the same bucket. Before Java 8 → one giant LinkedList → O(n) for every `get()`. After Java 8 → once chain hits 8, it treeifies → O(log n). HashMap still **works correctly**, just degrades to a tree/list. This is why a good `hashCode()` distribution is critical for O(1) performance.

> 💡 **Answer to Q2 — Why 0.75 load factor:**
> It's a **time-space tradeoff** sweet spot. Lower LF (e.g. 0.5) = fewer collisions, more memory wasted. Higher LF (e.g. 0.9) = more collisions, less memory wasted. 0.75 was empirically determined to minimize collision probability while keeping memory reasonable — based on Poisson distribution of key distribution.

---

**Ready for Chapter 4?** We can go to **`LinkedHashMap`** (insertion-order map, LRU cache implementation) or jump straight to **`TreeMap`** (sorted map, NavigableMap internals). Which one? 🎯
