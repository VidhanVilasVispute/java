# 📘 Chapter 3 — Polymorphism

## 3.1 What Is Polymorphism? (The Real Definition)

Most people say *"polymorphism = one method, many forms."*
That is **incomplete**. Here's the real definition:

> **Polymorphism** is the ability of a **single reference type** to behave differently depending on the **actual object** it points to at runtime — allowing you to write code that works on the parent type but executes the child's behavior automatically.

The keyword is **single reference, multiple behaviors**. This is what makes large systems extensible without modification.

---

## 3.2 The Two Types of Polymorphism in Java

```
Polymorphism in Java
├── Compile-Time Polymorphism (Static)
│     └── Method Overloading
│           └── Resolved by compiler based on method signature
│
└── Runtime Polymorphism (Dynamic)
      └── Method Overriding
            └── Resolved by JVM at runtime based on actual object type
```

These are fundamentally different mechanisms. Let's master both.

---

## 3.3 Compile-Time Polymorphism — Method Overloading

The **compiler** decides which method to call based on the **number, type, and order of parameters** at compile time.

```java
public class Calculator {

    // Same name — different signatures
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {   // different param types
        return a + b;
    }

    public int add(int a, int b, int c) {     // different param count
        return a + b + c;
    }

    public String add(String a, String b) {   // different param types
        return a + b;
    }
}

Calculator calc = new Calculator();
calc.add(1, 2);          // → int version    (compile-time decision)
calc.add(1.0, 2.0);      // → double version (compile-time decision)
calc.add(1, 2, 3);       // → 3-arg version  (compile-time decision)
calc.add("Hi", " Java"); // → String version (compile-time decision)
```

### What Does NOT Make a Valid Overload

```java
// ❌ Return type alone is NOT enough
public int    getValue() { return 1; }
public double getValue() { return 1.0; } // compile error — ambiguous

// ❌ Access modifier alone is NOT enough
public    void show() { }
private   void show() { } // compile error — duplicate method

// ✅ Parameter order IS enough (though bad practice)
public void print(int a, String b) { }
public void print(String a, int b) { } // valid — different order
```

### Type Promotion in Overloading — The Sneaky Behavior

```java
public class Demo {
    public void method(int x)    { System.out.println("int"); }
    public void method(long x)   { System.out.println("long"); }
    public void method(double x) { System.out.println("double"); }
}

Demo d = new Demo();
d.method(5);      // → "int"    (exact match)
d.method(5L);     // → "long"   (exact match)
d.method('A');    // → "int"    ← char promoted to int!
d.method(5.0f);   // → "double" ← float promoted to double!
```

Java promotes in this order when no exact match:
```
byte → short → int → long → float → double
                char ↗
```

---

## 3.4 Runtime Polymorphism — The Real Power

This is where Java gets powerful. The JVM decides at **runtime** which method to call based on the **actual object type**, not the reference type.

```java
class Animal {
    public String sound() { return "..."; }
    public void describe() {
        System.out.println("I am " + getClass().getSimpleName()
                           + " and I say: " + sound());
    }
}

class Dog extends Animal {
    @Override public String sound() { return "Woof"; }
}

class Cat extends Animal {
    @Override public String sound() { return "Meow"; }
}

class Duck extends Animal {
    @Override public String sound() { return "Quack"; }
}
```

Now watch runtime polymorphism in action:

```java
// Reference type = Animal (parent)
// Actual object  = Dog / Cat / Duck (child)
Animal a1 = new Dog();
Animal a2 = new Cat();
Animal a3 = new Duck();

// Same call — different behavior — decided at RUNTIME
a1.describe();  // → "I am Dog and I say: Woof"
a2.describe();  // → "I am Cat and I say: Meow"
a3.describe();  // → "I am Duck and I say: Quack"

// Process ANY animal without knowing its type:
List<Animal> animals = List.of(new Dog(), new Cat(), new Duck());
for (Animal a : animals) {
    a.describe();  // ← JVM figures it out — you don't care what type it is
}
```

