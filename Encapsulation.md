
# 📘 Chapter 1 — Encapsulation

## 1.1 What Is Encapsulation? (The Real Definition)

Most people say *"encapsulation = private fields + getters/setters."*
That is **incomplete**. Here's the real definition:

> **Encapsulation** is the concept of wrapping data and methods together in a single unit (class), and restricting direct access to data by making variables `private` and providing controlled access through methods.

> *"Encapsulation is achieved by enforcing invariants, controlling state transitions through behavior methods, and protecting internal representation using defensive copying."*

---

## 1.2 The Problem It Solves

Imagine you have this class with no encapsulation:

```java
class BankAccount {
    public double balance;  // ❌ anyone can touch this
}
```

Now someone writes:

```java
BankAccount acc = new BankAccount();
acc.balance = -99999;  // ❌ Invalid state! Bank is broken.
```

Your object is now in an **invalid state** and you have zero control over it.

Encapsulation **prevents this entirely**.

---


## 1.3 How Java Implements Encapsulation

Java gives you **4 access modifiers** to control visibility:

```
┌──────────────────┬──────────┬───────────┬─────────────┬─────────────────┐
│ Modifier         │ Same     │ Same      │ Different   │ Different       │
│                  │ Class    │ Package   │ Package     │ Package         │
│                  │          │           │ (Subclass)  │ (Non-subclass)  │
├──────────────────┼──────────┼───────────┼─────────────┼─────────────────┤
│ private          │  ✅      │  ❌       │  ❌         │  ❌             │
│ default (none)   │  ✅      │  ✅       │  ❌         │  ❌             │
│ protected        │  ✅      │  ✅       │  ✅         │  ❌             │
│ public           │  ✅      │  ✅       │  ✅         │  ✅             │
└──────────────────┴──────────┴───────────┴─────────────┴─────────────────┘
```

For encapsulation, **fields = `private`**, methods = `public` (when needed).

---

Simple Definition (super easy version):
Encapsulation = Hide the important stuff inside a box, and only let people use special “buttons” to touch it.
The “box” is the class.
The “important stuff” = the data (like balance).
The “special buttons” = methods (like deposit and withdraw).

# Real-Life Analogy (the best one for this example):
Think of your BankAccount class as a real bank.

- The money (balance) is kept inside a strong vault in the bank basement.
- You (the customer) cannot go downstairs and touch the money directly.
- You can only talk to the bank teller (the methods).
- The teller checks every rule before doing anything:
  - → You want to put money? Teller checks “Is it positive amount?”
  - → You want to take money? Teller checks “Do you have enough?”
- There is no button that says “Just change the balance to whatever you want”. That would be dangerous!

This is exactly what the Java code is doing.


## 1.4 Proper Encapsulation — Step by Step

```java
public class BankAccount {

    // Step 1: Make fields private — no one touches them directly
    private String owner;
    private double balance;

    // Step 2: Constructor enforces initial valid state
    public BankAccount(String owner, double initialBalance) {
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Initial balance cannot be negative");
        }
        this.owner = owner;
        this.balance = initialBalance;
    }

    // Step 3: Controlled READ access (getter)
    public double getBalance() {
        return balance;
    }

    public String getOwner() {
        return owner;
    }

    // Step 4: Controlled WRITE — not a setter, but a meaningful operation
    public void deposit(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Deposit must be positive");
        }
        this.balance += amount;
    }

    public void withdraw(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        if (amount > balance) throw new IllegalStateException("Insufficient funds");
        this.balance -= amount;
    }

    // No setBalance() ← THIS IS INTENTIONAL
    // There's no valid business reason to just "set" a balance directly
}
```

Notice: there is **no `setBalance()`**. That's not an accident — that's encapsulation doing its job. The only way to change `balance` is through `deposit()` or `withdraw()`, which enforce rules.

---

## 1.5 Getters and Setters — The Common Trap

Beginners generate getters + setters for every field using their IDE and call it "encapsulation." This is **wrong**. Here's why:

