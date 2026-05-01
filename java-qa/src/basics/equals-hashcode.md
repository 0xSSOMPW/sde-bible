# Q: What is the contract between `==`, `.equals()`, and `hashCode()`?

**Answer:**

### `==` (Reference Equality)
Compares **memory addresses**. Returns `true` only if both variables point to the exact same object on the heap.

```java
String a = new String("hello");
String b = new String("hello");
System.out.println(a == b);  // false — different objects
```

### `.equals()` (Content Equality)
Compares the **logical content** of two objects. The default implementation in `Object` uses `==`, so you must **override** it in your classes.

```java
System.out.println(a.equals(b));  // true — String overrides equals()
```

### `hashCode()`
Returns an integer hash used by hash-based collections (`HashMap`, `HashSet`). Must be consistent with `equals()`.

### The Contract (Critical!)

1. **If `a.equals(b)` is `true`, then `a.hashCode() == b.hashCode()` MUST be `true`.**
2. If `a.hashCode() != b.hashCode()`, then `a.equals(b)` MUST be `false`.
3. If `a.hashCode() == b.hashCode()`, `a.equals(b)` may or may not be `true` (hash collisions are allowed).

### What Happens If You Break the Contract?

```java
// ❌ BROKEN: overrides equals() but NOT hashCode()
public class Employee {
    private int id;
    private String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Employee)) return false;
        Employee e = (Employee) o;
        return id == e.id && Objects.equals(name, e.name);
    }
    // hashCode NOT overridden — uses default Object.hashCode() (memory address)
}

Employee e1 = new Employee(1, "Alice");
Employee e2 = new Employee(1, "Alice");

e1.equals(e2);  // true ✅

Set<Employee> set = new HashSet<>();
set.add(e1);
set.contains(e2);  // false! 💥 — different hashCode → looks in wrong bucket
```

### Correct Implementation

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Employee e)) return false;
    return id == e.id && Objects.equals(name, e.name);
}

@Override
public int hashCode() {
    return Objects.hash(id, name);
}
```

> [!CAUTION]
> **Always override `hashCode()` when you override `equals()`**. This is the #1 source of subtle bugs with `HashMap` and `HashSet` — objects that are logically equal but have different hash codes end up in different buckets and are treated as different entries.
