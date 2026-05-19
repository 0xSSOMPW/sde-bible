# Q: What are generators and iterators in JavaScript? When are they useful?

**Answer:**

An **iterator** is any object with a `next()` method that returns `{ value, done }`. An **iterable** is any object with a `[Symbol.iterator]()` method that returns an iterator. `for...of`, spread, destructuring, and `Array.from` all operate on iterables.

A **generator function** (`function*`) is the easiest way to build both — it returns an object that is *both* an iterator and an iterable, and it can `yield` values lazily.

### Basic Generator

```javascript
function* range(start, end, step = 1) {
  for (let i = start; i < end; i += step) yield i;
}

for (const n of range(0, 5)) console.log(n); // 0 1 2 3 4
[...range(0, 3)];                            // [0, 1, 2]
```

Execution pauses at each `yield` and resumes on the next `.next()` call — no buffering of the full sequence.

### The Iterator Protocol — by Hand

```javascript
const counter = {
  [Symbol.iterator]() {
    let i = 0;
    return {
      next() { return i < 3 ? { value: i++, done: false } : { value: undefined, done: true }; }
    };
  }
};
for (const n of counter) console.log(n); // 0 1 2
```

### Why Generators Matter

**1. Infinite/lazy sequences.** You can describe streams that would never fit in memory.

```javascript
function* naturals() { let n = 1; while (true) yield n++; }
function* take(iter, k) { for (const v of iter) { if (k-- <= 0) return; yield v; } }
[...take(naturals(), 5)]; // [1,2,3,4,5]
```

**2. Pipeline composition.** Each step yields one item at a time:

```javascript
function* map(iter, f)     { for (const v of iter) yield f(v); }
function* filter(iter, f)  { for (const v of iter) if (f(v)) yield v; }
[...take(filter(map(naturals(), x => x * x), x => x % 2), 5)];
// [1, 9, 25, 49, 81]
```

**3. Two-way communication via `.next(value)`.** A consumer can push data **back into** a paused generator. This is the foundation of older coroutine-style async (`co`, `redux-saga`).

```javascript
function* dialog() {
  const name = yield 'What is your name?';
  yield `Hello, ${name}`;
}
const g = dialog();
g.next();            // { value: 'What is your name?', done: false }
g.next('Ada');       // { value: 'Hello, Ada', done: false }
g.next();            // { value: undefined, done: true }
```

### Async Iterators (`Symbol.asyncIterator`)

Async generators (`async function*`) yield Promises and are consumed with `for await...of`:

```javascript
async function* lines(stream) {
  let buf = '';
  for await (const chunk of stream) {
    buf += chunk;
    let i; while ((i = buf.indexOf('\n')) >= 0) {
      yield buf.slice(0, i);
      buf = buf.slice(i + 1);
    }
  }
  if (buf) yield buf;
}
for await (const line of lines(fileStream)) processLine(line);
```

Streaming pagination, server-sent events, and reading large files become elegant one-shot loops.

### Generators vs Async/Await

```
  async/await   = generator + auto-runner that awaits each yielded promise
  generators    = manual control flow primitive
```

Most application code should reach for `async/await`. Reach for generators when you need:
- Pull-based lazy streams.
- Custom iteration protocols.
- Pausable / cancellable coroutines.

> [!NOTE]
> Generators are *not* parallel. They cooperatively pause; the engine is still single-threaded.

### Termination

- `return(value)` ends the generator early as if `return value;` ran in place of the next `yield`.
- `throw(err)` injects an error at the suspended `yield`, allowing a `try/catch` inside the generator to handle it.

### Real-World Sightings

- Node streams expose async iterators (`for await (const chunk of req)`).
- Redux-Saga uses synchronous generators for testable side-effect orchestration.
- Web Streams' `ReadableStream` works with `for await`.
