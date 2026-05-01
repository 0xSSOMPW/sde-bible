# Q: What is Inversion of Control (IoC) and Dependency Injection (DI)?

**Answer:**

### Inversion of Control (IoC)
IoC is a **design principle** where the framework controls the flow of the program and the creation of objects, instead of the application code. The "control is inverted" — you don't call the framework, the framework calls you.

### Dependency Injection (DI)
DI is the most common **implementation of IoC**. Instead of a class creating its own dependencies, they are **injected from the outside** by the Spring container.

### Without DI (Tight Coupling)
```java
// ❌ OrderService creates its own dependency — hard to test, hard to swap
public class OrderService {
    private final OrderRepository repo = new MySQLOrderRepository(); // Hardcoded

    public void createOrder(Order order) {
        repo.save(order);
    }
}
```

### With DI (Loose Coupling)
```java
// ✅ Dependency is injected — testable, swappable
@Service
public class OrderService {
    private final OrderRepository repo;  // Interface, not implementation

    @Autowired  // Spring injects the concrete implementation
    public OrderService(OrderRepository repo) {
        this.repo = repo;
    }

    public void createOrder(Order order) {
        repo.save(order);
    }
}
```

### Types of Injection

**1. Constructor Injection (Preferred)**
```java
@Service
public class OrderService {
    private final OrderRepository repo;

    public OrderService(OrderRepository repo) { // @Autowired optional for single constructor
        this.repo = repo;
    }
}
```

**2. Setter Injection**
```java
@Service
public class OrderService {
    private OrderRepository repo;

    @Autowired
    public void setRepo(OrderRepository repo) { this.repo = repo; }
}
```

**3. Field Injection (Avoid)**
```java
@Service
public class OrderService {
    @Autowired  // ❌ Makes testing hard, hides dependencies
    private OrderRepository repo;
}
```

### Why Constructor Injection is Best

| Aspect | Constructor | Setter | Field |
|---|---|---|---|
| **Immutability** | ✅ `final` fields | ❌ Mutable | ❌ Mutable |
| **Required deps** | ✅ Enforced at compile time | ❌ Can be null | ❌ Can be null |
| **Testability** | ✅ Easy (just pass mocks) | ⚠️ Need setter | ❌ Need reflection |
| **Circular deps** | Fails fast (detected) | Can mask issues | Can mask issues |

> [!TIP]
> Since Spring 4.3, if a class has **only one constructor**, `@Autowired` is optional. This makes constructor injection even cleaner and framework-agnostic.
