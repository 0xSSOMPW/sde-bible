# Q: Explain the JavaScript Event Loop. How does single-threaded JavaScript handle asynchronous operations?

**Answer:**

JavaScript is **single-threaded** — it has one call stack and can do exactly one thing at a time. Yet it handles network calls, timers and user input concurrently. This is possible because the runtime (browser or Node.js) provides extra machinery around the engine: a **Call Stack**, **Web/C++ APIs**, **Task Queues**, and the **Event Loop** that ties them together.

### The Big Picture

```
   +-------------------+        +-----------------------+
   |   Call Stack      |        |   Web APIs / libuv    |
   |  (synchronous)    |        |  setTimeout, fetch,   |
   |                   |        |  DOM events, I/O      |
   +---------+---------+        +-----------+-----------+
             |                              |
             | pushes/pops frames           | when work is done,
             v                              v callback is enqueued
   +-------------------+        +-----------------------+
   |     Heap          |        |   Macrotask Queue     |
   |   (objects)       |        |   (timers, I/O, UI)   |
   +-------------------+        +-----------+-----------+
                                            |
                                +-----------v-----------+
                                |   Microtask Queue     |
                                |  (promises, nextTick) |
                                +-----------+-----------+
                                            |
                                +-----------v-----------+
                                |     EVENT LOOP        |
                                |  pulls work back into |
                                |     the call stack    |
                                +-----------------------+
```

### The Loop's Algorithm

The event loop runs a simple, deterministic cycle:

1. Execute the entire current synchronous task on the call stack until it is empty.
2. Drain the **microtask queue** completely (promise reactions, `queueMicrotask`, `MutationObserver`).
3. Run **one** macrotask (timer callback, I/O completion, UI event).
4. If in a browser, possibly render a frame.
5. Repeat.

> [!NOTE]
> The microtask queue is drained to empty between each macrotask. A long chain of `.then(...)` callbacks can starve timers and rendering.

### Worked Example

```javascript
console.log('A');

setTimeout(() => console.log('B'), 0);

Promise.resolve()
  .then(() => console.log('C'))
  .then(() => console.log('D'));

console.log('E');
```

Output: `A, E, C, D, B`.

- `A` and `E` are synchronous — they run during the initial script task.
- The `setTimeout` callback is queued as a macrotask.
- Both `.then` reactions are microtasks. After sync code, the microtask queue drains: `C` then `D`.
- Only then is the next macrotask dequeued: `B`.

### Node.js Specifics

Node.js uses **libuv** and has additional phases inside one macrotask "tick":

```
   timers -> pending -> idle/prepare -> poll -> check -> close
```

- `setTimeout` callbacks fire in the **timers** phase.
- `setImmediate` fires in the **check** phase.
- `process.nextTick()` is a *higher-priority* microtask, drained before promise microtasks.

### Common Interview Traps

1. **Blocking the loop.** A long synchronous loop blocks every queue. Offload CPU work to a Worker.
2. **Microtask starvation.** Recursive `Promise.resolve().then(...)` chains can prevent timers from firing.
3. **`setTimeout(fn, 0)` is not zero.** Browsers clamp to ~4ms after nested timeouts; the callback is still macrotask-scheduled, not immediate.
4. **Rendering happens between macrotasks.** Mutating the DOM 1000 times in one synchronous loop produces exactly one paint, not 1000.
