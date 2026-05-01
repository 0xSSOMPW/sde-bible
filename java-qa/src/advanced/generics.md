# Q: How do Generics work in Java? What is Type Erasure?

**Answer:**

### Generics
Generics enable **type-safe, parameterized** classes, interfaces, and methods. They catch type errors at compile time instead of runtime.

```java
// Without generics: runtime ClassCastException risk
List list = new ArrayList();
list.add("hello");
Integer x = (Integer) list.get(0); // 💥 ClassCastException at RUNTIME

// With generics: compile-time safety
List<String> list = new ArrayList<>();
list.add("hello");
// list.add(42);  // ❌ Compilation error — caught EARLY
String x = list.get(0); // No cast needed
```

### Type Erasure
Java generics are a **compile-time** feature only. The compiler uses generic type information for type checking, then **erases** all generic types and replaces them with their bounds (or `Object`).

```java
// What you write:
List<String> strings = new ArrayList<>();
List<Integer> ints = new ArrayList<>();

// After type erasure (what the JVM sees):
List strings = new ArrayList();  // Just "List" — type info is GONE
List ints = new ArrayList();

// At runtime:
strings.getClass() == ints.getClass(); // true! Both are just ArrayList
```

### Consequences of Type Erasure

```java
// ❌ Cannot do these at runtime:
if (obj instanceof List<String>) { }  // Compilation error
new T();                                // Cannot instantiate type parameter
T[] array = new T[10];                  // Cannot create generic array

// ❌ Cannot overload with different generic types:
void process(List<String> list) { }
void process(List<Integer> list) { }    // Compilation error — same erasure!
```

### Bounded Type Parameters

```java
// Upper bound: T must be Comparable or its subtype
public <T extends Comparable<T>> T findMax(List<T> list) {
    return list.stream().max(Comparator.naturalOrder()).orElseThrow();
}

// Multiple bounds
public <T extends Serializable & Comparable<T>> void process(T item) { }
```

### Wildcards

```java
// Upper-bounded: read-only (producer)
void printAll(List<? extends Number> numbers) {
    for (Number n : numbers) { System.out.println(n); }
    // numbers.add(42); ❌ Cannot add — compiler doesn't know the exact type
}

// Lower-bounded: write-only (consumer)
void addIntegers(List<? super Integer> list) {
    list.add(42);  // ✅ Can add Integer or subtypes
    // Integer x = list.get(0); ❌ Can only read as Object
}

// Unbounded: completely read-only
void countElements(List<?> list) {
    System.out.println(list.size());
}
```

### PECS: Producer Extends, Consumer Super
The mnemonic for remembering wildcard usage:
- **`? extends T`** → Read from the collection (it **produces** items).
- **`? super T`** → Write to the collection (it **consumes** items).

> [!TIP]
> In interviews, type erasure is the key insight. "Generics provide compile-time safety but are erased at runtime. This means you can't do runtime type checks on generic types or create generic arrays — it's all synthetic compiler enforcement."
