# Q: When should you use a Map over a plain Object in JavaScript? What about Set vs Array?

**Answer:**

`Map` and `Set` were introduced in ES2015 to address long-standing gaps in using plain `Object` and `Array` as hash tables and uniqueness containers.

### Map vs Object

| Concern                 | `Map`                                     | `Object`                              |
|--------------------------|-------------------------------------------|---------------------------------------|
| Key types                | **Any** value (objects, functions, NaN)   | Only strings / symbols                |
| Insertion order          | Guaranteed across all iterations          | Mostly guaranteed for string keys     |
| Size                     | `map.size` — O(1)                         | `Object.keys(obj).length` — O(n)      |
| Default prototype keys   | None (truly empty)                        | Inherits `toString`, `hasOwnProperty`, etc. |
| Iteration                | Built-in iterator: `for...of map`         | Need `Object.entries(obj)` first      |
| Performance for frequent add/delete | Optimized                       | Slower; hidden-class churn in V8      |
| Serialization            | No native JSON support                    | `JSON.stringify` works out of the box |

```javascript
const obj = {};
obj['toString'];      // [Function: toString] — inherited!
obj['__proto__'];     // dangerous key

const map = new Map();
map.set('toString', 1);
map.get('toString');  // 1 — no inheritance leak

const userKey = { id: 42 };
map.set(userKey, 'admin');     // object as key — impossible with plain {}
```

> [!NOTE]
> Use a `Map` when keys are dynamic, untrusted user input, or non-strings. Use an `Object` when keys are a known fixed set known at code time (a struct).

### When Object Is Still Better

- Static, known-at-development keys (config records, DTOs).
- JSON-compatible payloads.
- Better destructuring ergonomics: `const { id, name } = user`.
- Tooling/IDE support for property names.

### Set vs Array

| Concern             | `Set`                              | `Array`                       |
|----------------------|------------------------------------|-------------------------------|
| Uniqueness          | Enforced by definition             | Manual (`indexOf`, `includes`)|
| Lookup              | `has(x)` — O(1)                    | `includes(x)` — O(n)          |
| Order               | Insertion order                    | Insertion order               |
| Random access       | No (`set[0]` doesn't work)         | Yes (`arr[0]`)                |
| Dup detection cost  | Free                               | O(n^2) loop, or extra Set     |

```javascript
const uniqueIds = [...new Set(ids)];       // de-dupe an array

const visited = new Set();
function dfs(node) {
  if (visited.has(node)) return;
  visited.add(node);
  node.children.forEach(dfs);
}
```

### WeakMap and WeakSet

When **keys are objects** and you want garbage collection to reclaim entries once nothing else references the key, use `WeakMap`/`WeakSet`.

- Keys must be objects (or, in modern engines, registered symbols).
- Not iterable; no `size`.
- Ideal for cache/metadata keyed by DOM nodes or per-request objects.

```javascript
const meta = new WeakMap();
function tag(node, data) { meta.set(node, data); }
// When the DOM node is removed and GC'd, meta entry disappears automatically.
```

### Decision Cheatsheet

```
   keys are arbitrary or non-string?         -> Map
   keys are objects and GC should clean up?  -> WeakMap
   need uniqueness + fast contains?          -> Set
   keys live in DOM/runtime objects?         -> WeakSet / WeakMap
   static known fields, serialized to JSON?  -> Object
   ordered random access by index?           -> Array
```
