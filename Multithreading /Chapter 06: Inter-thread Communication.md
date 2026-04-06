
# Chapter 6: Inter-thread Communication — `wait()`, `notify()`, `notifyAll()`

---

## 6.1 The Problem — Busy Waiting

Before `wait()`/`notify()`, developers used **busy waiting** to coordinate threads:

```java
// ❌ Busy waiting — thread keeps checking in a loop
while (queue.isEmpty()) {
    // do nothing — just spin
    // wastes 100% CPU on one core doing zero useful work
}
processItem(queue.poll());
```

This is wasteful. The consumer thread burns CPU doing nothing useful. You need a way for a thread to say: **"I'll sleep until something is ready — wake me up when it is."**

That's exactly what `wait()` and `notify()` provide.

---

## 6.2 The Three Methods — On `Object`, Not `Thread`

`wait()`, `notify()`, and `notifyAll()` are **NOT** methods of the `Thread` class.

They are methods of the **Object** class.

That means **every Java object** can be used as a meeting point for threads to talk to each other.

Example:
- A simple `Object lock = new Object();` can be used for communication.
- Even `this`, a String, or any custom object can be used.

| Method          | What it does (in very easy words)                                                                 | Who calls it          |
|-----------------|----------------------------------------------------------------------------------------------------|-----------------------|
| **wait()**      | "I am going to sleep and release the lock. Wake me up when someone notifies me."                  | The waiting thread    |
| **notify()**    | "Wake up **one** thread that is waiting on this object." (JVM chooses which one)                   | The notifying thread  |
| **notifyAll()** | "Wake up **all** threads that are waiting on this object."                                        | The notifying thread  |

### The Iron Rule
> **All three must be called inside a `synchronized` block on the same object.**
> Calling them outside `synchronized` throws `IllegalMonitorStateException`.

```java
synchronized (lock) {
    lock.wait();      // ✅ inside synchronized on lock
    lock.notify();    // ✅ inside synchronized on lock
    lock.notifyAll(); // ✅ inside synchronized on lock
}

lock.wait();          // ❌ IllegalMonitorStateException
```
### Simple Real-Life Analogy

Imagine a **restaurant kitchen** with only **one waiter** (the lock):

- Customers (threads) want food.
- The cook (one thread) is preparing food.
- When the cook is busy, customers wait.

How they communicate:

- Customer says: `"I will wait()"` → sits down and releases the table (releases lock).
- Cook finishes food and says: `"notify()"` → wakes up **one** waiting customer.
- Or cook says: `"notifyAll()"` → wakes up **all** waiting customers.

But nobody can shout "wait" or "notify" while standing outside the restaurant.  
They can only do it **while holding the table** (inside synchronized block).
---

## 6.3 What `wait()` Actually Does — Step by Step

```
Thread calls lock.wait() inside synchronized(lock):

Step 1: Thread RELEASES the lock on `lock`
        (unlike sleep() which holds the lock)

Step 2: Thread enters WAITING state
        (sits in lock's wait set — doing nothing, using no CPU)

Step 3: Another thread acquires the lock
        (because step 1 released it)

Step 4: That other thread calls lock.notify()
        (waking up the waiting thread)

Step 5: Waiting thread moves to BLOCKED
        (it wants the lock back but must wait for notifier to release it)

Step 6: Notifier exits synchronized block → releases lock

Step 7: Waiting thread acquires lock → continues after wait()
```

```
Thread A                              Thread B
synchronized(lock) {                 synchronized(lock) {
    lock.wait()                          // doing work
        │                                lock.notify()
        │ releases lock ──────────────►      │
        │                                    │ releases lock
        ◄────────────────────────────── woken up
        │ re-acquires lock
        │ continues
}                                    }
```

---

## 6.4 The Producer-Consumer Problem

This is the canonical use case for `wait()`/`notify()`. It appears in almost every Java interview.

### What is the Producer-Consumer Problem?
    It's a classic situation where two threads need to work together using a shared buffer (like a box or queue).

- Producer → Makes things (puts items into the buffer)
- Consumer → Takes things (removes items from the buffer)

Rules:

