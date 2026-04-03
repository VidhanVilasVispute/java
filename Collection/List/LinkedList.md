
# 📚 Java Collections — Chapter 2: `LinkedList`
### Full Stack: Theory → Internals → Code → Interview

---

## 🧠 1. THEORY — What is LinkedList?

`LinkedList` is a **doubly-linked list** implementation that implements both `List` and `Deque` interfaces. Unlike ArrayList's contiguous memory block, LinkedList stores elements in **nodes** scattered across the heap, each pointing to the next and previous node.

It lives in `java.util` and implements:
```
Iterable → Collection → List       → LinkedList
                      → Deque      → LinkedList
                      → Queue      → LinkedList
```

Key characteristics:
- **Ordered** — maintains insertion order
- **No index-based backing array** — traversal required for access
- **Allows duplicates**
- **Allows `null`**
- **NOT thread-safe**
- **Higher memory per element** than ArrayList (node overhead)

---

## ⚙️ 2. INTERNALS — What's happening under the hood?

### The Node — Heart of LinkedList

```java
// Inside java.util.LinkedList source code
private static class Node<E> {
    E item;       // the actual data
    Node<E> next; // pointer to next node
    Node<E> prev; // pointer to previous node

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

Every element you add creates a **new Node object** on the heap. There is no pre-allocated array.

---

### The Structure

```java
// Inside LinkedList
transient int size = 0;
transient Node<E> first;  // head pointer
transient Node<E> last;   // tail pointer
```

```
┌─────────────────────────────────────────────────────────────────┐
│                  DOUBLY LINKED LIST STRUCTURE                    │
│                                                                  │
│  first                                              last         │
│   ↓                                                  ↓           │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ prev │ null  │    │ prev │   ●───┼────│ prev │   ●   │       │
│  │ item │  "A"  │    │ item │  "B"  │    │ item │  "C"  │       │
│  │ next │   ●───┼───→│ next │   ●───┼───→│ next │  null │       │
│  └──────────────┘  ↗ └──────────────┘  ↗ └──────────────┘       │
│              ←────┘                ←────┘                        │
│                                                                  │
│  Each node lives at a RANDOM heap address (not contiguous)       │
└─────────────────────────────────────────────────────────────────┘
```

---

### `add(element)` — Adding to Tail

```java
// Simplified source of linkLast()
void linkLast(E e) {
    final Node<E> l = last;
    final Node<E> newNode = new Node<>(l, e, null); // prev=last, next=null
    last = newNode;
    if (l == null)
        first = newNode;  // list was empty
    else
        l.next = newNode; // previous tail points to new node
    size++;
    modCount++;
}
```

**O(1)** — because `last` pointer is always maintained. No traversal needed.

---

### `add(index, element)` — Inserting in Middle

```java
// Simplified source
void linkBefore(E e, Node<E> succ) {
    final Node<E> pred = succ.prev;
    final Node<E> newNode = new Node<>(pred, e, succ);
    succ.prev = newNode;
    if (pred == null)
        first = newNode;
    else
        pred.next = newNode;
    size++;
}
```

```
Before: [A] ↔ [C] ↔ [D]
Insert "B" at index 1

Step 1: Traverse to index 1 node (C)      → O(n)
Step 2: Create new Node("B")              → O(1)
Step 3: Rewire 4 pointers                 → O(1)

After:  [A] ↔ [B] ↔ [C] ↔ [D]
```

> The traversal to find the index is the expensive part — O(n).

---

### 🔍 Smart Traversal Optimization

LinkedList has a clever trick when you call `get(index)` or `node(index)`:

```java
// Actual source code
Node<E> node(int index) {
    if (index < (size >> 1)) {
        // index in first half → traverse from HEAD
        Node<E> x = first;
        for (int i = 0; i < index; i++)
            x = x.next;
        return x;
    } else {
        // index in second half → traverse from TAIL
        Node<E> x = last;
        for (int i = size - 1; i > index; i--)
            x = x.prev;
        return x;
    }
}
```

```
size = 10, get(index=7)

index 7 >= size/2 (5) → traverse from TAIL
Tail → 9 → 8 → 7  (only 3 hops instead of 7)
```

Still O(n) worst case, but **halved in practice**. This is why LinkedList is **doubly** linked.

---

### Memory Layout — The Real Cost

```
ArrayList holding 3 elements:
┌────────────────────────────┐
│ Object[]  [A] [B] [C] [_] │  ← 1 array object + elements
│ size = 3                   │
└────────────────────────────┘

LinkedList holding 3 elements:
┌──────────┐   ┌──────────┐   ┌──────────┐
│ Node {   │   │ Node {   │   │ Node {   │
│  item=A  │   │  item=B  │   │  item=C  │
│  next=●──┼──→│  next=●──┼──→│  next=null│
│  prev=null│ ←┼──●=prev  │ ←┼──●=prev  │
│ }        │   │ }        │   │ }        │
└──────────┘   └──────────┘   └──────────┘
+ LinkedList object (first, last, size)

