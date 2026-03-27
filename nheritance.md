What is Inheritance?
=> Inheritance in Java is a mechanism where a subclass acquires the non-private members of a superclass, allowing code reuse, extension of behavior,using the extends keyword. It models an IS-A relationship and enables runtime method overriding, but constructors are not inherited and Java supports single, multilevel, and hierarchical inheritance, but does not support multiple inheritance through classes.


If a method is private, it is not inherited by subclasses and therefore cannot be overridden. If a subclass defines a method with the same signature, it is treated as a completely new method, and method calls are resolved at compile time.



Good. Now you stop reading and **start writing**. I’ll give you **progressive practice**, not copy-paste junk.

---

# 🔥 Level 1 — Basic Inheritance (Warm-up)

## 👉 Task:

Create:

* `Vehicle` → method `start()`
* `Car` → method `drive()`

## ✅ Expected Code

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

# 🔥 Level 2 — Method Overriding (Important)

## 👉 Task:

* `Animal` → `sound()`
* `Dog` → override `sound()`

## ✅ Expected Code

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
        obj.sound(); // Dog's method
    }
}
```

---

# 🔥 Level 3 — Constructor + `super` (Most Asked)

## 👉 Task:

* Parent has constructor with parameter
* Child calls it using `super`

## ✅ Expected Code

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

public class Main {
    public static void main(String[] args) {
        Employee e = new Employee("John", 50000);
        e.display();
    }
}
```

---

# 🔥 Level 4 — Multilevel Inheritance

## 👉 Task:

* `Animal` → `eat()`
* `Dog` → `bark()`
* `Puppy` → `weep()`

## ✅ Expected Code

```java
class Animal {
    void eat() {
        System.out.println("Eating...");
    }
}

class Dog extends Animal {
    void bark() {
        System.out.println("Barking...");
    }
}

class Puppy extends Dog {
    void weep() {
        System.out.println("Weeping...");
    }
}

public class Main {
    public static void main(String[] args) {
        Puppy p = new Puppy();
        p.eat();
        p.bark();
        p.weep();
    }
}
```

---

# 🔥 Level 5 — Real-World Thinking (Important)

## 👉 Task:

Design:

* `Account` (base)
* `SavingsAccount`, `CurrentAccount`

## ✅ Expected Code

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

# 🚨 Brutal Practice Rules

Don’t just read this.

### You MUST:

1. Write all 5 examples yourself
2. Change names and logic slightly
3. Break code intentionally → understand errors
4. Practice `super`, `@Override`, polymorphism

---

# ⚠️ Common Mistakes (Fix Now)

* Forgetting `@Override` ❌
* Not using `super()` ❌
* Thinking inheritance = copying code ❌
* Wrong relationship (not IS-A) ❌

---

# 🎯 Next Step (Critical)

Now you should move to:
👉 **Method Overriding + Runtime Polymorphism (deep)**

Because inheritance without polymorphism = incomplete understanding.

---

If you want:
I’ll give you **interview-level tricky inheritance questions** that most people fail.





How does the SuperKeywords works in Java?
=> The super keyword in Java refers to the immediate superclass and is used to call superclass constructors, access overridden methods, and reference hidden fields. It ensures proper initialization and helps resolve ambiguity between superclass and subclass members. It must be used as the first statement when calling a superclass constructor.




What is the difference between this and super? Give rules and examples.”
this refers to the current object and is used to access current class members or call another constructor in the same class, while super refers to the immediate superclass and is used to call the superclass constructor or access overridden methods and hidden fields. Both must follow strict rules and cannot be used in static contexts.


## Rules (MEMORIZE THESE)

### Rule 1: Constructor calls

* `this()` → calls **another constructor of the same class**
* `super()` → calls **constructor of parent class**
* **Must be the first statement**
* **Cannot be used together**

```java
this();   // OK
super();  // OK
this(); super(); // ❌ compile-time error
```

---

### Rule 2: Method access

* `this.method()` → calls **current class method**
* `super.method()` → calls **parent class method** (useful when overridden)

```java
class Parent {
    void show() {
        System.out.println("Parent");
    }
}

class Child extends Parent {
    void show() {
        System.out.println("Child");
    }

    void test() {
        this.show();   // Child
        super.show();  // Parent
    }
}
```

---

### Rule 3: Variable access

* `this.x` → current class variable
* `super.x` → parent class variable (when hidden)

