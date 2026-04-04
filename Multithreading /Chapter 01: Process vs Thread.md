
# Chapter 1: Process vs Thread — The Foundation

---

## 1.1 What is a Process?

A **process** is a program in execution. When you run your ShopSphere `user-service` JAR, the OS creates a process for it. Each process gets its own isolated memory space — no two processes share memory by default.

Think of a process as a **restaurant**. It has its own kitchen, its own staff, its own address. Another restaurant can't just walk into your kitchen.

```
Your OS right now:
├── Process: IntelliJ IDEA        (its own memory)
├── Process: user-service.jar     (its own memory)
├── Process: product-service.jar  (its own memory)
└── Process: Chrome               (its own memory)
```

Each of these is completely isolated. If `user-service` crashes, `product-service` doesn't care.

---

## 1.2 What is a Thread?

A **thread** is a unit of execution *inside* a process. One process can have many threads. All threads inside the same process **share the same heap memory** but each gets its **own stack**.

Extending the restaurant analogy — threads are the **waiters** inside that restaurant. They all work in the same kitchen (shared heap), but each waiter carries their own notepad (their own stack).

```
Process: user-service.jar
├── Thread 1: main thread          (own stack)
├── Thread 2: handling request /login    (own stack)
├── Thread 3: handling request /profile  (own stack)
└── Thread 4: GC thread            (own stack)
         └── All share the same HEAP (objects, static variables)
```

This is exactly how Spring Boot handles HTTP requests — Tomcat gives each incoming request its own thread.

---

## 1.3 JVM Memory Model — The Most Important Diagram

```
┌─────────────────────────────────────────────────┐
│                   JVM Memory                    │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │              HEAP (Shared)               │   │
│  │  - All objects (new User(), new Order()) │   │
│  │  - Static variables                      │   │
│  │  - String pool                           │   │
│  │  ← ALL THREADS READ/WRITE HERE →         │   │
│  └──────────────────────────────────────────┘   │
│                                                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐      │
│  │ Thread 1 │  │ Thread 2 │  │ Thread 3 │      │
│  │  Stack   │  │  Stack   │  │  Stack   │      │
│  │          │  │          │  │          │      │
│  │ - local  │  │ - local  │  │ - local  │      │
│  │   vars   │  │   vars   │  │   vars   │      │
│  │ - method │  │ - method │  │ - method │      │
│  │   calls  │  │   calls  │  │   calls  │      │
│  └──────────┘  └──────────┘  └──────────┘      │
│                                                 │
│  ┌──────────────────────────────────────────┐   │
│  │         Method Area (Shared)             │   │
│  │  - Class definitions, bytecode           │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Key rules to burn into memory:**
- Local variables → **stack** → thread-private → automatically safe
- Objects (anything created with `new`) → **heap** → shared → needs careful handling
- Primitives inside objects → **heap** → shared → needs careful handling

---

## 1.4 Why Multithreading?

Two distinct reasons — don't confuse them:

### Reason 1: Responsiveness (don't block the user)
If `user-service` handled one request at a time, request #2 would wait for request #1 to finish. With threads, request #1 and request #2 run simultaneously.

### Reason 2: Performance (use all CPU cores)
Your server has 8 cores. A single-threaded program uses 1 core — 7 cores sit idle. Multithreading lets you utilize all cores.

---

## 1.5 Concurrency vs Parallelism — The Interview Favourite

These are NOT the same thing.

```
CONCURRENCY (1 core, 2 threads):
Core 1: [Thread1]--[Thread2]--[Thread1]--[Thread2]
         ^ rapid switching (context switching)
         Looks simultaneous, isn't truly simultaneous
         
PARALLELISM (2 cores, 2 threads):
Core 1: [Thread1]────────────────────────────────
Core 2: [Thread2]────────────────────────────────
         ^ truly simultaneous execution
