# Q: How does the String Pool work? Why are Strings immutable?

**Answer:**

### String Pool (String Intern Pool)
The String Pool is a special memory region inside the **heap** (moved from PermGen to heap in Java 7) where Java caches string literals to save memory.

```java
String a = "hello";   // Created in the String Pool
String b = "hello";   // Reuses the SAME object from the pool
String c = new String("hello"); // Creates a NEW object on the heap (outside pool)

System.out.println(a == b);       // true  (same reference in pool)
System.out.println(a == c);       // false (different objects)
System.out.println(a.equals(c));  // true  (same content)
```

You can explicitly add a string to the pool:
```java
String d = c.intern();  // Returns the pooled reference
System.out.println(a == d);  // true
```

### Why Are Strings Immutable?

**1. String Pool Requires It**
If strings were mutable, changing one reference would corrupt every other reference pointing to the same pooled object.

**2. Thread Safety**
Immutable objects are inherently thread-safe. Multiple threads can share the same string without synchronization.

**3. Security**
Strings are used for class loading, network connections, file paths, database URLs. If a string could be modified after creation, it would be a massive security hole.

**4. hashCode Caching**
Since a String's content never changes, its `hashCode()` is computed once and cached. This makes Strings extremely efficient as `HashMap` keys.

```java
// String class internally:
public final class String {
    private final char[] value;   // final → cannot be reassigned
    private int hash;             // cached hashCode
}
```

### Common Follow-Up: StringBuilder vs StringBuffer

| Feature | String | StringBuilder | StringBuffer |
|---|---|---|---|
| **Mutability** | Immutable | Mutable | Mutable |
| **Thread-safe** | Yes (immutable) | ❌ No | ✅ Yes (synchronized) |
| **Performance** | Slow for concatenation | Fast | Slower than StringBuilder |
| **Use case** | Constants, keys | Single-threaded string building | Multi-threaded string building |

```java
// ❌ Bad: Creates a new String object each iteration
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;  // O(n²) — each += creates a new String
}

// ✅ Good: Modifies in-place
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);  // O(n) — appends to the same buffer
}
```

> [!TIP]
> In modern Java (9+), the JIT compiler often optimizes string concatenation with `+` into `StringBuilder` or `invokedynamic` calls. But in loops, explicit `StringBuilder` is still the right approach.
