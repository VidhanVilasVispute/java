# MAP - 
A Map in Java stores key–value pairs where each key is unique. It provides efficient lookup, insertion, and deletion based on keys. Ordering and null support are implementation-specific, and Map does not extend the Collection interface.
# HashMap in Java — Interview Notes

`HashMap` is a hash table–based implementation of `Map` that stores key-value pairs using an array of buckets. It uses hashing to compute the index and handles collisions using linked lists or red-black trees (Java 8+). Provides **O(1) average** time complexity for `put` and `get`, but depends on proper `hashCode` and `equals` implementations.

---

## 🔹 Internal Working

### Step-by-step on `put(key, value)`

1. `key.hashCode()` is computed and spread using a bitwise operation to reduce collisions
2. Bucket index is calculated: `(n - 1) & hash` where `n` is current capacity (always a power of 2)
3. If bucket is empty → element inserted directly
4. If collision → stored in a **linked list** at that bucket
5. If bucket size exceeds **8** AND capacity ≥ **64** → linked list converts to a **red-black tree** (worst case O(n) → O(log n))

### Resizing

- Triggers when: `elements > capacity × load factor` (default load factor = **0.75**)
- Capacity **doubles** on resize
- Elements redistributed without recomputing full hashes

### Backing structure (simplified)

```java
class Node {
    int hash;
    K key;
    V value;
    Node next; // linked list chain for collisions
}
```

---

## 🔹 Time Complexity

| Operation | Average | Worst Case |
|---|---|---|
| `get(key)` | O(1) | O(log n) after Java 8 treeification |
| `put(key, value)` | O(1) | O(log n) |
| `remove(key)` | O(1) | O(log n) |
| `containsKey` | O(1) | O(log n) |

---

## 🔥 Practice Problems

### 1. Basic CRUD

```java
Map<String, Integer> map = new HashMap<>();

map.put("A", 1);
map.put("B", 2);

System.out.println(map.get("A")); // 1

map.put("A", 10);          // overwrite
map.remove("B");

System.out.println(map.containsKey("A")); // true
```

---

### 2. Frequency Count *(very common)*

```java
int[] arr = {1, 2, 2, 3, 1, 4};

Map<Integer, Integer> map = new HashMap<>();

for (int num : arr) {
    map.put(num, map.getOrDefault(num, 0) + 1);
}

System.out.println(map); // {1=2, 2=2, 3=1, 4=1}
```

> If you don't know `getOrDefault`, you're behind.

---

### 3. First Non-Repeating Character

```java
String s = "aabbcde";

Map<Character, Integer> map = new HashMap<>();

for (char c : s.toCharArray()) {
    map.put(c, map.getOrDefault(c, 0) + 1);
}

for (char c : s.toCharArray()) {
    if (map.get(c) == 1) {
        System.out.println(c); // 'c'
        break;
    }
}
```

---

### 4. Two Sum *(asked everywhere)*

```java
int[] arr = {2, 7, 11, 15};
int target = 9;

Map<Integer, Integer> map = new HashMap<>();

for (int i = 0; i < arr.length; i++) {
    int complement = target - arr[i];

    if (map.containsKey(complement)) {
        System.out.println(map.get(complement) + " " + i); // 0 1
        return;
    }

    map.put(arr[i], i);
}
```

---

### 5. Group Anagrams

```java
String[] strs = {"eat", "tea", "tan", "ate", "nat", "bat"};

Map<String, List<String>> map = new HashMap<>();

for (String word : strs) {
    char[] chars = word.toCharArray();
    Arrays.sort(chars);
    String key = new String(chars);

    map.computeIfAbsent(key, k -> new ArrayList<>()).add(word);
}

System.out.println(map.values()); // [[eat, tea, ate], [tan, nat], [bat]]
```

> Uses `computeIfAbsent` — know this method cold.

---

### 6. Sort Map by Value

```java
Map<String, Integer> map = new HashMap<>();
map.put("A", 3);
map.put("B", 1);
map.put("C", 2);

List<Map.Entry<String, Integer>> list = new ArrayList<>(map.entrySet());
list.sort(Map.Entry.comparingByValue());

for (Map.Entry<String, Integer> e : list) {
    System.out.println(e.getKey() + " " + e.getValue());
}
// B 1 → C 2 → A 3
```

---

### 7. LRU Cache *(production-level)*

