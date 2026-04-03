# Java Collections: List, ArrayList & LinkedList — Interview Notes

---

## 🔹 List

A `List` in Java is an **ordered collection** that:
- Allows duplicate elements
- Provides index-based access
- Preserves insertion order
- Can dynamically grow in size

Performance characteristics (e.g., random access) depend on the specific implementation.

**Hierarchy:** `Iterable → Collection → List → ArrayList / LinkedList`

---

## 🔹 ArrayList

`ArrayList` is a **resizable-array implementation** of the `List` interface. It maintains insertion order, allows duplicate and null elements, and provides fast index-based access.

### Internal Working

```java
// Inside java.util.ArrayList source code
transient Object[] elementData;  // the actual storage
private int size;                // logical size (not capacity)
```
*`ArrayList` is literally just a plain *`Object[]` array internally. All the "dynamic" magic is just array copying.


- Default capacity = **10**
- When full → resizes automatically
- Resize formula: `newCapacity = oldCapacity + (oldCapacity >> 1)` → effectively **×1.5**

```
10 → 15 → 22 → 33 → ...
```


> ⚠️ Resizing is expensive: creates a new array and copies all data → **O(n)**. If you know the size upfront, always set initial capacity:
> ```java
> List<Integer> list = new ArrayList<>(1000);
> ```

### Time Complexity

| Operation | Complexity | Reason |
|---|---|---|
| `get(index)` | O(1) | Direct array access |
| `add(element)` | O(1)* | Amortized |
| `add(index, element)` | O(n) | Shift elements |
| `remove(index)` | O(n) | Shift elements |
| Search | O(n) | Linear scan |

---

## 🔥 ArrayList — Practice Code

### 1. Basic CRUD

```java
import java.util.*;

public class ArrayListCRUD {
    public static void main(String[] args) {
        List<String> list = new ArrayList<>();

        list.add("A");
        list.add("B");
        list.add("C");
        list.add(1, "X");         // insert at index

        System.out.println(list.get(2)); // B

        list.set(2, "Y");         // update
        list.remove(1);           // delete by index
        list.remove("C");         // delete by value

        System.out.println(list); // [A, Y]
    }
}
```

---

### 2. Iteration — Know All Ways

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

### 3. Remove While Iterating — Interview Trap

```java
// ❌ Wrong — throws ConcurrentModificationException
for (Integer num : list) {
    if (num == 2) list.remove(num);
}

// ✅ Correct — Option 1: removeIf
list.removeIf(num -> num == 2);

// ✅ Correct — Option 2: Iterator
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    if (it.next() == 2) it.remove();
}
```

---

### 4. Sorting

```java
List<Integer> list = new ArrayList<>(Arrays.asList(5, 2, 8, 1));

Collections.sort(list);           // ascending
list.sort((a, b) -> b - a);       // descending

System.out.println(list);         // [8, 5, 2, 1]
```

---

### 5. Custom Object Sorting

```java
class Student {
    int id;
    String name;
    Student(int id, String name) { this.id = id; this.name = name; }
}

list.sort(Comparator.comparingInt(s -> s.id));
list.forEach(s -> System.out.println(s.id + " " + s.name));
```

---

### 6. Convert Array ↔ ArrayList

```java
// Array → List
String[] arr = {"A", "B"};
List<String> list = new ArrayList<>(Arrays.asList(arr));

// List → Array
String[] newArr = list.toArray(new String[0]);
```

---

### 7. Find Duplicates

```java
List<Integer> list = Arrays.asList(1, 2, 3, 2, 4, 1);

Set<Integer> seen = new HashSet<>();
Set<Integer> duplicates = new HashSet<>();

for (int num : list) {
    if (!seen.add(num)) duplicates.add(num);
}

System.out.println(duplicates); // [1, 2]
```

---

### 8. Filter + Transform (Java 8 Streams)

```java
List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

List<Integer> result = list.stream()
        .filter(n -> n % 2 == 0)
        .map(n -> n * 2)
        .toList();

System.out.println(result); // [4, 8]
```

---

### 9. Remove Duplicates (Keep Order)

```java
List<Integer> list = Arrays.asList(1, 2, 2, 3, 1, 4);

List<Integer> result = new ArrayList<>();
Set<Integer> set = new HashSet<>();

for (int num : list) {
    if (set.add(num)) result.add(num);
}

System.out.println(result); // [1, 2, 3, 4]
```

