# Q: What are the differences between Promise.all, Promise.allSettled, Promise.race, and Promise.any?

**Answer:**

All four are **promise combinators** — static methods that take an iterable of promises and return a single promise. They differ in *when* they settle and *what* they resolve or reject with.

### Quick Comparison

| Method                 | Resolves when                           | Rejects when                          | Result                          |
|------------------------|------------------------------------------|---------------------------------------|---------------------------------|
| `Promise.all`          | **All** fulfill                          | **Any** rejects (fail-fast)           | Array of values                 |
| `Promise.allSettled`   | **All** settle (fulfilled or rejected)   | Never                                 | Array of `{status, value/reason}` |
| `Promise.race`         | **First** to settle (either way)         | First rejection                       | Single value or reason          |
| `Promise.any`          | **First** fulfillment                    | All reject (with `AggregateError`)    | Single value                    |

### Promise.all — fail-fast aggregation

```javascript
const [user, posts, settings] = await Promise.all([
  fetchUser(),
  fetchPosts(),
  fetchSettings(),
]);
```

If any single promise rejects, the entire `Promise.all` rejects immediately. The other promises **keep running** in the background (they aren't cancelled), but their results are discarded.

> [!NOTE]
> Use `Promise.all` only when partial failure is unacceptable. If a single optional resource fails, the whole batch is lost.

### Promise.allSettled — never rejects

Introduced in ES2020. Waits for every promise regardless of outcome.

```javascript
const results = await Promise.allSettled([fetchA(), fetchB(), fetchC()]);
results.forEach(r => {
  if (r.status === 'fulfilled') console.log('value:', r.value);
  else                          console.error('reason:', r.reason);
});
```

Ideal for dashboards where you want to render whatever succeeded and show errors for the rest.

### Promise.race — first to finish

Returns whichever promise settles first, success or failure. Common for adding a timeout:

```javascript
function withTimeout(promise, ms) {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error('timeout')), ms)
  );
  return Promise.race([promise, timeout]);
}
```

### Promise.any — first success

Introduced in ES2021. Resolves on the first fulfillment; rejects only if **every** promise rejects, with an `AggregateError` containing all reasons.

```javascript
try {
  const fastest = await Promise.any([
    fetch('https://mirror1/api'),
    fetch('https://mirror2/api'),
    fetch('https://mirror3/api'),
  ]);
} catch (e) {
  // e instanceof AggregateError; e.errors is the list of reasons
}
```

### Decision Flow

```
   need every result?
        |
        +-- yes --> tolerate partial failure?
        |              |
        |              +-- yes --> Promise.allSettled
        |              +-- no  --> Promise.all
        |
        +-- no  --> need any success?
                       |
                       +-- yes --> Promise.any
                       +-- no  --> Promise.race  (first settle wins)
```

### Common Pitfalls

- `Promise.all([])` resolves immediately with `[]`. `Promise.any([])` **rejects** with empty `AggregateError`.
- Non-promise values are wrapped via `Promise.resolve(value)` — passing raw numbers is legal.
- Combinators **do not cancel** in-flight work. Use `AbortController` to actually abort fetches when a race winner is determined.
