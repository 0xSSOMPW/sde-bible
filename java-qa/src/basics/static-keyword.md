# Q: What does the `static` keyword do? Where can it be used?

**Answer:**

`static` = belongs to the **class**, not to instances. One copy shared across all instances. Loaded when the class is loaded.

### 1. Static Variables (Class Variables)
```java
class Counter {
    static int count = 0;   // shared across all instances
    int id;                 // per instance

    Counter() { id = ++count; }
}

new Counter();  // count=1
new Counter();  // count=2 — shared
```

### 2. Static Methods
```java
class MathUtil {
    static int square(int x) { return x * x; }
}
MathUtil.square(5);  // call without instance
```

Rules:
- Cannot access instance fields/methods (no `this`).
- Cannot be overridden — **hidden** (resolved at compile time, not polymorphic).
- Can be called via instance, but discouraged: `obj.staticMethod()`.

### 3. Static Block
```java
class Config {
    static final Map<String, String> MAP;
    static {
        MAP = new HashMap<>();
        MAP.put("env", "prod");
    }
}
```
Runs once at class load. Multiple blocks run top-to-bottom.

### 4. Static Nested Class
```java
class Outer {
    static class Nested {
        // does NOT hold reference to Outer instance
    }
}
Outer.Nested n = new Outer.Nested();
```
vs. **inner class** (non-static) which holds implicit `Outer.this` reference.

### 5. Static Imports
```java
import static java.lang.Math.PI;
import static java.lang.Math.sqrt;

double r = sqrt(PI);  // no Math. prefix
```

### Static Method Hiding vs Overriding
```java
class Parent { static void hi() { System.out.println("parent"); } }
class Child extends Parent { static void hi() { System.out.println("child"); } }

Parent p = new Child();
p.hi();  // "parent" — static binding (hidden, not overridden)
```
Compare with instance methods — would print `"child"` (dynamic dispatch).

### Common Pitfalls
- **Mutable static state** = global state = thread-safety nightmare.
- **Static + Spring** = bypass DI; static fields not injected by default.
- **Memory leaks**: static collections keep references alive for class lifetime.
- **Test isolation**: static state leaks across tests.

### When to Use
- Constants (`public static final`).
- Pure utility functions (`Math.max`, `Collections.sort`).
- Factory methods (`List.of`, `Optional.of`).
- Singletons (carefully).

### When to Avoid
- Anything stateful that's not a constant.
- Anything you want to mock in tests.
- Replace with dependency injection where possible.
