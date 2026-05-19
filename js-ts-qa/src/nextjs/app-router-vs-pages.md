# Q: What's the difference between Next.js App Router and Pages Router? What changed and why?

**Answer:**

Next.js historically organized routes via the `pages/` directory: each file became a route, with `getStaticProps`/`getServerSideProps` controlling data fetching. The **App Router** (introduced in Next 13, stable since 13.4) replaces this with the `app/` directory and embraces React Server Components, layouts, streaming, and Server Actions natively.

### Side-by-Side

| Aspect                  | Pages Router (`pages/`)                  | App Router (`app/`)                              |
|--------------------------|------------------------------------------|--------------------------------------------------|
| Components by default   | Client                                   | **Server** (`"use client"` opt-in)               |
| Data fetching            | `getStaticProps`, `getServerSideProps`   | Async server components + `fetch` with caching   |
| Layouts                  | Manual `_app.js` + `_document.js`        | Nested, file-system based                        |
| Streaming                | Limited                                  | Built-in via Suspense                            |
| Loading state            | Manual                                   | `loading.tsx` co-located file                    |
| Error UI                 | `_error.js`                              | `error.tsx` per route                            |
| Mutations                | API routes                               | **Server Actions** (`"use server"`)              |
| Caching model            | Full-route SSG/ISR                       | Per-fetch cache + per-route segment cache        |
| Dynamic routes           | `[id].js`                                | `[id]/page.tsx`                                  |

### File Conventions (App Router)

```
   app/
     layout.tsx        ← root layout, wraps every route
     page.tsx          ← "/"
     loading.tsx       ← Suspense fallback for siblings
     error.tsx         ← error boundary
     not-found.tsx     ← 404 UI
     posts/
       layout.tsx      ← nested layout for /posts/*
       page.tsx        ← "/posts"
       [slug]/
         page.tsx      ← "/posts/:slug"
```

Layouts are **nested and preserved** across navigations — a shared sidebar doesn't unmount when you click between two routes under the same layout.

### Server Components by Default

```jsx
// app/posts/page.tsx — runs only on the server
import { db } from '@/lib/db';
export default async function Page() {
  const posts = await db.post.findMany();
  return <PostList posts={posts}/>;
}
```

No `getServerSideProps`. Data fetching is just `await` inside the component. The result is serialized to the client; the server-side code never ships.

### `fetch` Is Cache-Aware

```javascript
// statically cached at build time (default in route segments)
fetch(url);

// revalidate every 60 seconds (ISR equivalent)
fetch(url, { next: { revalidate: 60 } });

// no cache — always live
fetch(url, { cache: 'no-store' });

// tag for on-demand revalidation
fetch(url, { next: { tags: ['posts'] } });
revalidateTag('posts');
```

The App Router collapses SSG, ISR, and SSR into a single mental model controlled per `fetch`.

### Server Actions Replace Most API Routes

```jsx
// actions.ts
'use server';
import { db } from '@/lib/db';
export async function createPost(formData: FormData) {
  await db.post.create({ data: { title: formData.get('title') as string } });
  revalidatePath('/posts');
}
```

```jsx
// component.tsx
import { createPost } from './actions';
<form action={createPost}>
  <input name="title"/><button>Create</button>
</form>
```

Forms post directly to the server function; no manual `fetch`, no API route, no parsing.

### Streaming and `loading.tsx`

```
   app/dashboard/
     layout.tsx
     loading.tsx      ← shown while page.tsx + children load
     page.tsx
```

`loading.tsx` is sugar for wrapping the segment in `<Suspense fallback={<Loading/>}>`. Slow data inside `page.tsx` streams in, replacing the fallback when ready — no full-page spinner needed.

### Error Boundaries Per Segment

```jsx
// app/dashboard/error.tsx
'use client';
export default function Error({ error, reset }) {
  return <div>Failed: {error.message}<button onClick={reset}>retry</button></div>;
}
```

Each route segment can isolate its failures. The shell stays interactive even if one panel crashes.

### When Pages Router Still Makes Sense

- Mature apps with large existing investments in `getStaticProps`/`getServerSideProps`.
- Apps that need pure static export with no server runtime.
- Tooling that hasn't fully migrated (some plugins/adapters lag the App Router).

You can mix both routers in one project — `pages/` and `app/` coexist while you migrate.

> [!NOTE]
> The App Router's caching model is powerful but easy to misconfigure. The four caches (full-route, router, data, request memoization) interact in ways that catch newcomers. When in doubt, opt out with `dynamic = 'force-dynamic'` until you understand each layer.

### Migration Checklist

1. Move static pages first (no data) — pure JSX.
2. Replace `getStaticProps`/`getServerSideProps` with async server components.
3. Convert interactive widgets to `"use client"` components.
4. Replace REST/API routes with Server Actions where possible.
5. Audit caching defaults — set explicit `cache`/`revalidate` per fetch.
6. Replace `_app.tsx` providers with `app/layout.tsx` + client-side providers wrapping `{children}`.