```java
class LRUCache extends LinkedHashMap<Integer, Integer> {
    private int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // true = access order
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity;
    }
}

LRUCache cache = new LRUCache(2);
cache.put(1, 1);
cache.put(2, 2);
cache.get(1);       // access key 1
cache.put(3, 3);    // evicts key 2 (least recently used)

System.out.println(cache); // {1=1, 3=3}
```

> `LinkedHashMap` with `accessOrder = true` maintains LRU order automatically. `removeEldestEntry` handles eviction.

---

### 8. Detect Duplicates

```java
int[] arr = {1, 2, 3, 2, 4, 1};

Set<Integer> set = new HashSet<>();

for (int num : arr) {
    if (!set.add(num)) {
        System.out.println("Duplicate: " + num);
    }
}
// Duplicate: 2
// Duplicate: 1
```

---

### 9. Subarray Sum Equals K *(separates average vs strong)*

```java
int[] nums = {1, 1, 1};
int k = 2;

Map<Integer, Integer> map = new HashMap<>();
map.put(0, 1); // base case: empty prefix sum

int sum = 0, count = 0;

for (int num : nums) {
    sum += num;

    if (map.containsKey(sum - k)) {
        count += map.get(sum - k);
    }

    map.put(sum, map.getOrDefault(sum, 0) + 1);
}

System.out.println(count); // 2
```

> Core idea: if `sum - k` exists in the map, there's a subarray ending here that sums to `k`. `map.put(0, 1)` handles subarrays starting from index 0.

---

### 10. Custom Object as Key *(critical understanding)*

```java
class User {
    int id;

    User(int id) { this.id = id; }

    @Override
    public int hashCode() {
        return id;
    }

    @Override
    public boolean equals(Object o) {
        return this.id == ((User) o).id;
    }
}

Map<User, String> map = new HashMap<>();
map.put(new User(1), "A");

System.out.println(map.get(new User(1))); // "A"
```

> Without overriding BOTH `hashCode` AND `equals`, two `new User(1)` objects would never match in the map.

---

## 🔹 Critical Contract: hashCode + equals

| Scenario | Result |
|---|---|
| `hashCode` overridden, `equals` not | Multiple entries for the same logical key |
| `equals` overridden, `hashCode` not | Object lands in wrong bucket — never found |
| Both overridden correctly | Works as expected |

> **Rule:** If two objects are `equals()`, they MUST have the same `hashCode()`. The reverse is not required.

---

## 🔹 HashMap vs Variants

| | `HashMap` | `LinkedHashMap` | `TreeMap` |
|---|---|---|---|
| Order | No order | Insertion / access order | Sorted by key |
| `get` / `put` | O(1) | O(1) | O(log n) |
| Use case | General purpose | LRU cache, ordered iteration | Sorted data |
| Null keys | ✅ 1 allowed | ✅ | ❌ |

---

## ⚠️ Common Mistakes

- Using a mutable object as a key — if it changes after insertion, its hash changes and the entry can never be found ❌
- Forgetting to override both `hashCode` AND `equals` for custom keys ❌
- Assuming `HashMap` iteration order is consistent ❌ (use `LinkedHashMap` if order matters)
- Not setting initial capacity when size is known — causes unnecessary resize cycles ❌

===========================================================================================================================================

# LinkedHashMap in Java — Interview Notes

`LinkedHashMap` is an extension of `HashMap` that maintains **insertion or access order** using a doubly linked list in addition to the hash table. It provides O(1) performance for basic operations while preserving order, and is commonly used to implement LRU caches using access-order mode.

---

## 🔹 Key Difference from HashMap

| | `HashMap` | `LinkedHashMap` |
|---|---|---|
| Order | No order | Insertion order (default) or access order |
| Internal structure | Array of buckets | Array of buckets + doubly linked list |
| Performance | O(1) | O(1) |
| Use case | General purpose | Order-preserving, LRU cache |

---

## 🔹 Two Modes

### Insertion Order (default)

```java
new LinkedHashMap<>()
```

Iterates in the order keys were first inserted.

### Access Order

```java
new LinkedHashMap<>(initialCapacity, loadFactor, true) // true = access order
```

Every `get()` or `put()` moves that entry to the **end** of the list. Oldest (least recently used) stays at the front. This is the foundation of LRU cache.

---

## 🔥 Practice Problems

### 1. Basic Usage — Insertion Order