- If the buffer is full, Producer must wait.
- If the buffer is empty, Consumer must wait.

This is exactly where `wait()` and `notify()` are used.


### Simple Real-Life Analogy

Imagine a **small dining table** (the buffer) that can hold only **5 plates**.

- **Producer** = Chef (keeps cooking and putting plates on the table)
- **Consumer** = Waiter (keeps taking plates from the table to serve customers)

What happens?
- If the table is full (5 plates), Chef should **stop cooking** and wait.
- If the table is empty (0 plates), Waiter should **stop** and wait.

They need to **communicate** with each other:
- Chef says: "Table is full, you take some!"
- Waiter says: "Table is empty, please cook more!"

This communication is done using `wait()` and `notify()`.

---

### ShopSphere Context

Your **Order Service** (producer) places orders into a queue. Your **Notification Service** (consumer) picks them up and sends emails. The queue has a max size to prevent memory overflow during traffic spikes.

---

## 6.5 Implementation — Producer Consumer with `wait()`/`notifyAll()`


### Simple Code Example (Very Easy to Understand)

```java
public class ProducerConsumerExample {

    private final List<String> buffer = new ArrayList<>();
    private final int MAX_SIZE = 5;           // Buffer can hold only 5 items
    private final Object lock = new Object(); // Communication object

    // Producer Thread
    class Producer implements Runnable {
        public void run() {
            for (int i = 1; i <= 10; i++) {
                synchronized (lock) {
                    while (buffer.size() == MAX_SIZE) {        // Buffer full?
                        try {
                            System.out.println("Producer: Buffer full, waiting...");
                            lock.wait();                       // Release lock and sleep
                        } catch (InterruptedException e) {}
                    }

                    buffer.add("Order-" + i);
                    System.out.println("Producer: Added Order-" + i + " | Buffer size: " + buffer.size());
                    
                    lock.notify();   // Wake up Consumer
                }
            }
        }
    }

    // Consumer Thread
    class Consumer implements Runnable {
        public void run() {
            for (int i = 1; i <= 10; i++) {
                synchronized (lock) {
                    while (buffer.isEmpty()) {                 // Buffer empty?
                        try {
                            System.out.println("Consumer: Buffer empty, waiting...");
                            lock.wait();                       // Release lock and sleep
                        } catch (InterruptedException e) {}
                    }

                    String order = buffer.remove(0);
                    System.out.println("Consumer: Processed " + order);
                    
                    lock.notify();   // Wake up Producer
                }
            }
        }
    }

    public static void main(String[] args) {
        ProducerConsumerExample example = new ProducerConsumerExample();
        
        Thread producerThread = new Thread(example.new Producer(), "Producer");
        Thread consumerThread = new Thread(example.new Consumer(), "Consumer");

        producerThread.start();
        consumerThread.start();
    }
}
```

---

### How It Works (Super Simple Steps)

1. Producer wants to add an item.
2. It checks: Is buffer full?  
   → If yes → `wait()` (goes to sleep and releases lock)
3. Consumer wants to take an item.
4. It checks: Is buffer empty?  
   → If yes → `wait()` (goes to sleep)
5. When Producer adds an item → calls `notify()` to wake Consumer.
6. When Consumer removes an item → calls `notify()` to wake Producer.

Both threads keep talking using the same `lock` object.

---

### Key Points to Remember

- Always use **`while`** loop with `wait()` (not `if`).  
  Reason: After waking up, the condition might still not be true.
- `wait()` always releases the lock so the other thread can work.
- `notify()` wakes only **one** waiting thread.
- Both Producer and Consumer use the **same lock object** for communication.

---

## 6.6 `while` vs `if` — The Spurious Wakeup Problem

This is a critical subtlety that separates junior from mid-level developers.

### What is a Spurious Wakeup?

The JVM/OS is allowed to wake up a waiting thread **without anyone calling `notify()`**. This is rare but permitted by the Java spec and does happen on some platforms.

If you use `if`:

```java
// ❌ WRONG — if
synchronized (lock) {
    if (queue.isEmpty()) {  // checked once
        wait();             // woken up spuriously — queue still empty!
    }
    String item = queue.poll(); // ← NullPointerException or wrong behavior
}
```

