
# 📘 Chapter 2 — Inheritance

## 2.1 What Is Inheritance? (The Real Definition)

Most people say *"inheritance = one class gets the properties of another."*
That is **incomplete**. Here's the real definition:

> **Inheritance** is a mechanism where a child class **acquires non-private the state and behavior** of a parent class, and can **extend or override** that behavior — establishing an **IS-A relationship** between the two.

    ⚠️ If a method is private, it is not inherited and cannot be overridden. If a subclass defines a method with the same signature, it is treated as a completely new method and resolved at compile time.

The keyword is **IS-A**. Before using inheritance anywhere, ask:

```
"Is a Dog actually an Animal in every possible context?"
  → YES → Inheritance is valid

"Is a Stack actually a Vector?"
  → NO  → Java's java.util.Stack got this wrong. Don't repeat it.
```

---

## 2.2 The Problem It Solves

Without inheritance:

```java
class Dog {
    String name;
    void eat()   { System.out.println("Dog eats"); }
    void sleep() { System.out.println("Dog sleeps"); }
    void bark()  { System.out.println("Dog barks"); }
}

class Cat {
    String name;
    void eat()   { System.out.println("Cat eats"); }   // ← DUPLICATE
    void sleep() { System.out.println("Cat sleeps"); } // ← DUPLICATE
    void meow()  { System.out.println("Cat meows"); }
}
```

