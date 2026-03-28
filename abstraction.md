# Abstraction in Java — Interview Notes

Abstraction is the process of defining a **contract** that exposes only essential behavior while hiding implementation details. It is achieved using **abstract classes** and **interfaces**, and it helps in building flexible, loosely coupled, and maintainable systems.

---

## 🔹 Example Walkthrough

### Interface (Contract)

```java
interface PaymentService {
    void pay();
}
```

### Implementation

```java
class UpiPaymentService implements PaymentService {
    public void pay() {
        // UPI payment logic
    }
}
```

### Controller (Client)

```java
class PaymentController {
    private PaymentService paymentService;

    public PaymentController(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    public void makePayment() {
        paymentService.pay();
    }
}
```

---

## 🔹 Reality Breakdown

| Layer | Role |
|---|---|
| **Controller (Client)** | Only knows about `pay()` — doesn't care if it's UPI, Card, or NetBanking |
| **Service (Implementation)** | Contains actual business logic; handles how payment is processed |

---

## 🔥 What is a Contract?

> **Contract = a set of rules or method signatures that an implementation must follow**

- In Java, **Interface = Contract**
- It says: *"हे methods असायलाच हवेत"* (these methods must exist)

### Clean Interview Line

> *"A contract in Java is an interface that defines expected behavior through method signatures, without providing implementation."*

### Summary

- **Controller** → uses the contract
- **Service** → implements the contract
- Controller is **unaware** of the internal implementation

---

## 🔹 Why Abstraction Exists (REAL Reason)

> ❌ Stop saying *"code reusability"* — give these real reasons instead:

### 1. Decoupling

```java
Payment p = new UpiPayment();
// Can switch to:
Payment p = new CreditCardPayment();
// No code break.
```

### 2. Flexibility (Open/Closed Principle)

- Add a new class **without modifying** existing logic.

### 3. Maintainability

- Change internal logic → external code is **unaffected**.

### 4. Testability

- You can **mock interfaces** easily in unit tests.

---

## 🔹 Abstract Class

An abstract class **cannot be instantiated** and may contain both abstract and concrete methods. Used to define a common base with shared behavior for related subclasses.

```java
abstract class Shape {
    abstract double area();       // abstract method

    void display() {              // concrete method
        System.out.println("Calculating area");
    }
}

class Circle extends Shape {
    double radius;

    Circle(double radius) {
        this.radius = radius;
    }

    @Override
    double area() {
        return Math.PI * radius * radius;
    }
}
```

### Usage

```java
Shape s = new Circle(5);
s.display();
System.out.println(s.area());
```

This single example demonstrates:
- Abstraction
- Inheritance
- Polymorphism
- Code reuse

---

## 🔹 Interface

An interface defines a **behavioral contract** that implementing classes must follow. Since Java 8, it can also include `default` and `static` methods.

### What an Interface Can Contain

#### 1. Abstract Methods (default)

```java
interface Payment {
    void pay(double amount); // public & abstract implicitly
}
```

- Methods are `public` by default
- You **cannot reduce visibility** when implementing

#### 2. Default Methods (Java 8+)

```java
default void log() {
    System.out.println("Logging");
}
```

- Exist for **backward compatibility**
- Allows interface evolution without breaking existing implementations

#### 3. Static Methods (Java 8+)

```java
static void validate() {
    System.out.println("Validating");
}
```

- Belong to the interface, **cannot be overridden**
- Called using `InterfaceName.method()`

#### 4. Constants (Fields Only)

```java
int MAX_LIMIT = 100; // implicitly public static final
```

- No instance variables allowed

---

## 🔹 Abstract Class vs Interface — Interview Weapon

| Feature | Abstract Class | Interface |
|---|---|---|
| Purpose | Base class | Contract |
| Methods | Abstract + Concrete | Abstract + Default + Static |
| Multiple Inheritance | ❌ No | ✅ Yes |
| Fields | Instance variables allowed | Constants only (`public static final`) |
| Constructor | ✅ Yes | ❌ No |
| Use Case | Code reuse | Decoupling |

### Core Rule

- **Interface** → *"CAN DO"* (capability / contract)
- **Abstract Class** → *"IS A"* (base class with shared logic)

---

## 🔹 When to Use What

### Use Interface when:
- You need multiple implementations (UPI, Card, NetBanking)
- You want loose coupling
- You want to support multiple inheritance
- You're defining a contract only

```java
interface PaymentService {
    void pay();
}
// UpiPayment, CardPayment — controller doesn't change
```

### Use Abstract Class when:
- You have shared logic
- You want default behavior + override option
- You need state (fields, constructors)
- Classes are closely related

```java
abstract class Payment {
    void validate() {
        System.out.println("Common validation");
    }
    abstract void pay();
}

class UpiPayment extends Payment {
    void pay() {
        System.out.println("UPI payment");
    }
}
// Reuses validate() from base class
```

### Real-World Mapping

- **Interface** → Spring `@Service` contract (e.g., `PaymentService`)
- **Abstract class** → Base service with shared logic (logging, validation)

---

## 🔹 Design Scenarios

### Q: Payment System — UPI, Credit Card, NetBanking

```java
interface Payment {
    void pay(double amount);
}

class UpiPayment implements Payment { ... }
class CardPayment implements Payment { ... }
```

👉 Use interface because: multiple payment methods, no shared logic initially, easy to extend.

#### Follow-up: What if all payments need logging?

Evolve the design:

```java
abstract class Payment {
    void log() {
        System.out.println("Logging...");
    }
    abstract void pay(double amount);
}
```

> 💡 Real thinking: **start with interface → move to abstract class when shared logic appears**

---

### Q: Notification System — Email, SMS, Push

```java
interface Notification {
    void send(String message);
}

class EmailNotification implements Notification { ... }
class SmsNotification implements Notification { ... }
```

> Tomorrow WhatsApp notification added? → No change in existing classes → **Open/Closed Principle** ✅

---

### Q: Why use Abstraction in Spring Boot?

```java
@Autowired
PaymentService paymentService; // you don't know the implementation
```

- Enables **loose coupling**
- Supports **dependency injection**
- Allows **switching implementations** without code change

---

## 🔥 Trap Questions (Interviewer Tests Depth)

### Can an interface have implementation?

> Yes (Java 8+) — via `default` and `static` methods.

---

### Why not always use abstract class instead of interface?

- Java doesn't support **multiple inheritance** with classes
- Interface allows **multiple inheritance**
- Interface is better for **pure contracts**

---

### What happens if abstraction is overused?

- Too many layers → **hard to debug**
- Overengineering
- Unnecessary complexity

> 💡 Say this confidently — most candidates don't.

---

### What's the biggest mistake developers make with abstraction?

- Creating abstraction **without need**
- Designing for **future assumptions**
- Not understanding the **real use case**

---

## 🔥 Code Debug Question

```java
interface A {
    void show();
}

class B implements A {
    public void show() {
        System.out.println("B");
    }
}

A obj = new B();
obj.show(); // Output: "B"
```

**What happens?**
- **Runtime polymorphism** — method of `B` executes
- This is **Abstraction + Polymorphism** working together
