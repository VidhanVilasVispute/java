# Encapsulation in Java — Interview Notes

Encapsulation is the concept of wrapping data and methods together in a single unit (class), and restricting direct access to data by making variables `private` and providing controlled access through methods.

> *"Encapsulation is achieved by enforcing invariants, controlling state transitions through behavior methods, and protecting internal representation using defensive copying."*

---

## 🔑 Key Terms (Use These in Interview)

### 1. Invariant *(MOST IMPORTANT)*

> A condition that must **always** remain true for an object.

```java
balance >= 0
```

> *"Encapsulation ensures invariants are never violated by restricting how state is modified."*

If you don't mention invariants → your answer is incomplete.

---

### 2. Data Hiding

> Restricting direct access to internal fields using `private`.

- This is just a **mechanism**, not the goal.
- Weak candidates stop here — strong ones go beyond.

---

### 3. Controlled Access

> Access to data is only allowed through validated methods.

- Use `deposit()` instead of `setBalance()`

> *"Encapsulation exposes behavior, not raw data."*

---

### 4. Abstraction Boundary

> The line between what is exposed and what is hidden.

```java
public void withdraw() // internal logic is hidden
```

---

### 5. Internal Representation

> How data is stored inside the class (List vs Set, Array vs Map).

- Should be hidden so you can **change it without breaking external code**.

---

### 6. Defensive Copying

> Returning a copy instead of the original mutable data.

```java
public List<String> getItems() {
    return new ArrayList<>(items);
}
```

---

### 7. Mutability / Immutability

| Type | Meaning |
|---|---|
| **Mutable** | State can change |
| **Immutable** | State cannot change after creation |

- `String` → immutable
- `StringBuilder` → mutable

> *"Immutability is the strongest form of encapsulation."*

---

### 8. State Transition Control

> Restricting how an object moves from one state to another.

```
CREATED → SUCCESS  ✅ (allowed)
SUCCESS → CREATED  ❌ (not allowed)
```

> *"Encapsulation enforces valid state transitions."*

---

### 9. Coupling

> Degree of dependency between classes. Encapsulation reduces tight coupling and direct field dependency.

---

### 10. Integrity

> Ensuring data remains correct and consistent (e.g., no negative balance, valid email format).

---

## 🔥 Problem 1: Order System

### Requirement
- An `Order` has a list of items and a total price
- No negative price
- Total must always match sum of items
- External code cannot modify items directly

### ❌ What most people write

```java
class Order {
    public List<Item> items;
    public double total;
}
```

Immediate rejection in interview.

### ✅ Correct Approach

```java
class Order {
    private List<Item> items = new ArrayList<>();
    private double total;

    public void addItem(Item item) {
        if (item == null || item.getPrice() <= 0) {
            throw new IllegalArgumentException("Invalid item");
        }
        items.add(item);
        total += item.getPrice();
    }

    public void removeItem(Item item) {
        if (items.remove(item)) {
            total -= item.getPrice();
        }
    }

    public double getTotal() {
        return total;
    }

    public List<Item> getItems() {
        return new ArrayList<>(items); // defensive copy
    }
}
```

**What this tests:**
- You didn't expose `items`
- You controlled modification
- You protected the invariant: `total = sum(items)`

---

## 🔥 Problem 2: Employee Salary System

### Requirement
- Salary should never be negative
- Salary increment allowed only via method
- No direct modification

### ❌ Weak Solution

```java
class Employee {
    private double salary;

    public void setSalary(double salary) {
        this.salary = salary; // fake encapsulation
    }
}
```

### ✅ Strong Solution

```java
class Employee {
    private double salary;

    public Employee(double salary) {
        if (salary < 0) {
            throw new IllegalArgumentException("Invalid salary");
        }
        this.salary = salary;
    }

    public void incrementSalary(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Invalid increment");
        }
        salary += amount;
    }

    public double getSalary() {
        return salary;
    }
}
```

**What this tests:** No setter, controlled updates only, invariant protected.

---