A spurious wakeup skips the check entirely and proceeds even though the condition isn't met.

If you use `while`:

```java
// ✅ CORRECT — while
synchronized (lock) {
    while (queue.isEmpty()) { // rechecked every time thread wakes up
        wait();               // woken up spuriously? loop rechecks, waits again
    }
    String item = queue.poll(); // ← safe, queue is guaranteed non-empty
}
```

### The Pattern — Always

```java
synchronized (lock) {
    while (!conditionMet()) {  // ← ALWAYS while, NEVER if
        lock.wait();
    }
    // condition is guaranteed to be met here
    doWork();
    lock.notifyAll();
}
```

---

## 6.7 `notify()` vs `notifyAll()` — When to Use Which

```
notify()     → wakes ONE arbitrary waiting thread
               ⚠ Dangerous: what if you wake the wrong thread?
               Example: wake a producer when you need a consumer

notifyAll()  → wakes ALL waiting threads
               They all compete for the lock — only one wins
               Others go back to waiting (while loop rechecks condition)
               ✅ Safer — always use this unless you have a specific reason
```

### When `notify()` Causes a Bug

```
Queue is EMPTY. Two consumers waiting, one producer waiting.

Someone calls notify() → wakes a CONSUMER (random choice)
Consumer sees queue.isEmpty() → goes back to wait()
Producer never woken up → system stalls forever

Deadlock-like situation from a single bad notify()
```

### Rule
> **Default to `notifyAll()`.** Use `notify()` only when you are certain all waiting threads are identical and interchangeable — and even then, be very careful.

---

## 6.8 Missed Notification — Another Subtle Bug

```java
// ❌ Classic bug — notification before wait()

// Thread 1 (consumer):
synchronized (lock) {
    // ... not yet waiting ...
}

// Thread 2 (producer) — runs first:
synchronized (lock) {
    queue.offer(item);
    lock.notify(); // ← fires BEFORE consumer is waiting
                   //   notification is LOST
}

// Thread 1 (consumer) — runs after:
synchronized (lock) {
    while (queue.isEmpty()) {
        lock.wait(); // ← waits forever — notification already fired
    }
}
```

The fix is always the same — check the condition with `while`. If the queue already has items when consumer enters, the `while` loop won't even call `wait()`:

```java
// ✅ Fixed — while loop handles this
synchronized (lock) {
    while (queue.isEmpty()) { // if items already there, skip wait entirely
        lock.wait();
    }
    process(queue.poll());
}
```

---

## 6.9 `wait()` with Timeout

```java
synchronized (lock) {
    long timeout = 5000; // 5 seconds max
    long deadline = System.currentTimeMillis() + timeout;

    while (queue.isEmpty()) {
        long remaining = deadline - System.currentTimeMillis();
        if (remaining <= 0) {
            throw new TimeoutException("Queue empty after 5 seconds");
        }
        lock.wait(remaining); // wait at most `remaining` ms
    }
    return queue.poll();
}
```

Unlike plain `wait()`, `wait(ms)` moves the thread to TIMED_WAITING and automatically wakes up after the timeout — even if nobody calls `notify()`.

---

## 6.10 Full State Diagram During Producer-Consumer

```
PRODUCER when queue is FULL:

  synchronized(queue) ──► queue.size() == MAX? ──YES──► queue.wait()
                                │                            │
                               NO                    releases lock
                                │                            │
                          queue.offer()              WAITING state
                                │                            │
                          notifyAll()          consumer calls notifyAll()
                                │                            │
                          release lock                 re-acquires lock
                                                            │
                                                    recheck while loop
                                                            │
                                                    queue not full → proceed


CONSUMER when queue is EMPTY:

  synchronized(queue) ──► queue.isEmpty()? ──YES──► queue.wait()
                                │                        │
                               NO                 releases lock
                                │                        │
                          queue.poll()           WAITING state
                                │                        │
                          notifyAll()      producer calls notifyAll()
                                │                        │
                          release lock            re-acquires lock
                                                        │
                                                recheck while loop
                                                        │
                                                queue not empty → proceed
```

---

## 6.11 ShopSphere — Real World Equivalent

