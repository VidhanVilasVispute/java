# 📘 Chapter 4 — Abstraction

## 4.1 What Is Abstraction? (The Real Definition)

Most people say *"abstraction = hiding implementation using abstract classes and interfaces."*
That is **incomplete**. Here's the real definition:

> **Abstraction** is the process of defining a contract that exposes only essential behavior while hiding implementation details. It is achieved using abstract classes and interfaces, and it helps in building flexible, loosely coupled, and maintainable systems.

The keyword is **what vs how**. The caller only needs to know the contract. The implementation is irrelevant to them.

Real-world analogy:
```
You press the brake pedal in a car.
  WHAT  → car slows down
  HOW   → hydraulic fluid, brake pads, ABS, friction — you don't care

You call paymentGateway.charge(amount)
  WHAT  → money is charged
  HOW   → Stripe API, Razorpay SDK, retry logic — caller doesn't care
```

---

## 4.2 The Problem It Solves

Without abstraction, callers are **tightly coupled** to implementation:

```java
// ❌ No abstraction — caller knows everything
public class OrderService {

    public void placeOrder(Order order) {
        // Directly knows it's MySQL, knows the query, knows the connection
        Connection conn = DriverManager.getConnection(
            "jdbc:mysql://localhost:3306/shopsphere", "root", "password"
        );
        PreparedStatement stmt = conn.prepareStatement(
            "INSERT INTO orders (customer_id, total) VALUES (?, ?)"
        );
        stmt.setString(1, order.getCustomerId());
        stmt.setDouble(2, order.getTotal());
        stmt.executeUpdate();
    }
}
```

Now if you switch from MySQL to PostgreSQL, or want to unit test this — you must **rewrite `OrderService`** entirely. The caller is glued to the implementation.

With abstraction:

```java
// ✅ OrderService knows nothing about storage details
public class OrderService {
    private final OrderRepository repository;  // ← abstraction

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

    public void placeOrder(Order order) {
        repository.save(order);  // ← just the "what". Never the "how".
    }
}
```

Switch MySQL → PostgreSQL → In-memory → nothing in `OrderService` changes.

---

## 4.3 Two Mechanisms of Abstraction in Java

```
Abstraction in Java
├── Abstract Classes  (0% to 100% abstraction)
│     └── Can have abstract + concrete methods
│         Can have state (fields)
│         Single inheritance only
│
└── Interfaces        (100% abstraction by contract)
      └── Pure contract — only what, never how
          Multiple implementation allowed
          Can have default/static methods (Java 8+)
```

---

## 4.4 Abstract Classes — Deep Dive

### Syntax and Rules

```java
public abstract class Vehicle {              // abstract class — cannot be instantiated

    // STATE — abstract classes CAN have fields
    protected String brand;
    protected int year;
    private int mileage = 0;                 // private state — fully encapsulated

    // CONSTRUCTOR — abstract classes CAN have constructors
    // (called via super() from child)
    public Vehicle(String brand, int year) {
        this.brand = brand;
        this.year = year;
    }

    // ABSTRACT METHOD — no body, must be overridden by child
    public abstract double fuelEfficiency();  // every vehicle calculates differently
    public abstract String fuelType();

    // CONCRETE METHOD — shared behavior, no override needed
    public void drive(int km) {
        mileage += km;
        System.out.printf("%s drove %d km. Total: %d km%n", brand, km, mileage);
    }

    // CONCRETE METHOD using abstract method — Template Method pattern
    public void printStats() {
        System.out.printf(
            "%s (%d) | Fuel: %s | Efficiency: %.2f km/l | Mileage: %d km%n",
            brand, year, fuelType(), fuelEfficiency(), mileage
        );
    }

    public int getMileage() { return mileage; }
}
```

