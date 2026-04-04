
# Chapter 3: Thread Lifecycle — 6 States, sleep(), join(), yield()

---

## 3.1 The 6 Thread States

In Java, a thread is always in exactly one of these states, defined in the `Thread.State` enum:

```
                    ┌─────────────────────────────────────────────────────────┐
                    │                  THREAD LIFECYCLE                       │
                    └─────────────────────────────────────────────────────────┘

                              new Thread(() -> {...})
                                        │
                                        ▼
                                    ┌───────┐
                                    │  NEW  │
                                    └───┬───┘
                                        │  thread.start()
                                        ▼
                                  ┌──────────┐
                              ┌──►│RUNNABLE  │◄──────────────────┐
                              │   └──┬───────┘                   │
                              │      │  CPU scheduler picks it   │
                              │      ▼                           │
                              │  [running on CPU]                │
                              │      │                           │
                    lock       │      ├──── sleep(ms) ──────►┌───────────────┐
                  released     │      │                       │ TIMED_WAITING │
                              │      │◄─── time expires ─────└───────────────┘
                              │      │
                              │      ├──── wait() ─────────►┌─────────┐
                              │      │                       │ WAITING  │
                              │      │◄─── notify() ─────────└─────────┘
                              │      │
                              │      ├──── needs lock ──────►┌─────────┐
                              └──────┤      (unavailable)    │ BLOCKED  │
                                     │◄─── lock available ───└─────────┘
                                     │
                                     │  run() returns
                                     ▼
                               ┌────────────┐
                               │ TERMINATED │
                               └────────────┘
```

---

## 3.2 Each State Explained

### State 1: NEW
Thread object is created but `start()` hasn't been called yet.

```java
Thread t = new Thread(() -> System.out.println("hello"));
System.out.println(t.getState()); // NEW
```

Thread exists in memory. No OS thread created yet.

---

### State 2: RUNNABLE
After `start()` is called. The thread is either:
- **actually running** on a CPU core right now, OR
- **ready to run**, waiting for the CPU scheduler to pick it

Java combines both into one state called RUNNABLE because from Java's perspective, you can't tell the difference — the OS scheduler decides.

```java
t.start();
System.out.println(t.getState()); // RUNNABLE
```

---

### State 3: BLOCKED
Thread tried to enter a `synchronized` block/method but another thread holds that lock. It sits here doing nothing, waiting for the lock to be released.

```java
// Thread A holds the lock on `this`
// Thread B tries to enter — goes BLOCKED until A releases
synchronized(this) {
    // Thread A is in here
}
```

You'll see this more in Chapter 4. For now: **BLOCKED = waiting for a lock.**

---

### State 4: WAITING
Thread is waiting **indefinitely** for another thread to explicitly wake it up. It called `wait()`, `join()` with no timeout, or `LockSupport.park()`.

No timeout. It will wait forever until notified.

```java
synchronized(lock) {
    lock.wait(); // WAITING — someone must call lock.notify()
}

t.join(); // calling thread goes WAITING until t finishes
```

---

### State 5: TIMED_WAITING
Same as WAITING but with a timeout. Thread wakes up either when notified OR when time expires — whichever comes first.

```java
Thread.sleep(2000);       // TIMED_WAITING for 2 seconds
t.join(1000);             // TIMED_WAITING — wait for t, max 1 second
lock.wait(500);           // TIMED_WAITING for 500ms
```

---

### State 6: TERMINATED
`run()` has completed — either normally or via an uncaught exception. Thread is dead. You cannot restart it.

```java
// After t's run() finishes:
System.out.println(t.getState()); // TERMINATED

t.start(); // ❌ throws IllegalThreadStateException — can't restart
```

---

## 3.3 Observing All States in One Program

