# 📚 Java Collections — Chapter 1: `ArrayList`
### Full Stack: Theory → Internals → Code → Interview

---

## 🧠 1. THEORY — What is ArrayList?

`ArrayList` is a **resizable-array implementation** of the `List` interface. It maintains insertion order, allows duplicate and null elements, and provides fast index-based access.

It lives in `java.util` and implements:
```
Iterable → Collection → List → ArrayList
```

Key characteristics:
- **Ordered** — maintains insertion order
- **Index-based** — O(1) random access
- **Allows duplicates**
- **Allows `null`**
- **NOT thread-safe**

---

## ⚙️ 2. INTERNALS — What's happening under the hood?

This is where most people lose. Let's go deep.

### The backing array

```java
// Inside java.util.ArrayList source code
transient Object[] elementData;  // the actual storage
private int size;                // logical size (not capacity)
```

`ArrayList` is literally just a **plain `Object[]` array** internally. All the "dynamic" magic is just array copying.

---

### Default Capacity & Construction

```java
ArrayList<String> list = new ArrayList<>();        // capacity = 10 (lazy)
ArrayList<String> list = new ArrayList<>(50);      // capacity = 50
ArrayList<String> list = new ArrayList<>(otherList); // capacity = otherList.size()
```

> ⚠️ **Important:** When you do `new ArrayList<>()`, the backing array is actually `EMPTY_ELEMENTDATA = {}` — it's **lazily initialized** to capacity 10 only on the **first `add()`**. This is an optimization added in Java 8.

---

### 📈 Growth Algorithm — The Most Asked Internal

When you add an element and the array is full, ArrayList **grows**:

```
newCapacity = oldCapacity + (oldCapacity >> 1)
```

That's: **new capacity = old capacity × 1.5** (right shift by 1 = divide by 2)

```
Capacity 0  → first add → 10
Capacity 10 → grows to  → 15
Capacity 15 → grows to  → 22
Capacity 22 → grows to  → 33
... and so on
```

Internally it calls `Arrays.copyOf()` which uses `System.arraycopy()` (native, very fast).

```
┌─────────────────────────────────────────────┐
│         ARRAYLIST GROWTH VISUALIZATION      │
│                                             │
│  [A][B][C][D][E][F][G][H][I][J]   ← full    │
│   0  1  2  3  4  5  6  7  8  9              │
│                                             │
│           add("K") called                   │
│                ↓                            │
│  [A][B][C][D][E][F][G][H][I][J][_][_][_]    │
│   ← old 10 copied ──────────────→ 5 new     │
│                                             │
│  new capacity = 10 + (10>>1) = 15           │
└─────────────────────────────────────────────┘
```

---

### `add(index, element)` — Shifting Cost

```
list = [A, B, C, D, E]
list.add(1, "X")  // insert at index 1

Before: [A][B][C][D][E]
After:  [A][X][B][C][D][E]
              ↑ B,C,D,E all shifted right by 1
```

This is O(n) — every element after the index must shift.

---

## ⏱️ 3. TIME COMPLEXITY

| Operation | Time | Why |
|---|---|---|
| `get(index)` | O(1) | Direct array access |
| `add(element)` (end) | O(1) amortized | Occasional resize is O(n) but rare |
| `add(index, element)` | O(n) | Shift elements right |
| `remove(index)` | O(n) | Shift elements left |
| `remove(object)` | O(n) | Linear search + shift |
| `contains(object)` | O(n) | Linear scan |
| `size()` | O(1) | Just returns `size` field |
| `iterator()` | O(n) | Full traversal |

---

## 💻 4. CODE — Real World, Production-Grade

### Basic Operations

```java
import java.util.ArrayList;
import java.util.List;
import java.util.Collections;

public class ArrayListDemo {
    public static void main(String[] args) {

        // Always code to interface, not implementation
        List<String> cities = new ArrayList<>();

        // add
        cities.add("Mumbai");
        cities.add("Pune");
        cities.add("Delhi");
        cities.add(1, "Bangalore"); // insert at index 1

        // get
        System.out.println(cities.get(0)); // Mumbai

        // remove
        cities.remove("Delhi");           // by object — O(n)
        cities.remove(0);                 // by index  — O(n) shift

        // iterate — prefer for-each or iterator, never index-based in production
        for (String city : cities) {
            System.out.println(city);
        }

        // size vs isEmpty
        System.out.println(cities.size());    // logical size
        System.out.println(cities.isEmpty()); // true if size == 0

        // contains
        System.out.println(cities.contains("Pune")); // true — uses .equals()

        // sort
        Collections.sort(cities);
        cities.sort(String::compareTo); // Java 8+ way
    }
}
```

---

### Pre-sizing — A Production Habit

```java
// ❌ BAD — if you know the size, don't let it resize multiple times
List<Order> orders = new ArrayList<>();

// ✅ GOOD — pre-size when you know approximate count
List<Order> orders = new ArrayList<>(expectedSize);
```

Pre-sizing avoids repeated `Arrays.copyOf()` calls → major performance gain in loops.

---
### Iteration — Know All Ways

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4);

// 1. Index for loop
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}

// 2. Enhanced for
for (int num : list) {
    System.out.println(num);
}

// 3. Iterator (safe for removal)
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    int val = it.next();
    if (val == 2) it.remove();
}