---

## 🔥 ConcurrentModificationException (Deep Dive)

### What's Really Happening

The enhanced for loop is syntactic sugar for an `Iterator`. Internally it becomes:

```java
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    Integer num = it.next();
    if (num == 2) list.remove(num); // ← Problem
}
```

- `Iterator` tracks `expectedModCount`
- `ArrayList` tracks `modCount` (incremented on every structural change)
- When you call `list.remove()` directly, `modCount` updates but `expectedModCount` doesn't
- On next `it.next()` → `modCount != expectedModCount` → **throws exception**

> This is called **fail-fast behavior** — a design safety mechanism, not a concurrency issue. Also happens in `HashMap` and `HashSet`.

---

### Correct Fixes

```java
// ✅ Option 1: Iterator.remove() — syncs both counts
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    if (it.next() == 2) it.remove();
}

// ✅ Option 2: removeIf — modern and clean
list.removeIf(num -> num == 2);
```

### What NOT to Do

```java
// ❌ Index loop without adjustment — skips elements after removal
for (int i = 0; i < list.size(); i++) {
    if (list.get(i) == 2) list.remove(i); // elements shift left
}

// ✅ Fix: adjust index after removal
for (int i = 0; i < list.size(); i++) {
    if (list.get(i) == 2) { list.remove(i); i--; }
}

// ❌ forEach lambda — still uses iterator internally
list.forEach(num -> {
    if (num == 2) list.remove(num); // still throws
});
```

### Safe Edge Case — Reverse Index Loop

```java
for (int i = list.size() - 1; i >= 0; i--) {
    if (list.get(i) == 2) list.remove(i);
}
```

> Safe because removing from the end avoids shifting issues and no iterator is involved.

---

## 🔹 LinkedList

`LinkedList` is a **doubly linked list** implementation of both `List` and `Deque` interfaces.

### Internal Structure

```java
class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;
}
```

Each element = a node. No contiguous memory — each node connects forward and backward.

**Key Characteristics:**
- Allows duplicates ✅
- Maintains insertion order ✅
- Not synchronized ❌
- No fast index-based access ❌

### Time Complexity

| Operation | Complexity | Reality |
|---|---|---|
| `get(index)` | O(n) | Traversal required |
| `addFirst()` / `addLast()` | O(1) | Fast — direct pointer update |
| `add(index, element)` | O(n) | Traverse then insert |
| `remove` | O(n) | Traverse required |

> ⚠️ `addFirst()` / `addLast()` are O(1) only when **adding at the ends**. Insertion in the middle still requires O(n) traversal.

---

## 🔥 LinkedList — Practice Code

### 1. Basic Operations

```java
LinkedList<Integer> list = new LinkedList<>();

list.add(10);
list.add(20);
list.addFirst(5);
list.addLast(30);

System.out.println(list); // [5, 10, 20, 30]

list.removeFirst();
list.removeLast();

System.out.println(list); // [10, 20]
```

---

### 2. Reverse — Easy Way

```java
Collections.reverse(list);
```

### 3. Reverse — Manual

```java
LinkedList<Integer> result = new LinkedList<>();
for (int num : list) {
    result.addFirst(num);
}
System.out.println(result); // [4, 3, 2, 1]
```

---

### 4. Remove While Iterating (Safe)

```java
Iterator<Integer> it = list.iterator();
while (it.hasNext()) {
    if (it.next() == 2) it.remove();
}
```

---

### 5. Find Middle Element

```java
LinkedList<Integer> list = new LinkedList<>(Arrays.asList(1, 2, 3, 4, 5));

int slow = 0, fast = 0;
while (fast < list.size() - 1) {
    slow++;
    fast += 2;
}

System.out.println(list.get(slow)); // 3
```

> Real DS version uses node pointers (slow/fast pointer technique).

---

### 6. Merge Two Sorted Lists

```java
LinkedList<Integer> l1 = new LinkedList<>(Arrays.asList(1, 3, 5));
LinkedList<Integer> l2 = new LinkedList<>(Arrays.asList(2, 4, 6));

LinkedList<Integer> merged = new LinkedList<>();
merged.addAll(l1);
merged.addAll(l2);
Collections.sort(merged);

System.out.println(merged); // [1, 2, 3, 4, 5, 6]
```