This is the **Open/Closed Principle** in action — you can add a new `Animal` subclass without changing the loop or the `describe()` method at all.

---

## 3.5 Upcasting and Downcasting

### Upcasting — Automatic, Always Safe

```java
Dog dog = new Dog("Rex", 3, "Labrador");
Animal animal = dog;   // ✅ Implicit upcast — no cast operator needed
                       // Dog IS-A Animal, so always safe

// What can you do with 'animal' reference?
animal.eat();          // ✅ inherited method
animal.sound();        // ✅ overridden — calls Dog's version at runtime
animal.bark();         // ❌ compile error — Animal ref doesn't know about bark()
```

When you upcast, you **lose access to child-specific methods** at the reference level — but the **actual object is still a Dog** in memory. The JVM still dispatches overridden methods correctly.

### Downcasting — Manual, Can Fail

```java
Animal animal = new Dog();  // upcast

// To access Dog-specific methods, downcast:
Dog dog = (Dog) animal;     // ✅ safe — actual object IS a Dog
dog.bark();                 // ✅ works

// DANGEROUS downcast:
Animal animal2 = new Cat();
Dog dog2 = (Dog) animal2;   // ❌ ClassCastException at RUNTIME
                             // Cat is NOT a Dog
```

### Safe Downcasting with `instanceof`

```java
// Classic way (Java 1–15)
if (animal instanceof Dog) {
    Dog dog = (Dog) animal;   // safe — checked first
    dog.bark();
}

// Modern way — Pattern Matching instanceof (Java 16+)
if (animal instanceof Dog dog) {   // ← declares variable in one step
    dog.bark();                    // 'dog' is scoped to this block
}

// Even cleaner with switch (Java 21 — Pattern Matching for switch)
String result = switch (animal) {
    case Dog d  -> d.breed + " says " + d.sound();
    case Cat c  -> "Cat says " + c.sound();
    case Duck k -> "Duck says " + k.sound();
    default     -> "Unknown animal";
};
```

---

## 3.6 Under the Hood — How JVM Does Dynamic Dispatch (vtable deep dive)

When `animal.sound()` is called where `animal` holds a `Dog`:

```
STEP 1 — At class loading time, JVM builds vtables:

Animal vtable:
┌─────────────┬──────────────────────┐
│ Slot 0      │ Animal.sound()       │
│ Slot 1      │ Animal.describe()    │
│ Slot 2      │ Object.toString()    │
│ Slot 3      │ Object.equals()      │
└─────────────┴──────────────────────┘

Dog vtable (inherits Animal's layout, replaces overridden slots):
┌─────────────┬──────────────────────┐
│ Slot 0      │ Dog.sound()     ← REPLACED (overridden)
│ Slot 1      │ Animal.describe()    ← SAME (not overridden)
│ Slot 2      │ Object.toString()    │
│ Slot 3      │ Object.equals()      │
└─────────────┴──────────────────────┘

STEP 2 — At call site: animal.sound()
  Compiler emits: invokevirtual #slot0

STEP 3 — At runtime:
  JVM reads object header → finds class pointer → Dog.class
  Dog.class vtable → slot 0 → Dog.sound()
  Executes Dog.sound()
```

**`invokevirtual`** is the JVM bytecode instruction for all instance method calls — it always checks the actual object's vtable at runtime. This is why overriding works transparently.

The other JVM method invocation bytecodes:

```
invokevirtual   → instance methods (polymorphic — uses vtable)
invokespecial   → private methods, constructors, super calls (non-polymorphic)
invokestatic    → static methods (non-polymorphic)
invokeinterface → interface methods (similar to invokevirtual but for interfaces)
invokedynamic   → lambdas, dynamic languages (Java 7+)
```

---

## 3.7 Polymorphism Through Interfaces — The Production Pattern

In real systems, polymorphism is used **primarily through interfaces**, not class hierarchies. This is because interfaces define contracts without forcing inheritance chains.

