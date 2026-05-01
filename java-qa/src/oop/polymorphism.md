# Q: Explain Polymorphism. What is the difference between Method Overloading and Overriding?

**Answer:**

**Polymorphism** = "many forms." It allows a single interface to represent different underlying types.

### Compile-Time Polymorphism (Method Overloading)
Same method name, **different parameter lists** in the same class. Resolved at **compile time** by the compiler based on the method signature.

```java
public class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; }
    public int add(int a, int b, int c) { return a + b + c; }
}
```

**Rules:**
- Must differ in parameter count, type, or order.
- Return type alone is NOT sufficient to overload.
- Access modifiers can differ.

### Runtime Polymorphism (Method Overriding)
Subclass provides a **specific implementation** of a method already defined in its parent class. Resolved at **runtime** via dynamic dispatch based on the actual object type.

```java
public class Shape {
    public double area() { return 0; }
}

public class Circle extends Shape {
    private double radius;

    @Override
    public double area() { return Math.PI * radius * radius; }
}

public class Rectangle extends Shape {
    private double width, height;

    @Override
    public double area() { return width * height; }
}

// Runtime polymorphism in action:
Shape shape = new Circle(5);   // Reference type: Shape, Object type: Circle
shape.area();                   // Calls Circle.area() — resolved at RUNTIME
```

**Rules:**
- Same method signature (name + parameters).
- Return type must be the same or a **covariant** (subclass) return type.
- Access modifier must be the same or **less restrictive**.
- Cannot override `static`, `final`, or `private` methods.
- Must use `@Override` annotation (not required but strongly recommended).

### Overloading vs Overriding

| Feature | Overloading | Overriding |
|---|---|---|
| **Where** | Same class | Subclass |
| **Method name** | Same | Same |
| **Parameters** | Must differ | Must be identical |
| **Return type** | Can differ | Same or covariant |
| **Resolved at** | Compile time | Runtime |
| **Polymorphism type** | Static | Dynamic |
| **`@Override`** | N/A | Yes |

> [!IMPORTANT]
> The most common interview trick question: "Can you override a static method?" **No.** Static methods belong to the class, not the instance. You can *hide* a static method (by defining one with the same signature in a subclass), but this is **method hiding**, not overriding — there's no dynamic dispatch.