---

### 7. As Queue

```java
Queue<Integer> q = new LinkedList<>();
q.offer(10);
q.offer(20);
System.out.println(q.poll()); // 10
System.out.println(q.peek()); // 20
```

---

### 8. As Stack

```java
Deque<Integer> stack = new LinkedList<>();
stack.push(1);
stack.push(2);
System.out.println(stack.pop()); // 2
```

---

### 9. Remove Duplicates

```java
LinkedList<Integer> list = new LinkedList<>(Arrays.asList(1, 2, 2, 3, 1));

Set<Integer> set = new HashSet<>();
Iterator<Integer> it = list.iterator();

while (it.hasNext()) {
    if (!set.add(it.next())) it.remove();
}

System.out.println(list); // [1, 2, 3]
```

---

### 10. Rotate List

```java
LinkedList<Integer> list = new LinkedList<>(Arrays.asList(1, 2, 3, 4, 5));
int k = 2;

for (int i = 0; i < k; i++) {
    list.addFirst(list.removeLast());
}

System.out.println(list); // [4, 5, 1, 2, 3]
```

---

## 🔹 ArrayList vs LinkedList — Truth, Not Theory

| Scenario | Winner |
|---|---|
| Random access (`get`) | ArrayList |
| Memory efficiency | ArrayList |
| Insert / delete at beginning | LinkedList |
| Iteration performance | ArrayList (CPU cache advantage) |
| Queue / Deque usage | LinkedList |

> **Brutal truth:** `LinkedList` is rarely used in production unless there is a very specific need. ArrayList's contiguous memory gives it a real-world performance advantage due to CPU cache locality.

### Interview Trap Questions

**Q: Why is LinkedList slower than expected?**
- No contiguous memory → poor CPU cache usage

**Q: Why no resizing needed?**
- Nodes are allocated dynamically

**Q: Why more memory?**
- Each node carries extra pointers (`prev`, `next`)

**Q: Why is insertion not always O(1)?**
- Only O(1) at the ends. Middle insertion still requires O(n) traversal to find the position.

---

## ⚠️ Common Mistakes

- Using `LinkedList` for search-heavy applications ❌
- Assuming `LinkedList` insertion is always O(1) ❌
- Modifying a list directly while iterating ❌
- Not setting initial capacity when size is known (ArrayList) ❌


Good — this is clean thinking. Now present it like someone who knows what they’re doing, not like scattered notes.

---

# ArrayList vs LinkedList (Structured Table)

| Aspect                   | ArrayList                                | LinkedList                                  |
| ------------------------ | ---------------------------------------- | ------------------------------------------- |
| **Underlying Structure** | Dynamic array (`Object[] elementData`)   | Doubly linked list (nodes with prev + next) |
| **Memory Layout**        | Contiguous memory                        | Non-contiguous (nodes scattered in heap)    |
| **Initial Capacity**     | Default = 10, grows ×1.5 when full       | No capacity concept                         |
| **Element Storage**      | Stored in array indexes `[0],[1],[2]...` | Each node stores data + pointers            |
| **Access (get index)**   | O(1)                                     | O(n)                                        |
| **Add at End**           | O(1) amortized                           | O(1)                                        |
| **Add at Beginning**     | O(n) (shifting required)                 | O(1)                                        |
| **Add in Middle**        | O(n) (shift elements)                    | O(n) (traverse)                             |
| **Remove at End**        | O(1)                                     | O(1)                                        |
| **Remove at Beginning**  | O(n)                                     | O(1)                                        |
| **Remove in Middle**     | O(n)                                     | O(n)                                        |
| **Memory Usage**         | Low                                      | High (extra 2 pointers per node)            |
| **Iteration Speed**      | Faster (cache-friendly)                  | Slower (cache misses)                       |
| **Resizing Cost**        | Yes (copy array)                         | No resizing needed                          |
| **Use as Queue/Deque**   | Not ideal                                | Ideal                                       |
| **CPU Cache Efficiency** | High                                     | Low                                         |
| **Real-world Usage**     | Default choice                           | Rare, specific cases                        |

---

# Rule of Thumb (Don’t mess this up)

* Use **ArrayList** by default
* Use **LinkedList** only when:

  * Frequent **add/remove at front**
  * Implementing **Queue / Deque**

---

