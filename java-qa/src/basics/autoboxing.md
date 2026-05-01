# Q: Explain autoboxing, unboxing, and the Integer cache.

**Answer:**

### Autoboxing & Unboxing
- **Autoboxing**: automatic conversion of primitive → wrapper (`int` → `Integer`).
- **Unboxing**: wrapper → primitive (`Integer` → `int`).

```java
Integer boxed = 10;       // autoboxing: Integer.valueOf(10)
int unboxed = boxed;      // unboxing: boxed.intValue()

List<Integer> nums = new ArrayList<>();
nums.add(5);              // autoboxing — primitive can't go in generic
int first = nums.get(0);  // unboxing
```

### The Integer Cache (`-128` to `127`)
`Integer.valueOf(int)` caches values in `[-128, 127]`. Same reference returned for cached values.

```java
Integer a = 100;
Integer b = 100;
System.out.println(a == b);   // true  — same cached reference

Integer c = 200;
Integer d = 200;
System.out.println(c == d);   // false — new objects, outside cache

System.out.println(c.equals(d)); // true — always use .equals() for wrappers
```

> Cache upper bound configurable via `-XX:AutoBoxCacheMax=N`.

### Pitfalls

**1. NullPointerException on unboxing**
```java
Integer x = null;
int y = x;  // 💥 NPE — unboxing null
```

**2. Performance — boxing in tight loops**
```java
Long sum = 0L;                    // ❌ Long, not long
for (long i = 0; i < 1_000_000; i++) {
    sum += i;                     // boxes/unboxes every iteration
}
```
Use `long` primitive → 10x+ faster.

**3. `==` vs `.equals()` on wrappers**
```java
Integer a = 1000, b = 1000;
if (a == b) { ... }        // ❌ reference compare — false outside cache
if (a.equals(b)) { ... }   // ✅ value compare
```

**4. Conditional expression unboxing**
```java
Integer i = null;
int x = true ? i : 0;  // 💥 NPE — ternary unboxes Integer
```

### When Boxing Happens
- Generics: `List<Integer>`, `Map<String, Long>`
- Object params: `Object o = 5;`
- Collection ops: `set.contains(42)`
- Reflection: `method.invoke(obj, 1)` — args are `Object[]`

### Best Practices
- Primitives in hot paths.
- Wrappers only when nullability or generics needed.
- Always `.equals()` for wrapper comparison.
- Watch ternary + null returns for NPE.
