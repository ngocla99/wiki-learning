# React & Rendering — Interview Questions

> Sources: Nadia Makarevich, Next.js docs, personal experience
> Updated: 2026-04-21

## Overview

Interview questions covering React rendering strategies (CSR, SSR, SSG, ISR), the trade-offs between them, and the architectural differences between the Next.js Page Router and App Router.

---

## Rendering Strategies

### Explain CSR, SSR, SSG, and ISR — when would you use each?

#### Client-Side Rendering (CSR)

The server sends a minimal HTML shell with a `<script>` tag. The browser downloads JavaScript, React mounts, and then generates the DOM.

```html
<!-- What the server sends -->
<body>
  <div id="root"></div>
  <script src="/bundle.js"></script>
</body>
```

**Performance profile:**
- FCP and LCP are delayed until JS executes
- No content without JavaScript (SEO unfriendly)
- After initial load, navigation is instant (SPA experience)

**When to use:** Dashboards, admin panels, authenticated apps where SEO doesn't matter and interactivity is heavy.

---

#### Server-Side Rendering (SSR)

The server executes React on every request, generates full HTML, and sends it to the browser. The browser shows content immediately (fast FCP/LCP), then hydrates (attaches event listeners).

```jsx
// Next.js Page Router
export async function getServerSideProps(context) {
  const user = await fetchUser(context.params.id);
  return { props: { user } };
}

// Next.js App Router (Server Component — SSR by default)
export default async function Page({ params }) {
  const user = await fetchUser(params.id);
  return <UserProfile user={user} />;
}
```

**Performance profile:**
- Fast FCP/LCP (full HTML arrives)
- Higher TTFB (server must fetch data and render before responding)
- Higher server load (render per request)

**When to use:** Pages with personalized or frequently changing data (user profiles, news feeds, cart pages).

---

#### Static Site Generation (SSG)

HTML is pre-built at **build time**. The same static file is served to all users, cached at the CDN edge — the fastest possible delivery.

```jsx
// Next.js Page Router
export async function getStaticProps() {
  const posts = await fetchPosts();
  return { props: { posts } };
}

export async function getStaticPaths() {
  const posts = await fetchPosts();
  return {
    paths: posts.map(p => ({ params: { id: p.id } })),
    fallback: false,
  };
}

// Next.js App Router (static = no dynamic data fetching)
export const dynamic = 'force-static';
```

**Performance profile:**
- Fastest TTFB and LCP (pre-built HTML from CDN)
- Zero server computation per request
- Data is stale until next build

**When to use:** Marketing pages, blog posts, documentation — any content that doesn't change per-request.

---

#### Incremental Static Regeneration (ISR)

Extends SSG by revalidating pages in the background after a configured interval. Stale-while-revalidate: the first visitor after expiry gets the cached (stale) page, and the server regenerates in the background. Next visitor gets the fresh version.

```jsx
// Page Router
export async function getStaticProps() {
  const data = await fetchData();
  return { props: { data }, revalidate: 60 }; // regenerate after 60s
}

// App Router
const data = await fetch('/api/data', { next: { revalidate: 60 } });
```

**On-demand ISR** — revalidate immediately when data changes (e.g., CMS webhook):

```js
// App Router — Server Action or Route Handler
import { revalidatePath, revalidateTag } from 'next/cache';
revalidatePath('/blog');
revalidateTag('posts');
```

**When to use:** Blog with frequent updates, product pages, anything that benefits from SSG speed but needs reasonably fresh data.

---

#### Comparison Table

| Strategy | TTFB | FCP/LCP | SEO | Data freshness | Server load |
|----------|------|---------|-----|----------------|-------------|
| CSR | Fast | Slow | Poor | Real-time | Low |
| SSR | Slow | Fast | Good | Real-time | High |
| SSG | Fastest | Fastest | Best | Stale (until rebuild) | None |
| ISR | Fastest | Fastest | Best | Configurable staleness | Low |

---

## Next.js Page Router vs App Router

### What are the key differences between the Page Router and App Router?

Next.js introduced the App Router in v13 as a fundamentally new architecture built on React Server Components. Both routers can coexist in the same project during migration.

