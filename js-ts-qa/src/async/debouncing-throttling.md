# Q: What is the difference between debouncing and throttling? When would you use each?

**Answer:**

Both are techniques to **rate-limit** how often a function runs in response to a high-frequency event (scroll, resize, keypress, mousemove). They differ in *which* invocations get through.

- **Debounce:** wait until the event has stopped firing for `N` ms, *then* run once.
- **Throttle:** allow the function to run at most once every `N` ms, regardless of how many events fire.

### Visual Timeline

```
events:   x x x x x   x x   x x x x x x   x      x
debounce  -----------|-----|-------------|-----|-X      (fires only at end of bursts)
throttle  |---|---|---|---|---|---|---|---|---|---|     (steady cadence)
```

### Use Cases

| Situation                                | Pick      |
|------------------------------------------|-----------|
| Search-as-you-type API call              | Debounce  |
| Save draft after typing stops            | Debounce  |
| Window resize layout recalculation       | Debounce  |
| Scroll-driven analytics or infinite scroll | Throttle |
| Mousemove drag preview                   | Throttle  |
| Buttons protected from double-click      | Throttle (leading) |

### Implementation — Debounce

```javascript
function debounce(fn, wait) {
  let timer;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), wait);
  };
}

const onSearch = debounce(query => api.search(query), 300);
input.addEventListener('input', e => onSearch(e.target.value));
```

Each new event resets the timer; only the **last** invocation in a burst actually runs after the wait window.

### Implementation — Throttle (leading + trailing)

```javascript
function throttle(fn, wait) {
  let last = 0;
  let timer;
  return function (...args) {
    const now = Date.now();
    const remaining = wait - (now - last);
    if (remaining <= 0) {
      clearTimeout(timer);
      timer = null;
      last = now;
      fn.apply(this, args);
    } else if (!timer) {
      timer = setTimeout(() => {
        last = Date.now();
        timer = null;
        fn.apply(this, args);
      }, remaining);
    }
  };
}
```

This variant fires immediately on the first call (leading edge) and ensures a final trailing call so the last event isn't dropped.

> [!NOTE]
> Libraries like Lodash expose `_.debounce(fn, wait, { leading, trailing, maxWait })`. The `maxWait` option turns debounce into a "throttle floor", guaranteeing the function runs at least every `maxWait` ms even during continuous activity.

### React Gotcha

Defining the debounced function inside a component creates a *new* debounced wrapper on every render — defeating the timer. Wrap with `useMemo` or `useCallback`:

```javascript
const onSearch = useMemo(
  () => debounce(q => api.search(q), 300),
  []
);
useEffect(() => () => onSearch.cancel?.(), [onSearch]);
```

Remember to cancel pending timers on unmount to avoid setting state on an unmounted component.

### When to Pick Which

Ask: *do I need every Nth event, or only the final event after silence?*

- "User stopped typing" — debounce.
- "Update the UI at a smooth 60fps while user drags" — throttle to 16ms.
- "Don't double-submit when user mashes the button" — throttle leading with no trailing.
