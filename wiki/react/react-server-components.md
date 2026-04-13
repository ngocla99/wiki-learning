# React Server Components

> Sources: Nadia Makarevich, 2025
> Raw: [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)

## Overview

React Server Components (RSC) represent a fundamental shift in how React applications are architected. Rather than treating the server as an afterthought purely for pre-rendering, RSC flips the default: all components are server components unless explicitly marked otherwise with `"use client"`. Server Components execute on the server, produce a serialized tree of React elements (not raw HTML), and never ship their JavaScript to the browser. Combined with streaming, this architecture addresses long-standing tensions between initial load performance, bundle size, and data fetching ergonomics. In real-world measurements, Server Components with streaming can outperform both client-side fetching and classic SSR by a factor of two on critical metrics.

## The Problem: Data Fetching on the Client

Traditional client-side data fetching creates a sequential chain of delays:

1. Download the JavaScript bundle
2. Parse and evaluate JavaScript
3. Mount the component
4. Trigger `fetch` inside `useEffect`
5. Wait for the server response
6. Re-render with the received data

Each step must complete before the next can begin. The data fetch cannot even start until the component is mounted, which itself depends on JavaScript being fully downloaded and evaluated.

```jsx
const Sidebar = ({ ...props }) => {
  const [sidebarData, setSidebarData] = useState(undefined);
  const [isLoading, setIsLoading] = useState(true);

  useEffect(() => {
    const fetchSidebarData = async () => {
      const response = await fetch('/api/sidebar');
      const data = await response.json();
      setSidebarData(data);
      setIsLoading(false);
    };
    fetchSidebarData();
  }, []);

  return (
    <NavigationGroup header="general" {...props}>
      {isLoading ? <SidebarSkeleton /> : renderSidebar(sidebarData)}
    </NavigationGroup>
  );
};
```

In performance measurements of a study app, this pattern yields an LCP of around 730 ms (for the page title), but dynamic sidebar items do not appear until ~1.4 s and table data not until ~1.8 s. Users see loading skeletons for roughly 700 ms. See [Data Fetching Patterns](data-fetching-patterns.md) for more on client-side optimization techniques like prefetching.

### Why Prefetching on the Client Has Limits

You can move the `fetch` call earlier -- outside the component, outside `useEffect`, even outside of React entirely via an inline `<script>` in the HTML `<head>`. Each step moves the fetch initiation further left on the timeline:

```jsx
// Prefetching: trigger fetch before the component mounts
const preloadPromise = fetch('/api/sidebar');

const Sidebar = ({ ...props }) => {
  useEffect(() => {
    const fetchSidebarData = async () => {
      const response = await preloadPromise;
      const data = await response.json();
      // ...
    };
    fetchSidebarData();
  }, []);
};
```

However, every client-side prefetch technique has a cost. Moving fetch calls to `<script>` tags in the HTML head starts them early, but those requests compete for bandwidth with critical resources, potentially worsening LCP (from 730 ms to ~790 ms in measurements). Libraries like TanStack Query or SWR provide cleaner APIs, but are still bound by the same fundamental rules: the fetch cannot begin until its JavaScript is evaluated, and updating state from the response always triggers a [re-render](react-re-renders.md).

## The Problem: SSR with Data Fetching

Adding SSR without changing the data fetching approach barely helps. The server pre-renders the HTML shell, improving LCP from ~730 ms to ~480 ms, but `useEffect` is client-only. Data fetching still happens entirely on the client after hydration. The sidebar and table appear at the same time as without SSR.

### Moving Data Fetching to the Server

To truly benefit from the server, data must be fetched there and passed to React for rendering:

```jsx
app.get("/*", async (c) => {
  const sidebarPromise = fetch('/api/sidebar').then(res => res.json());
  const statisticsPromise = fetch('/api/statistics').then(res => res.json());

  const [sidebar, statistics] = await Promise.all([
    sidebarPromise,
    statisticsPromise,
  ]);

  const reactHtml = renderToString(
    <App sidebar={sidebar} statistics={statistics} />,
  );

  // Inject data into the page for the client to pick up
  const htmlWithData = `
    <div id="root">${reactHtml}</div>
    <script>window.__SSR_DATA__ = ${JSON.stringify({ sidebar, statistics })}</script>
  `;
  // ...
});
```

This requires three steps: (1) pass data as props to `renderToString`, (2) inject the data into the HTML via a `<script>` tag on `window`, and (3) read it from `window` in `hydrateRoot` on the client. If you forget step 2 or 3, React will wipe the server-rendered content and re-fetch everything on the client, producing a jarring flash of skeleton states.

### The Cost of Server-Side Data Fetching with Classic SSR

