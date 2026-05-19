# Q: What's the difference between ESM and CommonJS modules in JavaScript? Why is interop tricky?

**Answer:**

CommonJS (CJS) is Node.js's original module system — synchronous `require`/`module.exports`. ES Modules (ESM) is the official JavaScript standard introduced in ES2015 — `import`/`export`, static structure, async loading. They look similar but behave very differently.

### Key Differences

| Concern              | CommonJS                                | ES Modules                             |
|-----------------------|-----------------------------------------|----------------------------------------|
| Syntax                | `require` / `module.exports`            | `import` / `export`                    |
| Loading               | **Synchronous**                         | **Asynchronous**, statically analysed  |
| Bindings              | **Copy of values** at require time      | **Live read-only bindings**            |
| Execution timing      | When `require` runs                     | After full graph is parsed             |
| `this` at top         | `module.exports`                        | `undefined`                            |
| Conditional load      | `if (x) require('y')` works             | Must use dynamic `import('y')`         |
| File extension        | `.js` / `.cjs`                          | `.mjs` / `.js` with `"type":"module"`  |
| Tree-shaking          | Hard (dynamic)                          | Excellent (static)                     |

### Live Bindings vs Snapshots

```javascript
// counter.cjs
let count = 0;
module.exports = { count, inc: () => count++ };

// app.cjs
const { count, inc } = require('./counter');
inc();
console.log(count); // 0 — destructured a snapshot of the number
```

```javascript
// counter.mjs
export let count = 0;
export function inc() { count++; }

// app.mjs
import { count, inc } from './counter.mjs';
inc();
console.log(count); // 1 — `count` is a live binding
```

You cannot reassign an imported binding from the consumer side — they're read-only views into the exporter's scope.

### Static vs Dynamic Structure

ESM `import`/`export` must be at the top level. The module graph is built **before** any code runs, enabling:

- Tree-shaking (unused exports dropped).
- Top-level circular dependency resolution via live bindings.
- Async preloading by the host.

For conditional/lazy loads, use `import('module')` which returns a `Promise<Namespace>`.

### Interop Pain Points

```javascript
// ESM consuming CJS
import pkg from 'lodash';      // works — default import binds to module.exports
import { map } from 'lodash';  // sometimes works, sometimes throws — depends on bundler/Node version
```

Node's strategy: when ESM imports CJS, `module.exports` becomes the default export, and named exports are detected with a static analyzer (`cjs-module-lexer`). It often gets named exports right, but not always.

```javascript
// CJS consuming ESM — must be async!
const esm = await import('./mod.mjs');
```

You **cannot** `require()` an ESM module synchronously in older Node. Node 22+ adds limited synchronous `require()` of ESM under a flag, but writing async-safe code is still safer.

> [!NOTE]
> If you're publishing a library, ship **both** formats with the `exports` field:
> ```json
> "exports": {
>   ".": {
>     "import": "./dist/index.mjs",
>     "require": "./dist/index.cjs",
>     "types": "./dist/index.d.ts"
>   }
> }
> ```

### `__dirname` and `__filename`

CJS has them as globals. ESM does not. Use:

```javascript
import { fileURLToPath } from 'node:url';
import { dirname } from 'node:path';
const __filename = fileURLToPath(import.meta.url);
const __dirname  = dirname(__filename);
```

### Circular Dependencies

- CJS: partial exports are returned mid-evaluation, leading to surprising `undefined` values.
- ESM: live bindings make most cycles work as long as you don't *use* the circular value before the cycle completes.

### When to Pick Which (in 2026)

- New code: **ESM** unless you're inside a legacy CJS codebase.
- Libraries: ship dual builds and gate on the `exports` field.
- Tools (CLIs) that need synchronous startup: CJS is still simpler.