```java
// Define the contract
public interface PaymentGateway {
    PaymentResult charge(double amount, String currency, String token);
    boolean refund(String transactionId);
}

// Multiple implementations
public class StripeGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(double amount, String currency, String token) {
        // call Stripe API
        System.out.println("Charging via Stripe: " + amount + " " + currency);
        return new PaymentResult("stripe-txn-001", true);
    }

    @Override
    public boolean refund(String transactionId) {
        System.out.println("Refunding Stripe txn: " + transactionId);
        return true;
    }
}

public class RazorpayGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(double amount, String currency, String token) {
        System.out.println("Charging via Razorpay: " + amount + " " + currency);
        return new PaymentResult("rzp-txn-001", true);
    }

    @Override
    public boolean refund(String transactionId) {
        System.out.println("Refunding Razorpay txn: " + transactionId);
        return true;
    }
}

// Service uses the interface — doesn't care which gateway
public class OrderService {
    private final PaymentGateway paymentGateway;  // ← reference to interface

    // Injected at runtime — Spring does this via @Autowired
    public OrderService(PaymentGateway paymentGateway) {
        this.paymentGateway = paymentGateway;
    }

    public void placeOrder(Order order) {
        PaymentResult result = paymentGateway.charge(
            order.getTotal(), "INR", order.getPaymentToken()
        );
        if (!result.isSuccess())
            throw new PaymentFailedException("Payment failed for order: " + order.getId());
        // proceed with order
    }
}
```

In Spring Boot, you simply swap implementations via configuration:
```java
@Bean
public PaymentGateway paymentGateway() {
    return new RazorpayGateway();  // swap to Stripe with one line change
}
```

`OrderService` never changes. This is polymorphism solving real problems.

---

## 3.8 Polymorphism + Overloading Together — The Confusion Zone

This is a classic interview trap where overloading and overriding interact:

```java
class Animal {
    public void eat(Animal a) { System.out.println("Animal eats Animal"); }
}

class Dog extends Animal {
    public void eat(Animal a) { System.out.println("Dog eats Animal"); }  // OVERRIDE
    public void eat(Dog d)    { System.out.println("Dog eats Dog"); }     // OVERLOAD
}

Animal a = new Dog();
Dog d    = new Dog();
Animal ad = new Dog();

a.eat(d);    // → "Dog eats Animal"  ← runtime picks Dog.eat(Animal)
             //   - reference is Animal → only eat(Animal) is visible
             //   - at runtime, Dog.eat(Animal) is called (override)

d.eat(d);    // → "Dog eats Dog"     ← Dog ref, Dog arg
             //   - compile time picks eat(Dog) overload (exact match)

d.eat(ad);   // → "Dog eats Animal"  ← Dog ref, Animal ref
             //   - compile time picks eat(Animal) (reference type is Animal)
             //   - runtime doesn't re-evaluate overload — overloading is compile-time
```

**The key insight:**
- **Which overload** to call → decided at **compile time** by reference type
- **Which override** to execute → decided at **runtime** by actual object type

---

## 3.9 The Deadly `instanceof` Trap — Why Polymorphism Replaces It

If you find yourself writing this, polymorphism is broken:

```java
// ❌ Anti-pattern — every new Animal type breaks this
public void processSound(Animal animal) {
    if (animal instanceof Dog) {
        Dog dog = (Dog) animal;
        System.out.println("Dog says: " + dog.bark());
    } else if (animal instanceof Cat) {
        Cat cat = (Cat) animal;
        System.out.println("Cat says: " + cat.meow());
    } else if (animal instanceof Duck) {
        Duck duck = (Duck) animal;
        System.out.println("Duck says: " + duck.quack());
    }
    // Add Eagle? Must modify this method. Violates Open/Closed.
}

// ✅ Polymorphism eliminates this entirely
public void processSound(Animal animal) {
    System.out.println(animal.getClass().getSimpleName()
                       + " says: " + animal.sound());
    // Add Eagle? Just create Eagle extends Animal with sound(). Done.
}
```

`instanceof` chains are a code smell that signal missing polymorphism.

---

## 3.10 Polymorphism with Abstract Classes

