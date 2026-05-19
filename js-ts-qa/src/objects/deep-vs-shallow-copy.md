# Q: What's the difference between a shallow copy and a deep copy in JavaScript? What are the trade-offs of each cloning technique?

**Answer:**

A **shallow copy** duplicates only the top-level structure: nested objects/arrays are shared by reference between original and clone. A **deep copy** recursively duplicates every nested value, so mutating the clone never affects the original.

### Visualizing the Difference

```
   original          shallow copy          deep copy
   +----------+      +----------+          +----------+
   | a: 1     |      | a: 1     |          | a: 1     |
   | b: *-----+--+   | b: *-----+--+       | b: *-----+--+
   +----------+  |   +----------+  |       +----------+  |
                 v                  v                     v
              +------+   (same)  +------+               +------+
              | x: 1 |  <--------| x: 1 |   (NEW)       | x: 1 |
              +------+           +------+               +------+
```

### Shallow Copy Techniques

```javascript
const a = { x: 1, nested: { y: 2 } };

const b = { ...a };               // spread
const c = Object.assign({}, a);   // Object.assign
const d = Array.from(arr);        // for arrays
const e = arr.slice();            // for arrays

b.nested.y = 99;
console.log(a.nested.y); // 99 — shared reference!
```

### Deep Copy Techniques

1. **`structuredClone`** — built into modern browsers and Node 17+. The right default.
   ```javascript
   const deep = structuredClone(original);
   ```
   Handles cycles, Maps, Sets, Dates, RegExps, ArrayBuffers, typed arrays. Cannot clone functions, DOM nodes (mostly), class prototype identity, or symbol-keyed properties (with caveats).

2. **`JSON.parse(JSON.stringify(obj))`** — quick and ubiquitous but lossy.
   - Loses `undefined`, functions, symbols.
   - Converts `Date` to string, `NaN`/`Infinity` to `null`.
   - Throws on cycles.
   - Drops the prototype chain.

3. **Library helpers** — `lodash.cloneDeep` is battle-tested if `structuredClone` isn't available.

4. **Hand-rolled recursion** — only when you need custom semantics (e.g., skip certain keys, preserve class identity).

### Trade-off Table

| Technique             | Speed     | Handles cycles | Preserves types          | Loses           |
|-----------------------|-----------|----------------|--------------------------|-----------------|
| `{...obj}` / spread   | Fastest   | Shallow only   | Top-level only           | Nested sharing  |
| `JSON.parse(stringify)` | Medium  | No (throws)    | No (Date/Map/etc lost)   | Functions, undefined |
| `structuredClone`     | Fast      | Yes            | Maps/Sets/Date/regex/etc | Functions, DOM  |
| `lodash.cloneDeep`    | Slower    | Yes            | Most types               | Some custom classes |

> [!NOTE]
> Prefer **shallow + immutability discipline** in apps using Redux/Zustand. Most state libraries assume new references at every level you change, so a controlled spread-per-level pattern is faster than a global deep clone.

### Common Bug

```javascript
function reset(state) {
  const copy = { ...state };
  copy.user.name = '';      // mutates state.user.name too!
  return copy;
}
```

Either deep-clone, or spread layer-by-layer:

```javascript
return { ...state, user: { ...state.user, name: '' } };
```

### When Each Matters

- React/Redux state updates — use targeted shallow clones to maintain referential equality.
- Caching API responses — `structuredClone` to fully isolate cached values from callers.
- Sending data to a Worker via `postMessage` — the structured clone algorithm is applied automatically.
