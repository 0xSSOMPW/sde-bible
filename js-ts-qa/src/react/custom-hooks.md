# Q: What are custom React hooks and what are the rules of hooks? Show examples of useful custom hooks.

**Answer:**

A **custom hook** is any JavaScript function whose name starts with `use` and which calls other hooks. It is a mechanism for **reusing stateful logic** (not UI) between components, replacing the older mixin/HOC/render-prop patterns.

### The Rules of Hooks

1. **Call hooks at the top level.** Never inside loops, conditions, or nested functions.
2. **Call hooks only from React function components or other custom hooks** — not from regular functions or class methods.

Why? React identifies which state slot belongs to which `useState` by **call order**, not by name. Any conditional or out-of-order call shifts subsequent slots and corrupts state. The ESLint plugin `eslint-plugin-react-hooks` enforces both rules.

```javascript
// Wrong
function Comp({ enabled }) {
  if (enabled) {
    const [v, setV] = useState(0); // call order changes when `enabled` flips
  }
}

// Right
function Comp({ enabled }) {
  const [v, setV] = useState(0);
  if (!enabled) return null;
  // use v
}
```

### Custom Hooks Are Just Functions

Two components calling the same custom hook get **independent state**. The hook is the *recipe*; the components are the *instances*.

```javascript
function useToggle(initial = false) {
  const [on, setOn] = useState(initial);
  const toggle = useCallback(() => setOn(o => !o), []);
  return [on, toggle];
}
```

### Useful Custom Hooks

**1. `useDebouncedValue`** — debounce a value:

```javascript
function useDebouncedValue(value, delay) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}
```

**2. `useLocalStorage`** — sync state to localStorage:

```javascript
function useLocalStorage(key, initial) {
  const [val, setVal] = useState(() => {
    try { return JSON.parse(localStorage.getItem(key)) ?? initial; }
    catch { return initial; }
  });
  useEffect(() => { localStorage.setItem(key, JSON.stringify(val)); }, [key, val]);
  return [val, setVal];
}
```

**3. `usePrevious`** — read the previous value of a prop or state:

```javascript
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; });
  return ref.current;
}
```

**4. `useEventListener`** — attach a listener with auto-cleanup:

```javascript
function useEventListener(target, event, handler) {
  const saved = useRef(handler);
  useEffect(() => { saved.current = handler; }, [handler]);
  useEffect(() => {
    const fn = e => saved.current(e);
    target.addEventListener(event, fn);
    return () => target.removeEventListener(event, fn);
  }, [target, event]);
}
```

The `ref` trick is React's idiomatic way to read the *latest* handler from inside an effect without re-binding the listener on every render.

**5. `useIsMounted`** — guard against setState after unmount:

```javascript
function useIsMounted() {
  const ref = useRef(false);
  useEffect(() => {
    ref.current = true;
    return () => { ref.current = false; };
  }, []);
  return () => ref.current;
}
```

**6. `useFetch` (simplified)** — but prefer TanStack Query for real apps:

```javascript
function useFetch(url) {
  const [state, set] = useState({ status: 'idle', data: null, error: null });
  useEffect(() => {
    const ctrl = new AbortController();
    set(s => ({ ...s, status: 'loading' }));
    fetch(url, { signal: ctrl.signal })
      .then(r => r.json())
      .then(data => set({ status: 'success', data, error: null }))
      .catch(error => { if (error.name !== 'AbortError') set({ status: 'error', data: null, error }); });
    return () => ctrl.abort();
  }, [url]);
  return state;
}
```

> [!NOTE]
> Always provide an abort/cleanup path in custom hooks that start async work — otherwise unmounting during a request leaks setState calls and listeners.

### Composition

Custom hooks compose just like functions. A `useUser` can call `useFetch`, which calls `useEffect`, etc. The state-slot mechanism preserves correctness as long as the call order within each invocation is deterministic.

### Common Mistakes

- **Returning new object references each render** — destructured consumers re-render unnecessarily. Wrap return objects in `useMemo` if components destructure them and rely on referential equality.
- **Calling hooks inside callbacks** — `onClick={() => useState(...)}` is illegal.
- **Forgetting dependencies** — let the lint rule auto-fix; ignoring it leads to stale closures.
- **Hidden side effects in the body** — anything not wrapped in `useEffect` runs during render and can cause double-execution in StrictMode.
