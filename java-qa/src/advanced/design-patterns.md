# Q: Explain the Singleton, Factory, and Builder design patterns.

**Answer:**

### 1. Singleton — One Instance, Global Access
Ensures a class has **exactly one instance** and provides a global point of access.

**Thread-Safe Singleton (Bill Pugh idiom):**
```java
public class DatabaseConnection {
    private DatabaseConnection() {} // Private constructor

    private static class Holder {
        private static final DatabaseConnection INSTANCE = new DatabaseConnection();
    }

    public static DatabaseConnection getInstance() {
        return Holder.INSTANCE; // Lazy, thread-safe (class loading guarantees)
    }
}
```

**Enum Singleton (simplest, recommended by Effective Java):**
```java
public enum DatabaseConnection {
    INSTANCE;

    public void query(String sql) { /* ... */ }
}
// Usage: DatabaseConnection.INSTANCE.query("SELECT 1");
```

**When to use:** Configuration managers, connection pools, caches, logging.

### 2. Factory — Delegate Object Creation
Encapsulates object creation logic, returning instances of a common interface without exposing the concrete class.

```java
public interface Notification {
    void send(String message);
}

public class EmailNotification implements Notification {
    @Override public void send(String msg) { /* send email */ }
}

public class SmsNotification implements Notification {
    @Override public void send(String msg) { /* send SMS */ }
}

// Factory
public class NotificationFactory {
    public static Notification create(String type) {
        return switch (type) {
            case "email" -> new EmailNotification();
            case "sms"   -> new SmsNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Usage
Notification n = NotificationFactory.create("email");
n.send("Hello!");
```

**When to use:** When the exact class to instantiate depends on runtime conditions (config, user input, environment).

### 3. Builder — Complex Object Construction
Separates the construction of a complex object from its representation. Avoids telescoping constructors.

```java
// ❌ Telescoping constructor hell
new User("Alice", "alice@mail.com", 25, "NYC", "Engineer", true, false);
// What is true? What is false? Unreadable.

// ✅ Builder pattern
public class User {
    private final String name;
    private final String email;
    private final int age;
    private final String city;

    private User(Builder builder) {
        this.name = builder.name;
        this.email = builder.email;
        this.age = builder.age;
        this.city = builder.city;
    }

    public static class Builder {
        private final String name;   // Required
        private final String email;  // Required
        private int age;             // Optional
        private String city;         // Optional

        public Builder(String name, String email) {
            this.name = name;
            this.email = email;
        }

        public Builder age(int age) { this.age = age; return this; }
        public Builder city(String city) { this.city = city; return this; }
        public User build() { return new User(this); }
    }
}

// Usage: clean and readable
User user = new User.Builder("Alice", "alice@mail.com")
    .age(25)
    .city("NYC")
    .build();
```

### Summary

| Pattern | Problem It Solves | Real-World Example |
|---|---|---|
| **Singleton** | Need exactly one shared instance | `Runtime.getRuntime()`, Spring beans (default scope) |
| **Factory** | Object creation depends on conditions | `Calendar.getInstance()`, `LoggerFactory.getLogger()` |
| **Builder** | Complex object with many optional params | `StringBuilder`, `HttpRequest.newBuilder()`, Lombok `@Builder` |

> [!TIP]
> In modern Java, **Lombok's `@Builder`** generates the Builder pattern automatically. And in Spring, most "singletons" are managed by the IoC container rather than the traditional pattern — so you rarely need to implement Singleton yourself.
