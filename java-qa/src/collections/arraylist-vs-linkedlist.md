# Q: ArrayList vs LinkedList — when to use which?

**Answer:**

**Short answer**: use `ArrayList` 99% of the time. `LinkedList` rarely wins in practice despite textbook claims.

| Op | `ArrayList` | `LinkedList` |
|---|---|---|
| `get(i)` | **O(1)** | O(n) — walk from head/tail |
| `add(e)` (end) | Amortized O(1) | O(1) |
| `add(0, e)` (front) | O(n) — shift all | O(1) |
| `add(i, e)` (middle) | O(n) — shift | O(n) — walk + insert |
| `remove(i)` | O(n) — shift | O(n) — walk |
| `iterator.remove()` | O(n) — shift remaining | O(1) |
| Memory per element | 4–8 bytes (array slot) | ~40 bytes (node + 2 refs) |
| Cache locality | excellent (contiguous) | poor (scattered nodes) |

### Why ArrayList Usually Wins Even For "LinkedList Cases"
Modern CPUs love contiguous memory. ArrayList shifts cost less than LinkedList pointer chasing because of cache lines + branch prediction. Bjarne Stroustrup's famous talk: linked lists lose for sizes < 100k even on insertion-heavy workloads.

### Internals

**ArrayList**:
```java
Object[] elementData;  // backing array
int size;
// add(): resize when full → Arrays.copyOf with 1.5x growth
```

**LinkedList**:
```java
class Node<E> {
    E item;
    Node<E> next;
    Node<E> prev;     // doubly-linked
}
```
Implements both `List` and `Deque`.

### Choose LinkedList When
- Heavy use as a **queue/deque** (`addFirst`, `removeFirst`, `pollLast`).
- Frequent insertions/removals via **iterator** (`O(1)`).
- Need a deque but `ArrayDeque` doesn't fit (rare).

### Choose ArrayList When
- Random access by index.
- Iteration-heavy.
- Default choice unless profiling proves otherwise.

### Better Alternatives
- Need a deque? → `ArrayDeque` beats `LinkedList`.
- Need fixed-size? → `Arrays.asList(...)` or `List.of(...)`.
- Need thread-safe? → `CopyOnWriteArrayList` (read-heavy) or wrap with `Collections.synchronizedList`.

### Common Pitfall
```java
List<Integer> list = new LinkedList<>();
for (int i = 0; i < list.size(); i++) {
    list.get(i);              // O(n) per call → O(n²) total
}
// Use iterator or for-each instead → O(n)
```

### Capacity Hint
```java
new ArrayList<>(10_000);  // pre-size if you know the count
// Avoids ~14 resize+copy cycles when growing from 10 → 10,000
```
