# Q: Explain the four kinds of nested classes in Java.

**Answer:**

| Type | Static? | Holds outer ref? | Use case |
|---|---|---|---|
| Static nested | yes | no | Logical grouping; helper attached to enclosing class |
| Inner (member) | no | yes | Tightly bound to outer instance |
| Local | no | yes (if in instance method) | One-method-only helper |
| Anonymous | no | yes | One-shot interface/abstract impl |

### 1. Static Nested
```java
class Outer {
    static class Builder {
        Outer build() { return new Outer(); }
    }
}
Outer o = new Outer.Builder().build();
```
No outer instance needed. Same as a top-level class but namespaced.

### 2. Inner (Member) Class
```java
class Outer {
    private int x = 10;
    class Inner {
        int read() { return x; }   // implicit Outer.this reference
    }
}
Outer.Inner i = new Outer().new Inner();  // needs outer instance
```

> [!WARNING]
> Inner classes hold a hidden reference to the outer instance — common cause of memory leaks (e.g., non-static `Handler` holding `Activity` in Android, or non-static inner classes referenced by long-lived collections).

### 3. Local Class (Inside Method)
```java
void process(List<String> items) {
    class LengthFilter {
        boolean keep(String s) { return s.length() > 3; }
    }
    LengthFilter f = new LengthFilter();
    items.removeIf(s -> !f.keep(s));
}
```
Captures effectively final local variables.

### 4. Anonymous Class
```java
Runnable r = new Runnable() {
    @Override public void run() { System.out.println("hi"); }
};
```
Mostly replaced by lambdas (Java 8+) for single-method interfaces:
```java
Runnable r = () -> System.out.println("hi");
```

### Lambdas vs Anonymous Classes
| | Lambda | Anonymous class |
|---|---|---|
| `this` refers to | enclosing class | the anonymous instance |
| Compiled to | invokedynamic / synthetic method | new `.class` file |
| Can hold state | no | yes (instance fields) |
| Multiple methods | no (single abstract method only) | yes |

### Effectively Final Capture
```java
void demo() {
    int x = 10;
    Runnable r = () -> System.out.println(x);  // ✅ x not reassigned
    // x = 20; ← would break the lambda
}
```

### Memory Leak Example
```java
class Repository {
    private List<Listener> listeners = new ArrayList<>();
    void register() {
        listeners.add(new Listener() {     // anonymous → holds Repository.this
            public void onEvent() { ... }
        });
    }
}
// Listener pinned in `listeners` → Repository instance never GC'd if list outlives it.
```
Fix: make it `static` nested, or store no reference, or use weak refs.