```java
// ❌ This is NOT encapsulation — it's just a private field with a public backdoor
class Person {
    private int age;

    public int getAge() { return age; }

    public void setAge(int age) {
        this.age = age;  // No validation = no protection
    }
}

// Someone can still do:
person.setAge(-500);  // Still broken!
```

**The rule:** Only add a setter if there's a genuine need to change that field after construction — and always add validation inside it.

```java
// ✅ Setter with guard
public void setAge(int age) {
    if (age < 0 || age > 150) {
        throw new IllegalArgumentException("Invalid age: " + age);
    }
    this.age = age;
}
```

---

## 1.6 Deep Dive — Encapsulation with Mutable Objects (The Sneaky Bug)

This is where 90% of Java developers get caught in interviews.

```java
public class Student {
    private List<String> courses;

    public Student(List<String> courses) {
        this.courses = courses;  // ❌ Storing the reference directly!
    }

    public List<String> getCourses() {
        return courses;  // ❌ Returning the internal reference!
    }
}
```

Now watch what happens:

```java
List<String> myCourses = new ArrayList<>(List.of("Math", "Physics"));
Student s = new Student(myCourses);

// The caller still holds the reference!
myCourses.add("Hacking101");  // ← mutates s's internal list!

// And via getter:
s.getCourses().clear();  // ← destroys all courses!
```

Your `private` field is **not actually protected** because you're sharing mutable references.

**The fix — Defensive Copying:**

```java
public class Student {
    private List<String> courses;

    // Defensive copy IN (constructor)
    public Student(List<String> courses) {
        this.courses = new ArrayList<>(courses);  // ✅ own copy
    }

    // Defensive copy OUT (getter)
    public List<String> getCourses() {
        return Collections.unmodifiableList(courses);  // ✅ caller can't modify
    }
}
```

Now:
```java
List<String> myCourses = new ArrayList<>(List.of("Math", "Physics"));
Student s = new Student(myCourses);

myCourses.add("Hacking101");       // ✅ doesn't affect s
s.getCourses().add("Evil");        // ✅ throws UnsupportedOperationException
```

---

## 1.7 Encapsulation + Immutability

The **strongest form** of encapsulation is making objects **immutable** — they cannot change state at all after construction. Java's `String`, `Integer`, `LocalDate` are all immutable.

Recipe for an immutable class:

```java
public final class Money {           // 1. final class — no subclassing

    private final double amount;     // 2. all fields private + final
    private final String currency;

    public Money(double amount, String currency) {
        if (amount < 0) throw new IllegalArgumentException("Negative money");
        this.amount = amount;
        this.currency = currency;
    }

    public double getAmount() { return amount; }
    public String getCurrency() { return currency; }

    // 3. No setters at all

    // 4. Return NEW objects for "modifications"
    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Currency mismatch");
        return new Money(this.amount + other.amount, this.currency);
    }

    @Override
    public String toString() {
        return amount + " " + currency;
    }
}
```

```java
Money m1 = new Money(100, "INR");
Money m2 = new Money(50, "INR");
Money m3 = m1.add(m2);  // m1 and m2 unchanged — m3 is new

System.out.println(m1);  // 100.0 INR
System.out.println(m3);  // 150.0 INR
```

---

## 1.8 Under the Hood — How JVM Enforces Access Control

Access modifiers are **enforced at compile time** by `javac`. At the bytecode (`.class`) level, the JVM also checks access at **load/link time** using flags stored in the class file:

```
Each field/method in bytecode has an access_flags value:
  ACC_PUBLIC    = 0x0001
  ACC_PRIVATE   = 0x0002
  ACC_PROTECTED = 0x0004
```

When you write `acc.balance = -99` where `balance` is `private`, `javac` refuses to compile it. If somehow illegal bytecode was crafted to bypass this, the **JVM verifier** would throw a `IllegalAccessError` at runtime.

However — `Reflection` can break encapsulation:

```java
Field f = BankAccount.class.getDeclaredField("balance");
f.setAccessible(true);  // bypasses access control!
f.set(account, -99999);
```