```

**Concurrency** is about *dealing with* multiple things at once (structure).
**Parallelism** is about *doing* multiple things at once (execution).

You can have concurrency without parallelism (single-core context switching), but parallelism always implies concurrency.

In your ShopSphere Gateway — when 100 requests hit simultaneously, Tomcat is doing concurrency. If your machine has 4 cores, some of those threads may truly run in parallel.

---

## 1.6 ShopSphere Context

Right now in your ShopSphere, multithreading is already happening without you explicitly writing any:

```
Incoming HTTP request to user-service
         │
         ▼
   Tomcat Thread Pool
   (default: 200 threads)
         │
    picks a free thread
         │
         ▼
  UserController.login()  ← runs on that thread
         │
         ▼
  UserService.authenticate()
         │
         ▼
  UserRepository.findByEmail()  ← JDBC call blocks this thread
         │
         ▼
  response returned → thread goes back to pool
```

Every `@RestController` method you wrote runs on a Tomcat-managed thread. The thread's stack holds your local variables. The `UserRepository` bean lives on the heap — shared across all threads.

This is why, when you'll later learn race conditions, it will matter — two threads can simultaneously touch the same object on the heap.

---

## 1.7 Code — Observing Threads in Action

```java
public class Chapter1_ProcessVsThread {

    public static void main(String[] args) {

        // The main method itself runs on a thread — the "main" thread
        System.out.println("Current thread: " + Thread.currentThread().getName());
        // Output: Current thread: main

        // Get the current process ID (Java 9+)
        long pid = ProcessHandle.current().pid();
        System.out.println("Process ID: " + pid);

        // See all active threads in this JVM process right now
        System.out.println("\n--- All active threads in this JVM ---");
        Thread.getAllStackTraces()
              .keySet()
              .forEach(t -> System.out.println(
                  "Thread: " + t.getName() +
                  " | Daemon: " + t.isDaemon() +
                  " | State: " + t.getState()
              ));
    }
}
```

**Sample output:**
```
Current thread: main
Process ID: 18472

--- All active threads in this JVM ---
Thread: main           | Daemon: false | State: RUNNABLE
Thread: GC Thread#0    | Daemon: true  | State: RUNNABLE
Thread: G1 Main Marker | Daemon: true  | State: RUNNABLE
Thread: Signal Catcher | Daemon: true  | State: WAITING
Thread: Finalizer      | Daemon: true  | State: WAITING
Thread: Reference Handler | Daemon: true | State: WAITING
```

Notice — you wrote zero thread code, yet your JVM already has 6+ threads running. The GC runs on its own threads, the JVM signal handler runs on its own thread. Multithreading is the default reality of the JVM.

---

## 1.8 Key Takeaways

| Concept | Remember This |
|---|---|
| Process | Isolated program, own memory, OS-managed |
| Thread | Unit of execution inside a process, shares heap |
| Heap | Shared across all threads — source of all thread bugs |
| Stack | Per-thread — always safe, holds local vars |
| Concurrency | Structuring work for multiple tasks |
| Parallelism | Actually executing tasks simultaneously |
| Spring Boot | Already multithreaded — each request gets a Tomcat thread |

---

## Interview Questions from This Chapter

**Q1. What is the difference between a process and a thread?**
> A process is an independent program with its own memory space. A thread is a lightweight unit of execution within a process. Threads share the heap but have their own stack.

**Q2. What is shared between threads and what is not?**
> Shared: heap (objects, static variables), method area (class definitions). Not shared: stack (local variables, method call frames), program counter register.

**Q3. What is the difference between concurrency and parallelism?**
> Concurrency is the ability to handle multiple tasks by interleaving them (even on a single core). Parallelism is actually executing multiple tasks simultaneously on multiple cores. Parallelism is a subset of concurrency.

---

Ready for **Chapter 2: Creating Threads** — `Thread`, `Runnable`, lambdas, and the critical `start()` vs `run()` distinction?