## 🔥 Problem 3: Student Grades

### Requirement
- No one can remove grades directly
- No invalid grades (`< 0` or `> 100`)

### ✅ Expected Design

```java
class Student {
    private List<Integer> grades = new ArrayList<>();

    public void addGrade(int grade) {
        if (grade < 0 || grade > 100) {
            throw new IllegalArgumentException("Invalid grade");
        }
        grades.add(grade);
    }

    public double getAverage() {
        return grades.stream()
                     .mapToInt(Integer::intValue)
                     .average()
                     .orElse(0);
    }

    public List<Integer> getGrades() {
        return new ArrayList<>(grades); // defensive copy
    }
}
```

---

## 🔥 Problem 4: Payment Service (State Transitions)

### Requirement
- Status lifecycle: `CREATED → SUCCESS / FAILED`
- Once `SUCCESS` → cannot change
- No external class should change status randomly

### ✅ Strong Design

```java
class Payment {
    private String status;

    public Payment() {
        this.status = "CREATED";
    }

    public void markSuccess() {
        if (!status.equals("CREATED")) {
            throw new IllegalStateException("Invalid transition");
        }
        status = "SUCCESS";
    }

    public void markFailed() {
        if (!status.equals("CREATED")) {
            throw new IllegalStateException("Invalid transition");
        }
        status = "FAILED";
    }

    public String getStatus() {
        return status;
    }
}
```

**What this tests:** State transition control (very important in microservices), no setter, business rules enforced.

---

## 🔥 Master Example: Wallet System

```java
import java.util.*;

class Wallet {

    // 🔹 DATA HIDING → private fields
    private double balance;               // INVARIANT: balance >= 0
    private final List<String> transactions; // INTERNAL REPRESENTATION

    // 🔹 CONSTRUCTOR → enforce initial invariant
    public Wallet(double initialBalance) {
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Balance cannot be negative");
        }
        this.balance = initialBalance;
        this.transactions = new ArrayList<>();
    }

    // 🔹 CONTROLLED ACCESS → no direct setter
    public void addMoney(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Invalid amount");
        }
        balance += amount;
        transactions.add("Added: " + amount);
    }

    // 🔹 CONTROLLED ACCESS + STATE TRANSITION CONTROL
    public void deductMoney(double amount) {
        if (amount <= 0 || amount > balance) {
            throw new IllegalArgumentException("Invalid deduction");
        }
        balance -= amount;
        transactions.add("Deducted: " + amount);
    }

    // 🔹 READ-ONLY ACCESS
    public double getBalance() {
        return balance;
    }

    // 🔹 DEFENSIVE COPYING → protect internal representation
    public List<String> getTransactions() {
        return new ArrayList<>(transactions);
    }

    // 🔹 ABSTRACTION BOUNDARY → expose behavior, not data structure
    public int getTransactionCount() {
        return transactions.size();
    }
}
```

### What This Demonstrates

| Principle | How |
|---|---|
| Invariant | `balance >= 0` always maintained |
| Data Hiding | All fields are `private` |
| Controlled Access | No `setBalance()` — only `addMoney()` / `deductMoney()` |
| State Transition Control | Balance changes only through validated methods |
| Internal Representation Hidden | List not exposed directly |
| Defensive Copying | Returns `new ArrayList<>(transactions)` |
| Abstraction Boundary | External world doesn't know how transactions are stored |
| Integrity | No invalid state possible |

> ⚠️ Remove validation → invariant breaks. Return original list → external mutation. Add setter → uncontrolled state.

---

## 🔥 Real Backend Entity: Spring Boot Style

### ✅ Without Lombok

