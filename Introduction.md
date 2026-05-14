# Java Fundamentals — Interview & Study Notes

---

## 1. Why Learn Java in Today's Era?

Java remains highly relevant because it is widely used for building scalable and reliable backend systems, especially in enterprise domains like banking and fintech. It has a strong ecosystem with frameworks like Spring Boot, supports multithreading, and performs well under high-load conditions. Its widespread industry adoption ensures strong job opportunities and long-term career stability.

---

## 2. What Are Programming Languages and Algorithms?

A **programming language** is a formal language used to write instructions for a computer (e.g., Java, Python). An **algorithm** is a step-by-step procedure to solve a problem, independent of any programming language. Programming languages focus on syntax and implementation; algorithms focus on logic and efficiency.

---

## 3. History of Java

Java was developed in 1991 by **James Gosling** at Sun Microsystems as part of the Green Project, initially targeting embedded systems and originally named **Oak**. In 1995, it was renamed Java and gained popularity with the rise of the internet due to its platform-independent nature via the JVM. Oracle Corporation acquired Sun Microsystems in 2010, after which Java continued to evolve into a dominant language for backend and enterprise development.

---

## 4. Anatomy of a Basic Java Class

```java
public class MyFirstApp {
    public static void main(String[] args) {
        System.out.print("I Rule!");
    }
}
```

A basic Java class consists of:

- **Class declaration** — defined using the `class` keyword with an access modifier like `public`.
- **`main` method** — the entry point of execution, declared as `public static void main(String[] args)`.
  - `public` — allows JVM access.
  - `static` — enables execution without creating an object.
  - `String[] args` — accepts command-line arguments.
- **Statements** — such as `System.out.print()`, each ending with a semicolon.

**Common Interview Mistakes:**
- Saying "main is mandatory" — it is only required for execution, not class definition.
- Not knowing why `static` is used.
- Thinking `System.out` is a method — it is a field of type `PrintStream`.

---

## 5. What Is Java and Its Key Features?

Java is a **statically typed, object-oriented** programming language that compiles source code into platform-independent **bytecode**, which runs on the JVM.

Key features include:

- Platform independence through the JVM
- Automatic memory management via garbage collection
- Strong multithreading support with a well-defined memory model
- Rich standard libraries
- Built-in security mechanisms
- High performance via JIT (Just-In-Time) compilation

---

## 6. Write Once, Run Anywhere

Java achieves this by compiling source code into **platform-independent bytecode**. Platform-specific JVM implementations then translate this bytecode into native machine code using interpretation and JIT compilation.

---

## 7. JVM, JRE, and JDK

| Component | Role |
|-----------|------|
| **JVM** | Executes Java bytecode; manages memory and garbage collection |
| **JRE** | Provides runtime environment; includes JVM and core libraries |
| **JDK** | Complete development kit; includes JRE plus compiler (`javac`) and debugger |

**Quick Checks:**
- Can we run Java without JDK? → **Yes** (using JRE)
- Can we compile without JDK? → **No**
- Is JVM platform independent? → **No**
- Is bytecode platform independent? → **Yes**

---

## 8. Why Is `main` Static? Can It Be Overloaded?

The `main` method is declared `static` so the JVM can invoke it **without creating an instance of the class**, avoiding the need for object instantiation at program startup.

Yes, `main` can be **overloaded** with different parameter types, but the JVM only executes the standard signature `public static void main(String[] args)`. All other overloaded versions must be called explicitly from within the program.

**Common Mistakes:**
- "main must always be static" — only required as the JVM entry point.
- "Overloaded main runs automatically" — it does not.

---

## 9. JVM Memory Areas

JVM memory is divided into **shared** areas (common to all threads) and **thread-specific** areas (each thread has its own).

| Memory Area | Scope | What It Stores | Key Points |
|-------------|-------|----------------|------------|
| **Heap** | Shared | Objects, instance variables | Managed by Garbage Collector; divided into Young & Old generations; `OutOfMemoryError` if full |
| **Stack** | Thread-specific | Stack frames, local variables, method parameters, return addresses | LIFO structure; not GC-managed; `StackOverflowError` on deep recursion |
| **Metaspace** (Java 8+) | Shared | Class metadata, method bytecode, static variables, runtime constant pool | Uses native memory (not heap); replaced the old Method Area |
| **PC Register** | Thread-specific | Address of current bytecode instruction | Required for multithreading; enables pause and resume |
| **Native Method Stack** | Thread-specific | Native method call data (C/C++ via JNI) | Used during JNI calls; separate from Java Stack |

---

## 10. Heap vs Stack Memory

| Property | Heap | Stack |
|----------|------|-------|
| Scope | Shared across all threads | Per thread |
| Stores | Objects, instance variables | Local variables, references, method calls |
| Management | Garbage Collector | Automatic (LIFO) |
| Speed | Slower (dynamic allocation + GC) | Faster (contiguous memory, simple push/pop) |
| Error | `OutOfMemoryError` | `StackOverflowError` |