```java
Map<Integer, String> map = new LinkedHashMap<>();

map.put(3, "C");
map.put(1, "A");
map.put(2, "B");

for (Map.Entry<Integer, String> e : map.entrySet()) {
    System.out.println(e.getKey() + " " + e.getValue());
}
// Output: 3 C → 1 A → 2 B  (insertion order preserved)
```

---

### 2. Access Order *(most people forget this)*

```java
Map<Integer, String> map = new LinkedHashMap<>(16, 0.75f, true);

map.put(1, "A");
map.put(2, "B");
map.put(3, "C");

map.get(1); // accessing key 1 → moves it to the end

System.out.println(map); // {2=B, 3=C, 1=A}
```

> If you don't understand this, you cannot build LRU.

---

### 3. LRU Cache *(interview must)*

```java
class LRUCache extends LinkedHashMap<Integer, Integer> {
    private int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // access-order = true
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
        return size() > capacity; // evict when over capacity
    }
}

LRUCache cache = new LRUCache(2);
cache.put(1, 1);
cache.put(2, 2);
cache.get(1);    // access key 1 → moves to end
cache.put(3, 3); // capacity exceeded → evicts key 2 (least recently used)

System.out.println(cache); // {1=1, 3=3}
```

> `removeEldestEntry` is called after every `put`. Returns `true` → eldest entry is removed automatically.

---

### 4. Frequency Count + Preserve Insertion Order

```java
String s = "aabbcde";

Map<Character, Integer> map = new LinkedHashMap<>();

for (char c : s.toCharArray()) {
    map.put(c, map.getOrDefault(c, 0) + 1);
}

System.out.println(map); // {a=2, b=2, c=1, d=1, e=1} — in insertion order
```

> Use `LinkedHashMap` here instead of `HashMap` when the order of first appearance matters.

---

### 5. Remove Eldest Manually

```java
LinkedHashMap<Integer, String> map = new LinkedHashMap<>();

map.put(1, "A");
map.put(2, "B");
map.put(3, "C");

Iterator<Integer> it = map.keySet().iterator();
if (it.hasNext()) {
    it.next();
    it.remove(); // removes oldest (first inserted)
}

System.out.println(map); // {2=B, 3=C}
```

---

### 6. First Unique Character *(order matters)*

```java
String s = "aabbcde";

Map<Character, Integer> map = new LinkedHashMap<>();

for (char c : s.toCharArray()) {
    map.put(c, map.getOrDefault(c, 0) + 1);
}

for (Map.Entry<Character, Integer> e : map.entrySet()) {
    if (e.getValue() == 1) {
        System.out.println(e.getKey()); // 'c'
        break;
    }
}
```

> `HashMap` would work for counting, but `LinkedHashMap` guarantees we find the first unique character in original order.

---

### 7. Sort HashMap by Value → Store in LinkedHashMap

```java
Map<String, Integer> map = new HashMap<>();
map.put("A", 3);
map.put("B", 1);
map.put("C", 2);

List<Map.Entry<String, Integer>> list = new ArrayList<>(map.entrySet());
list.sort(Map.Entry.comparingByValue());

Map<String, Integer> result = new LinkedHashMap<>();
for (Map.Entry<String, Integer> e : list) {
    result.put(e.getKey(), e.getValue());
}

System.out.println(result); // {B=1, C=2, A=3}
```

> Pattern: sort into a `List<Entry>`, then pour into a `LinkedHashMap` to preserve the sorted order.

---

## 🔹 Next Level: LRU Without LinkedHashMap

> This is where real engineers stand out.

Implement LRU using **HashMap + custom Doubly Linked List** — gives you full control and is the expected answer in system design / senior interviews.

```java
class LRUCache {
    private final int capacity;
    private final Map<Integer, Node> map = new HashMap<>();

    private final Node head = new Node(0, 0); // dummy head
    private final Node tail = new Node(0, 0); // dummy tail

    public LRUCache(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }

    public int get(int key) {
        if (!map.containsKey(key)) return -1;
        Node node = map.get(key);
        remove(node);
        insertToFront(node);
        return node.val;
    }

    public void put(int key, int value) {
        if (map.containsKey(key)) remove(map.get(key));
        if (map.size() == capacity) {
            Node lru = tail.prev;
            remove(lru);
            map.remove(lru.key);
        }
        Node node = new Node(key, value);
        insertToFront(node);
        map.put(key, node);
    }

    private void remove(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    private void insertToFront(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }

    static class Node {
        int key, val;
        Node prev, next;
        Node(int key, int val) { this.key = key; this.val = val; }
    }
}
```

