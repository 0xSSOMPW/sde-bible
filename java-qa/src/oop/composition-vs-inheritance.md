# Q: Composition vs Inheritance — when to use which?

**Answer:**

**Rule of thumb: "Favor composition over inheritance"** (Effective Java, Item 18).

| | Inheritance (`extends`) | Composition (has-a) |
|---|---|---|
| Relationship | "is-a" | "has-a" |
| Coupling | Tight — child depends on parent's impl | Loose — depends on interface |
| Flexibility | Fixed at compile time | Swap at runtime |
| Encapsulation | Breaks (subclass sees parent internals) | Preserves |
| Multiple types | Single inheritance only | Compose any number |

### Inheritance Example (When It Goes Wrong)
```java
// Classic broken example from Effective Java
class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;

    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);  // 💥 HashSet.addAll calls add() internally
    }
}
// addCount double-counts because addAll → add → addCount++
```
Subclass broke when parent's internal call chain changed. **Fragile base class problem.**

### Composition Fix
```java
class InstrumentedSet<E> implements Set<E> {
    private final Set<E> delegate;        // composed, not extended
    private int addCount = 0;

    InstrumentedSet(Set<E> delegate) { this.delegate = delegate; }

    @Override public boolean add(E e) {
        addCount++;
        return delegate.add(e);
    }
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return delegate.addAll(c);  // delegate's internal calls don't hit our add()
    }
    // ... forward other Set methods
}
```
Works regardless of which `Set` impl is wrapped (`HashSet`, `TreeSet`, etc).

### When Inheritance IS Right
- True "is-a" relationship (`Dog extends Animal`).
- Designed-for-extension classes (e.g., `AbstractList`).
- Within your own controlled hierarchy.
- Template Method pattern.

### When Composition Wins
- Reusing behavior across unrelated types.
- Need to swap implementations.
- Avoiding deep hierarchies.
- Multiple "behaviors" needed (Java has no multiple inheritance).

### Strategy Pattern (Composition Done Right)
```java
class PaymentService {
    private final PaymentGateway gateway;  // injected, swappable
    PaymentService(PaymentGateway g) { this.gateway = g; }
    void pay(Order o) { gateway.charge(o); }
}
// Swap Stripe ↔ PayPal without touching PaymentService
```

### Liskov Substitution Test
If subclass can't fully replace parent without breaking behavior → don't inherit.

```java
class Square extends Rectangle { ... }  // ❌ violates LSP
// Setting width changes height — surprises code that expects Rectangle
```