```java
public class ThreadStateObserver {

    public static void main(String[] args) throws InterruptedException {

        Object lock = new Object();

        Thread t = new Thread(() -> {
            try {
                // Will enter TIMED_WAITING
                Thread.sleep(1000);

                // Will enter WAITING
                synchronized (lock) {
                    lock.wait();
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }, "observed-thread");

        // State 1: NEW
        System.out.println("After new:     " + t.getState()); // NEW

        t.start();

        // State 2: RUNNABLE
        System.out.println("After start:   " + t.getState()); // RUNNABLE

        // Let it enter sleep
        Thread.sleep(100);

        // State 5: TIMED_WAITING (inside Thread.sleep)
        System.out.println("During sleep:  " + t.getState()); // TIMED_WAITING

        // Wait for sleep to finish, then it hits wait()
        Thread.sleep(1100);

        // State 4: WAITING (inside lock.wait())
        System.out.println("During wait:   " + t.getState()); // WAITING

        // Wake it up
        synchronized (lock) {
            lock.notify();
        }

        // Let it finish
        t.join();

        // State 6: TERMINATED
        System.out.println("After finish:  " + t.getState()); // TERMINATED
    }
}
```

**Output:**
```
After new:     NEW
After start:   RUNNABLE
During sleep:  TIMED_WAITING
During wait:   WAITING
After finish:  TERMINATED
```

---

## 3.4 `Thread.sleep(ms)` — Pause Execution

Puts the **current thread** into TIMED_WAITING for the given milliseconds. The thread releases the CPU but **does NOT release any locks it holds**.

```java
public class SleepDemo {

    public static void main(String[] args) throws InterruptedException {

        Thread t = new Thread(() -> {
            System.out.println("[" + Thread.currentThread().getName() + "] Starting task");

            try {
                // Simulate: calling external payment gateway
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                // IMPORTANT: always handle this properly
                System.out.println("Thread was interrupted during sleep!");
                Thread.currentThread().interrupt(); // restore interrupt flag
                return;
            }

            System.out.println("[" + Thread.currentThread().getName() + "] Task complete");
        }, "payment-thread");

        t.start();

        // Main thread is NOT blocked — it continues
        System.out.println("[main] Payment dispatched, doing other work...");
        Thread.sleep(500);
        System.out.println("[main] Still working...");
    }
}
```

### Critical Rules for `sleep()`

```
✅  Thread.sleep(1000)        — correct, sleeps current thread
❌  someOtherThread.sleep()  — misleading! still sleeps CURRENT thread
                                (sleep is static — avoid calling on instances)

✅  Always catch InterruptedException
✅  Always call Thread.currentThread().interrupt() in the catch block
    (restores the interrupt flag so callers can detect it)
```

### `sleep()` does NOT release locks

```java
synchronized (lock) {
    Thread.sleep(5000); // ← holds the lock for 5 full seconds
                        // other threads BLOCKED on this lock for 5s
}
```

This is a common performance bug — don't sleep inside synchronized blocks.

---

## 3.5 `join()` — Wait for Another Thread to Finish

`join()` makes the **calling thread** wait until the target thread finishes. The calling thread goes into WAITING state.

```java
public class JoinDemo {

    public static void main(String[] args) throws InterruptedException {

        // ShopSphere scenario:
        // Before sending order confirmation email,
        // we must wait for inventory to be reserved

        Thread inventoryThread = new Thread(() -> {
            System.out.println("[inventory] Reserving stock...");
            try { Thread.sleep(2000); } // simulate DB operation
            catch (InterruptedException e) { Thread.currentThread().interrupt(); }
            System.out.println("[inventory] Stock reserved.");
        }, "inventory-reserver");

        Thread emailThread = new Thread(() -> {
            System.out.println("[email] Waiting for inventory before sending email...");
            try {
                inventoryThread.join(); // email thread WAITS here
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
            System.out.println("[email] Inventory done! Sending confirmation email.");
        }, "email-sender");

        inventoryThread.start();
        emailThread.start();

        // Main waits for both
        inventoryThread.join();
        emailThread.join();

        System.out.println("[main] Order processing complete.");
    }
}
```

**Output (always in this order due to join):**
```
[inventory] Reserving stock...
[email] Waiting for inventory before sending email...
[inventory] Stock reserved.
[email] Inventory done! Sending confirmation email.
[main] Order processing complete.
```

### `join()` with timeout

```java
t.join(3000); // wait for t, but at most 3 seconds
              // after 3s, calling thread continues regardless
              // check t.isAlive() to know if it actually finished

if (t.isAlive()) {
    System.out.println("Thread didn't finish in time!");
}
```

