What is encapsulation in Java?
=> Encapsulation in Java is the concept of wrapping data and methods together in a single unit, typically a class, and restricting direct access to the data by making variables private and providing controlled access through public getter and setter methods. This helps in achieving data hiding, improving security, and maintaining flexibility in the code.

“In this design, encapsulation is achieved by enforcing invariants, controlling state transitions through behavior methods, and protecting internal representation using defensive copying.”



Good. Now we move from theory → pressure.

I’m not giving you toy questions. These are **interview-level + real-world design checks**. If you can’t solve these cleanly, your understanding of encapsulation is still weak.

---

# 🔥 Problem 1: Order System (Basic but revealing)

### Requirement:

* An `Order` has:

  * list of items
  * total price
* Rules:

  * No negative price
  * Total should always match sum of items
  * External code should NOT modify items directly

---

## ❌ Most people write this garbage:

```java
class Order {
    public List<Item> items;
    public double total;
}
```

Immediate rejection in interview.

---

## ✅ What you should design

Try before seeing answer.

---

### ✔️ Correct Approach

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

---

## 🔴 What this tests:

* You didn’t expose `items`
* You controlled modification
* You protected invariant: `total = sum(items)`

---

# 🔥 Problem 2: Employee Salary System (Trick Question)

### Requirement:

* Salary should NEVER be negative
* Salary increment allowed only via method
* No direct modification

---

## ❌ Weak Solution

```java
class Employee {
    private double salary;

    public void setSalary(double salary) {
        this.salary = salary;
    }
}
```

👉 This is fake encapsulation again.

---

## ✅ Strong Solution

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

---

## 🔴 What this tests:

* No setter
* Controlled updates only
* Invariant protected

---

# 🔥 Problem 3: Student Grades (Advanced)

### Requirement:

* Store grades
* No one can:

  * Remove grades directly
  * Add invalid grades (<0 or >100)

---

## ✅ Expected Design

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
        return new ArrayList<>(grades);
    }
}
```

---

# 🔥 Problem 4 (Important for YOU – Backend mindset)

### Scenario: Payment Service

### Requirement:

* Payment status:

  * CREATED → SUCCESS / FAILED
* Once SUCCESS → cannot change
* No external class should change status randomly

---

## ✅ Strong Design

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

---

## 🔴 What this tests:

* State transition control (very important in microservices)
* No setter
* Business rules enforced

---

# ⚡ Brutal Interview Insight

When interviewer asks:

> “Explain encapsulation”

They actually mean:
👉 “Can you design objects that **cannot be misused**?”

---

# 🚨 Your Weak Spots (most likely)

Be honest:

* You still think setter = encapsulation ❌
* You don’t think about invariants ❌
* You expose collections ❌
* You don’t control state transitions ❌

Fix these or you’ll fail LLD rounds.

---

# 🎯 Your Task (Don’t skip)

Design this yourself:

### 👉 “Wallet System”

* Balance cannot go negative
* Add money / deduct money
* No direct balance modification
* Transaction history should not be modifiable from outside

---
Good question. This is exactly where you separate yourself from average candidates.

If you can use these terms correctly in an interview, you sound like someone who actually designs systems—not just writes code.

---

# 🔑 Key Terms in Encapsulation (Interview-Level)

## 1. **Invariant** (MOST IMPORTANT)

> A condition that must ALWAYS remain true for an object.

### Example:

```java
balance >= 0
```

### Say this:

> Encapsulation ensures invariants are never violated by restricting how state is modified.

If you don’t mention invariants → your answer is incomplete.

---

## 2. **Data Hiding**

> Restricting direct access to internal fields using `private`.

### Important:

This is just a **mechanism**, not the goal.

👉 Weak candidates stop here. Strong ones go beyond.

---

## 3. **Controlled Access**

> Access to data is only allowed through validated methods.

### Example:

* `deposit()` instead of `setBalance()`

### Say this:

> Encapsulation exposes behavior, not raw data.

---

## 4. **Abstraction Boundary**

> The line between what is exposed and what is hidden.

### Example:

```java
public void withdraw()
```

Internal logic is hidden.

👉 You decide what the outside world can see.

---

## 5. **Internal Representation**

> How data is stored inside the class.

### Example:

* List vs Set
* Array vs Map

### Key Point:

> This should be hidden so you can change it without breaking external code.

---

## 6. **Defensive Copying**

> Returning a copy instead of original mutable data.

### Example:

```java
public List<String> getItems() {
    return new ArrayList<>(items);
}
```

### Why:

Prevents external modification.

---

## 7. **Mutability / Immutability**

* **Mutable** → state can change
* **Immutable** → state cannot change after creation

### Example:

* `String` → immutable
* `StringBuilder` → mutable

### Interview line:

> Immutability is the strongest form of encapsulation.

---

## 8. **State Control / State Transition**

> Restricting how an object changes from one state to another.

### Example:

```
CREATED → SUCCESS (allowed)
SUCCESS → CREATED (not allowed)
```

### Say this:

> Encapsulation enforces valid state transitions.

---

## 9. **Coupling**

> Degree of dependency between classes

### Encapsulation reduces:

* Tight coupling
* Direct field dependency

---

## 10. **Integrity**

> Ensuring data remains correct and consistent

### Example:

* No negative balance
* Valid email format

---


---

# 🔥 Encapsulation Master Example

```java
import java.util.*;