In your actual ShopSphere, you use RabbitMQ instead of a hand-rolled queue — but the concept is identical:

```
Hand-rolled Producer-Consumer          RabbitMQ equivalent
──────────────────────────────         ──────────────────────────
OrderQueue (shared buffer)        ←→   RabbitMQ Queue
queue.offer(orderId)              ←→   rabbitTemplate.convertAndSend()
queue.poll()                      ←→   @RabbitListener
wait() when full                  ←→   Publisher confirms + backpressure
wait() when empty                 ←→   Consumer idle (no messages)
notifyAll()                       ←→   RabbitMQ push notification to consumer
MAX_SIZE                          ←→   Queue max-length policy
```

RabbitMQ handles all the synchronization internally — but when you're asked in interviews how message queues work under the hood, this is the answer: `wait()`/`notifyAll()` on a shared buffer with capacity limits.

---

## 6.12 Common Mistakes — Summary

```
❌  Using if instead of while → spurious wakeup bug
❌  Calling wait()/notify() outside synchronized → IllegalMonitorStateException
❌  Using notify() when threads are not interchangeable → missed wakeup / stall
❌  Forgetting to call notifyAll() after state change → waiting threads never wake
❌  Holding lock during expensive work → others starved while you do slow I/O
❌  Different lock objects for wait() and notify() → notifications never received
```

```java
// ❌ This is broken — different lock objects
synchronized (lockA) { lockA.wait(); }    // Thread 1
synchronized (lockB) { lockB.notify(); }  // Thread 2 — notifies the WRONG lock
```

---

## 6.13 Key Takeaways

| Concept | Remember This |
|---|---|
| `wait()` | Releases lock + sleeps. Must be in `synchronized`. Goes WAITING. |
| `notify()` | Wakes ONE arbitrary waiter. Risky. |
| `notifyAll()` | Wakes ALL waiters. Safe default. |
| `while` not `if` | Spurious wakeups — always recheck condition after waking |
| Same lock | `wait()` and `notify()` must use the same object as the `synchronized` block |
| `wait()` vs `sleep()` | `wait()` releases lock. `sleep()` holds lock. |
| Producer-Consumer | Canonical use case — producer waits when full, consumer waits when empty |
| RabbitMQ | Production equivalent — same concept, battle-tested implementation |

---

## Interview Questions from This Chapter

**Q1. Why are `wait()`, `notify()`, `notifyAll()` on `Object` and not on `Thread`?**
> Because they operate on a lock, and every Java object can be a lock (monitor). The thread that calls `wait()` releases the lock on that specific object and waits. Since any object can be a lock, the methods belong on `Object` — not on `Thread`.

**Q2. Why must `wait()` always be inside a `while` loop, not an `if`?**
> Two reasons: (1) Spurious wakeups — the JVM can wake a thread without `notify()` being called. The `while` loop rechecks the condition and waits again if not met. (2) Multiple consumers — another consumer may have consumed the item before this thread reacquires the lock. `while` catches this and waits again.

**Q3. What is the difference between `notify()` and `notifyAll()`?**
> `notify()` wakes one arbitrary waiting thread — if the wrong thread is woken (e.g., a producer when a consumer is needed), the system can stall. `notifyAll()` wakes all waiting threads — they compete for the lock, the winner checks its condition, others re-enter the wait set if not met. `notifyAll()` is safer and should be the default.

**Q4. What happens to the lock when `wait()` is called?**
> The lock is released immediately and atomically. This is the key difference from `sleep()` which holds all locks while sleeping. After being notified, the thread must reacquire the lock before proceeding — it goes BLOCKED until the lock is available.

**Q5. What is a missed notification and how do you prevent it?**
> A missed notification happens when `notify()` is called before the other thread has started `wait()` — the signal is lost and the waiting thread sleeps forever. Prevention: always check the condition with a `while` loop. If the condition is already satisfied when the thread enters, it skips `wait()` entirely and proceeds.

---

Ready for **Chapter 7: The Executor Framework** — `ExecutorService`, thread pools, `submit()` vs `execute()`, and why you should almost never call `new Thread()` directly in production code?
