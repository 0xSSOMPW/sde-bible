# Q: What are common causes of memory leaks in JavaScript? How do you debug them?

**Answer:**

A **memory leak** in JS is memory that the GC cannot reclaim because something still holds a reference to it — usually unintentionally. The V8 garbage collector frees objects only when nothing reachable from the root set (globals, the current call stack, active closures) refers to them.

### Top Five Leak Patterns

**1. Accidental Globals**

```javascript
function init() {
  cache = {};        // missing `let`/`const`/`var` — attaches to globalThis
}
```

In strict mode this throws. In sloppy mode the variable lives forever on `window`. Always use strict mode and `"use strict"` modules.

**2. Forgotten Timers and Intervals**

```javascript
class Widget {
  constructor() {
    setInterval(() => this.tick(), 1000); // keeps `this` alive forever
  }
}
```

If the widget is removed from the DOM, the interval still references `this`, so the entire instance — and anything it transitively references — survives. Always store the handle and `clearInterval` on teardown.

**3. Detached DOM Nodes Held by JS**

```javascript
const cached = document.getElementById('panel');
document.body.removeChild(cached); // removed from DOM, still referenced
```

The element is "detached" — out of the DOM tree but pinned in memory because a JS variable holds it. Common in SPA frameworks if you cache DOM nodes globally.

**4. Listeners Without Cleanup**

```javascript
window.addEventListener('resize', this.onResize);
// component removed, but window still holds onResize -> the entire component
```

Always pair `addEventListener` with `removeEventListener` (or use `AbortController.signal` to detach all listeners atomically). React's `useEffect` cleanup is the canonical place.

**5. Closures Pinning Large Scope**

```javascript
function attach(big) {
  return function () { console.log('hi'); }; // doesn't use `big`...
}
```

V8 generally trims unused closure variables, but if any function in the same scope references `big`, all closures in that scope keep it alive. Move large values out of shared scope or null them when done.

### Debug Workflow in Chrome DevTools

```
   1. Memory tab -> "Heap snapshot" before suspect action
   2. Perform action that should free memory (e.g., navigate away)
   3. Force GC (trash-can icon)
   4. Take a second snapshot
   5. Compare snapshots -> filter by "Objects allocated between"
   6. Look for retained "Detached" DOM nodes, your custom classes,
      arrays growing unbounded
   7. Click an object -> "Retainers" pane shows what is pinning it
```

> [!NOTE]
> A single suspicious snapshot is not enough. Use the **three-snapshot technique**: idle, action, idle. Memory that grows and never shrinks across cycles is a leak.

### Allocation Timeline & Performance Profile

The **Allocation instrumentation on timeline** highlights bars where new objects survived multiple GCs — strong leak indicators. The Performance panel's "JS heap" line should sawtooth around a stable baseline; a steadily rising baseline equals a leak.

### Server-Side (Node.js)

- Use `--inspect` and Chrome DevTools to take heap snapshots of a running Node process.
- Tools: `heapdump`, `clinic.js doctor`, `node --heap-prof`.
- Watch out for module-level caches (`const cache = new Map()` at top level) that grow unboundedly under load — bound them with LRU.

### Prevention Checklist

- Use `WeakMap`/`WeakSet` for caches keyed by objects.
- Use `AbortController` for fetch/event listeners with a clear lifetime.
- Avoid global singletons holding per-request data.
- In React: cleanup in `useEffect`, never store DOM refs in module scope.
- Bound caches (LRU) and intern data deliberately.
