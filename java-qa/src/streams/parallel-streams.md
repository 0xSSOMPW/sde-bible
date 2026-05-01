# Q: How do parallel streams work? When should you avoid them?

**Answer:**

`stream().parallel()` or `parallelStream()` splits work across the **common ForkJoinPool** (`ForkJoinPool.commonPool()`).

### How
1. Source split into chunks (`Spliterator`).
2. Each chunk processed on a pool thread.
3. Results combined (depends on terminal op).

```java
list.parallelStream()
    .filter(x -> x > 0)
    .mapToInt(Integer::intValue)
    .sum();
```

### Common Pool — Important Caveats
- Default size = `Runtime.getRuntime().availableProcessors() - 1`.
- **Shared across all parallel streams in the JVM** — one slow task blocks others.
- Configure: `-Djava.util.concurrent.ForkJoinPool.common.parallelism=8`.

### When Parallel Streams Help
- Large dataset (rough rule: > 10k elements).
- CPU-bound work per element (heavy compute, not I/O).
- Stateless lambdas (no shared mutable state).
- Splittable source (`ArrayList`, arrays, `IntStream.range`) — splits cheaply.
- Associative reduction (`sum`, `max`, `min`).

### When To Avoid

**1. I/O or blocking work**
```java
urls.parallelStream().map(this::httpGet);  // ❌ blocks pool threads → starves the JVM
```
Use `CompletableFuture` with a dedicated `Executor`, or virtual threads.

**2. Small datasets**
Overhead of split + merge > savings.

**3. Order-sensitive work**
```java
list.parallelStream().forEach(System.out::println);          // unordered
list.parallelStream().forEachOrdered(System.out::println);   // ordered, kills parallelism gain
```

**4. Stateful or shared-mutable lambdas**
```java
List<Integer> result = new ArrayList<>();
list.parallelStream().forEach(result::add);   // 💥 race — ArrayList not thread-safe
// Correct: collect()
```

**5. LinkedList / Stream.iterate**
Bad splitters → poor parallelism.

**6. Unsplittable sources**
`Files.lines(path).parallel()` — IO bounded, hard to split.

### Cost Model
```
Useful = N * cost_per_element  >>  Splitting + merging + thread coordination overhead
```

### Custom Pool (Workaround)
Run parallel stream in your own pool:
```java
ForkJoinPool pool = new ForkJoinPool(16);
pool.submit(() -> list.parallelStream().map(...).collect(...)).get();
pool.shutdown();
```

### Reduction: Identity Must Be a True Identity
```java
int sum = list.parallelStream().reduce(0, Integer::sum);     // ✅ 0 + x = x
String s = list.parallelStream().reduce("", String::concat); // ✅
int prod = list.parallelStream().reduce(1, (a,b) -> a*b);    // ✅ 1 * x = x
int bad = list.parallelStream().reduce(1, Integer::sum);     // ❌ wrong identity
```

### Performance Reality
Parallel streams rarely scale linearly. Measure with JMH. Often a `for` loop or sequential stream is faster on real workloads.

### Decision Tree
```
Is work CPU-bound?              → no  → don't parallelize
Are elements > ~10k?            → no  → don't parallelize
Is operation associative?        → no  → don't parallelize
Is source efficiently splittable? → no → don't parallelize
Will it share the common pool with other work? → yes → use custom executor
```
