# Inheritance in Java — Interview Notes

Inheritance is a mechanism where a subclass acquires the non-private members of a superclass, allowing code reuse and extension of behavior using the `extends` keyword. It models an **IS-A relationship** and enables runtime method overriding. Constructors are not inherited, and Java supports single, multilevel, and hierarchical inheritance — but **not multiple inheritance through classes**.

> ⚠️ If a method is `private`, it is not inherited and cannot be overridden. If a subclass defines a method with the same signature, it is treated as a completely new method and resolved at **compile time**.

---

## 🔥 Level 1 — Basic Inheritance

```java
class Vehicle {
    void start() {
        System.out.println("Vehicle starts");
    }
}

class Car extends Vehicle {
    void drive() {
        System.out.println("Car drives");
    }
}

public class Main {
    public static void main(String[] args) {
        Car car = new Car();
        car.start(); // inherited
        car.drive();
    }
}
```

---

## 🔥 Level 2 — Method Overriding

```java
class Animal {
    void sound() {
        System.out.println("Animal makes sound");
    }
}

class Dog extends Animal {
    @Override
    void sound() {
        System.out.println("Dog barks");
    }
}

public class Main {
    public static void main(String[] args) {
        Animal obj = new Dog(); // polymorphism
        obj.sound();            // Dog's method executes
    }
}
```

---

## 🔥 Level 3 — Constructor + `super`

```java
class Person {
    String name;

    Person(String name) {
        this.name = name;
    }
}

class Employee extends Person {
    int salary;

    Employee(String name, int salary) {
        super(name); // calling parent constructor
        this.salary = salary;
    }

    void display() {
        System.out.println(name + " earns " + salary);
    }
}
```

---

## 🔥 Level 4 — Multilevel Inheritance

```java
class Animal {
    void eat() { System.out.println("Eating..."); }
}

class Dog extends Animal {
    void bark() { System.out.println("Barking..."); }
}

class Puppy extends Dog {
    void weep() { System.out.println("Weeping..."); }
}

public class Main {
    public static void main(String[] args) {
        Puppy p = new Puppy();
        p.eat();   // from Animal
        p.bark();  // from Dog
        p.weep();  // own method
    }
}
```

---

## 🔥 Level 5 — Real-World Design

```java
class Account {
    double balance;

    Account(double balance) {
        this.balance = balance;
    }

    void showBalance() {
        System.out.println("Balance: " + balance);
    }
}

class SavingsAccount extends Account {
    SavingsAccount(double balance) {
        super(balance);
    }

    void addInterest() {
        balance += balance * 0.05;
    }
}

class CurrentAccount extends Account {
    CurrentAccount(double balance) {
        super(balance);
    }

    void withdraw(double amount) {
        balance -= amount;
    }
}
```

---

## 🔑 `super` Keyword

The `super` keyword refers to the **immediate superclass** and is used to:
- Call superclass constructors
- Access overridden methods
- Reference hidden fields

> Must be used as the **first statement** when calling a superclass constructor.

---

## 🔑 `this` vs `super` — Interview Gold

| Aspect | `this` | `super` |
|---|---|---|
| Refers to | Current object | Parent part of object |
| Constructor call | `this()` | `super()` |
| Method call | Current class | Parent class |
| Variable access | Current class | Parent class |
| Static context | ❌ No | ❌ No |
| Access private | ✅ Same class | ❌ Parent |

### Rules (MEMORIZE)

**Rule 1 — Constructor calls**
- `this()` → calls another constructor of the same class
- `super()` → calls parent class constructor
- Must be the **first statement** — cannot be used together

```java
this();          // ✅ OK
super();         // ✅ OK
this(); super(); // ❌ compile-time error
```

**Rule 2 — Method access**

```java
class Parent {
    void show() { System.out.println("Parent"); }
}

class Child extends Parent {
    void show() { System.out.println("Child"); }

    void test() {
        this.show();  // Child
        super.show(); // Parent
    }
}
```

**Rule 3 — Variable access**

```java
class Parent {
    int x = 10;
}

class Child extends Parent {
    int x = 20;

    void print() {
        System.out.println(this.x);  // 20
        System.out.println(super.x); // 10
    }
}
```

**Rule 4 — Static context:** `this` and `super` cannot be used in static methods. Static members belong to the class, not the object.

**Rule 5 — Private members:** `super` cannot access `private` members of parent. `this` can access private members of the same class.

### Most Important Conceptual Line

> *"`this` and `super` do not create different objects — they are references to different levels of the same object in the inheritance hierarchy."*

### Common Interview Traps

- ❌ "`super` creates a parent object" → WRONG
- ❌ "`this` and `super` are objects" → WRONG
- ❌ "You can use both in a constructor" → WRONG