class Wallet {

    // 🔹 DATA HIDING → private fields
    private double balance; // 🔹 INVARIANT: balance >= 0
    private final List<String> transactions; // 🔹 INTERNAL REPRESENTATION (mutable list)

    // 🔹 CONSTRUCTOR → enforce initial INVARIANT
    public Wallet(double initialBalance) {
        if (initialBalance < 0) {
            throw new IllegalArgumentException("Balance cannot be negative");
        }
        this.balance = initialBalance;
        this.transactions = new ArrayList<>();
    }

    // 🔹 CONTROLLED ACCESS → no direct setter
    public void addMoney(double amount) {
        if (amount <= 0) { // 🔹 INVARIANT PROTECTION
            throw new IllegalArgumentException("Invalid amount");
        }
        balance += amount; // 🔹 STATE CHANGE
        transactions.add("Added: " + amount);
    }

    // 🔹 CONTROLLED ACCESS + STATE TRANSITION CONTROL
    public void deductMoney(double amount) {
        if (amount <= 0 || amount > balance) { // 🔹 INVARIANT PROTECTION
            throw new IllegalArgumentException("Invalid deduction");
        }
        balance -= amount; // 🔹 STATE CHANGE
        transactions.add("Deducted: " + amount);
    }

    // 🔹 READ-ONLY ACCESS (SAFE)
    public double getBalance() {
        return balance;
    }

    // 🔹 DEFENSIVE COPYING → protect INTERNAL REPRESENTATION
    public List<String> getTransactions() {
        return new ArrayList<>(transactions); 
        // prevents external modification → maintains ENCAPSULATION
    }

    // 🔹 ABSTRACTION BOUNDARY → expose behavior, not data structure
    public int getTransactionCount() {
        return transactions.size();
    }
}
```

---

# 🔥 What This Code Demonstrates (Don’t skip)

### ✅ Invariant

* `balance >= 0` always maintained

### ✅ Data Hiding

* All fields are `private`

### ✅ Controlled Access

* No `setBalance()`
* Only `addMoney()` / `deductMoney()`

### ✅ State Transition Control

* Balance changes only through validated methods

### ✅ Internal Representation Hidden

* List is not exposed directly

### ✅ Defensive Copying

* Returning `new ArrayList<>(transactions)`

### ✅ Abstraction Boundary

* External world doesn’t know how transactions are stored

### ✅ Integrity

* No invalid state possible

---

# ⚠️ If You Remove These → Design Breaks

* Remove validation → invariant breaks
* Return original list → external mutation
* Add setter → uncontrolled state



# 🔥 Real Backend Entity Design (Spring Boot Style)

We’ll design a **Payment entity** the way a serious engineer does it.

---

## ✅ Entity Design (Encapsulation Done Right)

```java
import jakarta.persistence.*;
import java.time.LocalDateTime;

