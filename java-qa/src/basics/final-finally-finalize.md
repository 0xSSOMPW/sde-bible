# Q: What is the difference between `final`, `finally`, and `finalize`?

**Answer:**

Despite the similar names, these are completely unrelated concepts.

### `final` — A Keyword for Immutability/Restriction

**1. final variable** — Value cannot be changed after assignment (constant).
```java
final int MAX = 100;
MAX = 200; // ❌ Compilation error
```

**2. final method** — Cannot be overridden by subclasses.
```java
public class Parent {
    public final void doWork() { /* cannot be overridden */ }
}
```

**3. final class** — Cannot be extended (no subclasses).
```java
public final class String { /* no one can extend String */ }
```

**4. final with references** — The reference can't change, but the object it points to CAN.
```java
final List<String> list = new ArrayList<>();
list.add("hello");       // ✅ Modifying the object is fine
list = new ArrayList<>(); // ❌ Reassigning the reference is not
```

### `finally` — Exception Handling Block
A block that **always executes** after a try-catch, whether an exception occurred or not. Used for cleanup (closing resources, releasing locks).

```java
try {
    riskyOperation();
} catch (Exception e) {
    log.error("Failed", e);
} finally {
    connection.close(); // Always runs, even if exception is thrown
}
```

> [!WARNING]
> `finally` does NOT execute in two edge cases:
> 1. `System.exit()` is called in the try/catch block.
> 2. The JVM crashes or the thread is killed.

### `finalize()` — Garbage Collection Hook (DEPRECATED)
A method called by the garbage collector before destroying an object. It was intended for cleanup of native resources.

```java
@Override
protected void finalize() throws Throwable {
    // Cleanup before GC — DON'T USE THIS
    super.finalize();
}
```

**Why it's deprecated (Java 9+):**
- No guarantee when (or if) it will be called.
- Causes significant GC performance overhead.
- Objects can be "resurrected" in `finalize()`, creating bugs.
- Not a replacement for proper resource management.

**Use instead:** `try-with-resources` + `AutoCloseable`, or `Cleaner` (Java 9+).

### Summary

| Concept | Type | Purpose |
|---|---|---|
| `final` | Keyword | Prevent modification/inheritance |
| `finally` | Block | Guaranteed cleanup after try-catch |
| `finalize()` | Method (deprecated) | GC hook before object destruction |
