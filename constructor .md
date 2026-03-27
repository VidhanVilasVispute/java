What is a constructor in Java?
=> A constructor in Java is a special method used to initialize an object when it is created. It has the same name as the class and does not have a return type. Constructors are automatically invoked during object creation and are used to set initial values for instance variables. Java supports default and parameterized constructors, and they can also be overloaded.

---

## Constructor Overloading – Java Example

```java
class User {

    String name;
    int age;

    // Default constructor
    User() {
        this.name = "Unknown";
        this.age = 0;
    }

    // Parameterized constructor
    User(String name) {
        this.name = name;
        this.age = 0;
    }

    // Another parameterized constructor
    User(String name, int age) {
        this.name = name;
        this.age = age;
    }
}
```

### What this shows

* Same constructor name (`User`)
* Different parameter lists
* Chosen at **object creation time**
* Used to initialize object state

Usage:

```java
User u1 = new User();
User u2 = new User("Vidhan");
User u3 = new User("Vidhan", 25);
```

---

## Method Overloading – Java Example

```java
class Calculator {

    int add(int a, int b) {
        return a + b;
    }

    double add(double a, double b) {
        return a + b;
    }

    int add(int a, int b, int c) {
        return a + b + c;
    }
}
```

### What this shows

* Same method name (`add`)
* Different parameter lists
* Resolved at **compile time**
* Used for behavior, not initialization

Usage:

```java
Calculator calc = new Calculator();

calc.add(2, 3);
calc.add(2.5, 3.5);
calc.add(1, 2, 3);
```

---

## Key Differences (this is what interviewers care about)

| Constructor Overloading | Method Overloading |
| ----------------------- | ------------------ |
| Initializes object      | Performs operation |
| Called during `new`     | Called explicitly  |
| No return type          | Has return type    |
| Same as class name      | Any valid name     |



What is the difference between constructor and method?
=> A constructor is a special class member used to initialize objects and is invoked automatically during object creation using new. It has the same name as the class and no return type. A method defines behavior that operates on an already created object, has a custom name, a return type, and is invoked explicitly using an object or class reference.


Is constructor inherited?
=> No, constructors are not inherited in Java because inheritance applies to methods and fields, not object initialization. However, superclass constructors are executed during object creation through constructor chaining using super().

What is the difference between constructor and method?
=> Constructor chaining allows one constructor to call another using this() within the same class or super() to invoke a superclass constructor. This ensures that object initialization happens in a proper order, avoids code duplication, and guarantees that parent class state is initialized before child class state.