---

### File System Structure

**Page Router** — files in `pages/` map directly to routes:

```
pages/
  index.tsx          → /
  about.tsx          → /about
  posts/[id].tsx     → /posts/:id
  _app.tsx           → global layout wrapper
  _document.tsx      → HTML document shell
  api/route.ts       → /api/route
```

**App Router** — files in `app/` use a folder-based convention:

```
app/
  layout.tsx         → root layout (wraps all pages)
  page.tsx           → /
  about/
    page.tsx         → /about
  posts/
    [id]/
      page.tsx       → /posts/:id
      loading.tsx    → automatic Suspense boundary
      error.tsx      → automatic Error Boundary
  api/route/
    route.ts         → /api/route (Route Handler)
```

---

### Data Fetching

**Page Router** — special export functions per page:

```jsx
// Runs server-side on every request
export async function getServerSideProps() {}

// Runs at build time
export async function getStaticProps() {}

// Client-side with useEffect / SWR / React Query
```

**App Router** — async/await directly in Server Components:

```jsx
// app/posts/page.tsx — Server Component
export default async function PostsPage() {
  const posts = await db.posts.findMany(); // runs on server, never shipped to client
  return <PostList posts={posts} />;
}

// Client Component — must opt-in with directive
'use client';
export function LikeButton({ postId }) {
  const [liked, setLiked] = useState(false);
  return <button onClick={() => setLiked(true)}>Like</button>;
}
```

---

### Server Components vs Client Components

This is the most significant conceptual change in the App Router:

| | Server Component | Client Component |
|---|-----------------|-----------------|
| **Runs on** | Server only | Browser (+ server for hydration) |
| **JS bundle** | Not shipped to client | Shipped to client |
| **Can use** | `async/await`, DB, file system, secrets | State, effects, event handlers, browser APIs |
| **Cannot use** | `useState`, `useEffect`, event handlers | Direct DB access |
| **Directive** | Default in App Router | `'use client'` at top of file |

```jsx
// Server Component (default)
async function ProductPage({ params }) {
  const product = await db.products.findById(params.id); // secure — never reaches client
  return (
    <div>
      <h1>{product.name}</h1>
      <AddToCartButton productId={product.id} /> {/* Client Component */}
    </div>
  );
}

// Client Component
'use client';
function AddToCartButton({ productId }) {
  const [added, setAdded] = useState(false);
  return (
    <button onClick={() => { addToCart(productId); setAdded(true); }}>
      {added ? 'Added!' : 'Add to Cart'}
    </button>
  );
}
```

---

### Layouts and Nested Routing

**Page Router:** Only one `_app.tsx` global wrapper. Nested layouts require manual composition.

**App Router:** Each folder can have its own `layout.tsx` that persists across navigation without re-mounting — this enables complex nested layouts without layout flicker:

```
app/
  layout.tsx         → always mounted (NavBar, Footer)
  dashboard/
    layout.tsx       → mounted when inside /dashboard (Sidebar)
    page.tsx         → /dashboard
    settings/
      page.tsx       → /dashboard/settings (both layouts active)
```

---

### Server Actions

App Router introduces Server Actions — functions that run on the server but can be called from client components. They replace many API route use cases:

```jsx
// app/actions.ts
'use server';
export async function createPost(formData: FormData) {
  const title = formData.get('title');
  await db.posts.create({ title });
  revalidatePath('/posts');
}

// app/new-post/page.tsx
import { createPost } from '../actions';
export default function NewPostPage() {
  return (
    <form action={createPost}>
      <input name="title" />
      <button type="submit">Create</button>
    </form>
  );
}
```

---

### Migration Considerations

| When to stay on Page Router | When to migrate to App Router |
|-----------------------------|-------------------------------|
| Stable, working app with no need to change | New project |
| Heavy reliance on `getServerSideProps` patterns | Need Server Components for data privacy |
| Third-party libraries not yet compatible with RSC | Complex nested layouts |
| Team unfamiliar with RSC mental model | Streaming / progressive loading required |

Both routers are supported in Next.js 14/15. Migration is incremental — you can move one route at a time.