```java
public class ElectricCar extends Vehicle {
    private double batteryCapacityKwh;
    private double rangeKm;

    public ElectricCar(String brand, int year,
                       double batteryCapacityKwh, double rangeKm) {
        super(brand, year);
        this.batteryCapacityKwh = batteryCapacityKwh;
        this.rangeKm = rangeKm;
    }

    @Override
    public double fuelEfficiency() {
        return rangeKm / batteryCapacityKwh;  // km per kWh
    }

    @Override
    public String fuelType() { return "Electric"; }
}

public class PetrolCar extends Vehicle {
    private double tankLitres;
    private double rangeKm;

    public PetrolCar(String brand, int year, double tankLitres, double rangeKm) {
        super(brand, year);
        this.tankLitres = tankLitres;
        this.rangeKm = rangeKm;
    }

    @Override
    public double fuelEfficiency() {
        return rangeKm / tankLitres;  // km per litre
    }

    @Override
    public String fuelType() { return "Petrol"; }
}
```

```java
Vehicle v1 = new ElectricCar("Tesla", 2023, 100, 600);
Vehicle v2 = new PetrolCar("Toyota", 2022, 50, 750);

v1.drive(120);
v2.drive(200);
v1.printStats();   // Tesla (2023) | Fuel: Electric | Efficiency: 6.00 km/l | Mileage: 120 km
v2.printStats();   // Toyota (2022) | Fuel: Petrol | Efficiency: 15.00 km/l | Mileage: 200 km

// Vehicle v = new Vehicle(); ← ❌ compile error — cannot instantiate abstract class
```

---

### Abstract Class Rules — Complete Reference

```
┌──────────────────────────────────────┬────────────┐
│ Feature                              │ Allowed?   │
├──────────────────────────────────────┼────────────┤
│ Can have abstract methods            │ ✅         │
│ Can have concrete methods            │ ✅         │
│ Can have fields (state)              │ ✅         │
│ Can have constructors                │ ✅         │
│ Can have static methods              │ ✅         │
│ Can have static fields               │ ✅         │
│ Can have private methods (Java 9+)   │ ✅         │
│ Can be instantiated directly         │ ❌         │
│ Can extend multiple abstract classes │ ❌         │
│ Must override all abstract methods   │ ✅         │
│   (unless child is also abstract)    │            │
└──────────────────────────────────────┴────────────┘
```

---

## 4.5 Interfaces — Deep Dive

### Basic Interface

```java
public interface Printable {
    void print();          // implicitly: public abstract void print()
    void preview();        // implicitly: public abstract void preview()
}

public interface Exportable {
    void exportToPdf();
    void exportToCsv();
}

// A class can implement MULTIPLE interfaces — solving multiple inheritance
public class Report implements Printable, Exportable {
    private String content;

    public Report(String content) { this.content = content; }

    @Override public void print()      { System.out.println("Printing: " + content); }
    @Override public void preview()    { System.out.println("Preview: " + content); }
    @Override public void exportToPdf(){ System.out.println("PDF: " + content); }
    @Override public void exportToCsv(){ System.out.println("CSV: " + content); }
}
```

---

### Java 8+ Interface Evolution — This Is Where It Gets Deep

Before Java 8, interfaces were **pure contracts** — no implementation at all. Java 8 added three new features:

#### `default` Methods — Backward-Compatible Implementation

```java
public interface Collection<E> {  // simplified

    boolean add(E e);
    boolean remove(Object o);
    int size();

    // default — implementation provided so existing classes
    // don't break when this method is added to the interface
    default boolean isEmpty() {
        return size() == 0;     // uses the abstract method
    }

    default void forEach(Consumer<? super E> action) {
        for (E e : this) {      // uses iterator()
            action.accept(e);
        }
    }
}
```

**Why `default` was added:** Before Java 8, adding a new method to an interface would break every class implementing it. `default` allows the interface to evolve without breaking existing implementations.

