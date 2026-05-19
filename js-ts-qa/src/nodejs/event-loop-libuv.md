# Q: How does Node.js's event loop differ from the browser's? What are the phases and when does each fire?

**Answer:**

Node.js uses **libuv** under the hood. While the browser event loop has only "tasks" and "microtasks", Node's loop is divided into **phases**, each with its own callback queue. The interaction between phases, `process.nextTick`, microtasks, and `setImmediate` produces some famously surprising ordering.

### Phases (in order, repeated forever)

```
   ┌───────────────────────────┐
   │      timers               │  setTimeout / setInterval callbacks whose threshold is met
   ├───────────────────────────┤
   │   pending callbacks       │  deferred system-level errors (e.g. TCP errors)
   ├───────────────────────────┤
   │   idle, prepare           │  internal use only
   ├───────────────────────────┤
   │      poll                 │  retrieve new I/O events; block here if no other work
   ├───────────────────────────┤
   │      check                │  setImmediate() callbacks
   ├───────────────────────────┤
   │   close callbacks         │  socket.on('close', ...) etc.
   └───────────────────────────┘
                      │
                      └──── microtasks + nextTick run **between** each callback,
                            not just between phases
```

### Microtasks in Node

Between every single callback (not just at phase boundaries), Node drains:

1. The `process.nextTick` queue — *higher* priority than promise microtasks.
2. The promise microtask queue (`.then`, `await` continuations, `queueMicrotask`).

So:

```javascript
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));
console.log('sync');
```

Output:

```
sync
nextTick
promise
timeout      // or immediate first — see below
immediate
```

`nextTick` and `promise` are drained after the sync phase finishes, before any I/O or timer phase runs.

### `setTimeout(fn, 0)` vs `setImmediate(fn)`

These two have an **indeterminate** order when scheduled from the main module:

```javascript
setTimeout(() => console.log('a'), 0);
setImmediate(() => console.log('b'));
// Order can be a-b or b-a — depends on how quickly the loop entered the timers phase
```

But when **scheduled from inside an I/O callback**, the order is guaranteed:

```javascript
fs.readFile('foo', () => {
  setTimeout(() => console.log('a'), 0);   // runs in next loop's timers phase
  setImmediate(() => console.log('b'));    // runs immediately after current poll, in check phase
  // Output: b, a
});
```

After the poll callback, the loop goes **check** before wrapping back to **timers** — so `setImmediate` wins.

### `process.nextTick` Is Not A Phase

It is a *microtask-like* queue, drained between every callback. Heavy use of `nextTick` can **starve I/O** because the loop won't advance to the poll phase while `nextTick` is still recursively enqueueing.

```javascript
function loop() { process.nextTick(loop); }
loop(); // I/O is now starved forever — server stops responding
```

> [!NOTE]
> `process.nextTick` exists mostly for backward compatibility and for cases where you need to **defer** to the end of the current operation without yielding to I/O. In new code, prefer `queueMicrotask`.

### Worker Threads vs The Event Loop

CPU-bound work blocks the loop. For heavy computation, use:

- `worker_threads` — true OS threads sharing the same process; communicate via `MessageChannel`/`postMessage` like browser workers.
- `child_process.fork` — separate Node process with IPC.

Network I/O does not block — libuv uses async system calls (`epoll`, `kqueue`, IOCP) and a thread pool for filesystem and DNS.

### The Thread Pool

By default libuv has a **4-thread pool** used for:

- `fs.*` filesystem operations.
- `crypto.pbkdf2`, `crypto.randomBytes`, `crypto.scrypt`.
- DNS lookup via `getaddrinfo`.
- `zlib` async functions.

If you saturate the pool with synchronous-style work, async filesystem calls queue up. Tune with `UV_THREADPOOL_SIZE` (max 1024) when needed.

### Differences vs Browser

| Aspect              | Browser                                | Node.js                                |
|----------------------|----------------------------------------|----------------------------------------|
| Phases               | Tasks + microtasks                     | Multi-phase libuv loop                 |
| Rendering            | Interleaved between tasks              | None (no DOM)                          |
| Highest priority queue | Microtasks                          | `process.nextTick` (then microtasks)   |
| `setImmediate`       | Not standard (IE-only)                 | Native, distinct phase                 |
| Thread pool          | Web Workers (manual)                   | libuv pool (implicit for fs/dns/crypto) |

### Debugging the Loop

- `node --inspect` + Chrome DevTools "Performance".
- `process.hrtime.bigint()` to measure callback duration.
- `perf_hooks.monitorEventLoopDelay()` — detects lag spikes.

```javascript
import { monitorEventLoopDelay } from 'node:perf_hooks';
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
setInterval(() => console.log('p99 lag ms', h.percentile(99) / 1e6), 5000);
```

Persistently high lag means something synchronous is blocking the loop — likely a hot JSON parse, a regex backtrack, or a sync `fs` call.
