# Q: What is the difference between Abstract Class and Interface?

**Answer:**

This distinction has evolved significantly across Java versions. The modern answer is more nuanced than the textbook version.

### Abstract Class
A class that **cannot be instantiated** and may contain both abstract (unimplemented) and concrete (implemented) methods. Represents an **"is-a"** relationship.

```java
public abstract class Animal {
    protected String name;

    public Animal(String name) { this.name = name; } // ✅ Can have constructors

    public abstract void makeSound(); // Must be implemented by subclasses

    public void breathe() { // Concrete method — inherited as-is
        System.out.println(name + " is breathing");
    }
}

public class Dog extends Animal {
    public Dog(String name) { super(name); }

    @Override
    public void makeSound() { System.out.println("Woof!"); }
}
```

### Interface
A **contract** that defines what a class can do, without specifying how. Represents a **"can-do" / "has-a-capability"** relationship.

```java
public interface Flyable {
    void fly();  // implicitly public abstract

    default void land() { // Default method (Java 8+)
        System.out.println("Landing...");
    }

    static boolean canFly(Animal a) { // Static method (Java 8+)
        return a instanceof Flyable;
    }
}

public class Bird extends Animal implements Flyable {
    public Bird(String name) { super(name); }
    @Override public void makeSound() { System.out.println("Tweet!"); }
    @Override public void fly() { System.out.println(name + " is flying"); }
}
```

### Key Differences

| Feature | Abstract Class | Interface |
|---|---|---|
| **Multiple inheritance** | ❌ Single `extends` only | ✅ Multiple `implements` |
| **Constructors** | ✅ Yes | ❌ No |
| **Instance fields** | ✅ Yes (any access modifier) | Only `public static final` constants |
| **Method types** | Abstract + concrete | Abstract + `default` + `static` + `private` (Java 9+) |
| **Access modifiers** | Any (`private`, `protected`, etc.) | Methods are implicitly `public` |
| **State** | ✅ Can maintain state (fields) | ❌ No instance state |

### When to Use Which?
- **Abstract class**: When subclasses share **common state** (fields) and behavior, and there's a clear "is-a" hierarchy (e.g., `Vehicle` → `Car`, `Truck`).
- **Interface**: When unrelated classes need a **shared capability** (e.g., `Comparable`, `Serializable`, `Flyable`).

> [!TIP]
> Since Java 8+, interfaces with `default` methods blurred the line significantly. The modern rule of thumb: **prefer interfaces** for defining contracts, and use abstract classes only when you need constructors, mutable instance fields, or non-public methods.
