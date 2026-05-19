# Q: What are React Server Components? How do they differ from SSR and from client components?

**Answer:**

**React Server Components (RSC)** are components that run **only on the server** and never ship to the client. They can read databases, hit secrets, or perform heavy work, then send a serialized UI tree to the browser. They are *not* the same as Server-Side Rendering — SSR runs a normal client component on the server to generate initial HTML; RSC defines a new component kind that lives entirely server-side.

### Mental Model

```
   ┌────────────────────────┐      ┌──────────────────────────┐
   │     Server             │      │     Browser              │
   │                        │      │                          │
   │ Server Component       │──────│► Streams RSC payload     │
   │ (DB, fs, env)          │      │  (JSON-ish tree)         │
   │   │                    │      │                          │
   │   ├─ embeds  Client    │      │ Client Component         │
   │   │  Component refs    │      │ hydrates, runs JS        │
   │   ▼                    │      │                          │
   │ HTML (optional)        │──────│► initial paint           │
   └────────────────────────┘      └──────────────────────────┘
```

### Key Properties of Server Components

- **No browser APIs.** No `useState`, `useEffect`, `useRef`, no event handlers, no `window`.
- **Async by default.** A Server Component can be an `async` function and `await` directly.
- **Zero JS to the client.** Their code stays on the server; only their *rendered output* is serialized.
- **Can import server-only deps.** Database drivers, secrets, big formatting libs — none of it ships to the browser.

```jsx
// app/posts/page.tsx — server component (default in Next App Router)
import { db } from '@/lib/db';
import { PostList } from './PostList'; // could be client or server
export default async function Page() {
  const posts = await db.post.findMany();
  return <PostList posts={posts} />;
}
```

### Client Components

Marked explicitly with `"use client"` at the top of the file. They are normal React components that ship JS and hydrate in the browser.

```jsx
'use client';
import { useState } from 'react';
export function LikeButton({ postId }) {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>Likes: {count}</button>;
}
```

### Composition Rules

- Server components can render client components.
- Client components **cannot import** server components directly — but they can receive them as **`children`** props.

```jsx
// server.tsx
<ClientShell>
  <ExpensiveServerWidget />   {/* passed as children — allowed */}
</ClientShell>
```

This pattern keeps client bundles small while reusing server-rendered subtrees inside interactive shells.

### Differences vs SSR

| Aspect              | Traditional SSR                 | React Server Components             |
|----------------------|---------------------------------|--------------------------------------|
| Code on client       | Yes (full component code)       | No (server components excluded)      |
| Output to browser    | HTML                            | Serialized RSC payload (+ HTML)      |
| Hydration            | Required for interactivity      | Only client components hydrate       |
| Data fetching        | In `getServerSideProps`/etc.    | Inline via `await`                   |
| Streaming            | Optional, manual                | Built-in via Suspense                |
| Bundle impact        | Heavy with framework code       | Slimmer — server-only libs excluded  |

> [!NOTE]
> Server Components do not replace SSR; they complement it. Frameworks like Next.js App Router run both: the server component tree is rendered, server-side, and the parts marked `"use client"` are SSR'd into HTML for first paint and then hydrated.

### Server Actions

Server Actions (`"use server"`) let a client component **call a server function** as if it were local. The runtime serializes the call, executes it on the server, and returns a result.

```jsx
'use server';
export async function deletePost(id: string) {
  await db.post.delete({ where: { id } });
  revalidatePath('/posts');
}
```

```jsx
'use client';
import { deletePost } from './actions';
<form action={deletePost.bind(null, post.id)}>...</form>
```

No fetch handler, no API route — the framework wires up the RPC for you.

### When to Reach for RSC

- Large static or read-heavy pages (dashboards, content sites).
- Pages that need access to databases/secrets but minimal interactivity.
- Reducing JS bundle size for first interaction.

### When to Stay Client-Side

- Highly interactive widgets (editors, drag-and-drop, games).
- Anything requiring browser APIs (Canvas, WebGL, geolocation).
- Realtime data via WebSockets (still possible via a server-pushed bridge but more involved).

### Common Pitfalls

- Forgetting `"use client"` and then trying to use `useState` — runtime error in development.
- Importing a server component from a client component — the compiler will error or the runtime will leak server code.
- Mutating shared module state on the server — RSCs run per request; module-level singletons must be thread-safe or per-request.
