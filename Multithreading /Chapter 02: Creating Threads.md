
# Chapter 2: Creating Threads — 3 Ways + The Critical Trap

---

## 2.1 Way 1 — Extending the `Thread` Class

This is the oldest and most basic way to create a new thread in Java.
You create your own class that inherits (extends) from Java's built-in `Thread` class, and then you override the `run()` method.
Whatever code you put inside the `run()` method will run on the new separate thread.

```java
class MyThread extends Thread {

    private String taskName;

    public MyThread(String taskName) {
        this.taskName = taskName;
    }

    @Override
    public void run() {
        // This code runs ON THE NEW THREAD
        System.out.println("[" + Thread.currentThread().getName() + "] Executing: " + taskName);
    }
}

public class Way1_ExtendThread {
    public static void main(String[] args) {
        MyThread t1 = new MyThread("Send Welcome Email");
        MyThread t2 = new MyThread("Update User Index");

        t1.start(); // spawns a new thread, calls run() on it
        t2.start(); // spawns another new thread

        System.out.println("[" + Thread.currentThread().getName() + "] main() continues immediately");
    }
}
```

**Sample output** (order is non-deterministic — that's the point):
```
[main] main() continues immediately
[Thread-0] Executing: Send Welcome Email
[Thread-1] Executing: Update User Index
```
Key Point:
`run()` is a special method. When you call `start()`, Java automatically calls `run()` on the new thread.

Why `start()` and not `run()`?

- `t1.start()` → Tells Java: "Please create a new thread and run the `run()` method on it."
- `t1.run()` → Wrong! This would run the code on the same main thread (no new thread is created).

`start()` is the magic that actually spawns a real new thread

Important Lesson:

- The main thread does not wait for t1 and t2 to finish.
- As soon as you call   `start()`, the main thread continues immediately.
- The two new threads run at the same time (concurrently).
- The order of output is not guaranteed — this is normal in multithreading.

### The Problem With This Approach

Java has **single inheritance**. If your class extends `Thread`, it can't extend anything else. In Spring Boot, your service classes often need to extend or be managed beans — you can't also extend `Thread`.

Meaning: A class can extend only one class.
```Java

class MyThread extends Thread {     // Already extending Thread
    ...
}
```
Now you cannot do this:

```Java

class MyService extends SomeOtherClass {   // Not allowed!

```

Real-life problem in Spring Boot / Enterprise projects:

- Your classes are usually Service, Repository, or Component classes.
- These are managed by Spring.
- You often need to extend other classes or implement multiple interfaces.
- If your task class extends Thread, you lose the ability to extend anything else.

Example of what you might want:

```Java

class EmailService extends BaseService {   // You want this
    ...
}
```
But if `EmailService` already extends `Thread`, it's impossible.

---

## 2.2 Way 2 — Implementing `Runnable`

`Runnable` is a functional interface with a single method: `run()`. You implement it and pass it to a `Thread` object.

```java
@FunctionalInterface
public interface Runnable {
    void run();        // Just this one method
}
```

Your job is to:

- Create a class that implements Runnable
- Write your task logic inside the run() method
- Pass this object to a Thread constructor

```java
class EmailTask implements Runnable {

    private String recipient;

    public EmailTask(String recipient) {
        this.recipient = recipient;
    }

    @Override
    public void run() {
        System.out.println("[" + Thread.currentThread().getName() + "] Sending email to: " + recipient);
    }
}

public class Way2_Runnable {
    public static void main(String[] args) {

        Runnable task = new EmailTask("vidhan@shopsphere.com");

        // Thread wraps the Runnable
        Thread t1 = new Thread(task);
        t1.start();

        // You can also give the thread a name — very useful for debugging
        Thread t2 = new Thread(new EmailTask("admin@shopsphere.com"), "notification-thread");
        t2.start();

        System.out.println("[" + Thread.currentThread().getName() + "] main continues");
    }
}
```

Key Point:
- `EmailTask` is not a Thread. It is just a task that can be executed by a thread.

### Why This Is Better

Your task (`EmailTask`) and the execution mechanism (`Thread`) are **separated**. `EmailTask` is just a plain class that implements `Runnable` — it can still extend `BaseService`, be a Spring bean, whatever you want. This separation is the right design.


```Java

@Service
public class EmailService extends BaseService implements Runnable {   // Possible now!
    
    // You can still extend BaseService and implement Runnable
    ...
}
```

Important Concepts Explained

1. Separation of Concerns (Best Design Principle)
- EmailTask = What to do (the job)
- Thread     = How to execute it (the worker)
They are now separate. This is good software design.

2. Thread Naming
```Java
new Thread(runnable, "notification-thread")
```

Giving meaningful names helps a lot when debugging. You can easily see which thread is doing what in logs.

3. Runnable is a Functional Interface
Since Java 8, you can write it using lambda expression (much shorter):

```Java

Runnable task = () -> {
    System.out.println("Sending email...");
};

Thread t = new Thread(task);
t.start();
```

Order is still non-deterministic.
---

## 2.3 Way 3 — Lambda (Most Common in Modern Java)

- Since Java 8, `Runnable` is a Functional Interface (it has only one abstract method: `run()`).
- Because of this, instead of creating a separate class that implements `Runnable`, you can directly write the task using a lambda expression (`() -> { ... }`).
- This makes the code much shorter, cleaner, and more readable.
- This is the way most developers write threads in modern Java (especially in Spring Boot projects).

```java
public class Way3_Lambda {
    public static void main(String[] args) {

        // Anonymous Runnable via lambda
        Thread t1 = new Thread(() -> {
            System.out.println("[" + Thread.currentThread().getName() + "] Processing order #1001");
        });

        Thread t2 = new Thread(() -> {
            System.out.println("[" + Thread.currentThread().getName() + "] Processing order #1002");
        }, "order-processor");  // thread name as second arg

        t1.start();
        t2.start();
    }
}
```
What does () -> { ... } mean?

- () → No parameters (because run() method takes no arguments)
- -> → "goes to" or "does this"
- { ... } → The actual code that will run on the new thread

It's like saying:
- "Create a thread that runs this block of code"


### ShopSphere Example — What This Looks Like in Real Code

When your Notification Service needs to send an email without blocking the current request thread:

```java
// Inside NotificationService.java (simplified)
public void sendWelcomeEmailAsync(String userEmail, String userName) {

    Thread emailThread = new Thread(() -> {
        // This runs on a separate thread
        // main request thread is already free to return response
        emailClient.send(userEmail, "Welcome " + userName);
        System.out.println("Email sent to " + userEmail);
    }, "email-sender");

    emailThread.setDaemon(true); // we'll cover daemon threads shortly
    emailThread.start();

    // This line executes IMMEDIATELY — doesn't wait for email to send
    System.out.println("Email dispatched asynchronously");
}
```
Why is this useful in real applications?

- When a user registers, you don’t want to make them wait for the welcome email to be sent.
- You want to send the email in the background (asynchronously).
- The main request thread stays free and can immediately return a success response to the user.

This improves user experience and application performance.
---

## 2.4 The Critical Trap — `start()` vs `run()`

This is the **most common beginner mistake** and a very frequent interview question.

```java
public class StartVsRun {
    public static void main(String[] args) {

        Thread t = new Thread(() -> {
            System.out.println("Running on: " + Thread.currentThread().getName());
        });

        // ✅ CORRECT
        t.start();
        // Output: Running on: Thread-0
        // → JVM spawns a NEW thread. run() executes ON THAT NEW THREAD.

        // ❌ WRONG
        t.run();
        // Output: Running on: main
        // → No new thread created. run() executes ON THE CURRENT (main) THREAD.
        // → This is just a regular method call. No concurrency at all.
    }
}
```

Let's make the difference crystal clear:

```
t.start()                          t.run()
─────────────────────────────      ─────────────────────────────
main thread                        main thread
    │                                  │
    ├── tells JVM: "create a           ├── directly calls run()
    │   new thread"                    │   like any other method
    │                                  │
    │         New Thread               │   (no new thread ever created)
    │             │                    │
    │         run() executes           │   run() executes on main
    │         HERE                     │
    │                                  │
    ▼                                  ▼
main continues                     main continues only after
immediately                        run() returns (BLOCKING)
```

### The Rule
> **Always call `start()`.** Calling `run()` directly gives you zero multithreading — it's a synchronous call on the current thread.

---

## 2.5 Setting Thread Properties

Before calling `start()`, you can configure the thread:

```java
public class ThreadProperties {
    public static void main(String[] args) throws InterruptedException {

        Thread t = new Thread(() -> {
            System.out.println("Thread running: " + Thread.currentThread().getName());
            System.out.println("Priority: " + Thread.currentThread().getPriority());
            System.out.println("Is Daemon: " + Thread.currentThread().isDaemon());
        });

        // 1. Give it a meaningful name (shows in logs, jstack, VisualVM)
        t.setName("shopsphere-email-sender");

        // 2. Set priority (1=MIN, 5=NORM default, 10=MAX)
        //    OS may or may not honor this — just a hint
        t.setPriority(Thread.NORM_PRIORITY);

        // 3. Daemon vs User thread (covered below)
        t.setDaemon(true);

        t.start();
        t.join(); // wait for t to finish before main exits
    }
}
```

---

## 2.6 Daemon Threads vs User Threads

This is subtle but important.

| | User Thread | Daemon Thread |
|---|---|---|
| Example | Your main thread, request handler threads | GC thread, JVM internal threads |
| JVM exit | JVM waits for ALL user threads to finish | JVM does NOT wait — kills daemon threads |
| Use case | Core application logic | Background housekeeping |
| Default | `setDaemon(false)` | `setDaemon(true)` |

```java
public class DaemonDemo {
    public static void main(String[] args) throws InterruptedException {

        Thread backgroundCleaner = new Thread(() -> {
            while (true) {
                try {
                    System.out.println("Cleaning expired sessions...");
                    Thread.sleep(1000);
                } catch (InterruptedException e) {
                    break;
                }
            }
        }, "session-cleaner");

        // Mark as daemon — JVM won't wait for this when main exits
        backgroundCleaner.setDaemon(true);
        backgroundCleaner.start();

        // Main thread does its work
        System.out.println("Main thread doing work...");
        Thread.sleep(2500); // simulate work
        System.out.println("Main thread done. JVM will exit now.");

        // When main thread (a user thread) finishes,
        // JVM exits and kills session-cleaner daemon mid-execution
    }
}
```

**Output:**
```
Main thread doing work...
Cleaning expired sessions...
Cleaning expired sessions...
Main thread done. JVM will exit now.
```
The cleaner only ran twice (in 2.5 seconds), then JVM killed it when main exited.

### ShopSphere Angle
In ShopSphere, Spring's scheduled tasks (`@Scheduled`) and background workers (like your RabbitMQ listener threads) are daemon threads managed by the Spring container. When the application shuts down gracefully, Spring stops them cleanly — but the key point is the JVM doesn't wait for them forever.

---

## 2.7 The Three Ways — Side-by-Side Summary

```java
// Way 1: Extend Thread (avoid in production)
class T1 extends Thread {
    public void run() { /* task */ }
}
new T1().start();

// Way 2: Implement Runnable (good, explicit separation)
class T2 implements Runnable {
    public void run() { /* task */ }
}
new Thread(new T2()).start();

// Way 3: Lambda (preferred, modern Java)
new Thread(() -> { /* task */ }).start();
```

---

## 2.8 Complete Practice Program

Run this and study the output carefully — specifically the non-deterministic ordering:

```java
public class Chapter2_Complete {

    public static void main(String[] args) throws InterruptedException {

        System.out.println("=== ShopSphere Order Processing Simulation ===\n");

        // Simulating: 3 orders arrive simultaneously
        // Each needs to: validate → reserve inventory → send confirmation

        Runnable processOrder = (/* order id via closure below */) -> {};

        for (int orderId = 1; orderId <= 3; orderId++) {
            final int id = orderId; // must be effectively final for lambda

            Thread orderThread = new Thread(() -> {
                String threadName = Thread.currentThread().getName();
                System.out.println("[" + threadName + "] Validating order #" + id);
                
                try { Thread.sleep(100); } // simulate DB call
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                
                System.out.println("[" + threadName + "] Reserving inventory for order #" + id);
                
                try { Thread.sleep(150); }
                catch (InterruptedException e) { Thread.currentThread().interrupt(); }
                
                System.out.println("[" + threadName + "] Confirmation sent for order #" + id);

            }, "order-processor-" + id);

            orderThread.start();
        }

        System.out.println("[main] All order threads launched. Main continues.\n");
    }
}
```

**Observe:** The output order of validation/reservation/confirmation across the 3 orders will differ every run. That unpredictability is the core challenge multithreading introduces — and exactly what Chapter 4 (synchronization) solves.

---

## 2.9 Key Takeaways

| Concept | Remember This |
|---|---|
| Extend `Thread` | Works but wastes single inheritance — avoid |
| Implement `Runnable` | Separates task from execution — good |
| Lambda | Modern standard — use this |
| `start()` | Creates new thread, calls `run()` on it |
| `run()` | Just a method call — no new thread, no concurrency |
| User thread | JVM waits for it to finish before exiting |
| Daemon thread | JVM kills it when all user threads are done |
| Thread name | Always set it — saves you in production debugging |

---

## Interview Questions from This Chapter

**Q1. What are the ways to create a thread in Java?**
> Extending `Thread`, implementing `Runnable`, or using a lambda (since `Runnable` is a functional interface). A fourth option — `Callable` with `Future` — is covered later.

**Q2. What is the difference between `start()` and `run()`?**
> `start()` tells the JVM to create a new OS thread and execute `run()` on it. `run()` is just a regular method call on the current thread — no new thread is created, no concurrency happens.

**Q3. What is a daemon thread?**
> A background thread that the JVM terminates automatically when all user threads have finished. Used for housekeeping tasks. Set via `thread.setDaemon(true)` before calling `start()`.

**Q4. Why is `Runnable` preferred over extending `Thread`?**
> Because Java has single inheritance. Implementing `Runnable` keeps your class free to extend other classes, and it separates the *task* (what to do) from the *execution mechanism* (how/when to run it) — better design.

---

Ready for **Chapter 3: Thread Lifecycle** — the 6 states, `sleep()`, `join()`, `yield()`, and how to visualize exactly what a thread is doing at any moment?