```java
public interface Auditable {
    String getCreatedBy();
    Instant getCreatedAt();

    // Default — implementing classes get this for free
    // but can override if they want custom formatting
    default String auditSummary() {
        return "Created by " + getCreatedBy()
               + " at " + getCreatedAt();
    }
}

public class Order implements Auditable {
    private String createdBy;
    private Instant createdAt;

    @Override public String getCreatedBy() { return createdBy; }
    @Override public Instant getCreatedAt() { return createdAt; }

    // auditSummary() inherited for free — no override needed
}
```

---

#### `static` Methods in Interfaces

```java
public interface Validator<T> {
    boolean validate(T value);

    // static utility — belongs to the interface, not any instance
    static Validator<String> nonNull() {
        return value -> value != null && !value.isBlank();
    }

    static Validator<Integer> positive() {
        return value -> value != null && value > 0;
    }
}

// Usage:
Validator<String> v = Validator.nonNull();
v.validate("hello");   // true
v.validate("");        // false
```

---

#### `private` Methods (Java 9+) — DRY Inside Interfaces

```java
public interface Logger {
    void logInfo(String message);
    void logError(String message);

    default void logWithTimestamp(String level, String message) {
        System.out.println(format(level, message));  // reuses private helper
    }

    // private — shared helper for default methods only
    // not visible to implementing classes at all
    private String format(String level, String message) {
        return "[" + Instant.now() + "] [" + level + "] " + message;
    }
}
```

---

### Interface Fields — The Hidden Trap

```java
public interface Constants {
    int MAX_RETRIES = 3;          // implicitly: public static final int MAX_RETRIES = 3
    String DEFAULT_CURRENCY = "INR"; // implicitly: public static final
}
```

Every field in an interface is **automatically** `public static final`. You cannot have instance fields in an interface. This is why interfaces cannot hold mutable state.

---

## 4.6 Abstract Class vs Interface — The Complete Decision Guide

This is the most important comparison in Java OOP:

```
┌────────────────────────────┬──────────────────────┬──────────────────────┐
│ Feature                    │ Abstract Class       │ Interface            │
├────────────────────────────┼──────────────────────┼──────────────────────┤
│ Instantiation              │ ❌                   │ ❌                   │
│ Fields (state)             │ ✅ Any type          │ ✅ Only static final │
│ Constructors               │ ✅                   │ ❌                   │
│ Abstract methods           │ ✅                   │ ✅ (implicit)        │
│ Concrete methods           │ ✅                   │ ✅ via default       │
│ static methods             │ ✅                   │ ✅ (Java 8+)         │
│ private methods            │ ✅                   │ ✅ (Java 9+)         │
│ Access modifiers           │ Any                  │ public only          │
│ Multiple inheritance       │ ❌ (single extends)  │ ✅ (multi implements)│
│ IS-A relationship          │ ✅ Strong            │ ✅ Capability-based  │
└────────────────────────────┴──────────────────────┴──────────────────────┘
```

### Decision Flowchart

```
Do you need to share STATE (fields) across implementations?
    YES → Abstract Class

Do you need a constructor to enforce initial state?
    YES → Abstract Class

Is this a capability/contract that unrelated classes should share?
    YES → Interface  (Serializable, Comparable, Runnable)

Do you need multiple inheritance?
    YES → Interface

Do you have a strong IS-A relationship with shared behavior?
    YES → Abstract Class

Are you defining an API contract for external consumers?
    YES → Interface

Can't decide? → Default to Interface.
               It gives more flexibility.
               You can always extract to abstract class later.
```

---

## 4.7 The Template Method Pattern — Abstract Class's Killer Use Case

This is where abstract classes truly shine. Define the **skeleton of an algorithm** in the parent, let children fill in the steps:

```java
// In ShopSphere — order processing has a fixed flow
// but each order type processes payment differently
public abstract class OrderProcessor {

    // TEMPLATE METHOD — defines the algorithm skeleton
    // final — children cannot reorder the steps
    public final void processOrder(Order order) {
        validateOrder(order);           // Step 1 — same for all
        reserveInventory(order);        // Step 2 — same for all
        processPayment(order);          // Step 3 — DIFFERENT per type ← abstract
        sendConfirmation(order);        // Step 4 — same for all
        updateAnalytics(order);         // Step 5 — optional hook
    }

    // Concrete steps — shared logic
    private void validateOrder(Order order) {
        if (order.getItems().isEmpty())
            throw new IllegalStateException("Order has no items");
        System.out.println("Order validated: " + order.getId());
    }

    private void reserveInventory(Order order) {
        System.out.println("Inventory reserved for: " + order.getId());
    }

    private void sendConfirmation(Order order) {
        System.out.println("Confirmation sent for: " + order.getId());
    }

    // Abstract step — each subclass defines HOW
    protected abstract void processPayment(Order order);

    // Hook method — optional override, does nothing by default
    protected void updateAnalytics(Order order) { }
}

// Stripe implementation
public class StripeOrderProcessor extends OrderProcessor {
    @Override
    protected void processPayment(Order order) {
        System.out.println("Processing payment via Stripe: ₹" + order.getTotal());
        // call Stripe SDK
    }
}

// COD implementation
public class CodOrderProcessor extends OrderProcessor {
    @Override
    protected void processPayment(Order order) {
        System.out.println("COD order — payment on delivery: ₹" + order.getTotal());
        // mark as pending payment
    }

    @Override
    protected void updateAnalytics(Order order) {   // overrides the hook
        System.out.println("COD analytics updated for: " + order.getId());
    }
}

// Usage — same call, different behavior
OrderProcessor processor = new StripeOrderProcessor();
processor.processOrder(order);

processor = new CodOrderProcessor();
processor.processOrder(order);
```

The flow is **fixed and final**. Only the variable step is abstract. This prevents child classes from breaking the algorithm structure.

---

## 4.8 Diamond Problem with Default Methods — And How Java Resolves It

```java
interface A {
    default void greet() { System.out.println("Hello from A"); }
}

interface B extends A {
    default void greet() { System.out.println("Hello from B"); }
}

interface C extends A {
    default void greet() { System.out.println("Hello from C"); }
}

// ❌ Ambiguous — B and C both provide greet()
class D implements B, C {
    // MUST override to resolve ambiguity — compiler forces this
    @Override
    public void greet() {
        B.super.greet();  // ← explicitly choose which parent's default
        // or write your own implementation entirely
    }
}
```

**Java's resolution rules (in order):**
```
1. Class wins over interface always
   (if D has its own greet(), it wins over all defaults)

2. More specific interface wins
   (if B extends A and both define default greet(),
    B's version wins over A's)

3. If still ambiguous → compiler error, must override explicitly
```

---

## 4.9 Under the Hood — JVM and Interface Dispatch

For interfaces, JVM uses `invokeinterface` instead of `invokevirtual`:

```
invokevirtual  → class method dispatch (vtable — O(1) lookup)
invokeinterface → interface method dispatch (itable — slightly more complex)
```

**Why a different instruction?**

A class has a single vtable with a fixed slot layout. But a class can implement multiple interfaces, and each interface defines its own slot numbering. So the JVM maintains an **itable (interface table)** — a secondary lookup structure per class that maps interface method references to the correct implementation.

```
Dog class memory layout:
┌──────────────────────────────┐
│ vtable (class methods)       │
│   slot 0 → Dog.sound()       │
│   slot 1 → Animal.eat()      │
├──────────────────────────────┤
│ itable (interface methods)   │
│   Swimmable::swim → Dog.swim()    │
│   Trainable::train → Dog.train()  │
└──────────────────────────────┘
```

`invokeinterface` does a search through the itable — historically slightly slower than `invokevirtual`, though modern JVMs optimize this heavily with inline caching.

---