**Why this matters:**
- `HashMap` → O(1) lookup by key
- Doubly linked list → O(1) move-to-front and evict-from-tail
- Combined → O(1) for both `get` and `put`

---

## ⚠️ Readiness Check

| What you can do | Level |
|---|---|
| Only know `put` / `get` | Weak |
| Understand insertion vs access order | Basic |
| Can write LRU using `LinkedHashMap` | Interview-ready |
| Can write LRU using `HashMap + DLL` | Strong |

=========================================================================================================

# TreeMap in Java — Interview Notes

`TreeMap` is a **Red-Black Tree–based** implementation of `NavigableMap` that maintains keys in **sorted order**. It provides **O(log n)** time complexity for all operations and supports navigation methods like `floor`, `ceiling`, and `subMap`, making it the go-to choice for range-based queries.

---

## 🔹 Key Difference from HashMap / LinkedHashMap

| | `HashMap` | `LinkedHashMap` | `TreeMap` |
|---|---|---|---|
| Order | No order | Insertion / access order | Sorted by key |
| Performance | O(1) | O(1) | O(log n) |
| Internal structure | Hash table | Hash table + DLL | Red-Black Tree |
| Navigation methods | ❌ | ❌ | ✅ floor, ceiling, subMap |
| Null keys | ✅ | ✅ | ❌ |
| Use case | General purpose | LRU cache | Range queries, sorted data |

---

## 🔹 Key Navigation Methods

| Method | Returns |
|---|---|
| `floorKey(k)` | Largest key ≤ k |
| `ceilingKey(k)` | Smallest key ≥ k |
| `lowerKey(k)` | Largest key strictly < k |
| `higherKey(k)` | Smallest key strictly > k |
| `firstKey()` | Minimum key |
| `lastKey()` | Maximum key |
| `subMap(from, to)` | Keys in range [from, to) |
| `descendingMap()` | View with keys in reverse order |

---

## 🔥 Practice Problems

### 1. Basic Sorting

```java
TreeMap<Integer, String> map = new TreeMap<>();

map.put(3, "C");
map.put(1, "A");
map.put(2, "B");

System.out.println(map); // {1=A, 2=B, 3=C} — sorted automatically
```

---

### 2. Floor / Ceiling *(interview favorite)*

```java
TreeMap<Integer, String> map = new TreeMap<>();

map.put(10, "A");
map.put(20, "B");
map.put(30, "C");

System.out.println(map.floorKey(25));   // 20 — largest key ≤ 25
System.out.println(map.ceilingKey(25)); // 30 — smallest key ≥ 25
```

> If you don't know these methods, you're missing the entire point of `TreeMap`.

---

### 3. Range Query — subMap

```java
TreeMap<Integer, String> map = new TreeMap<>();

map.put(10, "A");
map.put(20, "B");
map.put(30, "C");
map.put(40, "D");

System.out.println(map.subMap(20, 40));
// {20=B, 30=C} — inclusive start, exclusive end
```

---

### 4. Frequency Count in Sorted Order

```java
int[] arr = {3, 1, 2, 1, 3, 2, 4};

TreeMap<Integer, Integer> map = new TreeMap<>();

for (int num : arr) {
    map.put(num, map.getOrDefault(num, 0) + 1);
}

System.out.println(map); // {1=2, 2=2, 3=2, 4=1} — keys sorted
```

> Use `TreeMap` over `HashMap` here when you need to process frequencies in sorted key order.

---

### 5. Custom Sorting — Descending

```java
TreeMap<Integer, String> map = new TreeMap<>((a, b) -> b - a);

map.put(1, "A");
map.put(3, "C");
map.put(2, "B");

System.out.println(map); // {3=C, 2=B, 1=A}
```

---

### 6. Find Closest Value *(real problem — TreeMap beats HashMap)*

```java
TreeMap<Integer, String> map = new TreeMap<>();

map.put(10, "A");
map.put(20, "B");
map.put(30, "C");

int target = 25;

Integer floor = map.floorKey(target);
Integer ceil  = map.ceilingKey(target);

if (floor == null)      System.out.println(ceil);
else if (ceil == null)  System.out.println(floor);
else {
    int closest = (target - floor < ceil - target) ? floor : ceil;
    System.out.println(closest); // 30
}
```

