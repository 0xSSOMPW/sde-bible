# Q: What are Web Workers and when should you use them?

**Answer:**

JavaScript on the main thread shares the same event loop as rendering and user input. CPU-bound work there freezes the UI. **Web Workers** are background OS threads exposed to JS via a message-passing API — separate event loop, separate global scope, no shared DOM access. They let you do heavy work without dropping frames.

### Worker Flavors

| Kind            | Lifetime               | Shared between tabs/pages? | Typical use                                |
|------------------|------------------------|----------------------------|--------------------------------------------|
| Dedicated Worker | Same page as creator   | No                         | Per-page CPU work                          |
| Shared Worker    | As long as any tab open| Yes, same origin           | Cross-tab coordination                     |
| Service Worker   | Independent of pages   | Yes                        | Offline, caching, push, fetch interception |
| Worklet (Audio, Paint) | Tied to specific subsystem | No               | Realtime audio, custom paint, animations   |

### Hello, Worker

```javascript
// main.js
const worker = new Worker(new URL('./hash.worker.js', import.meta.url), { type: 'module' });
worker.postMessage({ payload: bigArrayBuffer }, [bigArrayBuffer]); // transfer ownership
worker.onmessage = e => console.log('hash =', e.data);
```

```javascript
// hash.worker.js
self.onmessage = async (e) => {
  const digest = await crypto.subtle.digest('SHA-256', e.data.payload);
  self.postMessage(new Uint8Array(digest));
};
```

### Communication Model

```
                  main thread                          worker thread
   +---------------------------------+        +----------------------------+
   | window, document, DOM           |        | self (DedicatedWorkerScope)|
   |                                 |        | no DOM, no window          |
   | worker.postMessage(data) ------>| structured clone or transfer --->  |
   |                                 |        | self.onmessage             |
   | worker.onmessage  <-------------| <-- self.postMessage(reply)        |
   +---------------------------------+        +----------------------------+
```

`postMessage` performs a **structured clone** of the payload (same algorithm as `structuredClone`). Pass the second arg, a list of *transferable* objects (`ArrayBuffer`, `MessagePort`, `ImageBitmap`), to give ownership without copying.

> [!NOTE]
> A 100 MB `ArrayBuffer` sent without transfer is duplicated; the same buffer marked transferable is moved in O(1).

### When to Reach for a Worker

- Image/video processing, parsing large CSV/JSON, encryption, compression.
- Pathfinding, simulation steps in a game.
- Spell-check, syntax highlighting on large documents.
- Anything that runs >16ms and blocks input.

### When Not to Bother

- The DOM is involved — workers cannot touch it.
- Small CPU work (<5ms). Marshalling overhead dominates.
- I/O-bound work — the network is already async on the main thread.

### SharedArrayBuffer + Atomics

For tight numerical loops, a `SharedArrayBuffer` lets multiple workers read/write the **same** memory. `Atomics.wait` / `Atomics.notify` provide low-level synchronization. Browsers require COOP/COEP cross-origin isolation headers to enable it.

```javascript
const sab = new SharedArrayBuffer(1024);
const view = new Int32Array(sab);
// share `sab` to a worker via postMessage; reads/writes are visible to both
```

### Worker Pool Pattern

Workers are not free — keep a pool sized to `navigator.hardwareConcurrency` and round-robin work over a job queue, instead of spawning per task.

```javascript
const pool = Array.from({ length: navigator.hardwareConcurrency }, makeWorker);
let next = 0;
function dispatch(job) {
  const w = pool[next = (next + 1) % pool.length];
  return new Promise(resolve => {
    w.onmessage = e => resolve(e.data);
    w.postMessage(job);
  });
}
```

### Service Workers vs Web Workers

A Service Worker is a special worker that sits between the page and the network. It handles `fetch` events, manages caches, and powers PWAs. It is **not** for offloading CPU work — use a Dedicated Worker for that.
