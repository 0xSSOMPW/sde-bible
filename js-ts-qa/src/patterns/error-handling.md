# Q: What are best practices for error handling in JavaScript and TypeScript?

**Answer:**

JavaScript's error model has rough edges — anything can be thrown, async errors are easy to lose, and the type system doesn't track "what can throw". Good error handling is therefore a combination of **conventions**, **boundary patterns**, and **observability**.

### Always Throw `Error` Instances

```javascript
throw new Error('User not found');         // good
throw 'User not found';                    // bad — no stack trace
```

Strings, numbers, plain objects work but lose stack info and break instanceof checks. Use a subclass for typed conditions:

```javascript
class NotFoundError extends Error {
  constructor(resource) {
    super(`${resource} not found`);
    this.name = 'NotFoundError';
    this.resource = resource;
  }
}
```

### Use `cause` for Wrapping

ES2022 added `Error.cause` to preserve the original cause when re-throwing:

```javascript
try {
  await db.query(sql);
} catch (e) {
  throw new Error('Failed to load user', { cause: e });
}
```

Modern runtimes print the cause chain in stack traces — invaluable for debugging.

### Never Swallow Errors Silently

```javascript
try { await save(); } catch {}     // bad — disappears the error
```

If a failure is genuinely safe to ignore, log it explicitly. Empty catches turn production bugs into mysteries.

### Promise Hygiene

```javascript
// 1. Always await or attach .catch on every promise
promise.catch(reportError);

// 2. Beware of "fire and forget" inside async functions
async function save() {
  doThingAsync(); // promise leak — rejection unhandled
}

// 3. Node: listen for unhandledRejection so you don't crash silently
process.on('unhandledRejection', err => log.fatal({ err }, 'unhandled'));
```

### Result Types vs Exceptions

Two competing styles:

**Throwing style** — terse, idiomatic, integrates with `async/await`:

```javascript
const user = await fetchUser(id);
```

**Result style** — explicit, type-safe, hints at every failure mode:

```typescript
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

async function fetchUser(id: string): Promise<Result<User, NotFoundError>> {
  const res = await fetch(`/api/u/${id}`);
  if (res.status === 404) return { ok: false, error: new NotFoundError('User') };
  if (!res.ok) throw new Error('Network error'); // unexpected vs expected failures
  return { ok: true, value: await res.json() };
}
```

> [!NOTE]
> A useful split: **expected** failures (validation, not-found, permission) -> Result type or domain error class. **Unexpected** failures (DB down, OOM) -> throw and let a boundary handle it.

### Boundaries

Place a single catch-all at architectural seams:

- React: an **Error Boundary** per page or section.
- Express/Fastify: an error middleware that maps exceptions to HTTP responses.
- Worker queue jobs: a wrapper that catches, logs, and retries with backoff.

```javascript
app.use((err, req, res, next) => {
  if (err instanceof NotFoundError) return res.status(404).json({ msg: err.message });
  log.error({ err, path: req.path }, 'unhandled');
  res.status(500).json({ msg: 'Internal Error' });
});
```

The boundary is where you decide: log, retry, surface a user-friendly message, alert an oncall.

### Async Stack Traces

Without `await`, stack traces show only synchronous frames. Always:

```javascript
return await innerCall();  // not `return innerCall()` — keeps the frame visible
```

V8 produces near-perfect async stacks when you `await` consistently and use `Error.cause` for re-throws.

### TypeScript and Errors

TS doesn't track *which* errors a function throws (no checked exceptions). Compensate by:

- Documenting failure modes in JSDoc / type aliases.
- Using **Result** types for expected, recoverable failures.
- Adding `instanceof` discriminators in catches:

```typescript
try { ... }
catch (e) {
  if (e instanceof ZodError)       return handleValidation(e);
  if (e instanceof NotFoundError)  return handleNotFound(e);
  throw e; // re-throw unknown
}
```

### Logging & Observability

Errors should always include:

- A message that names the operation that failed.
- Enough **context** (`userId`, `orderId`, `requestId`).
- The original error as `cause`.
- A correlation/trace ID so logs, metrics, and traces line up.

Tools: Sentry, Datadog, OpenTelemetry. Don't roll your own — they capture source maps, breadcrumbs, and grouping out of the box.

### Anti-Patterns Recap

- Throwing strings.
- Empty `catch` blocks.
- Wrapping every line in `try/catch` instead of placing boundaries.
- Catching `Error` just to log and re-throw without `cause`.
- Returning `null` on failure when a real domain error is more informative.
