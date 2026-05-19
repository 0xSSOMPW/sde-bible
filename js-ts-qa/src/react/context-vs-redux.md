# Q: When should you use React Context vs a state management library like Redux/Zustand?

**Answer:**

`React.Context` and external state managers (Redux Toolkit, Zustand, Jotai, Recoil) solve overlapping but distinct problems. Context is a **dependency injection** mechanism, not a state engine; treating it as one leads to performance issues.

### What Context Actually Does

Context propagates a value down the tree without prop drilling. When the `Provider`'s `value` reference changes, **every consumer re-renders** — there is no built-in selector or partial subscription.

```javascript
const ThemeCtx = createContext('light');
function App() { return <ThemeCtx.Provider value="dark"><Page/></ThemeCtx.Provider>; }
function Btn() { const theme = useContext(ThemeCtx); /* re-renders on any value change */ }
```

### What State Libraries Add

| Feature                       | Context | Redux Toolkit | Zustand | Jotai |
|--------------------------------|---------|---------------|---------|-------|
| Selector-based subscriptions   | No      | Yes (`useSelector`) | Yes (`useStore(s => s.x)`) | Atomic |
| Reference equality opt-in      | No      | Yes           | Yes (`shallow`) | N/A |
| Time-travel / DevTools        | No      | Yes           | Yes (via DevTools middleware) | Limited |
| Middleware (logging, async)    | No      | Yes (thunks, RTK Query) | Yes | No (uses atoms + effects) |
| Boilerplate                    | None    | Higher        | Low     | Low   |
| Bundle size                    | 0       | ~10 KB        | ~1 KB   | ~3 KB |

### Rule of Thumb

```
   Is the value rarely-changing config (theme, locale, current user, feature flags)?
        |-- Yes --> Context
        |
   Is the value updated frequently or read by many components?
        |-- Yes --> external store (Redux/Zustand/Jotai)
        |
   Is it server data (lists, entities, requests)?
        |-- Yes --> data-fetching cache (TanStack Query, RTK Query, SWR)
```

> [!NOTE]
> The single most common React perf bug in 2025 codebases: pushing rapidly-changing values (form input strings, mouse positions) through Context. Every keystroke re-renders the entire subtree under the Provider.

### Splitting Context to Mitigate

If you must use Context for a changing value, split by *change rate*:

```javascript
const StateCtx    = createContext(null);   // changes often
const DispatchCtx = createContext(null);   // stable reference

<StateCtx.Provider value={state}>
  <DispatchCtx.Provider value={dispatch}>  {/* never re-renders */}
    {children}
  </DispatchCtx.Provider>
</StateCtx.Provider>
```

Components that only call `dispatch` subscribe only to `DispatchCtx` and avoid the noise.

### Redux Toolkit — When It Earns Its Keep

- Complex domain logic with many reducers, selectors, and asynchronous workflows.
- Need for serializable state, replayable actions, audit logs.
- Team is large and benefits from explicit action contracts.
- Server cache management with RTK Query.

```javascript
const slice = createSlice({
  name: 'cart',
  initialState: { items: [] },
  reducers: {
    add(state, { payload }) { state.items.push(payload); },
    remove(state, { payload }) {
      state.items = state.items.filter(i => i.id !== payload);
    },
  },
});
```

Immer powers the "mutate state" syntax safely.

### Zustand — Pragmatic Middle Ground

```javascript
import { create } from 'zustand';
const useCart = create((set) => ({
  items: [],
  add: (item) => set(s => ({ items: [...s.items, item] })),
}));

function Count() {
  return <span>{useCart(s => s.items.length)}</span>; // re-renders only when count changes
}
```

No Provider needed; selectors give fine-grained subscriptions; full TS inference.

### Jotai / Recoil — Atomic State

State is decomposed into small *atoms*. Components subscribe to only the atoms they read, getting maximally fine-grained re-renders.

```javascript
const countAtom = atom(0);
function Counter() { const [n, set] = useAtom(countAtom); }
```

Useful for highly interactive UIs (graph editors, design tools) where the bottleneck is propagating updates.

### Server State Is Different

Caching, refetching, mutations, optimistic updates, and revalidation belong in **TanStack Query / SWR**, not in Redux. Mixing the two leads to duplicated, drifting data. Treat the server as the source of truth and the client store for *truly* client-only state (UI flags, drafts, selection).