**Example:**
```java
public void test() {
    int x = 10;       // x → Stack
    User u = new User(); // u (reference) → Stack; User object → Heap
}
```

---

## 11. Why Is Stack Faster? Stack Overflow? Objects in Stack?

Stack memory is faster because it uses a simple **LIFO structure with contiguous memory allocation**, allowing quick push and pop operations without complex memory management or garbage collection.

A **stack overflow** occurs when the call stack exceeds its limit, typically due to deep or infinite recursion, resulting in a `StackOverflowError`.

**Objects are not stored in the stack.** They are allocated on the heap; the stack stores only references to those objects and local primitive variables.

---

## 12. ClassLoader in Java

A **ClassLoader** is responsible for dynamically loading `.class` files into the JVM at runtime. It reads bytecode and converts it into `Class` objects. Java uses a **hierarchical ClassLoader system**:

- **Bootstrap ClassLoader** — loads core Java classes (e.g., `java.lang`)
- **Extension ClassLoader** — loads classes from the extensions directory
- **Application ClassLoader** — loads classes from the application's classpath

These follow a **parent-first delegation model** to ensure security and avoid duplicate class loading.

### Class Loading Process

**1. Loading** — Reads `.class` bytecode (from JAR, file system, network, etc.) and creates a `Class<?>` object in Metaspace.

**2. Linking** — Broken into three sub-steps:
- *Verification* — bytecode safety checks
- *Preparation* — memory allocation for static fields with default values
- *Resolution* — symbolic references resolved to direct references (lazy)

**3. Initialization** — Executes static blocks and assigns explicit static values.

### Custom ClassLoader

A custom ClassLoader is implemented by extending the `ClassLoader` class and overriding the `findClass()` method. The `loadClass()` method is generally not overridden to maintain the parent-first delegation model. Classes are only unloaded when their ClassLoader is garbage collected, so references must be carefully managed to avoid memory leaks. This is commonly used in **plugin-based architectures** for dynamic class loading.

---

## 13. Which Java Version to Use?

In production, use **Long-Term Support (LTS)** versions such as **Java 17** or **Java 21**. These provide stability, long-term security updates, and strong ecosystem support. Non-LTS versions have shorter support cycles and are not preferred for enterprise applications.

**Common Mistakes:**
- "I always use the latest Java" — careless; LTS is preferred in production.
- "Java 8 is best" — outdated thinking.
- Not mentioning LTS at all — incomplete answer.

---

## 14. Types of Variables in Java

| Type | Declared In | Storage | Default Value | Notes |
|------|-------------|---------|---------------|-------|
| **Local** | Inside method, constructor, or block | Stack | None (must initialize) | Temporary; exists only during method execution |
| **Instance** | Inside class, outside methods | Heap | Yes (0, null, false) | One copy per object |
| **Static** | Inside class with `static` keyword | Method Area / Metaspace | Yes | One shared copy per class across all objects |

**Example:**
```java
public class MemoryDemo {
    static int staticCount = 10;   // static variable
    int instanceCount = 5;         // instance variable

    public void calculate() {
        int localSum = 20;         // local variable
        System.out.println(localSum);
    }
}
```

### Default Values of Variables

Instance and static variables are automatically assigned default values by the JVM (e.g., `0` for numeric types, `false` for boolean, `null` for references). Local variables have **no default values** and must be explicitly initialized before use. This is enforced at compile time because local variables live on the stack, and Java prevents the use of undefined or garbage values.

---

## 15. Data Types and Literals

**Primitive types** store actual values directly (e.g., `int`, `float`, `boolean`). **Non-primitive types** store references to objects (e.g., arrays, class objects).

**Literals** are fixed constant values assigned to variables, such as integer literals (`10`), string literals (`"hello"`), and boolean literals (`true`, `false`).

---

## 16. Type Conversion and Casting

| Type | Direction | Manual? | Safe? |
|------|-----------|---------|-------|
| **Widening (Implicit)** | Small → Large | No | Yes |
| **Narrowing (Explicit)** | Large → Small | Yes | No (data loss possible) |
| **Upcasting** | Child → Parent | No (automatic) | Yes |
| **Downcasting** | Parent → Child | Yes | Risky (`ClassCastException` possible) |

**Widening order:** `byte → short → int → long → float → double`

**Widening example:**
```java
int a = 10;
double b = a; // automatic, no data loss
```

**Narrowing example:**
```java
double d = 10.5;
int x = (int) d; // data loss: x = 10
```

---

## 17. Type Promotion in Java Expressions

**Type promotion** is the automatic conversion of smaller data types into larger data types during expression evaluation. This prevents data loss and ensures consistent computation results when operands of different types appear in the same expression.

---

*End of Notes*
