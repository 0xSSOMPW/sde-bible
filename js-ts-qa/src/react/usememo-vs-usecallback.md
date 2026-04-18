# Q: What's the difference between `useMemo` and `useCallback`? When are they not worth using?

**Answer:**

Both `useMemo` and `useCallback` are React Hooks used for performance optimization (memoization). They both take a dependency array and re-compute only when those dependencies change. 

Their core difference lies in **what they return**:
*   `useMemo` caches the **result** of a function calculation.
*   `useCallback` caches the **function itself** (the reference to the function).

Essentially, `useCallback(fn, deps)` is exactly equivalent to `useMemo(() => fn, deps)`.

### 1. `useMemo` Use Cases
Use it to avoid repeating expensive, time-consuming calculations on every single render.

```javascript
// ✅ GOOD USE CASE: Expensive calculation
const expensiveResult = useMemo(() => {
    return bigArray.filter(item => item.value > threshold).map(heavyTransformation);
}, [bigArray, threshold]);
```

### 2. `useCallback` Use Cases
Use it when you need to keep a function reference completely stable between renders. This is almost exclusively needed when you are passing a callback function as a prop to a deeply nested child component that is wrapped in `React.memo()`, or if the function is used in a `useEffect` dependency array.

```javascript
// ✅ GOOD USE CASE: Passing to a pure child component
const handleSubmit = useCallback((data) => {
    api.submit(data);
}, []); // Function reference never changes

// MemoizedChild will NOT re-render since `handleSubmit` is identical across renders
return <MemoizedChild onSubmit={handleSubmit} />;
```

---

### When is it NOT worth it? (The "Gotcha")
A massive red flag in interviews is developers who wrap *every* function in `useCallback` and *every* variable in `useMemo`. 

**React components are designed to tear down and rebuild extremely fast.** Over-memoizing actually hurts performance because caching results and tracking dependency arrays takes up more memory and execution time than just letting React do its normal job.

**DON'T use them when:**
1.  **The operation is cheap:** Doing basic math or string concatenation? Don't use `useMemo`. `const fullName = firstName + lastName` is thousands of times faster to recalculate than wrapping it in `useMemo`.
2.  **Passing to native HTML elements:** Wrapping a function in `useCallback` just to pass it to `<button onClick={handleClick}>` is completely useless. Native DOM elements don't care about reference equality.
3.  **The child isn't memoized:** If you pass a `useCallback` function to a custom `<Button>` component, but that `<Button>` component is NOT wrapped in `React.memo`, the child is going to re-render anyway. You just wasted memory caching the function!

> [!TIP]
> **Interview Rule of Thumb:** Write the code without `useMemo` or `useCallback` first. Only add them if you explicitly identify a performance bottleneck (like a 500ms lag on typing) or a `useEffect` infinite loop caused by an unstable function dependency.