Abstract classes enforce polymorphism at the design level — child classes **must** provide certain behavior:

```java
public abstract class Shape {
    private String color;

    public Shape(String color) { this.color = color; }

    // Abstract — every shape MUST implement this
    public abstract double area();
    public abstract double perimeter();

    // Concrete — shared behavior
    public void describe() {
        System.out.printf("%s %s: area=%.2f, perimeter=%.2f%n",
            color, getClass().getSimpleName(), area(), perimeter());
    }
}

public class Circle extends Shape {
    private double radius;

    public Circle(String color, double radius) {
        super(color);
        this.radius = radius;
    }

    @Override public double area()      { return Math.PI * radius * radius; }
    @Override public double perimeter() { return 2 * Math.PI * radius; }
}

public class Rectangle extends Shape {
    private double width, height;

    public Rectangle(String color, double width, double height) {
        super(color);
        this.width = width;
        this.height = height;
    }

    @Override public double area()      { return width * height; }
    @Override public double perimeter() { return 2 * (width + height); }
}

// Polymorphic usage:
List<Shape> shapes = List.of(
    new Circle("Red", 5),
    new Rectangle("Blue", 4, 6),
    new Circle("Green", 3)
);

shapes.forEach(Shape::describe);
// Red Circle:    area=78.54, perimeter=31.42
// Blue Rectangle: area=24.00, perimeter=20.00
// Green Circle:  area=28.27, perimeter=18.85

double totalArea = shapes.stream()
    .mapToDouble(Shape::area)
    .sum();
```

---

## 3.11 Covariant Return Types — Polymorphism on Return Values

```java
class Animal {
    public Animal getInstance() {
        return new Animal();
    }
}

class Dog extends Animal {
    @Override
    public Dog getInstance() {   // ✅ Dog IS-A Animal — valid covariant return
        return new Dog();
    }
}

Animal a = new Dog();
Animal result = a.getInstance();  // runtime calls Dog.getInstance(), returns Dog
                                  // stored as Animal ref — still polymorphic
```

This is used heavily in Builder patterns and fluent APIs where subclass methods return the subclass type.

---

## 3.12 Real-World Pattern — Strategy Pattern (Polymorphism in Action)

The Strategy pattern is pure polymorphism — swap algorithms at runtime:

```java
// In ShopSphere — different discount strategies
public interface DiscountStrategy {
    double apply(double originalPrice);
}

public class NoDiscount implements DiscountStrategy {
    @Override
    public double apply(double price) { return price; }
}

public class PercentageDiscount implements DiscountStrategy {
    private final double percent;
    public PercentageDiscount(double percent) { this.percent = percent; }

    @Override
    public double apply(double price) {
        return price * (1 - percent / 100);
    }
}

public class FlatDiscount implements DiscountStrategy {
    private final double flatAmount;
    public FlatDiscount(double flatAmount) { this.flatAmount = flatAmount; }

    @Override
    public double apply(double price) {
        return Math.max(0, price - flatAmount);
    }
}

// Cart uses DiscountStrategy — doesn't know which one
public class Cart {
    private DiscountStrategy strategy = new NoDiscount(); // default

    public void setDiscountStrategy(DiscountStrategy strategy) {
        this.strategy = strategy;
    }

    public double checkout(double total) {
        return strategy.apply(total);   // ← polymorphic call
    }
}

// Usage:
Cart cart = new Cart();
cart.checkout(1000);                                     // 1000.0

cart.setDiscountStrategy(new PercentageDiscount(10));
cart.checkout(1000);                                     // 900.0

cart.setDiscountStrategy(new FlatDiscount(150));
cart.checkout(1000);                                     // 850.0
```

Swapped behavior at runtime. Cart code never changed.

---

## 3.13 Polymorphism Limitations in Java

```
┌───────────────────────────────────┬────────────────────────────────────┐
│ What IS polymorphic               │ What is NOT polymorphic            │
├───────────────────────────────────┼────────────────────────────────────┤
│ Instance method calls             │ Static method calls (hidden)       │
│ Overridden methods                │ Private methods (not inherited)    │
│ Interface default method dispatch │ Field access (always parent's)     │
│ Abstract method implementations   │ Constructors                       │
│ Covariant return types            │ Overloaded methods (compile-time)  │
└───────────────────────────────────┴────────────────────────────────────┘
```