```java
import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "payments")
public class Payment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private double amount; // INVARIANT: > 0

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Status status;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime completedAt;

    public enum Status {
        CREATED, SUCCESS, FAILED
    }

    protected Payment() {} // JPA requirement

    // 🔹 CONTROLLED CREATION (factory method)
    public static Payment create(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        Payment payment = new Payment();
        payment.amount = amount;
        payment.status = Status.CREATED;
        payment.createdAt = LocalDateTime.now();
        return payment;
    }

    // 🔹 STATE TRANSITION CONTROL
    public void markSuccess() {
        if (this.status != Status.CREATED) {
            throw new IllegalStateException("Invalid state transition");
        }
        this.status = Status.SUCCESS;
        this.completedAt = LocalDateTime.now();
    }

    public void markFailed() {
        if (this.status != Status.CREATED) {
            throw new IllegalStateException("Invalid state transition");
        }
        this.status = Status.FAILED;
        this.completedAt = LocalDateTime.now();
    }

    // 🔹 NO SETTERS → prevent misuse
    public Long getId() { return id; }
    public double getAmount() { return amount; }
    public Status getStatus() { return status; }
    public LocalDateTime getCreatedAt() { return createdAt; }
    public LocalDateTime getCompletedAt() { return completedAt; }
}
```

### ✅ With Lombok (Correct Way)

```java
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;

@Entity
@Table(name = "payments")
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
public class Payment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private double amount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Status status;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime completedAt;

    public enum Status { CREATED, SUCCESS, FAILED }

    public static Payment create(double amount) {
        if (amount <= 0) {
            throw new IllegalArgumentException("Amount must be positive");
        }
        Payment payment = new Payment();
        payment.amount = amount;
        payment.status = Status.CREATED;
        payment.createdAt = LocalDateTime.now();
        return payment;
    }

    public void markSuccess() {
        if (this.status != Status.CREATED) {
            throw new IllegalStateException("Invalid transition");
        }
        this.status = Status.SUCCESS;
        this.completedAt = LocalDateTime.now();
    }

    public void markFailed() {
        if (this.status != Status.CREATED) {
            throw new IllegalStateException("Invalid transition");
        }
        this.status = Status.FAILED;
        this.completedAt = LocalDateTime.now();
    }
}
```

### Why the Lombok version is correct

| Annotation | Why |
|---|---|
| `@Getter` only | Safe read-only exposure, no mutation |
| No `@Setter` | Prevents breaking invariants, forces controlled methods |
| `@NoArgsConstructor(PROTECTED)` | Required by JPA, prevents misuse |

---

## ⚠️ Common Lombok Mistakes (MUST AVOID)

### ❌ Using `@Setter`

```java
@Setter
private double amount; // anyone can break your invariant
```

### ❌ Using `@AllArgsConstructor`

```java
@AllArgsConstructor // bypasses validation → invalid objects created
```

### ❌ Using `@Builder` blindly

```java
@Builder // skips validation, allows invalid state creation
```

### ✅ If you really want a builder — use a factory method

```java
public static Payment create(double amount) {
    if (amount <= 0) {
        throw new IllegalArgumentException("Invalid amount");
    }
    Payment p = new Payment();
    p.amount = amount;
    p.status = Status.CREATED;
    p.createdAt = LocalDateTime.now();
    return p;
}
```

> Controlled builder = factory method ✅

---

## 🔥 Service Layer Usage

```java
@Service
public class PaymentService {

    public void processPayment() {
        Payment payment = Payment.create(1000);

        // business logic

        payment.markSuccess();
    }
}
```

- **Service** → orchestrates
- **Entity** → protects itself

---

## ❌ What Most Developers Do (WRONG)

```java
@Entity
class Payment {
    private double amount;
    private String status;

    public void setAmount(double amount) { this.amount = amount; }
    public void setStatus(String status) { this.status = status; }
}
```

> This is not encapsulation. This is a **data bag**. Interviewers reject this fast.

---

## 🚨 Your Weak Spots (Be Honest)

- Still thinking setter = encapsulation ❌
- Not thinking about invariants ❌
- Exposing collections ❌
- Not controlling state transitions ❌

Fix these or you'll fail LLD rounds.

---

## ⚡ Brutal Interview Insight

When an interviewer asks *"Explain encapsulation"*, they actually mean:

> *"Can you design objects that **cannot be misused**?"*
