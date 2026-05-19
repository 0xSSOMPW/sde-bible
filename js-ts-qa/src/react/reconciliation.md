# Q: How does React's reconciliation algorithm work? What is the Virtual DOM and how does Fiber change things?

**Answer:**

**Reconciliation** is the process of figuring out what changed between two renders and applying the minimum set of DOM mutations. React describes the desired UI as a tree of plain objects (the **Virtual DOM** / "React elements"), diffs the new tree against the previous one, and emits the diff to the host environment (DOM, native, canvas).

### The Diff Heuristic (O(n))

A full tree diff is O(n³). React makes it linear by leaning on two assumptions:

1. **Two elements of different types produce different trees.** A `<div>` swapped for a `<span>` is treated as "tear down and rebuild", not "compare children".
2. **Stable keys identify siblings across renders.** Without keys, React diffs by index; with keys, it matches by identity.

```
   Old tree                       New tree
     <ul>                            <ul>
       <li key="a"/>     match→        <li key="b"/>
       <li key="b"/>                   <li key="a"/>
                                       <li key="c"/>
```

With keys, React detects: keep `a` and `b`, just move them, insert `c`. Without keys (or with `key={index}`), it would re-render each `<li>` and rebuild internal state.

### Same-Type Component Update

```jsx
<Counter count={1} />   →   <Counter count={2} />
```

React keeps the instance, updates props, calls the component function, diffs its output. Local state (`useState`) is preserved.

### Element-Type Change Tears Down State

```jsx
{loading ? <Spinner/> : <Form/>}
```

When `loading` flips, React unmounts `Spinner` (running its cleanups) and mounts a fresh `Form`. Any state inside `Form` from a previous show is lost — it's a brand new component instance.

### Why Keys Matter So Much

Wrong keys are the most common subtle React bug. Using array index as key in a reorderable list causes:

- Inputs lose their values when items are inserted at the top.
- Animations attach to the wrong element.
- `useEffect` cleanup runs for the wrong identity.

> [!NOTE]
> A good key is a **stable, unique identifier of the underlying data**. Database IDs are ideal. Indexes are fine *only* if the list is append-only and items have no internal state.

### Fiber — Incremental Reconciliation

Prior to React 16, reconciliation was a synchronous recursive walk that blocked the main thread for large trees. **Fiber** rewrote this into a *cooperative*, interruptible scheduler.

Key ideas:

- Each component instance is a **fiber node** in a linked-list tree.
- Work is split into **units** that can be paused, resumed, or discarded.
- A **double-buffered tree** ("current" and "work-in-progress") lets React build the next render without disturbing the current one.
- The scheduler assigns **priorities** — urgent updates (input, hover) preempt low-priority ones (data fetch results, transitions).

```
   Render Phase (interruptible)
     ┌──────────────────────────────────────────┐
     │  begin work on each fiber                │
     │  build work-in-progress tree             │
     │  collect side-effects in a list          │
     └──────────────────┬───────────────────────┘
                        │
                        v
   Commit Phase (synchronous, atomic)
     ┌──────────────────────────────────────────┐
     │  apply DOM mutations                     │
     │  run layout effects (useLayoutEffect)    │
     │  schedule passive effects (useEffect)    │
     └──────────────────────────────────────────┘
```

### Concurrent Features Build on Fiber

- **`useTransition`** marks a state update as non-urgent — React can pause it to keep input responsive.
- **`useDeferredValue`** lets a heavy subtree lag behind a typing-driven input.
- **`Suspense`** lets a component "throw" a promise; React renders fallback content until the promise resolves, then continues.

### Render vs Commit

- The render phase calls your function components — it may run multiple times, can be aborted, must be **pure**.
- The commit phase applies the result. Side effects belong here (`useEffect`, `useLayoutEffect`).

### Common Perf Pitfalls That Fight the Reconciler

- New inline objects/functions every render that are then passed as props with `useMemo`-derived deps (defeats memoization).
- Conditional rendering that swaps component types (`A` vs `B`) when re-using a single component with a prop would preserve state.
- Putting whole-app state in a Context provider's `value` recreated each render — every consumer re-renders.

### What Reconciliation Does Not Do

- It does not minimize re-renders — it minimizes DOM mutations. Components still call your function every render unless memoized.
- It does not deduplicate state updates — multiple `setState` calls in the same event are batched, but state is not "diffed" before commit.
