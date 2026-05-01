# Q: What is the difference between List, Set, and Map?

**Answer:**

These are the three core interfaces of the Java Collections Framework.

### List — Ordered, Duplicates Allowed
An **ordered** collection (sequence). Elements are indexed by position and maintain insertion order. Duplicates are allowed.

```java
List<String> list = new ArrayList<>();
list.add("Alice");
list.add("Bob");
list.add("Alice");  // ✅ Duplicates allowed
list.get(0);        // "Alice" — indexed access
// [Alice, Bob, Alice]
```

| Implementation | Backed By | Access | Insert/Delete | Use Case |
|---|---|---|---|---|
| `ArrayList` | Dynamic array | O(1) random | O(n) middle | Default choice, random access |
| `LinkedList` | Doubly-linked list | O(n) | O(1) at head/tail | Frequent insert/remove, queues |
| `CopyOnWriteArrayList` | Array (copy on write) | O(1) | O(n) copies | Multi-threaded reads, rare writes |

### Set — No Duplicates
An **unordered** collection (by default) that does not allow duplicate elements.

```java
Set<String> set = new HashSet<>();
set.add("Alice");
set.add("Bob");
set.add("Alice");  // ❌ Ignored — already exists
// [Bob, Alice] — no guaranteed order
```

| Implementation | Ordered? | Sorted? | Performance | Use Case |
|---|---|---|---|---|
| `HashSet` | ❌ | ❌ | O(1) add/contains | Default choice |
| `LinkedHashSet` | ✅ Insertion order | ❌ | O(1) | When order matters |
| `TreeSet` | ✅ Sorted order | ✅ | O(log n) | Sorted, range queries |

### Map — Key-Value Pairs
Stores **key-value** pairs. Keys must be unique; values can be duplicated.

```java
Map<String, Integer> map = new HashMap<>();
map.put("Alice", 90);
map.put("Bob", 85);
map.put("Alice", 95);  // Overwrites previous value for "Alice"
map.get("Alice");       // 95
```

| Implementation | Ordered? | Sorted? | Thread-safe? | Use Case |
|---|---|---|---|---|
| `HashMap` | ❌ | ❌ | ❌ | Default choice |
| `LinkedHashMap` | ✅ Insertion order | ❌ | ❌ | LRU cache, ordered iteration |
| `TreeMap` | ✅ Sorted by key | ✅ | ❌ | Sorted keys, range queries |
| `ConcurrentHashMap` | ❌ | ❌ | ✅ | Multi-threaded access |

### Quick Decision Guide
```
Need indexed access?          → List (ArrayList)
Need uniqueness?              → Set (HashSet)
Need key-value lookup?        → Map (HashMap)
Need ordered + unique?        → LinkedHashSet or TreeSet
Need sorted + key-value?      → TreeMap
Need thread-safe map?         → ConcurrentHashMap
```
