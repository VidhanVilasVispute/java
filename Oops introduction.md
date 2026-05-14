
# Java OOP Fundamentals — Interview & Study Notes

---

## 1. What Is a Class and What Is an Object?

A **class** is a blueprint or template that defines the properties (variables) and behaviors (methods) of objects. An **object** is an instance of a class that represents a real-world entity and occupies memory at runtime. Objects are created using the `new` keyword and store actual data; the class only defines the structure.

---

## 2. Object Creation Process

Object creation in Java involves the following steps in order:

1. The **ClassLoader** loads the class into memory if not already loaded.
2. **Memory is allocated** in the heap for the new object.
3. The JVM **initializes the object with default values** (0, null, false).
4. The **constructor executes** to assign actual values.
5. A **reference is created** in the stack, pointing to the heap object.

This ensures proper memory management and initialization before the object is used.

---

## 3. Where Are Objects and References Stored?

Objects are always stored in **heap memory**. Where a reference is stored depends on the type of variable holding it:

| Variable Type | Reference Stored In |
|---------------|---------------------|
| Local variable | Stack |
| Instance variable | Heap (inside the containing object) |
| Static variable | Method Area (Metaspace) |

The common statement "objects are in the heap and references are in the stack" is only accurate for local variables, not universally.

### Key Distinction

| Aspect | Object | Reference |
|--------|--------|-----------|
| Meaning | Actual instance | Address of the object |
| Memory | Heap | Depends on variable type |
| Purpose | Stores data | Points to object |
| Example | `new User()` | `User u` |

### Multiple References to the Same Object

```java
User u1 = new User();
User u2 = u1;
```

Both `u1` and `u2` point to the **same object** in the heap. No new object is created — only the reference is copied. Modifying the object via either reference affects the same underlying object.

### Null Reference

```java
User u = null;
```

The reference variable exists but points to no object. Calling any method on it throws a `NullPointerException`.

---

## 4. `==` vs `.equals()`

The `==` operator compares **references** for objects (i.e., whether two variables point to the same memory address) and compares **actual values** for primitives. The `.equals()` method compares **content or logical equality** and can be overridden to define custom comparison logic. By default, `.equals()` behaves identically to `==`, but classes like `String` override it to compare values.

| Feature | `==` | `.equals()` |
|---------|------|-------------|
| Compares | Reference (objects) / value (primitives) | Content |
| Overridable | No | Yes |
| Use case | Identity check | Logical equality |

**Primitive comparison:**
```java
int a = 10, b = 10;
System.out.println(a == b); // true
```

**Object reference comparison:**
```java
User u1 = new User();
User u2 = new User();
System.out.println(u1 == u2); // false — different objects, different addresses
```

**Content comparison with `.equals()`:**
```java
String s1 = new String("hello");
String s2 = new String("hello");
System.out.println(s1.equals(s2)); // true — same content
```

**String pool special case:**
```java
String s1 = "hello";
String s2 = "hello";
System.out.println(s1 == s2); // true — same reference due to String pool
```

When string literals are used (not `new String(...)`), Java reuses the same object from the **String pool**, so `==` returns true even though it is a reference check.

---

## 5. Shallow Copy vs Deep Copy

A **shallow copy** creates a new object but copies references of nested objects. Both the original and the copy share the same internal objects, so a change made through one can affect the other.

A **deep copy** creates a completely independent object by duplicating all nested objects as well. Changes in one object do not affect the other.

By default, `clone()` performs a **shallow copy**. A deep copy must be implemented manually.

---

## 6. Root Superclass of All Classes

`java.lang.Object` is the root superclass of all classes in Java. Every class implicitly extends `Object` unless it explicitly extends another class. It provides common methods available to all objects:

- `toString()` — string representation
- `equals()` — logical equality
- `hashCode()` — hash value for collections
- `getClass()` — runtime class information
- `wait()`, `notify()`, `notifyAll()` — thread coordination

---

## 7. `equals()` and `hashCode()` Contract

These two methods follow a strict contract that is critical for correct behavior in hash-based collections like `HashMap` and `HashSet`.

**The contract rules:**

- If `equals()` returns true for two objects, they **must** return the same `hashCode()`.
- If two objects have the same `hashCode()`, they are **not necessarily** equal.
- If `equals()` returns false, the `hashCode()` may be the same or different.

**How `HashMap` uses both:**

1. Uses `hashCode()` to find the correct bucket.
2. Uses `equals()` to find the exact match within that bucket.

If the contract is violated, collections behave incorrectly — for example, duplicate entries in a `HashSet` or failed lookups in a `HashMap`.

### Problem When Neither Is Overridden

```java
class User {
    String name;
}

User u1 = new User("A");
User u2 = new User("A");

System.out.println(u1.equals(u2)); // false — default equals() compares reference
```

### Correct Implementation

```java
class User {
    String name;

    User(String name) {
        this.name = name;
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof User)) return false;
        User u = (User) o;
        return name.equals(u.name);
    }

    @Override
    public int hashCode() {
        return name.hashCode();
    }
}
```

### Effect on `HashSet`

```java
Set<User> set = new HashSet<>();
set.add(u1);
set.add(u2);
```

Without overriding: both objects are added (duplicate entry).
With correct override: only one object is stored.

**Always override `equals()` and `hashCode()` together. Never override just one.**

---

## 8. Complete Example — Class, Object, and References

```java
// Class = Blueprint
class User {

    // Fields — state of the object
    int id;
    String name;

    // Constructor — runs during object creation
    User(int id, String name) {
        this.id = id;
        this.name = name;
    }

    // Method — behavior of the object
    void display() {
        System.out.println("Id: " + id);
        System.out.println("Name: " + name);
    }
}

public class Main {
    public static void main(String[] args) {

        // Step 1: Declaration (reference variable created on stack)
        User u1;

        // Step 2: Instantiation (object created in heap)
        u1 = new User(1, "John");

        // Combined form (recommended)
        User u2 = new User(2, "Alice");

        // Calling methods on objects
        u1.display();
        u2.display();

        // Multiple references pointing to the same object
        User u3 = u1;
        u3.name = "Changed Name";
        u1.display(); // Prints "Changed Name" — same object affected

        // Null reference
        User u4 = null;
        // u4.display(); → NullPointerException
    }
}
```

---

*End of Notes*
