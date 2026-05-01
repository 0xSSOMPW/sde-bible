# Q: Explain the SOLID Principles with Java examples.

**Answer:**

SOLID is a set of five design principles for writing maintainable, scalable object-oriented code.

### S — Single Responsibility Principle
A class should have **only one reason to change**. It should do one thing and do it well.

```java
// ❌ Violates SRP: handles both user logic AND email sending
public class UserService {
    public void createUser(User user) { /* save to DB */ }
    public void sendWelcomeEmail(User user) { /* send email */ }
}

// ✅ Follows SRP: each class has one responsibility
public class UserService {
    public void createUser(User user) { /* save to DB */ }
}
public class EmailService {
    public void sendWelcomeEmail(User user) { /* send email */ }
}
```

### O — Open/Closed Principle
Classes should be **open for extension, closed for modification**. Add new behavior without changing existing code.

```java
// ❌ Must modify this class every time a new shape is added
public double calculateArea(Shape shape) {
    if (shape instanceof Circle) return Math.PI * ((Circle) shape).radius * ((Circle) shape).radius;
    if (shape instanceof Rectangle) return ((Rectangle) shape).w * ((Rectangle) shape).h;
}

// ✅ Add new shapes by extending, not modifying
public abstract class Shape {
    public abstract double area();
}
public class Circle extends Shape {
    @Override public double area() { return Math.PI * radius * radius; }
}
// Adding a new shape doesn't touch existing code
```

### L — Liskov Substitution Principle
Subtypes must be **substitutable** for their base types without breaking the program.

```java
// ❌ Violates LSP: Square changes Rectangle's expected behavior
public class Rectangle {
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
}
public class Square extends Rectangle {
    @Override public void setWidth(int w) { this.width = w; this.height = w; } // Surprise!
}

// Code expecting Rectangle behavior breaks:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(10);
assert r.area() == 50; // FAILS! Square made it 100
```

### I — Interface Segregation Principle
Clients should not be forced to depend on methods they don't use. Prefer **small, focused interfaces**.

```java
// ❌ Fat interface: forces all implementations to handle everything
public interface Worker {
    void work();
    void eat();
    void sleep();
}

// ✅ Segregated: each interface is focused
public interface Workable { void work(); }
public interface Feedable { void eat(); }

public class Robot implements Workable {
    @Override public void work() { /* ... */ }
    // Robot doesn't need eat() or sleep()
}
```

### D — Dependency Inversion Principle
High-level modules should not depend on low-level modules. Both should depend on **abstractions**.

```java
// ❌ High-level depends directly on low-level
public class OrderService {
    private MySQLOrderRepository repo = new MySQLOrderRepository(); // Tight coupling
}

// ✅ Both depend on abstraction
public interface OrderRepository { void save(Order order); }

public class OrderService {
    private final OrderRepository repo; // Depends on interface
    public OrderService(OrderRepository repo) { this.repo = repo; } // DI
}
```

> [!TIP]
> In interviews, don't just list the principles — give examples of **violations** and their **consequences**. This shows you've actually used them in practice rather than memorizing definitions.
