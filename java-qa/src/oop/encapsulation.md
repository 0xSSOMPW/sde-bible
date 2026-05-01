# Q: What is encapsulation? Why does it matter beyond "private fields + getters/setters"?

**Answer:**

Encapsulation = bundling state + behavior + **hiding internal representation** behind a stable interface. The point isn't "use private" — it's **decouple callers from implementation** so internals can change.

### Naive "Encapsulation" (Anti-pattern)
```java
class User {
    private String name;
    private List<Order> orders;

    public String getName() { return name; }
    public void setName(String n) { this.name = n; }
    public List<Order> getOrders() { return orders; }   // ❌ leaks mutable internal list
    public void setOrders(List<Order> o) { this.orders = o; }
}
```
Caller can mutate `getOrders().clear()` — bypasses class. This is a public field with extra steps.

### Real Encapsulation
```java
public final class User {
    private final String name;
    private final List<Order> orders = new ArrayList<>();

    public User(String name) { this.name = Objects.requireNonNull(name); }

    public String name() { return name; }

    public List<Order> orders() {
        return Collections.unmodifiableList(orders);  // defensive
    }

    public void placeOrder(Order o) {                  // behavior, not setter
        if (orders.size() >= 100) throw new IllegalStateException("max orders");
        orders.add(o);
    }
}
```

### Principles
1. **Hide state.** Fields private; never expose mutable internals.
2. **Defensive copies** for mutable inputs/outputs (or use immutable types).
3. **Validate at boundaries.** Reject bad input in constructor/method.
4. **Behavior over setters.** `placeOrder()` not `setOrders()` — captures invariants.
5. **Minimal API.** Only expose what callers need.

### Why It Matters
- **Refactor safely.** Change `orders` from `List` to `LinkedHashSet` without breaking callers.
- **Invariants hold.** "Max 100 orders" enforced — caller can't break it.
- **Concurrency.** Internal state can be made thread-safe in one place.
- **Testability.** Behavior methods describe domain rules; tests assert behavior, not field values.

### Java 16+ Records
```java
public record Money(BigDecimal amount, Currency currency) {
    public Money {
        Objects.requireNonNull(amount);
        if (amount.signum() < 0) throw new IllegalArgumentException("negative");
    }
}
```
Compact, immutable, validated. Fits encapsulation goals for value types.

### Encapsulation vs Information Hiding
- **Encapsulation**: bundling state + behavior in one unit.
- **Information hiding**: design decisions hidden behind interfaces.
- Java's `private` enforces both. Modules (`module-info.java`, JPMS) extend it across packages.
