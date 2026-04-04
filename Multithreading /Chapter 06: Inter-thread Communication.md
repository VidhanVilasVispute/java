
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

This surprises most developers. These methods are on `Object` — meaning **every Java object** can be used as a communication point between threads.

```
wait()       → current thread releases the lock and sleeps
               until another thread calls notify() on the same object

notify()     → wakes up ONE thread waiting on this object
               (which one — unpredictable, up to JVM)

notifyAll()  → wakes up ALL threads waiting on this object
               they all compete for the lock — one wins, others go BLOCKED
```

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

**The scenario:**
- **Producer** generates items and puts them in a shared buffer
- **Consumer** takes items from the buffer and processes them
- Buffer has a maximum capacity

**The rules:**
- Producer must wait if buffer is **full**
- Consumer must wait if buffer is **empty**

### ShopSphere Context

Your **Order Service** (producer) places orders into a queue. Your **Notification Service** (consumer) picks them up and sends emails. The queue has a max size to prevent memory overflow during traffic spikes.

---

## 6.5 Implementation — Producer Consumer with `wait()`/`notifyAll()`

```java
import java.util.LinkedList;
import java.util.Queue;

public class OrderNotificationPipeline {

    // ─── Shared Buffer ────────────────────────────────────────────────
    static class OrderQueue {

        private final Queue<String> queue = new LinkedList<>();
        private final int MAX_SIZE;

        public OrderQueue(int maxSize) {
            this.MAX_SIZE = maxSize;
        }

        // Called by Order Service threads
        public synchronized void produce(String orderId) throws InterruptedException {

            // ✅ ALWAYS use while, never if — explained in 6.6
            while (queue.size() == MAX_SIZE) {
                System.out.println("[" + Thread.currentThread().getName()
                    + "] Queue FULL (" + MAX_SIZE + "). Waiting...");
                wait(); // release lock, sleep until notified
            }

            queue.offer(orderId);
            System.out.println("[" + Thread.currentThread().getName()
                + "] Produced: " + orderId
                + " | Queue size: " + queue.size());

            notifyAll(); // wake up all waiting consumers
        }

        // Called by Notification Service threads
        public synchronized String consume() throws InterruptedException {

            // ✅ ALWAYS use while, never if
            while (queue.isEmpty()) {
                System.out.println("[" + Thread.currentThread().getName()
                    + "] Queue EMPTY. Waiting...");
                wait(); // release lock, sleep until notified
            }

            String orderId = queue.poll();
            System.out.println("[" + Thread.currentThread().getName()
                + "] Consumed: " + orderId
                + " | Queue size: " + queue.size());

            notifyAll(); // wake up all waiting producers
            return orderId;
        }

        public synchronized int size() {
            return queue.size();
        }
    }

    // ─── Producer (Order Service) ─────────────────────────────────────
    static class OrderService implements Runnable {

        private final OrderQueue orderQueue;
        private final String[] orders;

        public OrderService(OrderQueue queue, String[] orders) {
            this.orderQueue = queue;
            this.orders = orders;
        }

        @Override
        public void run() {
            try {
                for (String orderId : orders) {
                    orderQueue.produce(orderId);
                    Thread.sleep(300); // simulate order creation delay
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }
    }

    // ─── Consumer (Notification Service) ─────────────────────────────
    static class NotificationService implements Runnable {

        private final OrderQueue orderQueue;
        private final int totalToConsume;

        public NotificationService(OrderQueue queue, int totalToConsume) {
            this.orderQueue = queue;
            this.totalToConsume = totalToConsume;
        }

        @Override
        public void run() {
            try {
                for (int i = 0; i < totalToConsume; i++) {
                    String orderId = orderQueue.consume();
                    sendEmail(orderId);
                    Thread.sleep(500); // simulate email sending
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }

        private void sendEmail(String orderId) {
            System.out.println("[" + Thread.currentThread().getName()
                + "] ✉ Email sent for: " + orderId);
        }
    }

    // ─── Main ─────────────────────────────────────────────────────────
    public static void main(String[] args) throws InterruptedException {

        OrderQueue sharedQueue = new OrderQueue(3); // max 3 orders in queue

        String[] incomingOrders = {
            "ORD-001", "ORD-002", "ORD-003",
            "ORD-004", "ORD-005", "ORD-006"
        };

        // 2 producers (two instances of Order Service)
        Thread producer1 = new Thread(
            new OrderService(sharedQueue, new String[]{"ORD-001", "ORD-002", "ORD-003"}),
            "order-service-1"
        );
        Thread producer2 = new Thread(
            new OrderService(sharedQueue, new String[]{"ORD-004", "ORD-005", "ORD-006"}),
            "order-service-2"
        );

        // 2 consumers (two Notification Service workers)
        Thread consumer1 = new Thread(
            new NotificationService(sharedQueue, 3),
            "notification-worker-1"
        );
        Thread consumer2 = new Thread(
            new NotificationService(sharedQueue, 3),
            "notification-worker-2"
        );

        producer1.start();
        producer2.start();
        consumer1.start();
        consumer2.start();

        producer1.join();
        producer2.join();
        consumer1.join();
        consumer2.join();

        System.out.println("\n✅ All orders processed and emails sent.");
    }
}
```

**Sample Output:**
```
[order-service-1] Produced: ORD-001 | Queue size: 1
[order-service-2] Produced: ORD-004 | Queue size: 2
[notification-worker-1] Consumed: ORD-001 | Queue size: 1
[notification-worker-1] ✉ Email sent for: ORD-001
[order-service-1] Produced: ORD-002 | Queue size: 2
[order-service-2] Produced: ORD-005 | Queue size: 3
[order-service-1] Queue FULL (3). Waiting...
[order-service-2] Queue FULL (3). Waiting...
[notification-worker-2] Consumed: ORD-004 | Queue size: 2
[notification-worker-2] ✉ Email sent for: ORD-004
[order-service-1] Produced: ORD-003 | Queue size: 3
...
✅ All orders processed and emails sent.
```

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
