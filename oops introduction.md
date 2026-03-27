What is the class, and what is the object in Java?
=> A class in Java is a blueprint or template used to define the properties and behaviors of objects, including variables and methods. An object is an instance of a class that represents a real-world entity and occupies memory at runtime. Objects are created using the new keyword and store actual data, while the class defines the structure.

What is object creation process?
=> The object creation process in Java involves several steps. First, the class is loaded into memory by the ClassLoader if it is not already loaded. Then, memory is allocated in the heap for the object. The JVM initializes the object with default values, after which the constructor is executed to assign actual values. Finally, a reference to the object is created in the stack, pointing to the allocated memory in the heap.“This process ensures proper memory management and object initialization before usage.”


Does Java store objects in the heap and references in the stack?
=> In Java, objects are always stored in heap memory. References to those objects are stored based on the type of variable: local references are stored in the stack, instance variable references are stored within objects in the heap, and static references are stored in the method area. Therefore, while it is commonly said that objects are in the heap and references are in the stack, this is only true for local variables and not universally accurate.


🔴 Core Difference (Say This Clearly)
Aspect	Object	Reference
Meaning	Actual instance	Address of object
Memory	Heap	Depends (stack/heap/static)
Purpose	Stores data	Points to object
Example	new User()	User u

🔴 Critical Concept (Most Important)
User u1 = new User();
User u2 = u1;

👉 Both u1 and u2 point to same object

No new object created
Only reference copied

👉 Modify via one → affects same object

🔴 Null Case
User u = null;
Reference exists
No object in heap

👉 Calling method → NullPointerException




Difference between == and .equals()?
=-> In Java, the == operator compares references for objects and checks whether two references point to the same memory location, while for primitive types it compares actual values. The .equals() method is used to compare the content or logical equality of objects and can be overridden to define custom comparison logic. By default, .equals() behaves like ==, but classes like String override it to compare values instead of references.


# 🔹 Real Understanding (This is a classic trap)

## 🔗 `==` Operator


* Compares **references (memory addresses)** for objects
* Compares **values** for primitives

### Example:

```java
int a = 10, b = 10;
System.out.println(a == b); // true
```

```java
User u1 = new User();
User u2 = new User();
System.out.println(u1 == u2); // false
```

👉 Different objects → different addresses

---

## 📦 `.equals()` Method

* Used to compare **content (logical equality)**
* Defined in `Object` class
* Can be **overridden**

### Example:

```java
String s1 = new String("hello");
String s2 = new String("hello");

System.out.println(s1.equals(s2)); // true
```

👉 Same content → true

---

# 🔴 Critical Point (Most Important)

Default behavior:

```java
Object.equals() → same as ==
```

👉 Only becomes useful when **overridden** (e.g., String)

---

# 🔴 String Special Case (Interview Favorite)

```java
String s1 = "hello";
String s2 = "hello";

System.out.println(s1 == s2); // true (String pool)
```

👉 Same reference due to **String pool optimization**

---

# 🔴 Core Difference (Say This Clearly)

| Feature  | `==`                                     | `.equals()`      |
| -------- | ---------------------------------------- | ---------------- |
| Compares | Reference (objects) / value (primitives) | Content          |
| Override | No                                       | Yes              |
| Use case | Identity check                           | Logical equality |


# 🔥 Follow-up Traps (Be Ready)

* Why String overrides equals()?
* What happens if equals is not overridden?
* Difference between equals() and hashCode()?




Shallow vs deep copy
=> In Java, a shallow copy creates a new object but copies references of nested objects, meaning both objects share the same internal data. As a result, changes in one object can affect the other. A deep copy, on the other hand, creates a completely independent copy by duplicating all nested objects, ensuring that changes in one object do not impact the other. By default, methods like clone() perform shallow copying, and deep copy must be implemented manually.

What is the superclass (root class) of all classes in Java?
=> In Java, java.lang.Object is the root superclass of all classes. Every class implicitly extends Object if no other superclass is specified. It provides common methods such as toString, equals, hashCode, getClass, and thread coordination methods like wait and notify, enabling polymorphism and consistent behavior across all objects.


What is the equals() and hashCode() contract in Java? Explain with a real example.
In Java, the equals() and hashCode() methods follow a contract where if two objects are equal according to equals(), they must return the same hashCode(). However, two objects with the same hashCode() are not necessarily equal. These methods are especially important in hash-based collections like HashMap and HashSet, where hashCode() is used to locate a bucket and equals() is used to compare objects within that bucket. Therefore, both methods must be overridden together to ensure correct behavior.“Violating this contract can lead to incorrect behavior in collections, such as duplicate entries in HashSet or failed lookups in HashMap.”




🔹 Real Understanding (This is NOT optional)

You don’t “know Java” if you don’t understand this.

🧠 Why these exist
Used in hash-based collections:
HashMap
HashSet

👉 They decide:

Where object is stored
How equality is checked
🔴 The Contract (Memorize + Understand)
Rule 1:

👉 If equals() is true → hashCode() must be same

Rule 2:

👉 If hashCode() is same → equals() may or may not be true

Rule 3:

👉 If equals() is false → hashCode() can be same or different

🔴 Why this matters
4
How HashMap works:
Uses hashCode() → find bucket
Uses equals() → check exact match

👉 If contract breaks → collections behave incorrectly

🔴 Real Problem (If You Don’t Override)
class User {
    String name;
}
User u1 = new User("A");
User u2 = new User("A");

System.out.println(u1.equals(u2)); // false ❌

👉 Because default equals() compares reference

🔹 Correct Implementation (Real Example)
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
✅ Now:
User u1 = new User("A");
User u2 = new User("A");

System.out.println(u1.equals(u2)); // true ✅
🔴 What Happens in HashSet
Set<User> set = new HashSet<>();
set.add(u1);
set.add(u2);

👉 Without override:

Both added → duplicate ❌

👉 With correct override:

Only one stored ✅
=================================================
