The critical problem: `renderToString` is a synchronous JavaScript function. You must `await` all data before calling it. If your endpoints take ~700 ms, the entire HTML response is delayed by that amount. LCP drops from ~480 ms to ~1.45 s -- nearly a second worse.

This is the core tension that Server Components with streaming solve. Traditional SSR with `renderToString` is all-or-nothing: wait for everything, then send everything.

## How React Server Components Work

### The Conceptual Model

A normal React component is a function that returns a React Element. When React renders, it walks the component tree, calling each function, and constructs a tree of objects:

```js
{
  type: "button",
  props: {
    children: "I'm a button",
    className: "inline-flex gap-2 ..."
  }
}
```

With Server Components, this tree is constructed on the server in advance. The resulting serialized tree is injected into the page (similar to how SSR data is injected via `window.__SSR_DATA__`), and React on the client reads the pre-built object tree and constructs DOM nodes from it without needing to call any component functions. The component code itself never reaches the browser.

In Next.js, you can verify this by examining the built output. The `.next/server` folder contains the serialized Server Component data as `self.__next_f` entries in `<script>` tags. Searching for component text (like "I'm a button") inside these tags reveals the React element tree in serialized form.

### Server Components Stay on the Server

You can verify that Server Components are excluded from the client bundle:

1. Create a page that renders a `Button` component without `"use client"`.
2. Search for the Button's code in `.next/server` -- it will appear there.
3. Search in `.next/static/chunks` -- it will **not** appear. The Button's JavaScript is never sent to the browser.

Now add `"use client"` to the page file and rebuild. The Button's code appears in `.next/static/chunks/app/page-xxx.js` and is downloaded by the browser.

This is the first benefit: [bundle size](bundle-size-optimization.md) reduction. Components that exist only on the server contribute zero bytes to the client bundle.

### The `"use client"` Directive

The `"use client"` directive marks the boundary between server and client. Adding it to the top of a file turns that component (and all components it imports) into Client Components -- the traditional React components that are bundled, sent to the browser, and hydrated.

```jsx
'use client';
import { Button } from '@fe/components/button';

export default function App() {
  return <Button>I'm a button</Button>;
}
```

A critical subtlety: **components without `"use client"` are not necessarily Server Components.** Their parent determines their fate. If a parent has `"use client"`, all children imported by that parent become Client Components too, regardless of whether they have the directive themselves.

In the example above, `Button` has no directive, but because `App` is marked `"use client"`, the Button ends up in the client bundle. In another context where `App` is a Server Component, the same `Button` would remain on the server.

This means that in a large app with multiple developers, any component may or may not end up in the client bundle depending on its position in the render tree. Controlling this requires careful attention to where `"use client"` boundaries are placed.

### The `"use server"` Directive

There is also a `"use server"` directive, which is used for server functions (also called Server Actions) that can be called from Client Components. It is essentially a way to create REST API endpoints callable from the client. This is a separate concern from Server Components themselves.

## Async Server Components and Data Fetching

Since Server Components run on the server, they can be **asynchronous**. Async components cannot be Client Components (at least currently -- attempting it produces errors in dev mode). This provides a natural enforcement mechanism: if a component is `async`, it is guaranteed to be server-only.

```jsx
export default async function App() {
  const sidebarPromise = fetch('/api/sidebar').then(res => res.json());
  const statisticsPromise = fetch('/api/statistics').then(res => res.json());

  const [sidebar, statistics] = await Promise.all([
    sidebarPromise,
    statisticsPromise,
  ]);

  return (
    <>
      <p>Sidebar data: {JSON.stringify(sidebar)}</p>
      <p>Statistics data: {JSON.stringify(statistics)}</p>
    </>
  );
}
```

Benefits over traditional SSR data fetching:

- **Co-location**: data fetching lives inside the component that uses the data, not in a separate server handler or `getServerSideProps`
- **No props drilling**: each component fetches its own data directly
- **No client-side network requests**: data appears in the page without any `fetch` calls visible in the browser's Network panel
- **Access to server resources**: components can use `fs`, databases, or any Node-specific API directly

However, without streaming, this approach has the same performance cost as classic SSR: the server waits for all data before sending anything to the client.

### Next.js Pre-rendering Behavior

Next.js aggressively pre-builds and pre-caches Server Component output at build time, including executing data fetches. If your API server is not running during `next build`, the build will fail. This is ideal for static or semi-static data, but for truly dynamic data you must opt out:

```jsx
// Force dynamic rendering (no build-time pre-rendering)
export const dynamic = "force-dynamic";

export default async function App() {
  // data is fetched on every request, not at build time
}
```

## Streaming: The Missing Piece

Without streaming, Server Components with data fetching behave identically to classic SSR: the server blocks on all `await` calls, then sends the complete response. The page is blank until everything resolves.