---

## 🔥 Tricky Output Questions

### Q1 — Overriding vs Overloading Trap

```java
class A {
    void show(int x) { System.out.println("A"); }
}

class B extends A {
    void show(double x) { System.out.println("B"); }
}

public class Main {
    public static void main(String[] args) {
        A obj = new B();
        obj.show(10); // Output: A
    }
}
```

> `B.show(double)` is **overloading**, not overriding. Reference type is `A`, so `A.show(int)` is called. Most people say **B** — that's wrong.

---

### Q2 — Runtime Polymorphism

```java
class A {
    void show() { System.out.println("A"); }
}

class B extends A {
    void show() { System.out.println("B"); }
}

A obj = new B();
obj.show(); // Output: B
```

> Runtime polymorphism — actual object is `B`, so `B.show()` executes.

---

### Q3 — Variable Hiding

```java
class A { int x = 10; }
class B extends A { int x = 20; }

A obj = new B();
System.out.println(obj.x); // Output: 10
```

> Variables are resolved by **reference type** (compile time), not actual object type. Methods use runtime polymorphism — variables don't.

---

### Q4 — `super` Keyword

```java
class A { int x = 10; }

class B extends A {
    int x = 20;

    void print() {
        System.out.println(x);       // 20
        System.out.println(super.x); // 10
    }
}
```

---

### Q5 — Constructor Execution Order

```java
class A {
    A() { System.out.println("A Constructor"); }
}

class B extends A {
    B() { System.out.println("B Constructor"); }
}

new B();
// Output:
// A Constructor
// B Constructor
```

> Parent constructor always executes first.

---

### Q6 — Final Method

```java
class A {
    final void show() { System.out.println("A"); }
}

class B extends A {
    void show() { System.out.println("B"); } // ❌ Compile error
}
```

> `final` methods cannot be overridden.

---

### Q7 — Static Method (Method Hiding)

```java
class A {
    static void show() { System.out.println("A"); }
}

class B extends A {
    static void show() { System.out.println("B"); }
}

A obj = new B();
obj.show(); // Output: A
```

> Static methods are resolved by **reference type** (compile time). This is **method hiding**, not overriding — runtime polymorphism does NOT apply.

---

### Q8 — Private Method Trap

```java
class A {
    private void show() { System.out.println("A"); }

    void call() { show(); }
}

class B extends A {
    void show() { System.out.println("B"); }
}

A obj = new B();
obj.call(); // Output: A
```

> `A.show()` is `private` — not inherited, not overridden. `call()` always calls `A`'s own private `show()`.

---

## 🔹 Design Scenarios

### Vehicle System

```java
class Vehicle {
    void start() {}
}

class Car extends Vehicle {}
class Bike extends Vehicle {}
```

- Shared behavior → parent
- Specific behavior (e.g., AC for Car) → child only, don't pollute parent

---

### Employee System

```java
class Employee {
    double salary;
}

class Developer extends Employee {} // has coding skills
class Manager extends Employee {}   // has team size
```

- Common properties → parent
- Role-specific fields → keep in respective child classes

---

### Payment System

```java
// Weak:
class Payment {}
class Upi extends Payment {}

// Strong thinking:
// Payment types behave very differently → prefer interface (abstraction)
// Not everything should be inheritance
```

---

## 🔹 Interview Question Bank

### Should we use inheritance for code reuse?

> ❌ Not always. Prefer **composition over inheritance**.
> Inheritance creates tight coupling — changes in parent affect all children.

---

### Inheritance vs Composition

| | Inheritance | Composition |
|---|---|---|
| Relationship | IS-A | HAS-A |
| Coupling | Tight | Flexible |
| Preference | Use carefully | Prefer this |

---

### When should you NOT use inheritance?

- When the relationship is not IS-A
- When behavior changes frequently
- When multiple combinations are needed

```java
// ❌ Wrong
class EngineCar extends Car {} // Engine is NOT a Car

// ✅ Correct → Engine is HAS-A → use composition
```

---

### What is the biggest problem with inheritance?

- Tight coupling — changes in parent affect all children
- Deep chains are hard to debug and modify
- Creates ripple effects across the hierarchy

```
A → B → C → D → E  // ❌ avoid deep chains
```

### Biggest mistake developers make?

- Using inheritance for **reuse instead of relationship**
- Creating deep inheritance chains
- Not thinking about maintainability

---

## ⚠️ Common Mistakes (Fix Now)

- Forgetting `@Override` ❌
- Not using `super()` ❌
- Thinking inheritance = copying code ❌
- Using inheritance for a non IS-A relationship ❌
