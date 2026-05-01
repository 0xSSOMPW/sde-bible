# Q: How does the Stream API work?

**Answer:**

The Stream API (Java 8+) provides a **declarative, functional** way to process collections — focusing on *what* to do instead of *how*.

### Creating Streams
```java
List<String> names = List.of("Alice", "Bob", "Charlie", "David");

names.stream()            // From collection
Stream.of("a", "b", "c") // From values
IntStream.range(1, 10)    // Primitive stream
Files.lines(Path.of("f")) // From file
```

### Intermediate Operations (Lazy, Return a Stream)
```java
names.stream()
    .filter(n -> n.length() > 3)         // Predicate: keep if true
    .map(String::toUpperCase)             // Transform each element
    .sorted()                             // Natural ordering
    .distinct()                           // Remove duplicates
    .limit(10)                            // Take first N
    .peek(System.out::println)            // Debug: inspect without modifying
```

### Terminal Operations (Trigger Execution, Return a Result)
```java
.collect(Collectors.toList())             // Collect into a List
.collect(Collectors.toSet())              // Collect into a Set
.collect(Collectors.joining(", "))        // Join as String
.collect(Collectors.groupingBy(fn))       // Group into Map
.forEach(System.out::println)            // Side-effect per element
.count()                                  // Count elements
.findFirst()                              // Optional<T>
.reduce(0, Integer::sum)                  // Reduce to single value
.toArray(String[]::new)                   // To array
```

### Real-World Example
```java
// Get the names of the top 3 highest-paid employees in the Engineering department
List<String> topPaid = employees.stream()
    .filter(e -> "Engineering".equals(e.getDepartment()))
    .sorted(Comparator.comparing(Employee::getSalary).reversed())
    .limit(3)
    .map(Employee::getName)
    .collect(Collectors.toList());
```

### Parallel Streams
```java
list.parallelStream()
    .filter(...)
    .map(...)
    .collect(Collectors.toList());
```

> [!CAUTION]
> **Parallel streams are not always faster.** They use the common ForkJoinPool and add overhead for splitting, threading, and merging. Use them only for CPU-intensive operations on large datasets. For I/O-bound tasks or small collections, sequential streams are faster.

### Key Concepts

| Concept | Detail |
|---|---|
| **Lazy evaluation** | Intermediate operations are NOT executed until a terminal operation is called |
| **Short-circuit** | Operations like `findFirst()`, `limit()`, `anyMatch()` stop early |
| **Stateless vs Stateful** | `filter`/`map` are stateless (per-element); `sorted`/`distinct` are stateful (need all elements) |
| **One-time use** | A stream can only be consumed ONCE. Reuse requires creating a new stream |