@Entity
@Table(name = "payments")
public class Payment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // 🔹 DATA HIDING
    @Column(nullable = false)
    private double amount;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Status status;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime completedAt;

    // 🔹 ENUM → controlled STATE
    public enum Status {
        CREATED, SUCCESS, FAILED
    }

    // 🔹 PROTECTED constructor for JPA
    protected Payment() {}

    // 🔹 CONTROLLED CREATION (factory method)
    public static Payment create(double amount) {
        if (amount <= 0) { // 🔹 INVARIANT
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

    // 🔹 SAFE READ ACCESS (getters)
    public Long getId() {
        return id;
    }

    public double getAmount() {
        return amount;
    }

    public Status getStatus() {
        return status;
    }

    public LocalDateTime getCreatedAt() {
        return createdAt;
    }

    public LocalDateTime getCompletedAt() {
        return completedAt;
    }
}
```

---

# 🔥 What This Actually Shows (Real Understanding)

## ✅ Invariant

* `amount > 0`
* status must follow valid lifecycle

---

## ✅ Controlled Creation

* No public constructor
* `create()` enforces rules

👉 This is how you stop invalid objects from even existing.

---

## ✅ State Transition Control

* `CREATED → SUCCESS / FAILED`
* No reverse transitions

👉 Critical for payment/order systems

---

## ✅ No Setters

* You don’t allow random mutation

👉 This is where 90% of developers fail

---

## ✅ Enum for State

* Avoids string chaos
* Makes transitions explicit

---

## ✅ JPA Compatibility

* `protected constructor` (required by Hibernate)
* Fields mapped cleanly

---

# ⚠️ What Most Developers Do (WRONG)

```java
@Entity
class Payment {
    private double amount;
    private String status;

    public void setAmount(double amount) { this.amount = amount; }
    public void setStatus(String status) { this.status = status; }
}
```

👉 This is not encapsulation.
👉 This is a **data bag**.

Interviewers reject this fast.

---

# 🔥 Service Layer Usage (Important)

```java
@Service
public class PaymentService {

    public void processPayment() {
        Payment payment = Payment.create(1000);

        // some business logic

        payment.markSuccess();
    }
}
```

👉 Service orchestrates
👉 Entity protects itself

---

import jakarta.persistence.*;
import lombok.AccessLevel;
import lombok.Getter;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Entity
@Table(name = "payments")

@Getter // 🔹 READ-ONLY access (safe)
@NoArgsConstructor(access = AccessLevel.PROTECTED) // 🔹 JPA requirement

public class Payment {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private double amount; // 🔹 INVARIANT: > 0

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private Status status;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    private LocalDateTime completedAt;

    public enum Status {
        CREATED, SUCCESS, FAILED
    }

    // 🔹 CONTROLLED CREATION
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

🔥 Why This Is Correct
✅ @Getter only
Safe exposure
No mutation
✅ No @Setter
Prevents breaking invariants
Forces controlled methods
✅ @NoArgsConstructor(PROTECTED)
Required by JPA
Prevents misuse
⚠️ Common Lombok Mistakes (You MUST avoid)
❌ Using @Setter
@Setter
private double amount;

👉 Anyone can break your invariant

❌ Using @AllArgsConstructor
@AllArgsConstructor

👉 Bypasses validation → invalid objects created

❌ Using @Builder blindly
@Builder

👉 Looks clean, but:

Skips validation
Allows invalid state creation
🔥 If You REALLY Want Builder (Do It Right)
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

👉 Controlled builder = factory method