Each Node = 32 bytes on 64-bit JVM (header + 3 refs + item ref)
```

**LinkedList uses ~3–4× more memory per element than ArrayList.**

---

## ⏱️ 3. TIME COMPLEXITY

| Operation | LinkedList | ArrayList | Winner |
|---|---|---|---|
| `get(index)` | O(n) | O(1) | ✅ ArrayList |
| `add(end)` | O(1) | O(1) amortized | 🤝 Tie |
| `add(front)` | O(1) | O(n) | ✅ LinkedList |
| `add(middle)` | O(n) traversal + O(1) insert | O(n) shift | 🤝 Tie (LL slightly better) |
| `remove(first)` | O(1) | O(n) | ✅ LinkedList |
| `remove(last)` | O(1) | O(1) | 🤝 Tie |
| `remove(middle)` | O(n) traversal + O(1) unlink | O(n) shift | 🤝 Tie |
| `contains()` | O(n) | O(n) | 🤝 Tie |
| Memory | High (node overhead) | Low (array) | ✅ ArrayList |
| Cache performance | Poor (scattered heap) | Excellent (contiguous) | ✅ ArrayList |

> ⚠️ **The biggest misconception in Java interviews:** People say *"use LinkedList when you insert/delete a lot."* That's only true if you **already have the node reference**. If you have to `get(index)` first, it's still O(n) traversal before the O(1) operation — same as ArrayList's O(n) shift.

---

## 💻 4. CODE — Real World, Production-Grade

### As a List

```java
import java.util.LinkedList;
import java.util.List;

List<String> list = new LinkedList<>();

list.add("A");           // adds to tail — O(1)
list.add(0, "Z");        // adds to head — O(1) (index 0 = first)
list.get(2);             // O(n) traversal — avoid in loops!
list.remove(0);          // removes head  — O(1)
list.remove("A");        // search + unlink — O(n)
```

---

### As a Deque (Double-Ended Queue) — The Real Power

```java
import java.util.LinkedList;
import java.util.Deque;

Deque<String> deque = new LinkedList<>();

// Add to both ends — O(1)
deque.addFirst("B");   // [B]
deque.addLast("C");    // [B, C]
deque.addFirst("A");   // [A, B, C]

// Peek without removing — O(1)
deque.peekFirst();     // "A"
deque.peekLast();      // "C"

// Remove from both ends — O(1)
deque.pollFirst();     // removes "A"
deque.pollLast();      // removes "C"
```

---

### As a Stack — O(1) push/pop

```java
Deque<String> stack = new LinkedList<>();

stack.push("A");  // addFirst → [A]
stack.push("B");  // addFirst → [B, A]
stack.push("C");  // addFirst → [C, B, A]

stack.pop();      // removeFirst → "C"
stack.peek();     // peekFirst  → "B"
```

> ✅ **Prefer `Deque` over legacy `Stack` class.** `Stack` extends `Vector` (synchronized, slow). `LinkedList` as `Deque` is the modern, unsynchronized equivalent.

---

### As a Queue — FIFO — O(1) enqueue/dequeue

```java
import java.util.Queue;

Queue<String> queue = new LinkedList<>();

queue.offer("Task1");   // enqueue → tail
queue.offer("Task2");
queue.offer("Task3");

queue.poll();           // dequeue → "Task1" from head
queue.peek();           // look at head → "Task2" (no remove)
```

---

### Safe Iteration

```java
LinkedList<String> list = new LinkedList<>(List.of("A", "B", "C"));

// ✅ For-each — uses ListIterator internally
for (String s : list) {
    System.out.println(s);
}

// ✅ ListIterator — can traverse both directions
ListIterator<String> it = list.listIterator(list.size()); // start from end
while (it.hasPrevious()) {
    System.out.println(it.previous()); // C, B, A
}

// ❌ NEVER do this — it's O(n²)!
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i)); // each get(i) is O(n) traversal!
}
```

> 🔥 **Index-based loop on LinkedList = O(n²). This is a production killer.**

---

### Real Use Case — LRU Cache skeleton

```java
// LinkedList maintains access order perfectly for LRU
Deque<Integer> lruOrder = new LinkedList<>();
Map<Integer, String> cache = new HashMap<>();

void access(int key) {
    lruOrder.remove(Integer.valueOf(key)); // O(n) — in real LRU use LinkedHashMap
    lruOrder.addFirst(key);               // move to front = most recent
}