---

## 3.6 `yield()` — Give Up CPU (Politely)

`Thread.yield()` is a **hint** to the scheduler: "I'm willing to pause and let other threads of the same priority run." The scheduler may or may not honor it.

```java
public class YieldDemo {

    public static void main(String[] args) {

        Runnable task = () -> {
            for (int i = 0; i < 5; i++) {
                System.out.println(Thread.currentThread().getName() + " → " + i);
                Thread.yield(); // hint: let others run
            }
        };

        new Thread(task, "thread-A").start();
        new Thread(task, "thread-B").start();
    }
}
```

### Honest Assessment of `yield()`

```
Without yield():   A→0, A→1, A→2, A→3, A→4, B→0, B→1 ... (A might hog CPU)
With yield():      A→0, B→0, A→1, B→1 ... (more interleaved, but not guaranteed)
```

`yield()` is rarely used in production code. You'll almost never need it. It exists for very specific performance tuning scenarios. Know it for interviews, but don't reach for it in real code.

---

## 3.7 `sleep()` vs `wait()` vs `join()` — The Interview Table

This comparison comes up in almost every Java interview:

| | `sleep()` | `wait()` | `join()` |
|---|---|---|---|
| **Class** | `Thread` (static) | `Object` | `Thread` (instance) |
| **Lock released?** | ❌ No | ✅ Yes | N/A |
| **Where called** | Anywhere | Inside `synchronized` block | Anywhere |
| **Who wakes it?** | Time expires | `notify()` / `notifyAll()` | Target thread finishes |
| **State** | TIMED_WAITING | WAITING / TIMED_WAITING | WAITING / TIMED_WAITING |
| **Purpose** | Pause execution | Inter-thread communication | Wait for thread completion |

---

## 3.8 `interrupt()` — Requesting a Thread to Stop

You **cannot forcibly kill** a thread in Java (deprecated `stop()` was removed for good reason). Instead you **request** interruption, and the thread decides how to handle it.

```java
public class InterruptDemo {

    public static void main(String[] args) throws InterruptedException {

        Thread worker = new Thread(() -> {
            System.out.println("[worker] Started. Processing orders...");

            while (!Thread.currentThread().isInterrupted()) {
                // Do work
                System.out.println("[worker] Processing...");

                try {
                    Thread.sleep(500);
                } catch (InterruptedException e) {
                    // sleep() clears the interrupt flag when it throws
                    // so we must restore it
                    System.out.println("[worker] Sleep interrupted. Shutting down.");
                    Thread.currentThread().interrupt(); // restore flag
                    break; // exit the loop cleanly
                }
            }

            System.out.println("[worker] Cleanup done. Exiting.");
        }, "order-worker");

        worker.start();

        // Let it work for 2 seconds
        Thread.sleep(2000);

        // Request it to stop
        System.out.println("[main] Requesting worker to stop...");
        worker.interrupt();

        worker.join();
        System.out.println("[main] Worker stopped cleanly.");
    }
}
```

**Output:**
```
[worker] Started. Processing orders...
[worker] Processing...
[worker] Processing...
[worker] Processing...
[worker] Processing...
[main] Requesting worker to stop...
[worker] Sleep interrupted. Shutting down.
[worker] Cleanup done. Exiting.
[main] Worker stopped cleanly.
```

### The Interrupt Contract
```
interrupt()           → sets the interrupt flag on the target thread
isInterrupted()       → checks flag WITHOUT clearing it
Thread.interrupted()  → checks flag AND clears it (static, checks current thread)

If thread is sleeping/waiting when interrupted:
  → InterruptedException is thrown
  → interrupt flag is CLEARED automatically
  → you MUST restore it with Thread.currentThread().interrupt()
```

---

## 3.9 Complete ShopSphere Simulation

Putting it all together — a realistic order lifecycle:

