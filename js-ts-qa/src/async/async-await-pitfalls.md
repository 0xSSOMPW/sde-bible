# Q: What are common pitfalls when using async/await? How does it actually work under the hood?

**Answer:**

`async/await` is syntactic sugar over promises. An `async` function always returns a Promise; an `await` expression pauses the function until the awaited promise settles, then either resumes with the resolved value or throws the rejection reason. The function is conceptually transformed into a state machine using generators and the microtask queue.

### Mental Model

```javascript
async function foo() {
  const a = await stepA();
  const b = await stepB(a);
  return b;
}

// Roughly equivalent to:
function foo() {
  return stepA().then(a => stepB(a)).then(b => b);
}
```

Every `await` is a `then` boundary — a microtask hop. There is no thread; the function "pauses" by returning control to the event loop.

### Pitfall 1: Sequential vs Parallel

```javascript
// BAD: 6 seconds total
const user    = await fetchUser();      // 2s
const orders  = await fetchOrders();    // 2s  (independent of user!)
const reviews = await fetchReviews();   // 2s

// GOOD: 2 seconds total
const [user, orders, reviews] = await Promise.all([
  fetchUser(), fetchOrders(), fetchReviews(),
]);
```

> [!NOTE]
> Only chain `await`s sequentially when each call genuinely depends on the previous result.

### Pitfall 2: `await` Inside `forEach`

```javascript
// Does NOT wait! forEach ignores returned promises.
items.forEach(async item => {
  await process(item);
});
console.log('done'); // logs before any item finishes

// Use for...of for sequential
for (const item of items) {
  await process(item);
}

// Or Promise.all for parallel
await Promise.all(items.map(process));
```

### Pitfall 3: Forgetting to Await Means Unhandled Rejection

```javascript
async function save() {
  doNetworkCall();           // fire-and-forget; rejection becomes UnhandledPromiseRejection
  return 'ok';
}
```

If `doNetworkCall()` rejects, the error is lost. Either `await` it or attach `.catch(handleError)`.

### Pitfall 4: try/catch Only Catches Synchronous Awaited Errors

```javascript
async function load() {
  try {
    return fetchData();      // NO await: error escapes try/catch
  } catch (e) { /* unreachable */ }
}
```

The `return` here returns a promise; the function exits before the promise rejects, so the `catch` never sees it. Add `await` (`return await fetchData()`) or `.catch` the call.

### Pitfall 5: Mixing `await` and Loops in Hot Paths

Awaiting inside a tight loop serializes work. For independent tasks, build a promise array and `await Promise.all([...])`. For backpressure, batch with chunks:

```javascript
for (const chunk of chunks(items, 10)) {
  await Promise.all(chunk.map(process));
}
```

### Pitfall 6: Top-Level Await Blocks Module Graph

ES module top-level `await` is allowed but **delays** evaluation of every importing module. Avoid long awaits at module top-level.

### Under the Hood — State Machine

```
   async function f() {
     const x = await p1;   --> suspend, register .then on p1
     const y = await p2;   --> on resume with x, suspend on p2
     return x + y;         --> on resume with y, resolve f's promise
   }
```

Each `await` resumption is queued as a **microtask**. That means an `await` always yields at least once to the event loop, even if the value is already resolved.

### Pitfall 7: `await` on a Non-Promise

`await 42` is legal — the value is wrapped in `Promise.resolve(42)` and resumes on the next microtask tick. Useful in tests but introduces unnecessary microtask hops in hot code.