**Field access is NOT polymorphic — interview trap:**

```java
class Parent { String name = "Parent"; }
class Child extends Parent { String name = "Child"; }

Parent p = new Child();
System.out.println(p.name);  // → "Parent" ← field access uses reference type
                              //   NOT the actual object type!
```

Fields don't have vtable entries. Only methods do. Always access state through methods.

---

## 🎯 Chapter 3 — Interview Questions

**Q1. What is the difference between compile-time and runtime polymorphism?**
→ Compile-time: resolved by compiler via method overloading based on parameter signature. Runtime: resolved by JVM via method overriding based on actual object type using vtable dispatch (`invokevirtual`).

**Q2. What is the output?**
```java
class A { void show() { System.out.println("A"); } }
class B extends A { void show() { System.out.println("B"); } }
class C extends B { }

A obj = new C();
obj.show();
```
→ `"B"`. `C` doesn't override `show()`, so JVM walks up the vtable to `B.show()`.

**Q3. Can polymorphism be achieved without inheritance?**
→ Yes — through interfaces. A class implementing an interface is polymorphic with respect to that interface without extending any class.

**Q4. Why are fields not polymorphic in Java?**
→ Fields don't participate in vtable-based dynamic dispatch. Field access is resolved at compile time based on the reference type. Only method calls go through `invokevirtual`.

**Q5. What is the difference between overloading and overriding?**

```
┌──────────────────┬─────────────────────────┬──────────────────────────┐
│                  │ Overloading             │ Overriding               │
├──────────────────┼─────────────────────────┼──────────────────────────┤
│ Resolution       │ Compile time            │ Runtime                  │
│ Defined in       │ Same class              │ Parent + Child class      │
│ Signature        │ Must differ             │ Must be same             │
│ Return type      │ Can differ              │ Same or covariant        │
│ Polymorphism     │ Static                  │ Dynamic                  │
│ JVM bytecode     │ Best match at compile   │ invokevirtual + vtable   │
└──────────────────┴─────────────────────────┴──────────────────────────┘
```

**Q6. What is the output of overloading with type promotion?**
```java
void method(int i)    { System.out.println("int"); }
void method(Integer i){ System.out.println("Integer"); }

method(5);     // → "int"     (primitive preferred over autoboxing)
method(5L);    // → compile error — no long/Long overload
```
→ Java prefers widening over autoboxing over varargs in overload resolution order.

**Q7. How does the Strategy pattern use polymorphism?**
→ Defines a family of algorithms behind an interface, encapsulates each, and makes them interchangeable at runtime. The context holds an interface reference and calls through it — the JVM dispatches to the correct implementation.

---

## ✅ Chapter 3 Summary

```
Polymorphism
├── Compile-time (Static)
│     └── Method Overloading
│           ├── Resolved by compiler via parameter signature
│           ├── Type promotion: char→int, float→double etc.
│           └── Return type alone is NOT sufficient
│
├── Runtime (Dynamic)
│     └── Method Overriding
│           ├── Resolved by JVM via vtable (invokevirtual)
│           ├── Reference type → compile decides visibility
│           ├── Actual object type → runtime decides execution
│           └── Fields are NOT polymorphic — only methods
│
├── Upcasting — implicit, always safe
├── Downcasting — explicit, use instanceof first
│     └── Pattern matching instanceof (Java 16+)
│
├── Interface polymorphism — production standard
├── Abstract class — enforces polymorphic contracts
├── Covariant return types
├── Strategy pattern — swapping behavior at runtime
└── instanceof chains = code smell → replace with polymorphism
```

---

Ready for **Chapter 4 — Abstraction**? That's where we tie all 4 OOP pillars together — abstract classes vs interfaces, default methods, when to use which, and how Spring Boot uses abstraction everywhere. 🚀