Streaming changes this fundamentally. It allows the server to send HTML progressively, in chunks, as different parts of the component tree resolve. Think Netflix streaming vs. downloading the entire movie first -- applied to React rendering.

### Suspense Controls Streaming Boundaries

To enable streaming, you must tell React which parts of the tree can be deferred. This is done with [Suspense](lazy-loading-and-suspense.md) boundaries:

```jsx
export default async function App() {
  return (
    <div>
      <h1>Welcome to the App!</h1>
      <Suspense>
        <Sidebar />
      </Suspense>
      <Suspense>
        <Statistics />
      </Suspense>
    </div>
  );
}

const Sidebar = async () => {
  const sidebar = await fetch('/api/sidebar').then(res => res.json());
  return <>sidebar: {JSON.stringify(sidebar)}</>;
};

const Statistics = async () => {
  const statistics = await fetch('/api/statistics').then(res => res.json());
  return <>statistics: {JSON.stringify(statistics)}</>;
};
```

With this structure:
1. The `<h1>` appears immediately (it is on the critical path).
2. `Sidebar` data appears next (its endpoint is faster).
3. `Statistics` data appears last (its endpoint is slower).

Each Suspense boundary defines one streaming "chunk." The placement of Suspense boundaries directly controls the user experience.

### Suspense Grouping Matters

Wrapping multiple async components in a **single** Suspense boundary causes them to appear together, only after the slowest one resolves:

```jsx
<Suspense>
  <Sidebar />
  <Statistics />
</Suspense>
```

Here, both Sidebar and Statistics appear simultaneously once the slower endpoint finishes.

### Beware of Server-Side Waterfalls

Nesting async components inside each other creates sequential data fetching, just like client-side waterfalls, but on the server:

```jsx
const Statistics = async () => {
  const statistics = await fetch('/api/statistics').then(res => res.json());
  return (
    <>
      <Suspense>
        <Sidebar />  {/* This fetch cannot start until Statistics resolves */}
      </Suspense>
      statistics: {JSON.stringify(statistics)}
    </>
  );
};
```

Because `Sidebar` is a child of `Statistics`, its function is not called until `Statistics` finishes rendering. The data fetches are chained sequentially. This is a server-side waterfall -- the same problem that plagues client-side fetching, now relocated to the server. Keep async components as siblings wrapped in independent Suspense boundaries to allow parallel fetching.

## Performance Comparison: The Numbers

Measurements from the study project comparing three architectures for the same page (a dashboard with a title on the critical path, plus dynamic sidebar items and a statistics table):

| Metric | Client-Side Fetching | "Classic" SSR | Server Components + Streaming |
|--------|---------------------|---------------|-------------------------------|
| Critical Path (LCP) | 740 ms | 1.2 s | **410 ms** |
| Sidebar Items | 1.4 s | 1.2 s | **550 ms** |
| Statistics Table | 1.8 s | 1.2 s | **1 s** |

Server Components with streaming outperform both alternatives on every metric. The critical path renders nearly twice as fast as client-side fetching and nearly three times faster than classic SSR. Dynamic content appears significantly earlier because the server streams each chunk as soon as its data is ready, rather than waiting for everything.

Classic SSR has uniform timing (1.2 s for everything) because `renderToString` blocks until all data is fetched, then sends the complete page at once. Client-side fetching has fast LCP (the shell renders quickly) but slow dynamic content (each fetch waits for JavaScript evaluation plus network round-trip). Server Components get the best of both: fast shell via streaming, plus fast dynamic content via server-side fetching streamed incrementally.

## Server Components vs SSR: The Distinction

A common misconception is that Server Components are synonymous with SSR, or that they require SSR. They are distinct concepts:

| Aspect | SSR (`renderToString`) | Server Components |
|--------|----------------------|-------------------|
| What it produces | HTML string | Serialized React element tree |
| JS shipped to client | All component JS (for hydration) | Only Client Component JS |
| Data approach | Fetch all data before render | Fetch per-component, stream results |
| Bundle size impact | None -- all code still ships | Server-only code excluded from bundle |
| Interactivity | Full, after hydration | None (server-only); use Client Components for interactivity |
| State / Effects | Yes (after hydration) | No (`useState`, `useEffect`, event handlers are unavailable) |

Next.js couples SSR and Server Components (there is no way to disable SSR in Next.js), which reinforces the confusion. But conceptually, Server Components can exist independently: they produce a serialized tree that React can turn into DOM on the client, even without any SSR-generated HTML. You can verify this by manually removing pre-rendered HTML from Next.js output -- the page still renders correctly from the serialized Server Component data alone (with JavaScript enabled).

## Composition Rules: Server and Client Components

### Server Components Can Render Client Components

Server Components can import and render Client Components, passing serializable props:

```jsx
// Server Component (no directive)
import { InteractiveChart } from './InteractiveChart'; // has "use client"

export default async function Dashboard() {
  const data = await fetchData();
  return (
    <div>
      <h1>Dashboard</h1>
      <InteractiveChart data={data} />
    </div>
  );
}
```

### Client Components Cannot Import Server Components

Client Components cannot directly import Server Components. However, they can receive Server Components as `children` or other JSX props:

```jsx
'use client';
export function ClientWrapper({ children }) {
  const [isOpen, setIsOpen] = useState(true);
  return isOpen ? children : null;
}

// In a Server Component:
<ClientWrapper>
  <ServerComponent />  {/* This works -- passed as children */}
</ClientWrapper>
```

### The Boundary Propagation Rule

The `"use client"` directive creates a boundary. Everything imported by a Client Component file becomes part of the client bundle, even without its own `"use client"` directive. This propagation through the import tree is the primary mechanism that determines whether a component ends up on the server or the client.

Components that use Node-specific APIs (like `fs` or direct database access) will break the build if they accidentally end up in the client bundle via this propagation: `Module not found: Can't resolve 'fs'`. This serves as a hard guard against accidental client inclusion.

## Practical Considerations

### Bundle Size Benefits Are Real but Subtle

The bundle size reduction is the most frequently cited benefit but also the hardest to observe meaningfully. A single Button component is unlikely to weigh 1 MB. In practice, [bundle size](bundle-size-optimization.md) problems come from compounding small duplicated or unnecessary libraries. Since any component's server-or-client status depends on its position in the render tree, controlling what stays on the server in a large multi-developer codebase requires discipline and clear `"use client"` boundary conventions.

### Framework Requirement

Implementing proper Server Components support is extremely difficult. As of 2025, Next.js (App Router) is the only major framework among the top three (Next.js, React Router, TanStack) that implements them. This effectively makes "Server Components" synonymous with "Next.js" in practice, though that may change as other frameworks adopt the architecture.

### When Server Components Shine

- **Data-heavy pages with mixed static/dynamic content**: streaming lets the static shell appear immediately while dynamic sections load independently
- **SEO-sensitive pages**: content is server-rendered without the full JS payload cost
- **Pages with heavy server-only dependencies**: markdown parsers, syntax highlighters, or other large libraries that only need to run once stay on the server
- **Reducing time-to-interactive**: less JavaScript shipped means less to parse, evaluate, and hydrate

### When to Be Cautious

- **Highly interactive UIs**: components requiring state, effects, or event handlers must be Client Components regardless
- **Build-time data caching surprises**: Next.js pre-renders aggressively; dynamic data requires explicit opt-out (`export const dynamic = "force-dynamic"`)
- **Server-side waterfalls**: careless nesting of async components recreates the waterfall problem on the server

## Key Takeaways

- Server Components run on the server and never ship JavaScript to the client. They produce a serialized element tree, not HTML.
- The `"use client"` directive marks the boundary between server and client code. Components without it are Server Components by default, unless a parent with `"use client"` pulls them into the client bundle.
- Async components are guaranteed to be server-only (they cannot be Client Components), enabling direct data fetching with `await` inside the component body.
- Streaming, controlled by Suspense boundaries, is the key that unlocks the performance benefits. Without streaming, Server Components with data fetching perform identically to classic SSR.
- Each Suspense boundary defines a streaming chunk. Siblings in separate Suspense boundaries stream independently; siblings in one Suspense boundary wait for the slowest.
- Nesting async Server Components inside each other creates server-side data fetching waterfalls, just as `useEffect` chains create client-side waterfalls.
- In performance measurements, Server Components with streaming outperformed client-side fetching and classic SSR on every metric: 410 ms vs 740 ms vs 1.2 s for the critical path.
- Classic SSR ships all JavaScript for hydration and blocks on data fetching before sending any HTML. Server Components ship only Client Component JavaScript and stream content progressively.
- As of 2025, Next.js App Router is effectively the only production implementation of Server Components among major frameworks.

## See Also

- [Rendering Strategies](rendering-strategies.md) -- CSR, SSR, SSG, and how Server Components fit into the landscape
- [Data Fetching Patterns](data-fetching-patterns.md) -- client-side fetching, prefetching, and the evolution toward server-side approaches
- [Lazy Loading and Suspense](lazy-loading-and-suspense.md) -- Suspense boundaries serve double duty for lazy loading and streaming
- [Bundle Size Optimization](bundle-size-optimization.md) -- Server Components as a bundle size reduction strategy
- [Web Performance Metrics](web-performance-metrics.md) -- LCP, INP, and the metrics used to evaluate these approaches
- [React Re-renders](react-re-renders.md) -- state updates from data fetching trigger re-renders; Server Components avoid this entirely
- [Memoization in React](memoization.md) -- optimization techniques for Client Components that still need interactivity