// 4. forEach (Java 8)
list.forEach(System.out::println);
```

---
### `trimToSize()` and `ensureCapacity()`

```java
List<String> list = new ArrayList<>(1000);  // creates big array of 1000 slots

list.add("A");
list.add("B");
// Now list has only 2 elements, but internal array is still size 1000 (wasting memory)

list.trimToSize();   // Now internal array becomes exactly size 2

// When to use?
// - When you have finished adding elements and want to save memory.
// - Especially useful if you created ArrayList with a large initial capacity but ended up adding //   very few elements.

---------------------------------------------------------------------------------------

List<String> list = new ArrayList<>();   // starts with small array (usually 10)

list.ensureCapacity(500);   // Now it prepares array for at least 500 elements

// Now if you add 400-500 elements, no resizing will happen → faster
for(int i = 0; i < 400; i++) {
    list.add("item " + i);
}

// When to use?

// - When you know in advance that you will add many elements.
// - To avoid multiple slow resizing operations during adding elements in a loop.
```

---

### ConcurrentModificationException Trap

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C", "D"));

// ❌ WRONG — throws ConcurrentModificationException
for (String s : list) {
    if (s.equals("B")) list.remove(s);
}

// ✅ CORRECT — use iterator explicitly
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("B")) it.remove(); // safe remove
}

// ✅ ALSO CORRECT — Java 8+
list.removeIf(s -> s.equals("B"));
```
- `Iterator` tracks `expectedModCount`
- `ArrayList` tracks `modCount` (incremented on every structural change)
- When you call `list.remove()` directly, `modCount` updates but `expectedModCount` doesn't
- On next `it.next()` → `modCount != expectedModCount` → **throws exception**

> This is called **fail-fast behavior** — a design safety mechanism, not a concurrency issue. Also happens in `HashMap` and `HashSet`.


---

### Conversion Tricks

```java
// Array → ArrayList
String[] arr = {"X", "Y", "Z"};
List<String> list = new ArrayList<>(Arrays.asList(arr));

// ArrayList → Array
String[] back = list.toArray(new String[0]);

// List.of() → unmodifiable, NOT an ArrayList
List<String> immutable = List.of("A", "B"); // Java 9+
```

> ⚠️ `Arrays.asList()` returns a **fixed-size** list backed by the array. You can `set()` but not `add()` or `remove()`. Always wrap in `new ArrayList<>()` if you need full mutability.

---

## 🎯 5. INTERVIEW TRAPS & EDGE CASES

### Trap 1 — `remove(int)` vs `remove(Object)`

```java
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3));

nums.remove(1);          // removes INDEX 1 → list = [1, 3]
nums.remove(Integer.valueOf(1)); // removes OBJECT 1 → list = [2, 3]
```

Autoboxing gotcha. With primitives, `remove(int)` is the index overload.

---

### Trap 2 — `ArrayList` is not thread-safe

```java
// ❌ Race condition — two threads adding simultaneously corrupts the array
List<String> shared = new ArrayList<>();

// ✅ Options:
List<String> safe1 = Collections.synchronizedList(new ArrayList<>());
List<String> safe2 = new CopyOnWriteArrayList<>(); // better for read-heavy
```

---

### Trap 3 — `equals()` and `contains()`

```java
List<String> list = new ArrayList<>(List.of("hello"));
list.contains("hello"); // true  — uses .equals()
list.contains("HELLO"); // false — case sensitive
```

`contains()` calls `equals()` on elements — your custom objects need `equals()` overridden or `contains()` will always return `false`.

---

### Trap 4 — `subList()` returns a view, not a copy

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C", "D", "E"));
List<String> sub = list.subList(1, 3); // [B, C] — backed by original list

sub.clear(); // modifies the ORIGINAL list too!
System.out.println(list); // [A, D, E]
```

---

## 🔥 6. INTERVIEW QUESTIONS

**Theory Level:**
1. What is the default initial capacity of ArrayList, and when is it actually allocated?
2. How does ArrayList grow internally? What is the growth formula?
3. What is `modCount` and why does it exist?
4. What's the difference between `size()` and `capacity()`?

**Internals Level:**
5. Why is `add()` O(1) *amortized* and not simply O(1)?
6. What native method does ArrayList use during resizing?
7. Why is `elementData` marked `transient`? How does serialization work then?

**Code/Trap Level:**
8. Why does `Arrays.asList()` throw `UnsupportedOperationException` on `add()`?
9. How do you safely remove elements from an ArrayList during iteration?
10. What is `ConcurrentModificationException` and how does ArrayList detect it?
11. `list.remove(1)` vs `list.remove(Integer.valueOf(1))` — what's the difference?

**Design Level:**
12. When would you choose `LinkedList` over `ArrayList`?
13. When would you use `CopyOnWriteArrayList` instead of `synchronizedList`?
14. If you know you'll insert 10,000 elements, what should you do before the loop?

---

> 💡 **Answer to Q7 (transient):** `elementData` is transient because the backing array may have `null` slots beyond `size`. Custom `writeObject()`/`readObject()` methods in ArrayList only serialize elements up to `size`, avoiding serializing empty slots.

---

**Ready for the next chapter?** We can go to `LinkedList` next — and do a deep **ArrayList vs LinkedList** battle comparison. 🥊
