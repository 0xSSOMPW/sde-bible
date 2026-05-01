# Q: Explain fail-fast vs fail-safe iterators. What is `ConcurrentModificationException`?

**Answer:**

### Fail-Fast
Detect concurrent modification → throw `ConcurrentModificationException` immediately.

```java
List<Integer> list = new ArrayList<>(List.of(1, 2, 3));
for (int x : list) {
    if (x == 2) list.remove(Integer.valueOf(x));  // 💥 CME on next iteration
}
```

**How it works**: collection has `modCount`. Iterator captures `expectedModCount` at creation. On every `next()`, checks `modCount == expectedModCount`. Mismatch → CME.

```java
// ArrayList.Itr#next()
final void checkForComodification() {
    if (modCount != expectedModCount)
        throw new ConcurrentModificationException();
}
```

> [!IMPORTANT]
> CME is not a guarantee — it's **best-effort**. Don't rely on it for thread safety. It's a debugging aid for incorrect single-threaded code.

### Fail-Safe
Iterate over a **snapshot** or copy → no CME, but you may see stale data.

```java
List<Integer> list = new CopyOnWriteArrayList<>(List.of(1, 2, 3));
for (int x : list) {
    if (x == 2) list.remove(Integer.valueOf(x));  // ✅ no CME
    // iterator sees the old snapshot — won't see the removal
}
```

### Fail-Fast Collections
- `ArrayList`, `LinkedList`, `HashMap`, `HashSet`, `TreeMap`, `TreeSet`, `Vector` (mostly), `Hashtable` (mostly).

### Fail-Safe Collections
- `CopyOnWriteArrayList`, `CopyOnWriteArraySet` — full snapshot on write.
- `ConcurrentHashMap` — weakly consistent iterator (sees some, not all, concurrent updates without CME).

### Correct Removal During Iteration
```java
// ❌ Wrong
for (String s : list) {
    if (s.startsWith("x")) list.remove(s);  // CME
}

// ✅ Iterator.remove()
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().startsWith("x")) it.remove();
}

// ✅ Java 8+
list.removeIf(s -> s.startsWith("x"));
```

### Why `iterator.remove()` Works
It updates `expectedModCount` along with `modCount`. Other methods (`list.remove()`) bump `modCount` only.

### Map Iteration Pitfall
```java
Map<String, Integer> map = new HashMap<>();
for (var e : map.entrySet()) {
    map.put("new", 1);     // 💥 CME
}
```
Use:
```java
map.entrySet().removeIf(e -> e.getValue() < 0);
// or replaceAll for value updates
map.replaceAll((k, v) -> v * 2);
```

### Weakly Consistent (`ConcurrentHashMap`)
- No CME ever.
- Iterator may reflect updates after creation, may not.
- Safe across threads, but iteration is not a snapshot.
