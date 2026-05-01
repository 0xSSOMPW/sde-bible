# Q: Explain `Collectors`. Common patterns + what `groupingBy` actually does.

**Answer:**

`Collector<T, A, R>` = mutable reduction strategy: how to **accumulate** stream elements into a result container.

Built-in factory: `java.util.stream.Collectors`.

### Common Recipes

**To collection**
```java
list.stream().collect(Collectors.toList());        // mutable, post-Java 16: prefer .toList()
list.stream().toList();                            // unmodifiable, Java 16+
list.stream().collect(Collectors.toUnmodifiableList());
list.stream().collect(Collectors.toSet());
list.stream().collect(Collectors.toCollection(TreeSet::new));
```

**To map**
```java
users.stream().collect(Collectors.toMap(User::id, Function.identity()));

// duplicate-key merge
.collect(Collectors.toMap(User::email, u -> u, (a, b) -> a));

// pick map type
.collect(Collectors.toMap(User::id, u -> u, (a,b) -> a, LinkedHashMap::new));
```

**Joining strings**
```java
names.stream().collect(Collectors.joining(", ", "[", "]"));
// → "[alice, bob, carol]"
```

**Counting / summing / averaging**
```java
orders.stream().collect(Collectors.counting());
orders.stream().collect(Collectors.summingDouble(Order::amount));
orders.stream().collect(Collectors.averagingInt(Order::quantity));
orders.stream().collect(Collectors.summarizingDouble(Order::amount));
// → DoubleSummaryStatistics{count, sum, min, avg, max}
```

### `groupingBy` (The Big One)

```java
Map<Status, List<Order>> byStatus =
    orders.stream().collect(Collectors.groupingBy(Order::status));
```

Two- and three-arg forms take a **downstream collector**:

```java
// count per status
Map<Status, Long> counts =
    orders.stream().collect(Collectors.groupingBy(Order::status, Collectors.counting()));

// sum amount per customer
Map<Long, Double> totals =
    orders.stream().collect(Collectors.groupingBy(
        Order::customerId,
        Collectors.summingDouble(Order::amount)));

// nested grouping
Map<Status, Map<Long, List<Order>>> nested =
    orders.stream().collect(Collectors.groupingBy(
        Order::status,
        Collectors.groupingBy(Order::customerId)));

// pick map type
Collectors.groupingBy(Order::status, TreeMap::new, Collectors.toList());
```

### `partitioningBy`
Special-case `groupingBy` with a predicate → always returns map with `true`/`false` keys (both keys present even if one is empty).

```java
Map<Boolean, List<User>> adultsAndMinors =
    users.stream().collect(Collectors.partitioningBy(u -> u.age() >= 18));
```

### `mapping`, `filtering`, `flatMapping`
Apply transform inside the downstream:

```java
Map<Status, List<String>> idsByStatus =
    orders.stream().collect(Collectors.groupingBy(
        Order::status,
        Collectors.mapping(Order::id, Collectors.toList())));

Map<Status, List<Order>> highValuePerStatus =
    orders.stream().collect(Collectors.groupingBy(
        Order::status,
        Collectors.filtering(o -> o.amount() > 1000, Collectors.toList())));
```

### `reducing`
Lower-level than the typed sum/avg variants:
```java
Optional<Order> biggest =
    orders.stream().collect(Collectors.reducing(BinaryOperator.maxBy(Comparator.comparing(Order::amount))));
```

### Custom Collector
```java
Collector<Order, ?, BigDecimal> totalCollector = Collector.of(
    () -> new BigDecimal[]{ BigDecimal.ZERO },          // supplier
    (a, o) -> a[0] = a[0].add(o.amount()),              // accumulator
    (a, b) -> { a[0] = a[0].add(b[0]); return a; },     // combiner
    a -> a[0]                                            // finisher
);
```

### Pitfalls
- `toMap` with duplicate keys → `IllegalStateException`. Always pass merger.
- `Collectors.toList()` returns mutable list pre-16. Don't rely on immutability.
- `groupingBy` returns regular `HashMap` — no order guarantees. Use `LinkedHashMap` for insertion order.
- `null` values not allowed in `toMap` (uses `Map.merge`). Pre-filter or use `groupingBy`.