int evict() {
    return lruOrder.pollLast(); // least recently used = tail
}
```

> In production LRU: use `LinkedHashMap` with `accessOrder=true`. But LinkedList's deque operations are the concept behind it.

---

## ⚔️ 5. ArrayList vs LinkedList — The Full Battle

```
┌─────────────────────────────────────────────────────────────┐
│              WHEN TO USE WHAT                               │
│                                                             │
│  Use ArrayList when:           Use LinkedList when:         │
│  ─────────────────────         ──────────────────────────   │
│  • Random access needed        • Used as Queue / Deque      │
│  • Mostly read operations      • Frequent head insertions   │
│  • Memory is a concern         • Frequent head removals     │
│  • Bulk data processing        • Implementing Stack/Queue   │
│  • Cache-friendly iteration    • Size changes unpredictably │
│                                  and middle inserts common  │
│                                                             │
│  In practice: ArrayList wins 90% of the time.              │
│  LinkedList shines only as a Deque.                         │
└─────────────────────────────────────────────────────────────┘
```

---

### The Cache Miss Problem — Why ArrayList beats LinkedList in Real Benchmarks

```
ArrayList in memory (contiguous):
[A][B][C][D][E] ← CPU loads entire cache line at once
 ↑ Cache line (64 bytes) fetched in 1 memory access

LinkedList in memory (scattered):
[A] at 0x100   [B] at 0x5F0   [C] at 0x3A8
 ↑ Each node access = potential cache MISS = 100x slower than cache hit
```

This is why even for frequent middle insertions, **ArrayList with its O(n) shift often beats LinkedList in real benchmarks** — because CPU cache efficiency matters more than O-notation at typical sizes (<10k elements).

---

## 🎯 6. INTERVIEW TRAPS & EDGE CASES

### Trap 1 — `poll()` vs `remove()` on empty list

```java
Queue<String> q = new LinkedList<>();

q.remove(); // ❌ throws NoSuchElementException on empty queue
q.poll();   // ✅ returns null on empty queue — ALWAYS prefer poll()

q.element(); // ❌ throws NoSuchElementException if empty
q.peek();    // ✅ returns null — ALWAYS prefer peek()
```

---

### Trap 2 — `push()` vs `offer()` direction

```java
Deque<String> d = new LinkedList<>();
d.push("A");   // addFirst → inserts at HEAD (stack behavior)
d.offer("B");  // addLast  → inserts at TAIL (queue behavior)
// Result: [A, B]

d.pop();       // removeFirst → "A"
d.poll();      // removeFirst → "B"
```

---

### Trap 3 — `remove(Object)` needs proper `equals()`

```java
LinkedList<Integer> list = new LinkedList<>(List.of(1, 2, 3));
list.remove(Integer.valueOf(2)); // ✅ removes object 2

// With custom objects — equals() MUST be overridden
// or remove(Object) will never find the element
```

---

### Trap 4 — LinkedList implements both List AND Deque

```java
// Both are valid — different capabilities exposed
List<String>  l = new LinkedList<>();  // List view  — no deque methods
Deque<String> d = new LinkedList<>();  // Deque view — no index methods
LinkedList<String> ll = new LinkedList<>(); // all methods — but avoid coupling to impl
```

> Always code to the **narrowest interface** that fulfils your need. If you need a queue, use `Queue<>`. If you need a list, use `List<>`.

---

## 🔥 7. INTERVIEW QUESTIONS

**Theory Level:**
1. What data structure does LinkedList use internally?
2. Why does LinkedList implement both `List` and `Deque`?
3. How many pointers does each node in Java's LinkedList have?
4. Why is `get(index)` O(n) in LinkedList?

**Internals Level:**
5. Explain the smart traversal optimization in `node(index)`. What condition triggers forward vs backward traversal?
6. Why is LinkedList's memory footprint higher than ArrayList's?
7. Why does accessing LinkedList elements cause cache misses?
8. Why are `first` and `last` fields marked `transient`?

**Code/Trap Level:**
9. What is the difference between `poll()` and `remove()` on an empty LinkedList?
10. Why is index-based for loop on LinkedList O(n²)? How do you fix it?
11. What's the difference between `push()`/`pop()` and `offer()`/`poll()` on a LinkedList?
12. How does `ListIterator` allow bidirectional traversal?

**Design Level:**
13. *"LinkedList is better than ArrayList for frequent insertions"* — agree or disagree? Explain with cache behaviour.
14. When would you use `LinkedList` as a `Deque` vs `ArrayDeque`?
15. How would you implement an LRU cache using LinkedList?
16. Why is the legacy `Stack` class discouraged, and what should you use instead?

---

> 💡 **Answer to Q14:** `ArrayDeque` is almost always preferred over `LinkedList` as a Deque — it has no node overhead, is cache-friendly, and is faster in practice. **LinkedList as Deque is only preferred when you need null elements** (ArrayDeque doesn't allow null) or when you need `List` + `Deque` simultaneously from the same object.

---

**Ready for Chapter 3?** We can go to `HashMap` — the most internally complex and interview-heavy collection in Java. 🗺️🔥