## 4.10 Functional Interfaces — Abstraction Meets Lambdas

A **functional interface** has exactly **one abstract method**. This makes it the bridge between abstraction and lambda expressions:

```java
@FunctionalInterface            // annotation — compiler enforces single abstract method
public interface Transformer<T, R> {
    R transform(T input);       // single abstract method

    // default and static methods are fine — don't count
    default Transformer<T, R> andLog() {
        return input -> {
            R result = this.transform(input);
            System.out.println("Transformed: " + input + " → " + result);
            return result;
        };
    }
}

// Usage with lambda — lambda IS an implementation of Transformer
Transformer<String, Integer> lengthFinder = str -> str.length();
lengthFinder.transform("ShopSphere");  // → 10

// With andLog()
Transformer<String, String> upper = String::toUpperCase;
upper.andLog().transform("hello");
// Transformed: hello → HELLO
```

Java's built-in functional interfaces you use everywhere:

```
┌─────────────────────┬──────────────────┬───────────────────────────────┐
│ Interface           │ Method           │ Use case                      │
├─────────────────────┼──────────────────┼───────────────────────────────┤
│ Runnable            │ run()            │ Thread task, no args/return   │
│ Callable<V>         │ call()           │ Thread task with return value  │
│ Supplier<T>         │ get()            │ Produce a value               │
│ Consumer<T>         │ accept(T)        │ Consume a value, no return    │
│ Function<T,R>       │ apply(T)         │ Transform T → R               │
│ Predicate<T>        │ test(T)          │ Test condition → boolean      │
│ BiFunction<T,U,R>   │ apply(T,U)       │ Two inputs → one output       │
│ UnaryOperator<T>    │ apply(T)         │ T → T (same type)             │
│ Comparator<T>       │ compare(T,T)     │ Ordering                      │
└─────────────────────┴──────────────────┴───────────────────────────────┘
```

---

## 4.11 Abstraction in Spring Boot — How the Framework Uses It

Spring Boot is **built on abstraction**. Here's how it appears in your ShopSphere project:

```java
// 1. Repository abstraction — you never see SQL
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByCustomerId(String customerId);
    // Spring generates the implementation at runtime via proxy
}

// 2. Service layer — abstracts business logic from controller
public interface OrderService {
    OrderResponse placeOrder(PlaceOrderRequest request);
    OrderResponse getOrder(Long orderId);
    void cancelOrder(Long orderId);
}

@Service
public class OrderServiceImpl implements OrderService {
    // actual implementation hidden behind the interface
}

// 3. Controller sees only the interface
@RestController
public class OrderController {
    private final OrderService orderService;  // ← interface, not impl

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> place(@RequestBody PlaceOrderRequest req) {
        return ResponseEntity.ok(orderService.placeOrder(req));
    }
}

// 4. Messaging abstraction
public interface EventPublisher {
    void publish(String topic, Object event);
}

@Component
public class KafkaEventPublisher implements EventPublisher {
    private final KafkaTemplate<String, Object> kafkaTemplate;
    // kafka-specific details hidden from callers
}

// 5. Swapping implementations for testing — abstraction pays off
@TestConfiguration
public class TestConfig {
    @Bean
    public EventPublisher eventPublisher() {
        return (topic, event) -> System.out.println("Test event: " + event);
        // Lambda implements interface — no Kafka needed in tests
    }
}
```

Every layer in Spring Boot talks to the **abstraction above it**, never the implementation below it. This is the **Dependency Inversion Principle** — one of SOLID's core rules.



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

## 4.12 Tying All 4 OOP Pillars Together

Here's a single example that uses all four pillars:

```java
// ABSTRACTION — defines the contract (what)
public interface Notification {
    void send(String recipient, String message);

    default void sendBulk(List<String> recipients, String message) {
        recipients.forEach(r -> send(r, message));
    }
}

// ABSTRACTION + ENCAPSULATION — abstract class with shared private state
public abstract class BaseNotification implements Notification {
    private final List<String> sentLog = new ArrayList<>();  // ENCAPSULATION

    @Override
    public final void send(String recipient, String message) {
        String result = deliver(recipient, message);         // ABSTRACTION
        sentLog.add(recipient + ": " + result);             // protected state
    }

    // Abstract — child fills in HOW delivery happens
    protected abstract String deliver(String recipient, String message);

    public List<String> getSentLog() {
        return Collections.unmodifiableList(sentLog);        // ENCAPSULATION
    }
}

// INHERITANCE — reuses BaseNotification, fills in delivery
public class EmailNotification extends BaseNotification {
    private final String smtpServer;

    public EmailNotification(String smtpServer) {
        this.smtpServer = smtpServer;
    }

    @Override
    protected String deliver(String recipient, String message) {
        System.out.printf("Email via %s → %s: %s%n", smtpServer, recipient, message);
        return "EMAIL_SENT";
    }
}

public class SmsNotification extends BaseNotification {
    @Override
    protected String deliver(String recipient, String message) {
        System.out.printf("SMS → %s: %s%n", recipient, message);
        return "SMS_SENT";
    }
}

// POLYMORPHISM — caller doesn't care which implementation it is
public class NotificationService {
    private final Notification notification;  // ← abstract type

    public NotificationService(Notification notification) {
        this.notification = notification;
    }

    public void notifyAll(List<String> users, String msg) {
        notification.sendBulk(users, msg);    // ← polymorphic dispatch
    }
}

// Runtime wiring
NotificationService svc = new NotificationService(new EmailNotification("smtp.gmail.com"));
svc.notifyAll(List.of("vidhan@email.com", "admin@email.com"), "Order shipped!");
```

---

## 🎯 Chapter 4 — Interview Questions

**Q1. What is the difference between abstraction and encapsulation?**
→ Encapsulation hides **data** — protects internal state from direct access. Abstraction hides **implementation** — exposes only what an object does, not how. Encapsulation is about fields and access control. Abstraction is about method contracts and polymorphism.

**Q2. When do you choose an abstract class over an interface?**
→ Abstract class when: you need shared state (fields), you need constructors, you need non-public methods, or you have a strong IS-A with shared concrete behavior. Interface when: you need multiple inheritance, you're defining a capability/contract for unrelated classes, or you're designing an API for external consumers.

**Q3. Can an interface extend another interface?**
→ Yes. An interface can extend multiple interfaces. `interface C extends A, B {}` — this is valid. When a class implements C, it must implement all abstract methods from A, B, and C.

**Q4. Can an abstract class implement an interface?**
→ Yes — and it's not required to implement all interface methods. The unimplemented methods remain abstract and must be implemented by the first concrete subclass.
```java
abstract class Base implements Runnable {
    // doesn't implement run() — remains abstract
    // first concrete subclass must implement run()
}
```

**Q5. What happens if two interfaces have the same default method and a class implements both?**
→ Compile error. The class must explicitly override the conflicting method and optionally delegate with `InterfaceName.super.method()`.

**Q6. Can you instantiate an abstract class using an anonymous class?**
→ Yes — this is actually very common:
```java
// Anonymous class provides the missing abstract methods on the spot
Notification n = new BaseNotification() {
    @Override
    protected String deliver(String r, String m) {
        return "PUSH_SENT";
    }
};
n.send("user@app.com", "Hello!");
```
You're not instantiating the abstract class — you're creating a nameless subclass of it inline.

**Q7. What is a marker interface?**
→ An interface with no methods at all — used to mark a class for special treatment by the JVM or framework.
```java
public interface Serializable { }  // no methods
public interface Cloneable { }     // no methods
```
The JVM checks `instanceof Serializable` before serializing. With Java annotations, most marker interfaces have been replaced (e.g., `@FunctionalInterface`, `@Entity`).

