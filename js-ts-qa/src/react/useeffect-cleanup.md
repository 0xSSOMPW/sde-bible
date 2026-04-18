# Q: What does the cleanup function in `useEffect` do, and when does it run?

**Answer:**

In React, the cleanup function is the function you **return** from within a `useEffect` callback. Its primary job is to clean up side effects (like subscriptions, timers, or event listeners) to prevent memory leaks and unexpected behavior.

```javascript
useEffect(() => {
    // 1. Setup the side effect
    const timer = setInterval(() => console.log('Tick'), 1000);

    // 2. Return the cleanup function
    return () => {
        clearInterval(timer); 
    };
}, []); 
```

### When does the cleanup function run?
React runs the cleanup function in two specific scenarios:
1.  **Before an Effect runs again (on re-renders):** If your `useEffect` dependencies change and the component re-renders, React will first run the cleanup function from the *previous* render with the old state/props, and then run the newly updated effect.
2.  **When the component unmounts:** Before the component is removed from the DOM entirely, React fires the cleanup function to destroy any lingering processes.

### What happens if you forget to clean it up?
Forgetting to clean up effects (like event listeners, WebSockets, or `setInterval` timers) typically leads to **Memory Leaks**. 

If the component unmounts but a `setInterval` is still running in the background, it will continue executing forever. If that interval tries to update a React state variable (e.g., `setCount(c => c + 1)`) on an unmounted component, React used to aggressively throw a memory leak warning:
> *"Warning: Can't perform a React state update on an unmounted component. This is a no-op, but it indicates a memory leak in your application."*

Even worse, if the component is mounted and unmounted 10 times, you will now have 10 identical intervals running at the exact same time, severely degrading app performance and causing chaotic UI bugs.