> `HashMap` cannot do this efficiently — it has no concept of ordering. `TreeMap` gives you the two nearest keys in O(log n).

---

### 7. Sliding Window Maximum *(strong interview problem)*

```java
int[] nums = {1, 3, -1, -3, 5, 3, 6, 7};
int k = 3;

TreeMap<Integer, Integer> map = new TreeMap<>();

for (int i = 0; i < nums.length; i++) {
    map.put(nums[i], map.getOrDefault(nums[i], 0) + 1);

    if (i >= k - 1) {
        System.out.println(map.lastKey()); // max element in current window

        int out = nums[i - k + 1];
        map.put(out, map.get(out) - 1);
        if (map.get(out) == 0) map.remove(out);
    }
}
// Output: 3 3 5 5 6 7
```

> `map.lastKey()` gives the window max in O(log n). Removing the outgoing element and decrementing its count handles duplicates correctly.

---

### 8. Leaderboard — Highest Score First

```java
TreeMap<Integer, String> map = new TreeMap<>();

map.put(90, "A");
map.put(95, "B");
map.put(85, "C");

System.out.println(map.descendingMap());
// {95=B, 90=A, 85=C} — highest score first
```

---

## ⚠️ Common Mistakes

- Using `TreeMap` for general key-value storage where order doesn't matter — pay O(log n) for nothing ❌
- Only using `TreeMap` for basic sorting and ignoring navigation methods ❌
- Inserting `null` keys — `TreeMap` throws `NullPointerException` (unlike `HashMap`) ❌
- Forgetting that `subMap(from, to)` is **inclusive start, exclusive end** ❌

---

## ⚠️ Readiness Check

| What you can do | Level |
|---|---|
| Only use `TreeMap` for sorted output | Weak |
| Use `floor` / `ceiling` / `subMap` | Solid |
| Solve closest value and range problems | Interview-ready |
| Solve sliding window max with `TreeMap` | Strong |


====================================================================================


# Core Difference Table (No fluff)

| Feature                | HashMap                   | LinkedHashMap               | TreeMap                       |
| ---------------------- | ------------------------- | --------------------------- | ----------------------------- |
| **Ordering**           | No guarantee              | Insertion / Access order    | Sorted (natural / comparator) |
| **Internal Structure** | Array + LinkedList / Tree | HashMap + Doubly LinkedList | Red-Black Tree                |
| **Time Complexity**    | O(1) avg                  | O(1) avg                    | O(log n)                      |
| **Null Key**           | 1 allowed                 | 1 allowed                   | Not allowed ❌                 |
| **Null Values**        | Allowed                   | Allowed                     | Allowed                       |
| **Iteration Order**    | Random                    | Predictable                 | Sorted                        |
| **Memory Usage**       | Low                       | Slightly higher             | Higher                        |
| **Performance**        | Fastest                   | Slight overhead             | Slowest                       |
| **Use Case**           | General purpose           | Order-sensitive / LRU       | Sorted / range queries        |

---

# When to use (this is what interviewers care about)

## Use HashMap

👉 Default choice

* Fast lookup
* Order doesn’t matter

---

## Use LinkedHashMap

👉 When order matters

* Maintain insertion order
* Build **LRU cache**

---

## Use TreeMap

👉 When sorting is required

* Sorted keys
* Range queries (`floor`, `ceiling`)

---

# Real-world scenarios (don’t give theory answers)

| Scenario                      | Correct Choice      |
| ----------------------------- | ------------------- |
| Caching system                | LinkedHashMap (LRU) |
| Leaderboard                   | TreeMap             |
| Fast lookup (user data)       | HashMap             |
| Maintain insertion order logs | LinkedHashMap       |
| Range-based queries           | TreeMap             |

---

# Brutal Truth (pay attention)

* If you use **TreeMap for everything** → performance mistake
* If you ignore **LinkedHashMap for LRU** → weak design
* If you don’t default to **HashMap** → you don’t understand basics

---

# Interview-Level Answer (clean)

> HashMap provides O(1) average time complexity with no ordering guarantees. LinkedHashMap extends HashMap by maintaining insertion or access order using a doubly linked list. TreeMap is based on a Red-Black Tree and maintains sorted order of keys with O(log n) operations, making it suitable for range queries.

---

# Final mental model

* **HashMap → speed**
* **LinkedHashMap → order**
* **TreeMap → sorting**