```java
class Parent {
    int x = 10;
}

class Child extends Parent {
    int x = 20;

    void print() {
        System.out.println(this.x);   // 20
        System.out.println(super.x);  // 10
    }
}
```

---

### Rule 4: Static context

* ❌ `this` **cannot** be used in static context
* ❌ `super` **cannot** be used in static context

Why?

> Static members belong to the class, not the object.

---

### Rule 5: Private members

* ❌ `super` cannot access `private` members of parent
* `this` can access private members of the **same class**

---

## Side-by-side comparison (INTERVIEW GOLD)

| Aspect           | `this`         | `super`               |
| ---------------- | -------------- | --------------------- |
| Refers to        | Current object | Parent part of object |
| Constructor call | `this()`       | `super()`             |
| Method call      | Current class  | Parent class          |
| Variable access  | Current class  | Parent class          |
| Static context   | ❌ No           | ❌ No                  |
| Access private   | ✅ Same class   | ❌ Parent              |

---

## Most important conceptual line (say this)

> `this` and `super` do not create different objects; they are references to different levels of the same object in the inheritance hierarchy.

That sentence shows **real understanding**.

---

## Common interview traps (don’t fall)

❌ “`super` creates parent object” → WRONG
❌ “`this` and `super` are objects” → WRONG
❌ “You can use both in constructor” → WRONG

Good. Now we stop spoon-feeding and start testing.

These are the kinds of questions where candidates *think* they know inheritance—and then crash.

---

# 🔥 Question 1 — Method Overriding vs Overloading (Trap)

```java
class A {
    void show(int x) {
        System.out.println("A");
    }
}

class B extends A {
    void show(double x) {
        System.out.println("B");
    }
}

public class Main {
    public static void main(String[] args) {
        A obj = new B();
        obj.show(10);
    }
}
```

## ❓ What is the output?

👉 Most people say **B** → WRONG.

---

# 🔥 Question 2 — Runtime Polymorphism

```java
class A {
    void show() {
        System.out.println("A");
    }
}

class B extends A {
    void show() {
        System.out.println("B");
    }
}

public class Main {
    public static void main(String[] args) {
        A obj = new B();
        obj.show();
    }
}
```

## ❓ Output?

---

# 🔥 Question 3 — Variable Hiding (Very Important)

```java
class A {
    int x = 10;
}

class B extends A {
    int x = 20;
}

public class Main {
    public static void main(String[] args) {
        A obj = new B();
        System.out.println(obj.x);
    }
}
```

## ❓ Output?

👉 If you treat variables like methods, you fail.

---

# 🔥 Question 4 — `super` Keyword

```java
class A {
    int x = 10;
}

class B extends A {
    int x = 20;

    void print() {
        System.out.println(x);
        System.out.println(super.x);
    }
}

public class Main {
    public static void main(String[] args) {
        B obj = new B();
        obj.print();
    }
}
```

## ❓ Output?

---

# 🔥 Question 5 — Constructor Execution Order

```java
class A {
    A() {
        System.out.println("A Constructor");
    }
}

class B extends A {
    B() {
        System.out.println("B Constructor");
    }
}

public class Main {
    public static void main(String[] args) {
        new B();
    }
}
```

## ❓ Output order?

---

# 🔥 Question 6 — Final Method (Concept Check)

```java
class A {
    final void show() {
        System.out.println("A");
    }
}

class B extends A {
    void show() {
        System.out.println("B");
    }
}
```

## ❓ What happens?

---

# 🔥 Question 7 — Static Method (Hidden Trap)

```java
class A {
    static void show() {
        System.out.println("A");
    }
}

class B extends A {
    static void show() {
        System.out.println("B");
    }
}

public class Main {
    public static void main(String[] args) {
        A obj = new B();
        obj.show();
    }
}
```

## ❓ Output?

👉 If you think runtime polymorphism applies here, you're wrong.

---

# 🔥 Question 8 — Private Method (Advanced Trap)

```java
class A {
    private void show() {
        System.out.println("A");
    }

    void call() {
        show();
    }
}

class B extends A {
    void show() {
        System.out.println("B");
    }
}

public class Main {
    public static void main(String[] args) {
        A obj = new B();
        obj.call();
    }
}
```

## ❓ Output?

---

# 🚨 Brutal Instructions

Don’t guess.
Write answers like this:

```
Q1: ?
Q2: ?
...
Q8: ?
```

Then I’ll:

* Tear apart your mistakes
* Fix your thinking
* Explain ONLY where you’re weak

---

If you get more than 2 wrong → your inheritance understanding is still shallow.


