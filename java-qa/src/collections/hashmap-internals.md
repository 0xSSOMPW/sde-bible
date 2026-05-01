# Q: How does HashMap work internally in Java?

**Answer:**

This is one of the most asked Java interview questions. Understanding HashMap internals demonstrates deep knowledge of data structures.

### Structure (Java 8+)
A `HashMap` is internally an **array of buckets** (called `Node<K,V>[] table`). Each bucket can hold a linked list or a red-black tree of entries.

```
HashMap table (initial capacity = 16):
Index: [0] [1] [2] [3] [4] [5] ... [15]
              │
         Node("Alice", 90)
              │
         Node("Bob", 85)  ← collision: same bucket
```

### put() Operation — Step by Step

```java
map.put("Alice", 90);
```

1. **Compute hash**: `hash = key.hashCode()` → apply a bit-spreading function to reduce collisions.
2. **Find bucket index**: `index = hash & (table.length - 1)` (bitwise AND, equivalent to `hash % capacity`).
3. **Check bucket**:
   - If bucket is **empty** → create a new `Node` and place it there.
   - If bucket is **occupied** → traverse the linked list:
     - If a key with `.equals()` match is found → **replace** the value.
     - If no match → **append** a new node at the end.
4. **Treeify**: If a bucket's linked list exceeds **8 nodes** (and table size ≥ 64), convert it to a **red-black tree** for O(log n) lookup instead of O(n).

### get() Operation

```java
map.get("Alice");
```

1. Compute `hash("Alice")`.
2. Find bucket: `index = hash & (table.length - 1)`.
3. Traverse the bucket (linked list or tree) comparing with `.equals()`.
4. Return the value if found, `null` otherwise.

### Resizing (Rehashing)
When the number of entries exceeds `capacity × loadFactor` (default: 16 × 0.75 = 12), the table **doubles** in size (16 → 32) and ALL entries are **rehashed** into new bucket positions.

```java
new HashMap<>(initialCapacity, loadFactor);
// Default: capacity=16, loadFactor=0.75
```

### Why Load Factor Matters
- **Low load factor** (0.5): More empty buckets → fewer collisions → faster lookups → more memory.
- **High load factor** (1.0): More collisions → slower lookups → less memory.
- **Default (0.75)**: Good balance between time and space.

### Java 8+ Optimization: Treeification

| Bucket size | Structure | Lookup |
|---|---|---|
| ≤ 8 nodes | Linked list | O(n) |
| > 8 nodes | Red-black tree | O(log n) |
| ≤ 6 nodes (after deletion) | Untreeified back to list | O(n) |

> [!CAUTION]
> **Mutable keys break HashMap.** If you use a mutable object as a HashMap key and modify it after insertion, its `hashCode()` changes, but the entry stays in the old bucket. It becomes unreachable — a silent memory leak. Always use immutable objects (String, Integer, records) as keys.
