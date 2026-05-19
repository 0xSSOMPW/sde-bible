# Q: How do Suspense and Error Boundaries work in React? When should you use each?

**Answer:**

`Suspense` and Error Boundaries are React's two top-down primitives for handling **non-local UI states** — loading and errors — in a declarative, composable way. Both rely on a component "throwing" something during render that an ancestor catches.

### Error Boundaries — Catching Errors

An Error Boundary is any class component that implements `componentDidCatch` or `getDerivedStateFromError`. It catches errors thrown by descendants during rendering, lifecycle methods, and constructors.

```jsx
class Boundary extends React.Component {
  state = { error: null };
  static getDerivedStateFromError(error) { return { error }; }
  componentDidCatch(error, info) { logErrorToService(error, info); }
  render() {
    return this.state.error
      ? <Fallback error={this.state.error}/>
      : this.props.children;
  }
}

<Boundary><Dashboard /></Boundary>
```

What it does **not** catch:
- Errors in event handlers — wrap them with `try/catch` manually.
- Errors inside `setTimeout`/`Promise` callbacks unless they bubble back into render.
- Errors during SSR (use the streaming-aware variant on the server).
- Errors in the boundary itself.

> [!NOTE]
> There is no hook equivalent yet. Use `react-error-boundary` for the ergonomic functional API — it wraps the class internally.

### Suspense — Catching Loading

`Suspense` catches a thrown **promise** from a child. While that promise is pending, the nearest ancestor `<Suspense>` renders its `fallback`.

```jsx
<Suspense fallback={<Spinner/>}>
  <UserProfile id={42} />
</Suspense>
```

Inside `UserProfile`, something — typically a data-fetching library (`use` in React 19, TanStack Query, Relay, Apollo, Next.js fetching) — *suspends* by throwing the in-flight promise. React shows the fallback, awaits the promise, retries the render.

### React 19's `use` Hook

```jsx
function UserProfile({ id }) {
  const user = use(fetchUser(id)); // suspends until the promise resolves
  return <h1>{user.name}</h1>;
}
```

`use(promise)` integrates promises directly into render. Combined with React Server Components, it removes the need for `useEffect + useState` loading patterns.

### Composition

The two boundaries compose with each other and with each other repeatedly:

```jsx
<ErrorBoundary fallback={<Crashed/>}>
  <Suspense fallback={<PageSpinner/>}>
    <Layout>
      <ErrorBoundary fallback={<WidgetCrashed/>}>
        <Suspense fallback={<WidgetSpinner/>}>
          <RemoteWidget />
        </Suspense>
      </ErrorBoundary>
    </Layout>
  </Suspense>
</ErrorBoundary>
```

This produces graceful, fine-grained loading and isolated failure regions.

```
   ┌─────────── ErrorBoundary (page-level) ──────────────┐
   │  ┌──────── Suspense (page-level) ────────────────┐  │
   │  │   Layout                                       │  │
   │  │   ┌────── ErrorBoundary (widget) ─────────┐    │  │
   │  │   │ ┌──── Suspense (widget) ──────────┐   │    │  │
   │  │   │ │  RemoteWidget                   │   │    │  │
   │  │   │ └─────────────────────────────────┘   │    │  │
   │  │   └────────────────────────────────────────┘    │  │
   │  └────────────────────────────────────────────────┘  │
   └──────────────────────────────────────────────────────┘
```

### Streaming SSR

In React 18+ on the server, Suspense becomes the seam for streaming HTML. The shell renders synchronously; suspended subtrees render later and are streamed in `<template>` chunks that replace their fallback on the client. This unlocks **TTFB-fast** pages even with slow data.

### `useTransition` + Suspense

To avoid showing a fallback for a fast state change (e.g., navigating between tabs), mark the update as a transition:

```jsx
const [isPending, startTransition] = useTransition();
startTransition(() => setTab('orders'));
```

React keeps the old UI visible until the new one is ready (or shows the fallback only if it takes too long). Without this, every navigation would flash a spinner.

### Pitfalls

- **Throwing in event handlers** is not caught by Error Boundaries. Wrap with `try/catch` or surface into state.
- **Suspense without a data library that opts in** does nothing — vanilla `fetch + useState` never suspends.
- **Nested Suspense fallbacks cascade.** If you don't want a parent fallback to hide everything, push the fallback further down.
- **Hydration errors** (mismatched server vs client output) cannot be recovered by Error Boundaries until React 19's recovery mode.

### Quick Decision Guide

```
   Need to show a spinner / skeleton while data loads?     -> Suspense
   Need to recover from unexpected exceptions?              -> Error Boundary
   Both?                                                    -> Nest them: ErrorBoundary > Suspense > content
```
