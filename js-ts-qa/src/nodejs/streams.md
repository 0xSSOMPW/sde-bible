# Q: What are Node.js streams? Explain backpressure and the four stream types.

**Answer:**

Node **streams** are an abstraction for processing data **incrementally**, piece by piece, instead of loading entire payloads into memory. They are the foundation of `fs.createReadStream`, HTTP requests/responses, TCP sockets, child-process pipes, zlib, and crypto.

### The Four Types

| Type        | Reads | Writes | Example                                              |
|--------------|-------|--------|------------------------------------------------------|
| Readable     | yes   | no     | `fs.createReadStream`, `http.IncomingMessage`        |
| Writable     | no    | yes    | `fs.createWriteStream`, `http.ServerResponse`        |
| Duplex       | yes   | yes    | TCP socket — read and write are independent          |
| Transform    | yes   | yes    | `zlib.createGzip()` — write input -> read transformed output |

A `Transform` is a `Duplex` where output is a function of input.

### Two Reading Modes

**Flowing mode** — the stream pushes data via events:

```javascript
stream.on('data', chunk => process(chunk));
stream.on('end', () => done());
stream.on('error', err => fail(err));
```

**Paused mode** — the consumer pulls via `.read()` or `for await`:

```javascript
for await (const chunk of stream) {
  process(chunk);
}
```

Modern code overwhelmingly prefers the async-iterator form — it integrates with `await`, propagates errors naturally, and respects backpressure for free.

### Backpressure

Backpressure is the mechanism that prevents a fast producer from overwhelming a slow consumer. Each writable has an internal buffer; `write(chunk)` returns `false` when the buffer exceeds its `highWaterMark`.

```javascript
function copy(src, dst, cb) {
  src.on('data', chunk => {
    if (!dst.write(chunk)) src.pause();   // slow consumer — stop reading
  });
  dst.on('drain', () => src.resume());    // ready for more
  src.on('end', () => dst.end(cb));
  src.on('error', cb);
}
```

Writing without honoring the return value leads to unbounded memory growth — process RSS climbs forever while the buffer queues chunks.

### `pipe` and `pipeline`

`stream.pipe()` handles backpressure for you but **does not** forward errors:

```javascript
src.pipe(gzip).pipe(dst);
src.on('error', handle);
gzip.on('error', handle);
dst.on('error', handle);
```

Use `pipeline` from `node:stream/promises` — it cleans up on any error and resolves when done:

```javascript
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';

await pipeline(
  createReadStream('big.log'),
  createGzip(),
  createWriteStream('big.log.gz')
);
```

> [!NOTE]
> `pipeline` is to streams what `Promise.all` is to promises — the right default.

### Visual: Backpressure Across A Pipeline

```
   Source        Transform           Sink
   ┌──────┐     ┌──────────┐      ┌──────┐
   │ read │ ──► │  buffer  │ ──►  │ write│
   └──────┘     └──────────┘      └──────┘
        ▲           ▲                 │
        │           │  drain          │ slow disk
        │           └─────────────────┘
        │                              │
        └── pause when downstream full ┘
```

### Implementing a Transform

```javascript
import { Transform } from 'node:stream';

const upper = new Transform({
  transform(chunk, _enc, cb) {
    cb(null, chunk.toString().toUpperCase());
  }
});

process.stdin.pipe(upper).pipe(process.stdout);
```

For object-mode streams (chunks are JS objects, not buffers), set `objectMode: true`.

### Common Pitfalls

- **Listening for `data` and forgetting `error`.** Unhandled errors crash the process.
- **Not consuming the readable.** If nothing reads, the source stays in paused mode forever.
- **Mixing async iteration with `pipe`.** Choose one model per stream.
- **Mutating the chunk buffer in-place.** Cause: `Buffer.concat` reuses memory regions.
- **Calling `res.end()` while writes are still buffered.** Use `await pipeline(...)` or `await new Promise(r => dst.end(r))`.

### Web Streams Interop

Modern Node also supports the **Web Streams API** (`ReadableStream`, `WritableStream`, `TransformStream`) used by browsers and Service Workers. Convert with helpers:

```javascript
import { Readable } from 'node:stream';
const nodeStream = Readable.fromWeb(webStream);
const webStream  = Readable.toWeb(nodeStream);
```

Useful when integrating Fetch API (`Response.body` is a Web stream) with Node tooling.
