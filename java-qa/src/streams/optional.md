# Q: What is Optional and how should you use it?

**Answer:**

`Optional<T>` (Java 8+) is a container that may or may not contain a non-null value. It's designed to **eliminate `NullPointerException`** by making nullability explicit in the API.

### The Problem
```java
// ❌ NullPointerException waiting to happen
User user = userRepository.findById(id); // Could return null
String city = user.getAddress().getCity(); // 💥 NPE if user or address is null
```

### Using Optional
```java
// ✅ Explicit nullability
Optional<User> user = userRepository.findById(id);

// Safe access
String city = user
    .map(User::getAddress)
    .map(Address::getCity)
    .orElse("Unknown");
```

### Creating Optionals
```java
Optional<String> present = Optional.of("hello");       // Must be non-null
Optional<String> nullable = Optional.ofNullable(value); // May be null
Optional<String> empty = Optional.empty();               // Explicitly empty
```

### Consuming Optionals
```java
// Get with default value
String name = optional.orElse("default");

// Get with lazy default (only computed if empty)
String name = optional.orElseGet(() -> expensiveDefault());

// Throw if empty
String name = optional.orElseThrow(() -> new NotFoundException("Not found"));

// Execute if present
optional.ifPresent(value -> System.out.println(value));

// Java 9+: if-present-else
optional.ifPresentOrElse(
    value -> System.out.println("Found: " + value),
    () -> System.out.println("Not found")
);
```

### Transforming Optionals
```java
// map: transform the value if present
Optional<String> upper = optional.map(String::toUpperCase);

// flatMap: when the transformation itself returns Optional
Optional<String> city = userOpt.flatMap(User::getAddress)  // getAddress returns Optional<Address>
                               .map(Address::getCity);

// filter: keep value only if predicate matches
Optional<User> admin = userOpt.filter(u -> u.getRole() == Role.ADMIN);
```

### Anti-Patterns (Don't Do This!)

```java
// ❌ Using Optional as a glorified null check — defeats the purpose
if (optional.isPresent()) {
    return optional.get();
}

// ❌ Optional as a method parameter — confusing API
public void process(Optional<String> name) { } // Bad

// ❌ Optional as a field — not serializable, adds overhead
private Optional<String> name; // Bad

// ❌ Optional.of() with a nullable value — NPE!
Optional.of(null); // 💥 NullPointerException
```

### Best Practices

| ✅ Do | ❌ Don't |
|---|---|
| Use as **return type** to signal possible absence | Use as **method parameter** |
| Use `map`/`flatMap`/`orElse` chains | Use `isPresent()` + `get()` |
| Use `orElseThrow()` for required values | Use `Optional.get()` without checking |
| Use `Optional.ofNullable()` for nullable values | Use `Optional.of()` with nullable values |

> [!TIP]
> Think of `Optional` as a **single-element Stream**. It supports `map`, `flatMap`, `filter`, and `ifPresent` — the same functional operations. If you're comfortable with Streams, Optional follows the same patterns.
