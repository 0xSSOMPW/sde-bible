# Q: What design patterns come up frequently in JavaScript? Give examples for module, singleton, observer, factory, and strategy.

**Answer:**

Classic GoF patterns translate to JS, but most look quite different because the language has first-class functions, prototypes, dynamic typing, and modules. Idiomatic JS leans on **closures**, **higher-order functions**, and **dependency injection** more than on the explicit class hierarchies of Java or C++.

### 1. Module Pattern

Encapsulate private state behind a public API. Pre-ESM idiom uses IIFE + closures; today ES modules give it natively.

```javascript
// IIFE version (legacy)
const Counter = (function () {
  let count = 0;
  return {
    inc() { count++; },
    get() { return count; },
  };
})();

// ESM version — the file itself is the module
let count = 0;
export const inc = () => count++;
export const get = () => count;
```

Each importer sees the **same** module instance (modules are singletons at the loader level).

### 2. Singleton

A single instance shared across the program. In JS, an ES module already behaves like a singleton, so writing an explicit `Singleton` class is often unnecessary.

```javascript
// db.js
let instance;
export function getDb() {
  if (!instance) instance = createConnection();
  return instance;
}
```

> [!NOTE]
> Be cautious — global singletons make testing harder, hide dependencies, and break in environments that spin multiple module realms (SSR, workers). Prefer dependency injection where you can.

### 3. Observer (Pub/Sub, EventEmitter)

A producer notifies many subscribers without knowing who they are. Foundation of DOM events, Node's `EventEmitter`, and reactive libraries.

```javascript
class Emitter {
  constructor() { this.listeners = new Map(); }
  on(event, fn) {
    const set = this.listeners.get(event) ?? new Set();
    set.add(fn);
    this.listeners.set(event, set);
    return () => set.delete(fn); // unsubscribe handle
  }
  emit(event, payload) {
    this.listeners.get(event)?.forEach(fn => fn(payload));
  }
}
```

Returning the unsubscribe function is the standard ergonomic touch; it removes the need for a separate `off`.

### 4. Factory

A function that returns objects, hiding which concrete implementation is chosen. Reduces `new` coupling and enables polymorphism.

```javascript
function makeLogger(env) {
  if (env === 'prod')  return { log: msg => sendToCloud(msg) };
  if (env === 'test')  return { log: () => {} };
  return                       { log: msg => console.log(msg) };
}

const log = makeLogger(process.env.NODE_ENV);
log.log('hello');
```

Factories pair beautifully with closures — the returned object carries private state.

### 5. Strategy

Encapsulate interchangeable algorithms behind a common interface. Trivial in JS because functions are values.

```javascript
const strategies = {
  fifo: q => q.shift(),
  lifo: q => q.pop(),
  priority: q => q.sort((a, b) => a.p - b.p).shift(),
};

function process(queue, strategyName) {
  const next = strategies[strategyName] ?? strategies.fifo;
  while (queue.length) handle(next(queue));
}
```

### Other Patterns Worth Knowing

**Decorator (function form):**

```javascript
const withTiming = fn => async (...args) => {
  const t = performance.now();
  try { return await fn(...args); }
  finally { console.log(fn.name, performance.now() - t, 'ms'); }
};
const timedFetch = withTiming(fetchUser);
```

**Adapter / Facade:** wrap a verbose API in a simpler one.

```javascript
const storage = {
  get: key => JSON.parse(localStorage.getItem(key) ?? 'null'),
  set: (key, v) => localStorage.setItem(key, JSON.stringify(v)),
};
```

**Command:** package an action as data — undo/redo systems, queue workers.

```javascript
const command = { type: 'MOVE', from: [0, 0], to: [3, 4] };
dispatch(command);
```

**Iterator:** the language gives you it for free via `Symbol.iterator` and generators (see Generators chapter).

### Anti-Patterns to Avoid

- **God objects** — single 5000-line module that everyone imports.
- **Stringly typed Strategy** — `do('fifo')` with no type safety. In TS, use a union literal.
- **Subclass abuse** — JS classes encourage tall hierarchies. Prefer composition (`function compose(...fs)`).
- **Singleton + global mutable state** — turns testing into fixture archaeology.

### Why JS Has Fewer "Pattern Books"

Because functions are first-class and modules already provide encapsulation, many GoF patterns degenerate to "just write a function" or "use closure". The patterns above still apply; the rest are often over-engineering in idiomatic JavaScript.