This is why encapsulation is a **design-level protection**, not a cryptographic guarantee. The JVM module system (`module-info.java` in Java 9+) adds a stronger wall by allowing `--illegal-access` to be denied even for reflection.

---

## 1.9 Real-World Pattern — Builder + Encapsulation

When you have many fields, constructors get messy. The **Builder pattern** keeps encapsulation intact:

```java
public class Order {
    private final String orderId;
    private final String customerId;
    private final List<String> items;
    private final double total;

    private Order(Builder b) {   // private constructor!
        this.orderId = b.orderId;
        this.customerId = b.customerId;
        this.items = Collections.unmodifiableList(b.items);
        this.total = b.total;
    }

    public String getOrderId()    { return orderId; }
    public String getCustomerId() { return customerId; }
    public List<String> getItems(){ return items; }
    public double getTotal()      { return total; }

    public static class Builder {
        private String orderId;
        private String customerId;
        private List<String> items = new ArrayList<>();
        private double total;

        public Builder orderId(String id)      { this.orderId = id; return this; }
        public Builder customerId(String id)   { this.customerId = id; return this; }
        public Builder item(String item)       { this.items.add(item); return this; }
        public Builder total(double total)     { this.total = total; return this; }

        public Order build() {
            if (orderId == null) throw new IllegalStateException("orderId required");
            return new Order(this);
        }
    }
}

// Usage:
Order order = new Order.Builder()
    .orderId("ORD-001")
    .customerId("CUST-42")
    .item("Laptop")
    .item("Mouse")
    .total(75000)
    .build();
```

---

## 1.10 Encapsulation in Java Records (Java 14+)

Java Records are a **compact, immutable encapsulation** syntax:

```java
// This single line...
public record Point(int x, int y) {}

// ...is equivalent to a full class with:
//   private final int x, y
//   constructor with validation hooks
//   getters: x(), y()
//   equals(), hashCode(), toString()
```

You can add validation in a **compact constructor**:

```java
public record Money(double amount, String currency) {
    // Compact constructor — no parameter list needed
    public Money {
        if (amount < 0) throw new IllegalArgumentException("Negative amount");
        currency = currency.toUpperCase(); // can normalize here
    }
}
```

Records are essentially **immutable encapsulation baked into the language.**

---

## 🎯 Chapter 1 — Interview Questions

Here are the questions you'll face, with what the interviewer is actually testing:

**Q1. What is encapsulation? Is it just private + getter/setter?**
→ *Tests if you know the real definition.* Answer: No. It's about protecting invariants and controlling state change. Getters/setters without validation is a broken pattern.

**Q2. I have a `private List<String>` field and return it via getter. Is it encapsulated?**
→ *Tests knowledge of mutable object references.* Answer: No — you need defensive copy on the way out (`Collections.unmodifiableList`) and on the way in (constructor copy).

**Q3. Can Reflection break encapsulation?**
→ *Tests depth.* Answer: Yes, via `setAccessible(true)`. Java 9 modules can restrict this.

**Q4. How do you make a class fully immutable?**
→ `final` class, `private final` fields, no setters, defensive copies for mutable fields, return new objects on "mutation" operations.

**Q5. What's the difference between encapsulation and abstraction?**
→ *Classic confusion question.* Encapsulation = **hiding data** (HOW it's stored). Abstraction = **hiding implementation** (HOW it works). They work together but are different concepts.

**Q6. Why doesn't `String` have a `setCharAt()` method?**
→ Because `String` is immutable by design for thread-safety, security (classloaders use String), and caching (String Pool). This is encapsulation + immutability in action.

---

## ✅ Chapter 1 Summary

```
Encapsulation
├── Bundle data + behavior in one class
├── private fields — no direct access
├── Validate in constructors and setters
├── Avoid setters where not needed
├── Defensive copying for mutable fields
├── Immutability = strongest encapsulation
│   └── final class + final fields + no setters
├── Builder pattern for complex objects
├── Java Records = compact immutable encapsulation
└── Reflection can bypass — Java Modules restrict it
```

---

Ready for **Chapter 2 — Inheritance**? Or want to go deeper on any part of Encapsulation first? 🚀
