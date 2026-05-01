# Q: What are Lambda Expressions and Functional Interfaces?

**Answer:**

### Functional Interface
An interface with **exactly one abstract method**. Annotated with `@FunctionalInterface` (optional but recommended).

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);  // Single abstract method
    // Can have default and static methods
}
```

### Lambda Expressions (Java 8+)
A concise way to implement a functional interface **without boilerplate anonymous classes**.

```java
// ❌ Old way: Anonymous class (verbose)
Predicate<String> isLong = new Predicate<String>() {
    @Override
    public boolean test(String s) {
        return s.length() > 5;
    }
};

// ✅ Lambda: same thing, cleaner
Predicate<String> isLong = s -> s.length() > 5;
```

### Built-in Functional Interfaces (`java.util.function`)

| Interface | Method | Signature | Use Case |
|---|---|---|---|
| `Predicate<T>` | `test(T)` | `T → boolean` | Filtering, conditions |
| `Function<T,R>` | `apply(T)` | `T → R` | Transformation |
| `Consumer<T>` | `accept(T)` | `T → void` | Side effects (logging, saving) |
| `Supplier<T>` | `get()` | `() → T` | Factory, lazy evaluation |
| `BiFunction<T,U,R>` | `apply(T,U)` | `(T,U) → R` | Two-arg transformation |
| `UnaryOperator<T>` | `apply(T)` | `T → T` | Same-type transformation |

### Method References
Shorthand for lambdas that call an existing method:

```java
// Lambda                          →  Method Reference
s -> s.toUpperCase()               →  String::toUpperCase
s -> System.out.println(s)         →  System.out::println
s -> Integer.parseInt(s)           →  Integer::parseInt
() -> new ArrayList<>()            →  ArrayList::new
```

### Effectively Final
Lambdas can capture local variables, but they must be **effectively final** (not modified after initialization):

```java
int multiplier = 3;  // effectively final — never reassigned
Function<Integer, Integer> multiply = x -> x * multiplier; // ✅

int counter = 0;
Runnable task = () -> counter++; // ❌ Compilation error! counter is modified
```

> [!TIP]
> Think of lambdas as **data** rather than code. You're passing behavior as a parameter — the foundation of functional programming in Java. This enables powerful patterns like strategy pattern without a dozen classes.
