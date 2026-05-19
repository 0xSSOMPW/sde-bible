# Q: How do you reduce the JavaScript bundle size of a web app? Walk through your toolbox.

**Answer:**

Bundle size directly drives **time-to-interactive**, especially on slow networks and low-end mobile. The toolbox is layered: measure first, then attack imports, then build configuration, then runtime patterns.

### Measure First

- `webpack-bundle-analyzer` / `rollup-plugin-visualizer` / `vite-bundle-visualizer` — flame view of what's in each chunk.
- `source-map-explorer` — works on any final bundle with a source map.
- Lighthouse + WebPageTest — real-world TTI, LCP, and JS execution time.
- `npx bundlephobia <pkg>` — pre-flight check before adding a dependency.

### Layer 1: Pick Smaller Dependencies

| Heavy                 | Lighter alternative                  |
|------------------------|--------------------------------------|
| moment (~70 KB)       | `date-fns` (tree-shakable) or `Intl.DateTimeFormat` |
| lodash full import    | `lodash-es` + named imports, or one-off helpers |
| axios                 | native `fetch` for most cases        |
| `recharts`/`chart.js` | `uPlot` for line/bar use cases       |
| `jquery`              | DOM APIs are plenty in 2026          |
| Redux + RTK           | Zustand/Jotai if features fit        |

### Layer 2: Tree-Shaking Hygiene

For tree-shaking to work, packages and your own code must use **ES modules** with side-effect-free imports.

```jsonc
// package.json — tell the bundler files have no side effects
{
  "sideEffects": false,
  // or, scoped:
  "sideEffects": ["*.css", "./polyfills.js"]
}
```

In application code, prefer named imports from ESM packages:

```javascript
// good: only debounce ends up in the bundle
import { debounce } from 'lodash-es';

// bad: pulls in the entire library
import _ from 'lodash';
const f = _.debounce(...);
```

### Layer 3: Code Splitting

**Route-based**:

```javascript
const Settings = React.lazy(() => import('./Settings'));
<Suspense fallback={<Spinner/>}>
  <Route path="/settings" element={<Settings/>}/>
</Suspense>
```

**Component-based** for heavy widgets used rarely (rich-text editor, chart):

```javascript
const Editor = React.lazy(() => import('./Editor'));
```

**Vendor splitting** — keep stable third-party code in a long-cached chunk so app updates don't invalidate it.

### Layer 4: Dynamic Imports for Conditional Code

```javascript
async function exportCsv(rows) {
  const { stringify } = await import('csv-stringify/browser/esm');
  return stringify(rows);
}
```

The CSV library only ships when a user actually exports.

### Layer 5: Minification & Compression

- **Terser/SWC/esbuild** for minification — fine-tune `pure_funcs`, `passes`, drop console.
- **Brotli** static compression at the CDN edge often beats gzip by 15-20% on JS.
- **Preload** critical chunks (`<link rel="modulepreload">`) and **defer** non-critical ones.

### Layer 6: Polyfill Strategy

Sending a 100 KB pile of polyfills to a modern Chrome user is wasteful. Use **differential serving**: ship a modern bundle (`<script type="module">`) and a legacy fallback (`<script nomodule>`).

```html
<script type="module" src="/app.modern.js"></script>
<script nomodule src="/app.legacy.js" defer></script>
```

Tools: Vite legacy plugin, esbuild `target`, `@babel/preset-env` with `browserslist`.

> [!NOTE]
> Audit your `browserslist`. Defaults of `> 0.5%, last 2 versions` typically include browsers nobody in your audience uses, ballooning polyfills.

### Layer 7: Server Components / SSR

Move code that doesn't need the browser to the server (Next.js App Router, RSC). Server-only dependencies don't enter the client bundle at all — a huge win for date libraries, schema validators, markdown renderers, etc.

### Layer 8: Avoid Re-Bundling The Framework

- Keep React and major libs **external** if you use a CDN or Module Federation.
- Don't accidentally bundle two copies of React via mismatched peer deps.

### Layer 9: Images & Fonts

Not strictly JS, but the most under-optimized assets on most sites. Use `AVIF`/`WebP`, responsive `srcset`, `font-display: swap`, and only preload above-the-fold fonts.

### Realistic Targets

| Bundle stage          | Target (gzipped)         |
|------------------------|--------------------------|
| Initial app chunk      | < 100-170 KB             |
| Per-route chunk        | < 50 KB                  |
| Vendor chunk           | < 150 KB                 |
| Total JS on first load | < 250-300 KB             |

### Decision Cheatsheet

```
   pulling in moment / momentjs?           -> swap for date-fns or Intl
   importing whole library?                -> use named import, check tree-shake
   route only used by 5% of users?         -> React.lazy
   feature loaded after CTA click?         -> dynamic import()
   heavy code only on Node side?           -> move to server component / API
   legacy browser support cost > benefit?  -> drop them; raise browserslist
```