```java
public class OrderLifecycleSimulation {

    public static void main(String[] args) throws InterruptedException {

        System.out.println("=== ShopSphere Order Lifecycle ===\n");

        // Step 1: Validate payment (must complete first)
        Thread paymentValidator = new Thread(() -> {
            System.out.println("[payment] Calling payment gateway...");
            try { Thread.sleep(1500); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
            System.out.println("[payment] Payment validated ✓");
        }, "payment-validator");

        // Step 2: Reserve inventory (must complete before shipping)
        Thread inventoryReserver = new Thread(() -> {
            System.out.println("[inventory] Checking stock...");
            try { Thread.sleep(1000); }
            catch (InterruptedException e) { Thread.currentThread().interrupt(); return; }
            System.out.println("[inventory] Inventory reserved ✓");
        }, "inventory-reserver");

        // Step 3: Notify customer (only after payment + inventory done)
        Thread notifier = new Thread(() -> {
            try {
                System.out.println("[notifier] Waiting for payment & inventory...");
                paymentValidator.join();   // wait for payment
                inventoryReserver.join();  // wait for inventory
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return;
            }
            System.out.println("[notifier] Sending confirmation email ✓");
        }, "customer-notifier");

        // Start payment and inventory in parallel
        paymentValidator.start();
        inventoryReserver.start();
        notifier.start();

        // Main thread observes state
        Thread.sleep(200);
        System.out.println("\n[main] Thread states:");
        System.out.println("  payment-validator: " + paymentValidator.getState());
        System.out.println("  inventory-reserver: " + inventoryReserver.getState());
        System.out.println("  customer-notifier: " + notifier.getState());
        System.out.println();

        // Wait for everything
        notifier.join();
        System.out.println("\n[main] Order fully processed ✓");
    }
}
```

**Output:**
```
=== ShopSphere Order Lifecycle ===

[payment] Calling payment gateway...
[inventory] Checking stock...
[notifier] Waiting for payment & inventory...

[main] Thread states:
  payment-validator:  TIMED_WAITING
  inventory-reserver: TIMED_WAITING
  customer-notifier:  WAITING

[inventory] Inventory reserved ✓
[payment] Payment validated ✓
[notifier] Sending confirmation email ✓

[main] Order fully processed ✓
```

Notice the notifier is in WAITING (because of `join()`) while payment and inventory are in TIMED_WAITING (because of `sleep()`). This is the lifecycle in action.

---

## 3.10 Key Takeaways

| Concept | Remember This |
|---|---|
| 6 states | NEW → RUNNABLE → BLOCKED / WAITING / TIMED_WAITING → TERMINATED |
| BLOCKED | Waiting for a lock |
| WAITING | Waiting for explicit notify/join — no timeout |
| TIMED_WAITING | Same but with timeout |
| `sleep()` | Pauses current thread, holds locks, always catch InterruptedException |
| `join()` | Calling thread waits for target thread to finish |
| `yield()` | Hint to scheduler — rarely used in production |
| `interrupt()` | Request, not command — thread must cooperate |
| Interrupt flag | Cleared by InterruptedException — always restore it |

---

## Interview Questions from This Chapter

**Q1. What are the states of a thread in Java?**
> NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED — defined in the `Thread.State` enum.

**Q2. What is the difference between WAITING and TIMED_WAITING?**
> WAITING has no timeout — thread waits indefinitely until explicitly notified (via `notify()`) or joined. TIMED_WAITING has a maximum wait duration after which the thread returns to RUNNABLE automatically.

**Q3. Does `sleep()` release the lock?**
> No. A sleeping thread holds all its locks. This is a key difference from `wait()`, which releases the lock while waiting.

**Q4. How do you stop a thread in Java?**
> You request it to stop via `interrupt()`. You cannot forcibly kill a thread. The thread must check `isInterrupted()` or handle `InterruptedException` and exit cooperatively.

**Q5. What is the difference between `sleep()` and `wait()`?**
> `sleep()` is a static method on `Thread`, doesn't release locks, and can be called anywhere. `wait()` is on `Object`, releases the lock, and must be called inside a `synchronized` block. Both move the thread to a waiting state but serve different purposes.

---

Ready for **Chapter 4: Race Conditions & the `synchronized` keyword** — where we see exactly how shared heap memory causes bugs, and how synchronization fixes them?
