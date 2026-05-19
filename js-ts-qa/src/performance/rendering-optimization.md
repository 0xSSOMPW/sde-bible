# Q: How do you optimize React rendering performance? When should you reach for memo, useMemo, useCallback?

**Answer:**

React's default behavior is to re-render a component whenever its parent re-renders. Most of the time this is fast — diffing virtual DOM is cheap. Optimization should be **measured first, not preemptive.** Premature `useMemo` everywhere is itself a perf bug (extra closures, dep arrays, complexity).

### Step Zero: Measure

Use the React DevTools **Profiler** tab. Record an interaction, look at the flamegraph, and find components whose render time is large or whose render count is unexpected. The browser's Performance tab gives you long-task and scripting cost.

### The Three Memoization Tools

| Tool          | What it does                                  | Cost                   |
|---------------|-----------------------------------------------|------------------------|
| `React.memo`  | Skips a child render if props are shallow-equal | Comparison + extra HOC |
| `useMemo`     | Caches a value (object/array/computed result) | Comparison + closure   |
| `useCallback` | Caches a function identity                    | Same as useMemo        |

`useMemo` and `useCallback` are about **stabilizing references** to satisfy `React.memo` or hook dependency arrays — they are not magic speed boosters.

### When `React.memo` Actually Helps

A child wrapped in `memo` only avoids work if its props are *referentially stable* across parent renders. So `memo` and `useCallback`/`useMemo` come as a package:

```jsx
const Row = React.memo(function Row({ item, onSelect }) { /* ... */ });

function List({ items, onSelect }) {
  const onRowSelect = useCallback(id => onSelect(id), [onSelect]);
  return items.map(it => <Row key={it.id} item={it} onSelect={onRowSelect} />);
}
```

Without `useCallback`, `onRowSelect` would be a new function each render and `memo` would never bail out.

> [!NOTE]
> If the only prop is `key + primitive`, `memo` likely pays off. If props include big objects rebuilt every render, you need `useMemo` on them too — otherwise `memo` does nothing.

### Avoid Unnecessary Re-renders At The Source

Before reaching for memo, ask: *can I avoid the re-render entirely?*

1. **Move state down.** A parent that owns input state re-renders the whole subtree on every keystroke. Move the input + state into a child.
2. **Lift props as `children`.** Components passed via `children` to a parent don't re-render when that parent re-renders, because the `children` element reference is preserved by the *grandparent*.
   ```jsx
   <Layout><ExpensiveTree/></Layout>
   ```
   If `Layout` updates its own state, `ExpensiveTree` skips re-render because `children` is the same element.
3. **Split contexts** by change rate (see Context vs Redux). High-frequency values shouldn't share a provider with low-frequency ones.

### Big Lists — Virtualization

Rendering thousands of rows always hurts. Use windowing (`react-virtual`, `react-window`) to render only what's visible:

```
   viewport
   ┌──────────────┐
   │ visible rows │ ← rendered (~20 nodes)
   ├──────────────┤
   │ spacer       │ ← height matches off-screen rows
   └──────────────┘
```

DOM nodes drop from O(n) to O(viewport size) regardless of list length.

### Other Levers

- **Code splitting** with `React.lazy(() => import('./Big'))` + `<Suspense>` to defer heavy modules until needed.
- **Concurrent transitions** — wrap heavy state updates in `startTransition` so input stays responsive.
- **`useDeferredValue`** — let a slow subtree lag behind a fast input.
- **Move work off main thread** with a Web Worker when CPU-bound.
- **Avoid layout thrash** — batch DOM reads/writes; don't toggle classes inside a tight loop that triggers reflow.

### Common Anti-Patterns

```jsx
// 1. useMemo on a primitive — useless
const total = useMemo(() => items.length, [items]);

// 2. useCallback that's never compared — useless
const onClick = useCallback(() => doThing(), []);
return <button onClick={onClick}>x</button>;

// 3. memo with new prop objects every render — useless
<MemoChild config={{ size: 10 }} />   // {} is new each time
```

### Decision Tree

```
   Profiler shows slow component?
       |
       +-- yes -> Can I reduce work or split it? (move state down, children prop)
       |             |
       |             +-- still slow -> memoize stable props (useCallback/useMemo)
       |                              + wrap child in React.memo
       |
       +-- no  -> stop optimizing
```

### React Compiler (2025+)

The React Compiler (formerly "React Forget") auto-memoizes components and dependencies at compile time. With it enabled, hand-written `useMemo`/`useCallback` become largely unnecessary. Adopt it gradually and remove redundant memoization once you've verified equivalent behavior in production.