`eat()` and `sleep()` are duplicated across every animal. If the logic changes, you must update **every class**. This violates **DRY (Don't Repeat Yourself)**.

With inheritance:

```java
class Animal {
    String name;
    void eat()   { System.out.println(name + " eats"); }
    void sleep() { System.out.println(name + " sleeps"); }
}

class Dog extends Animal {
    void bark() { System.out.println(name + " barks"); }
}

class Cat extends Animal {
    void meow() { System.out.println(name + " meows"); }
}
```

`eat()` and `sleep()` live in **one place**. Change once, affects all.

---

## 2.3 Java Inheritance Syntax — Every Keyword Explained

```java

public class InheritanceExample {

    public static void main(String[] args) {
        
        // Creating a Dog object
        Dog tommy = new Dog("Tommy", 5, "Golden Retriever");

        // Calling methods
        tommy.eat();           // Inherited from Animal
        tommy.bark();          // Dog's own method
        tommy.showBreed();     // Dog's own method

        // Using inherited getter
        System.out.println("Age from getter: " + tommy.getAge());
    }
}

// ====================== PARENT CLASS ======================
class Animal {                     // Parent / Superclass / Base class

    protected String name;         // Subclasses can access this
    private int age;               // Subclasses CANNOT access this directly

    // Constructor
    public Animal(String name, int age) {
        this.name = name;
        this.age = age;             // Age is set here
    }

    // Method that will be inherited
    public void eat() {
        System.out.println(name + " is eating");
    }

    // Getter for private field (safe way for subclasses)
    public int getAge() {         // Public door to see age
        return age;
    }
}

// ====================== CHILD CLASS ======================
class Dog extends Animal {         // Child / Subclass / Derived class

    private String breed;          // Dog's own field

    // Constructor
    public Dog(String name, int age, String breed) {
        super(name, age);          // MUST call parent constructor first // Send age to parent to set it
        this.breed = breed;
    }

    // Dog's own method
    public void bark() {
        System.out.println(name + " barks!");           // 'name' is protected → OK
        System.out.println(name + " is " + getAge() + " years old."); // using getter
    }

    // Another method specific to Dog
    public void showBreed() {
        System.out.println(name + " is a " + breed);
    }
}
```

### Your Question:
> In the child class `Dog`, we cannot access `private int age` directly.  
> Then how did we print `"Tommy is 5 years old"` and `"Age from getter: 5"`?

---

### Simple Answer:

Even though `Dog` **cannot touch** the `age` variable directly,  
it **can ask** the parent class (`Animal`) to tell the age using a **public method** called `getAge()`.

This is the **safe and correct way** in Java.

---

### Let’s Break It Down Super Simply (Like a Story)

1. **What is `private int age`?**
   - It is locked inside the `Animal` class.
   - Only code written **inside** `Animal` can touch it directly.
   - `Dog` is outside, so it cannot write `age = 10;` or `System.out.println(age);`

2. **What is `public int getAge()`?**
   - This is a **public door** that `Animal` provides.
   - Anyone (including `Dog`) can call `getAge()` to **ask** for the age.
   - The `Animal` class itself opens the door and returns the value of `age`.

3. **How does the age become 5?**

   Look at this line carefully:

   ```java
   public Dog(String name, int age, String breed) {
       super(name, age);        // ← This line sets the age!
       this.breed = breed;
   }
   ```

   - When you create a Dog:
     ```java
     Dog tommy = new Dog("Tommy", 5, "Golden Retriever");
     ```

   - Java first goes to `Animal` constructor using `super(name, age)`
   - It passes `5` to Animal’s constructor.
   - Inside Animal constructor:
     ```java
     public Animal(String name, int age) {
         this.name = name;
         this.age = age;     // ← Age is set here to 5
     }
     ```

   So the age is set **inside the parent class**, not in the child.

4. **How does Dog see the age?**

   In `bark()` method:

   ```java
   System.out.println(name + " is " + getAge() + " years old.");
   ```

   - `Dog` calls `getAge()` → which is a public method **inherited** from `Animal`.
   - `getAge()` runs the code `return age;` (which is inside Animal).
   - So it safely returns the value 5.

---

## 2.4 What Gets Inherited — Exactly

This is where most developers are fuzzy. Here's the precise truth:

```
┌────────────────────────────────┬─────────────────┬──────────────────────────┐
│ Member                         │ Inherited?      │ Notes                    │
├────────────────────────────────┼─────────────────┼──────────────────────────┤
│ public fields                  │ ✅ Yes          │                          │
│ protected fields               │ ✅ Yes          │                          │
│ default (package) fields       │ ✅ If same pkg  │                          │
│ private fields                 │ ❌ No           │ Exist in memory but      │
│                                │                 │ not directly accessible  │
│ public methods                 │ ✅ Yes          │                          │
│ protected methods              │ ✅ Yes          │                          │
│ private methods                │ ❌ No           │                          │
│ static methods                 │ ✅ Inherited    │ But NOT overridden —     │
│                                │                 │ they are HIDDEN          │
│ Constructors                   │ ❌ No           │ Called via super()       │
│ static fields                  │ ✅ Inherited    │ Shared, not per-instance │
└────────────────────────────────┴─────────────────┴──────────────────────────┘
```

The most important nuance: **private fields ARE in memory** inside the child object. They are part of the object's layout. You just can't access them directly — only through inherited public/protected methods.

---

## 2.5 The Object in Memory — JVM Internals

When you create a `Dog` object:

```java
Dog tommy = new Dog("Tommy", 5, "Golden Retriever");
```

You are **not** creating two separate things (one Animal + one Dog).  
Instead, the JVM creates **ONE single object** in memory that contains **both**:

- Everything from the parent (`Animal`)
- Everything from the child (`Dog`)

It's like a **sandwich** — the Animal part is the bottom bread + filling, and Dog part is the top bread. But when you eat it, you eat **one sandwich**, not two separate things.

        This is exactly what happens in JVM memory.

---

### How the Dog Object Looks in Heap Memory

```
Heap Memory — One Single Dog Object
┌──────────────────────────────────────────────┐
│   Object Header (16 bytes)                   │  ← Every Java object has this
│   (contains mark word, class pointer, etc.)  │
├──────────────────────────────────────────────┤
│   Animal Part (Inherited)                    │
│     name   → reference to "Tommy"            │
│     age    → 5                               │
├──────────────────────────────────────────────┤
│   Dog Part (Added by Child)                  │
│     breed  → reference to "Golden Retriever" │
└──────────────────────────────────────────────┘
```

**Important Points:**

- Everything is stored in **one contiguous memory block**.
- There is **no separate Animal object** in memory.
- The `Animal` fields (`name` and `age`) are physically inside the `Dog` object.
- `private` fields like `age` are still present in memory — they are just **not accessible** by name from the `Dog` class code.

---

### Why `super` Does Not Create a New Object

Many students think `super()` creates a parent object.

**Wrong thinking:**
“First Animal object is created, then Dog object is created on top.”

**Correct thinking:**
`super(name, age)` just tells the JVM:  
“Please initialize the Animal part of this same Dog object with these values.”

It’s like telling the factory:  
“First put the Animal features properly, then add the Dog features.”

Everything stays in **one single object**.

---

## 2.6 `super` Keyword — All 3 Uses

```java
class Animal {
    String name;
    int age;

    Animal(String name, int age) {
        this.name = name;
        this.age = age;
    }

    void describe() {
        System.out.println("Animal: " + name);
    }
}

class Dog extends Animal {
    String breed;

    Dog(String name, int age, String breed) {
        super(name, age);        // USE 1: Call parent constructor
                                 // MUST be first line in child constructor
        this.breed = breed;
    }

    @Override
    void describe() {
        super.describe();        // USE 2: Call parent's version of overridden method
        System.out.println("Breed: " + breed);
    }

    void showAge() {
        System.out.println(super.name); // USE 3: Access parent field
                                        // (rarely needed — usually just use this.name
                                        //  since name is inherited)
    }
}
```

**Critical rule:** `super()` must always be the **first statement** in the child constructor. If you don't write it, Java inserts `super()` (no-arg) automatically. If the parent has no no-arg constructor, you **must** explicitly call the right `super(...)` or it won't compile.

---

## 2.7 Method Overriding — The Heart of Inheritance

Overriding lets a child class **redefine** a parent's method:

```java
class Animal {
    public String sound() {
        return "...";
    }
}

class Dog extends Animal {
    @Override                          // ← annotation — catches mistakes at compile time
    public String sound() {
        return "Woof";
    }
}

class Cat extends Animal {
    @Override
    public String sound() {
        return "Meow";
    }
}
```

**Rules for valid overriding:**

```
┌─────────────────────────────┬────────────────────────────────────────────┐
│ Rule                        │ Detail                                     │
├─────────────────────────────┼────────────────────────────────────────────┤
│ Same method name            │ Must match exactly                         │
│ Same parameter list         │ Must match exactly                         │
│ Return type                 │ Same OR covariant (subtype is allowed)     │
│ Access modifier             │ Same OR broader (can't reduce visibility)  │
│ Exceptions                  │ Can throw fewer/narrower checked exceptions│
│ static methods              │ Cannot be overridden — they are HIDDEN     │
│ private methods             │ Cannot be overridden — not inherited       │
│ final methods               │ Cannot be overridden — compiler blocks it  │
└─────────────────────────────┴────────────────────────────────────────────┘
```

**Covariant return type example:**

```java
class Animal {
    public Animal create() { return new Animal(); }
}

class Dog extends Animal {
    @Override
    public Dog create() {   // ✅ Dog IS-A Animal — covariant return is allowed
        return new Dog();
    }
}
```

```java
// Full Running Code - Method Overriding Example

public class MethodOverridingDemo {

    public static void main(String[] args) {
        
        System.out.println("=== Method Overriding Demo ===\n");

        // 1. Normal Animal object
        Animal animal = new Animal();
        System.out.println("Animal sound: " + animal.sound());

        // 2. Dog object with Animal reference (Polymorphism)
        Animal dog = new Dog();
        System.out.println("Dog sound: " + dog.sound());

        // 3. Cat object with Animal reference
        Animal cat = new Cat();
        System.out.println("Cat sound: " + cat.sound());

        // 4. Direct Dog and Cat references
        Dog d = new Dog();
        Cat c = new Cat();
        
        System.out.println("\nDirect calls:");
        System.out.println("Dog directly: " + d.sound());
        System.out.println("Cat directly: " + c.sound());

        // 5. Covariant Return Type Example
        System.out.println("\n=== Covariant Return Type Demo ===");
        Animal a1 = new Animal().create();
        Dog d1 = new Dog().create();           // Dog reference can hold Dog object
        
        System.out.println("Created: " + a1.getClass().getSimpleName());
        System.out.println("Created: " + d1.getClass().getSimpleName());
    }
}

// ====================== PARENT CLASS ======================
class Animal {

    public String sound() {
        return "... (Some generic animal sound)";
    }

    // Method to demonstrate covariant return type
    public Animal create() {
        return new Animal();
    }
}

// ====================== CHILD CLASS 1 ======================
class Dog extends Animal {

    @Override
    public String sound() {
        return "Woof! Woof!";
    }

    @Override
    public Dog create() {        // Covariant return type (Dog instead of Animal)
        return new Dog();
    }
}

// ====================== CHILD CLASS 2 ======================
class Cat extends Animal {

    @Override
    public String sound() {
        return "Meow! Meow!";
    }
}

output =>

=== Method Overriding Demo ===

Animal sound: ... (Some generic animal sound)
Dog sound: Woof! Woof!
Cat sound: Meow! Meow!

Direct calls:
Dog directly: Woof! Woof!
Cat directly: Meow! Meow!

=== Covariant Return Type Demo ===
Created: Animal
Created: Dog

```
### "Why do we use Covariant Return Type? Is it used to return the child class name?"

    - Yes, Covariant Return Type allows a child class to return its own type (Dog) instead of the parent type (Animal) when overriding a method.
---

## 2.8 Method Hiding vs Method Overriding

This is the **most common interview trap**:

```java
// Static Method Hiding vs Instance Method Overriding

public class StaticVsInstanceDemo {

    public static void main(String[] args) {

        System.out.println("=== Static vs Instance Method Demo ===\n");

        // Case 1: Parent reference, Parent object
        Parent p1 = new Parent();
        p1.staticMethod();
        p1.instanceMethod();

        System.out.println("--------------------------------");

        // Case 2: Child reference, Child object
        Child c1 = new Child();
        c1.staticMethod();
        c1.instanceMethod();

        System.out.println("--------------------------------");

        // Case 3: Most Important - Parent reference, Child object (Polymorphism)
        Parent p2 = new Child();
        p2.staticMethod();      // ← What will this print?
        p2.instanceMethod();    // ← What will this print?

        System.out.println("--------------------------------");

        // Case 4: Direct call using class name (for static)
        Parent.staticMethod();
        Child.staticMethod();
    }
}

// ====================== PARENT CLASS ======================
class Parent {

    public static void staticMethod() {
        System.out.println("Parent static");
    }

    public void instanceMethod() {
        System.out.println("Parent instance");
    }
}

// ====================== CHILD CLASS ======================
class Child extends Parent {

    // This is NOT overriding → This is called "Method Hiding"
    public static void staticMethod() {
        System.out.println("Child static");
    }

    // This is TRUE Method Overriding
    @Override
    public void instanceMethod() {
        System.out.println("Child instance");
    }
}


Expected Output:
text=== Static vs Instance Method Demo ===

Parent static
Parent instance
--------------------------------
Child static
Child instance
--------------------------------
Parent static          ← Static method from Parent (Reference type decides)
Child instance         ← Instance method from Child (Actual object decides)
--------------------------------
Parent static
Child static

```

### Simple Explanation (Like you're 10 years old)

| Situation                        | Which Method Runs?     | Who Decides?          | Type of Behavior      |
|----------------------------------|------------------------|-----------------------|-----------------------|
| `p2.instanceMethod()`            | Child's version        | **Runtime** (Object)  | **Overriding**        |
| `p2.staticMethod()`              | Parent's version       | **Compile time** (Reference) | **Hiding**         |

### Why This Happens?

- **Instance methods** (non-static):  
  Java waits until the program is **running** to decide which method to call.  
  It looks at the **actual object** (`new Child()`), so it calls Child’s `instanceMethod()` → **Polymorphism**

- **Static methods**:  
  Java decides at **compile time** (before running).  
  It only looks at the **reference type** (`Parent p2`), so it calls Parent’s `staticMethod()`.

That’s why we say:
> **Static methods are hidden**, not overridden.

---

### Key Rules to Remember

- You **cannot override** `static` methods.
- If child declares a static method with same name → it **hides** the parent’s static method.
- `static` methods belong to the **class**, not to the object.
- Only **instance methods** support true **Runtime Polymorphism**.

---

## 2.9 Constructor Chaining — Full Call Order

This is crucial and confuses everyone:

```java
class A {
    A() {
        System.out.println("A constructor");
    }
}

class B extends A {
    B() {
        // super() is inserted here automatically by compiler
        System.out.println("B constructor");
    }
}

class C extends B {
    C() {
        // super() is inserted here automatically by compiler
        System.out.println("C constructor");
    }
}

new C();
```

Output:
```
A constructor     ← grandparent runs first
B constructor     ← parent runs second
C constructor     ← child runs last
```

**Why this order?** Because `C()` calls `B()` via `super()` before doing anything else, and `B()` calls `A()` via `super()` before doing anything else. Constructors **unwind from top of the hierarchy downward**. This ensures parent state is initialized before child state tries to use it.

---

## 2.10 The `final` Keyword and Inheritance

`final` can block inheritance at three levels:

```java
// 1. final CLASS — cannot be extended at all
public final class String { }       // nobody can extend String
public final class Integer { }      // nobody can extend Integer

// 2. final METHOD — cannot be overridden
public class Animal {
    public final void breathe() {   // every animal breathes the same way
        System.out.println("Breathing...");
    }
}

class Dog extends Animal {
    // @Override void breathe() ← ❌ compile error
}

// 3. final FIELD — cannot be reassigned (unrelated to inheritance,
//    but often used with it in immutable parent classes)
public class Animal {
    public final String kingdom = "Animalia";  // all animals share this
}
```

---

## 2.11 Types of Inheritance in Java

```
Single:      A → B
             (Dog extends Animal)

Multilevel:  A → B → C
             (GoldenRetriever extends Dog extends Animal)

Hierarchical: A → B
              A → C
             (Dog extends Animal, Cat extends Animal)

Multiple:    ❌ NOT supported with classes in Java
             (Java avoids the Diamond Problem)

Multiple via Interfaces: ✅ Allowed
             class Dog extends Animal implements Swimmable, Trainable { }
```

**Why Java doesn't allow multiple class inheritance — The Diamond Problem:**

```
       Animal
      /      \
  Dog         Robot
      \      /
      RobotDog    ← which eat() does RobotDog inherit? Dog's or Robot's?
```

Java solves this by **allowing only single class inheritance** but **multiple interface implementation**. Interfaces (since Java 8) can have `default` methods, and Java has explicit rules for resolving diamond conflicts there.

---

## 2.12 Inheritance vs Composition — The Most Important Design Decision

This is what separates junior devs from senior devs.



- **Inheritance** = "IS-A" relationship  
  The child **is a** type of the parent.

- **Composition** = "HAS-A" relationship  
  One class **has** another class as a part (it contains it).

**Modern Rule of Thumb (Very Important):**  
**Prefer Composition over Inheritance**  
(You will hear this in almost every senior Java interview)

---

### Best Real-Life Analogy

**Inheritance (IS-A):**
- You are a **Human**.
- A **Doctor** IS-A Human.
- A **Teacher** IS-A Human.

The Doctor and Teacher inherit common things from Human (like breathing, eating, having a name).

**Composition (HAS-A):**
- A **Car** HAS-A Engine.
- A **Car** HAS-A Steering Wheel.
- A **Car** HAS-A Wheels.

The Car is **not** an Engine. It **contains** an Engine as a part.

If the Engine breaks, you can replace it easily without changing the whole Car.  
But if you use inheritance wrongly (Car extends Engine), changing the Engine breaks the Car badly.

---

### Side-by-Side Comparison (Very Clear)

| Feature                    | Inheritance ("IS-A")                          | Composition ("HAS-A")                          |
|---------------------------|-----------------------------------------------|------------------------------------------------|
| Relationship              | Dog **is an** Animal                          | Car **has an** Engine                          |
| Code Reuse                | Good                                          | Better & more flexible                         |
| Coupling                  | Tight (strong dependency)                     | Loose (easy to change)                         |
| Flexibility               | Low (changing parent affects all children)    | High (can change parts at runtime)             |
| When to Use               | When subclass is a true specialized version   | Almost always (unless clear IS-A)              |
| Problem with Deep Hierarchy | Fragile Base Class Problem                    | No such problem                                |

---

### 1. Inheritance Example (IS-A)

```java
class Animal {
    public void eat() {
        System.out.println("Eating food...");
    }
}

class Dog extends Animal {        // Dog IS-A Animal
    public void bark() {
        System.out.println("Woof Woof!");
    }
}
```

**Problem:** If you later change `Animal` class, it may break all subclasses (`Dog`, `Cat`, etc.).

---

### 2. Composition Example (HAS-A) — Better Approach

```java
// Engine as a separate class
class Engine {
    public void start() {
        System.out.println("Engine started...");
    }
    
    public void stop() {
        System.out.println("Engine stopped...");
    }
}

// Car uses composition
class Car {
    private final Engine engine;        // Car HAS-A Engine

    public Car() {
        this.engine = new Engine();     // Composition
    }

    public void startCar() {
        engine.start();                 // Delegate to Engine
        System.out.println("Car is moving!");
    }

    public void stopCar() {
        engine.stop();
        System.out.println("Car stopped.");
    }
}
```

**Advantages of Composition here:**
- You can easily replace Engine with ElectricEngine or DieselEngine.
- Changing Engine does **not** break the Car class.
- You have full control (you can add, remove, or change behavior).

---

### Full Running Demo Code (Single File)

```java
public class InheritanceVsCompositionDemo {

    public static void main(String[] args) {
        
        System.out.println("=== Inheritance Example (IS-A) ===");
        Dog dog = new Dog();
        dog.eat();
        dog.bark();

        System.out.println("\n=== Composition Example (HAS-A) ===");
        Car myCar = new Car();
        myCar.startCar();
        myCar.stopCar();
    }
}

// ====================== INHERITANCE ======================
class Animal {
    public void eat() {
        System.out.println("Eating food...");
    }
}

class Dog extends Animal {
    public void bark() {
        System.out.println("Woof Woof!");
    }
}

// ====================== COMPOSITION ======================
class Engine {
    public void start() {
        System.out.println("Engine started...");
    }
    
    public void stop() {
        System.out.println("Engine stopped...");
    }
}

class Car {
    private final Engine engine;   // Composition

    public Car() {
        this.engine = new Engine();
    }

    public void startCar() {
        engine.start();
        System.out.println("Car is running on road!");
    }

    public void stopCar() {
        engine.stop();
        System.out.println("Car is parked.");
    }
}
```

### When Should You Use Which?

**Use Inheritance when:**
- There is a clear **IS-A** relationship (Dog is an Animal, Square is a Shape).
- You need polymorphism heavily.
- The hierarchy is small and stable.

**Use Composition when:**
- You want flexibility.
- You are reusing behavior (not type).
- The relationship can change at runtime.
- You want to avoid fragile base class problem.

**Golden Rule (Remember this):**
> **"Favor Composition over Inheritance"** — This is one of the most important design principles in modern Java/Spring Boot development.


---

## 2.13 Real-World Pattern — Inheritance in Spring Boot

In your ShopSphere project, this pattern appears constantly:

```java
// Base entity — common audit fields in one place
@MappedSuperclass                        // JPA: not a table, but fields are inherited
public abstract class BaseEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @CreatedDate
    private Instant createdAt;

    @LastModifiedDate
    private Instant updatedAt;

    // getters only — no setters for audit fields
    public Long getId()         { return id; }
    public Instant getCreatedAt() { return createdAt; }
    public Instant getUpdatedAt() { return updatedAt; }
}

// Every service entity extends this — gets id, createdAt, updatedAt for free
@Entity
@Table(name = "orders")
public class Order extends BaseEntity {
    private String customerId;
    private double total;
    private OrderStatus status;

    // Order-specific fields and methods only
    public void cancel() {
        if (this.status == OrderStatus.SHIPPED)
            throw new IllegalStateException("Cannot cancel shipped order");
        this.status = OrderStatus.CANCELLED;
    }
}

@Entity
@Table(name = "products")
public class Product extends BaseEntity {
    private String name;
    private double price;
    // No id/createdAt/updatedAt — inherited from BaseEntity
}
```

This is inheritance doing exactly what it should — eliminating duplication of cross-cutting concerns across all entities.

---

## 2.14 Under the Hood — How JVM Resolves Method Calls (vtable)

When you write `animal.sound()` where `animal` is actually a `Dog`:

```
1. JVM looks at the object header in heap → finds class pointer → Dog.class

2. Dog.class has a vtable (Virtual Method Table):
   ┌──────────────────┬─────────────────────────────┐
   │ Method           │ Points to                   │
   ├──────────────────┼─────────────────────────────┤
   │ sound()          │ Dog.sound()    ← overridden  │
   │ eat()            │ Animal.eat()   ← inherited   │
   │ sleep()          │ Animal.sleep() ← inherited   │
   │ toString()       │ Object.toString() ← inherited│
   └──────────────────┴─────────────────────────────┘

3. JVM dispatches to Dog.sound() — this is dynamic dispatch
```

Every class has a vtable built at **class loading time**. Overriding = replacing an entry in the vtable. This is why `@Override` works — the child's vtable entry points to the child's method.

Static methods are NOT in the vtable — they're resolved by the **compiler** using the reference type, which is why they can't be overridden.

---

## 🎯 Chapter 2 — Interview Questions

**Q1. What is the difference between method overriding and method hiding?**
→ Overriding is for instance methods — resolved at runtime based on actual object type. Hiding is for static methods — resolved at compile time based on reference type. `@Override` cannot be used on static methods.

**Q2. Can a constructor be inherited?**
→ No. Constructors are not members and are not inherited. But the child can call the parent constructor using `super()`. If not called explicitly, the compiler inserts `super()` automatically, which fails if the parent has no no-arg constructor.

**Q3. What is the output of this code?**
```java
class A { A() { print(); } void print() { System.out.println("A"); } }
class B extends A { int x = 10; @Override void print() { System.out.println(x); } }
new B();
```
→ Output is `0`. When `new B()` is called, `A()` runs first (before B's fields are initialized), and during `A()`, `print()` resolves to `B.print()` via vtable. But `x` hasn't been initialized yet — it holds the default value `0`. This is the classic **"overriding in constructor"** trap.

**Q4. What is the diamond problem and how does Java solve it?**
→ When two parent classes have a method with the same signature and a child inherits both, it's ambiguous which to use. Java prevents this by disallowing multiple class inheritance. For interfaces with `default` methods, Java requires the class to explicitly override the conflicting method.

**Q5. Can you override a private method?**
→ No. Private methods are not inherited, so the child class defining a method with the same name creates a new, unrelated method — not an override. `@Override` on it would cause a compile error.

**Q6. When would you prefer composition over inheritance?**
→ When the relationship is HAS-A rather than IS-A, when you need flexibility to swap implementations at runtime, when the parent class may change and break child behavior (fragile base class problem), or when you want to avoid exposing all parent methods to callers.

**Q7. What is the fragile base class problem?**
→ When a parent class is modified (e.g., an internal method now calls another method that the child overrides), it can silently break child behavior without any API change. This is why `Effective Java` recommends designing for inheritance explicitly with `@Override`-safe methods, or prohibiting inheritance with `final`.

---

## ✅ Chapter 2 Summary

```
Inheritance
├── IS-A relationship — validate before using
├── extends keyword — single parent only
├── What's inherited: public + protected members
│   └── private fields exist in memory but not accessible directly
├── super keyword — 3 uses: constructor, method, field
├── Constructor chain: grandparent → parent → child
├── Method Overriding — runtime, instance methods, vtable dispatch
├── Method Hiding — compile time, static methods only
├── final — blocks class extension, method overriding
├── Types: Single, Multilevel, Hierarchical (no multiple class)
├── Diamond Problem → solved by disallowing multiple class inheritance
├── JVM vtable — how dynamic dispatch works internally
├── Real world: @MappedSuperclass for base entities in Spring Boot
└── Composition > Inheritance when IS-A doesn't truly hold
```

---

Ready for **Chapter 3 — Polymorphism**? That's where inheritance truly comes alive — runtime dispatch, `instanceof`, pattern matching, and how the JVM makes it all work. 🚀
