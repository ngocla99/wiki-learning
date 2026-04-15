# Lazy Loading and Suspense

> Sources: Nadia Makarevich, 2025
> Raw: [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)

## Overview

Lazy loading defers the downloading of non-critical JavaScript until it is actually needed, reducing the initial bundle that must be downloaded and evaluated. React provides `lazy()` and `Suspense` as first-class APIs for this. Combined with strategic code splitting, lazy loading allows precise control over what the user sees first and can improve LCP by hundreds of milliseconds. This article covers the mechanics of `React.lazy`, how `Suspense` works, route-based and component-based lazy loading, preloading strategies, framework integration, and the interaction between lazy loading and SSR.

## Lazy Loading and Code Splitting

Lazy loading and code splitting are complementary but distinct concepts. Code splitting (covered in [Bundle Size Optimization](bundle-size-optimization.md)) breaks a bundle into chunks. Lazy loading controls **when** those chunks are downloaded.

Without lazy loading, all manual chunks are preloaded as critical resources -- they download in parallel at page load, and the browser waits for all of them. Lazy loading changes this: the chunk associated with a lazy component is only downloaded when that component first mounts.

The typical candidate for lazy loading is a feature that is **heavy** and **invisible during initial load**: modal dialogs, drawers, WYSIWYG editors, below-the-fold content, and secondary pages.

## React.lazy and Dynamic Imports

`React.lazy` wraps a dynamic `import()` to create a lazily-loaded component. The four steps to implement lazy loading in React are:

1. Mark code as "unnecessary on the initial load" (use `lazy`)
2. Extract the marked code into its own chunk (bundler configuration)
3. Control when the download starts (conditional rendering)
4. Control what happens during download (Suspense fallback)

### Basic Usage

```jsx
import { lazy } from 'react';

// Direct import is replaced with lazy import
const MessageEditorLazy = lazy(async () => {
  return {
    default: (await import('@fe/patterns/message-editor')).MessageEditor,
  };
});
```

`lazy` accepts a "load" function that returns a Promise. When the Promise resolves, React renders the `default` export of the returned value. For named exports (which are more common than default exports), you wrap the dynamic import to reshape it:

```jsx
// For a named export:
const MyComponentLazy = lazy(async () => {
  return {
    default: (await import('./my-module')).MyComponent,
  };
});

// For a default export (simpler):
const MyComponentLazy = lazy(() => import('./my-module'));
```

### Chunk Extraction

The bundler must extract the lazy component into its own chunk. Some bundlers (like Vite) do this automatically when they detect a dynamic `import()`. However, if you have manual chunk rules that override this (e.g., putting all `node_modules` into a `vendor` chunk), the lazy component's dependencies may still end up in the vendor chunk.

In the study project, a lazy-loaded message editor produced a tiny 4.86 KB chunk for its own code, but the heavy editor libraries (prosemirror, tiptap -- 295 KB) stayed in the vendor chunk because of the `node_modules` rule. The fix was to create a separate chunk for editor dependencies:

```js
manualChunks: (id) => {
  if (id.includes('node_modules/@tiptap') || id.includes('node_modules/prosemirror')) {
    return 'editor-vendor';
  }
  if (id.includes('node_modules')) {
    return 'vendor';
  }
  return null;
},
```

After this, the vendor chunk was halved, and the lazy editor-vendor chunk was properly excluded from `index.html` -- the bundler recognized it originated from a lazy import.

**Always verify** that lazy chunks do not appear in the generated `index.html`. If they are preloaded there, the lazy loading is pointless.

### When the Download Starts

The dynamic import (and the download) is triggered only when the lazy component **mounts**. If the lazy component is rendered conditionally:

```jsx
{clickedMessage ? (
  <MessageEditorLazy onClose={() => setClickedMessage(null)} />
) : null}
```

Then the download starts only when `clickedMessage` becomes truthy (i.e., when the user clicks a message). Before that, the lazy chunks are completely absent from the network.

If the lazy component is rendered unconditionally (always mounted), the download triggers during React's initial render -- but sequentially after the critical JavaScript loads, not in parallel. This actually makes performance **worse** than not using lazy at all, because instead of downloading everything in parallel, you now have a two-step sequential process.

## Intro to Suspense

`Suspense` is the component that makes lazy loading work properly. It detects whether any of its children are in the process of lazy-loading, and if so, it "suspends" the entire subtree: React skips those children, renders a fallback component instead, and focuses on rendering everything else that is not suspended.

### How Suspense Works

```jsx
const Parent = () => {
  return (
    <>
      <h1>Welcome!</h1>
      <Suspense fallback={<div>Loading...</div>}>
        <LazyButton />
      </Suspense>
    </>
  );
};
```

1. React encounters `<h1>Welcome!</h1>` -- renders it immediately.
2. React encounters the `Suspense` boundary. Inside is `LazyButton`, which is loading.
3. React renders the fallback (`<div>Loading...</div>`) and continues.
4. When the lazy component's Promise resolves, React replaces the fallback with the actual component.

Everything inside a single `Suspense` boundary is suspended together. If you have both `<h3>This is suspended</h3>` and `<LazyButton />` inside the same Suspense, both are hidden until the lazy load completes.

If the child is not actually lazy-loaded, `Suspense` detects this and renders it immediately -- it is a no-op in that case.

### Understanding `lazy` as a Promise

Since `lazy` just accepts a Promise, you can simulate loading delays for development:

```jsx
const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms));

const LazyButton = lazy(async () => {
  await sleep(3000); // simulate slow download
  return { default: Button };
});
```

This is useful for testing Suspense fallback UI without network throttling.

## Applying Suspense in Practice

### The Wrong Way: Lazy Without Suspense

If you use `lazy` without `Suspense`, the lazy component mounts, triggers its download, and React waits -- showing nothing. The UI appears blank or broken until the download completes. In the study project, this made LCP 200 ms worse than having no lazy loading at all (1.8 s vs 1.6 s).

### The Right Way: Lazy Inside Suspense

```jsx
<Suspense fallback={<LoadingOverlay />}>
  <MessageEditorLazy onClose={() => setClickedMessage(null)} />
</Suspense>
```

With Suspense, React renders the non-suspended parts immediately. In the study project, adding Suspense around the lazy editor improved LCP from 1.6 s to 1.2 s -- a 400 ms improvement. The page was fully interactive while the lazy content was still downloading.

### Suspense with Fallback for Conditional Components

When a lazy component is rendered conditionally (e.g., on click), a meaningful fallback is essential:

```jsx
{clickedMessage ? (
  <Suspense fallback={
    <div className="w-full h-full fixed top-0 left-0 opacity-50 bg-gray-300 z-50" />
  }>
    <MessageEditorLazy onClose={() => setClickedMessage(null)} />
  </Suspense>
) : null}
```

Without a fallback, the user clicks a message and nothing happens for potentially seconds -- they assume the UI is broken. With the overlay fallback, they see immediate feedback that something is loading.

After the first click, subsequent clicks show the component instantly -- the JavaScript is already downloaded and cached.

## Manual Code Splitting per Route

Route-based code splitting extracts each page into its own chunk via the bundler configuration:

```js
manualChunks: (id) => {
  if (id.includes('pages/dashboard')) return 'dashboard';
  if (id.includes('pages/inbox')) return 'inbox';
  if (id.includes('pages/login')) return 'login';
  if (id.includes('pages/settings')) return 'settings';
  if (id.includes('node_modules')) return 'vendor';
  return null;
};
```

However, manual per-route chunks alone do not improve first-time visitor LCP if the bottleneck is the vendor chunk (which did not change). They can help with **returning visitors** when code changes are isolated to one page: only the changed page's chunk needs re-downloading (e.g., 620 ms vs 640 ms LCP for returning visitors). But the improvement is modest.

## Lazy-loading per Route

The more powerful approach: replace direct page imports with lazy imports in the router:

```jsx
const DashboardPageLazy = lazy(async () => {
  return { default: (await import('./pages/dashboard')).DashboardPage };
});

// In App component:
<Suspense>
  <DashboardPageLazy />
</Suspense>
```

By itself, naive route-based lazy loading can make LCP **slightly worse** (1.07 s vs 0.915 s in the study project) because it creates a sequential download pattern instead of parallel. The benefit comes when you combine it with critical path optimization.

## Loading Critical Elements First

The real power of lazy loading is **controlling what renders first**. By moving critical elements outside the Suspense boundary, you define the "critical path" from React's perspective.

### Identifying the LCP Element

Use Chrome DevTools: record performance, click the LCP green label, find the "related node" in the Summary tab, then trace which React component renders it.

### Extracting the Critical Path

In the study project, the LCP element was the "My Dashboards" title, rendered inside `AppLayout`. By moving `AppLayout` outside the Suspense boundary:

```jsx
export default function App() {
  return (
    <AppLayout>
      <Suspense>
        <DashboardPageLazy />
      </Suspense>
    </AppLayout>
  );
}
```

LCP dropped from 1.07 s to 763 ms -- almost 300 ms improvement. The full page content loads later, but the user sees the most important element immediately.

Going further, lazy-loading the Sidebar (not part of the LCP) brought LCP down to 695 ms:

```jsx
// In AppLayout:
const SidebarLazy = lazy(async () => {
  return { default: (await import('./sidebar')).Sidebar };
});

<Suspense fallback={<div className="sidebar-fallback" />}>
  <SidebarLazy />
</Suspense>
```

### Performance Comparison Table

| Scenario | Home LCP |
|---|---|
| Baseline (non-split, first visit) | 915 ms |
| Manual chunks via config (first visit) | 992 ms |
| Lazy-loading, LCP on critical path (first visit) | 695 ms |
| Framework (TanStack), LCP on critical path (first visit) | 726 ms |
| Baseline (repeated visit, code change) | 640 ms |
| Manual chunks (repeated visit, code change) | 620 ms |
| Lazy-loading, LCP on critical path (repeated visit) | 545 ms |
| Framework (TanStack), LCP on critical path (repeated visit) | 628 ms |

A 15-25% improvement in LCP numbers for manual lazy-loading. The framework adds slight overhead for first-time visitors (726 ms vs 695 ms) and loses more on repeated visits (628 ms vs 545 ms) due to its default chunking strategy (see [Framework Chunking Strategy](#framework-chunking-strategy) below).

## Preloading Lazy Chunks Manually

The trade-off of lazy loading: when navigating to a new page, there is a blank screen while the new page's JavaScript downloads. To fix this, preload chunks after the critical resources are loaded:

```js
// At the top of dashboard.tsx -- preload Settings page
import('./settings');
```

In Vite, this simple `import()` statement is enough -- the bundler handles the rest. The settings chunk downloads in the background without blocking the current page's content. Navigation from Home to Settings becomes nearly instantaneous.

For small projects, manual preloading is fine. For larger projects, it becomes unmanageable.

## Preloading Lazy Chunks with Link Components

A more scalable approach: tie preloading to navigation links. Create a mapping between paths and their dynamic imports:

```jsx
const preloadingMap = {
  '/': () => import('./pages/dashboard'),
  '/settings': () => import('./pages/settings'),
  '/inbox': () => import('./pages/inbox'),
  '/login': () => import('./pages/login'),
};

// Inside Link component:
useEffect(() => {
  if (href && preloadingMap[href]) {
    preloadingMap[href]();
  }
}, [href]);
```

When a Link component mounts and the sidebar renders, all reachable pages start preloading -- but only after the main content is rendered, so they do not compete with the critical path.

This is still rudimentary. A proper implementation would need to:

- Generate the path-to-chunks mapping from the bundler (manifest file)
- Preload only links in the viewport (not all links on the page)
- Support hover-based preloading for even better perceived performance
- Handle chunks that contain shared components like the Sidebar

For larger projects with dozens of links on a page, preloading all of them simultaneously is not only unnecessary but can clog the network and delay the loading of critical resources. More targeted strategies provide better results:

- **Viewport-based preloading**: start preloading when a link scrolls into view. Good for pages where the user is likely to click any visible link.
- **Hover/intent-based preloading**: wait until the user signals intent by mousing over a link. Most conservative network usage, works well when there are many links but users typically click only one.

For small projects, preloading all links on mount is acceptable. For anything beyond that, viewport or hover-based preloading avoids unnecessary network contention.

This is exactly what frameworks provide out of the box.

## Bundles, Loading, and Frameworks

Modern frameworks handle code splitting, lazy loading, and preloading automatically:

**TanStack Start** -- File-based routing with automatic code splitting per route. LCP on critical path: 726 ms (comparable to manual implementation at 695 ms). Default preloading strategy: "intent" (preload on hover). Also supports "viewport" and "render" strategies via a single prop change.

**React Router v7** (formerly Remix) -- Framework mode with SSR support, automatic code splitting, and built-in preloading.

**Next.js** -- SSR by default, automatic page-level code splitting, integrated preloading.

### Framework Evaluation Checklist

The specific framework does not matter if you know the fundamentals. Learn the underlying concepts first, and you can switch between frameworks quickly, understand their strengths and weaknesses, take advantage of their features, compensate for their flaws, and adjust their defaults to your needs.

When evaluating any framework's bundling and loading behavior, ask these five questions:

1. **How to initialize a project?** Typically there will be a CLI command or a starter repository to clone.
2. **SSR vs CSR by default?** This fundamentally changes the performance profile and the role of lazy loading (SSR makes LCP dependent on CSS, not JS).
3. **How does routing work?** File-based routing (each page is its own file), code-based routing (routes declared in code), or a hybrid of both. File-based routing makes the project structure immediately clear.
4. **How do Link components work and what is their preloading strategy?** Every framework provides a `Link` component and hooks like `useNavigate`, `useLocation`, `useRouter`. The preloading strategy attached to `Link` determines when route chunks are fetched.
5. **What is the default chunking strategy for vendor vs app code?** Whether the framework separates third-party libraries from application code into different chunks directly affects repeated-visitor caching performance.

Everything else in the framework's documentation can be learned when needed. These five questions give you enough to migrate a project and understand the performance implications.

### Framework Chunking Strategy

Frameworks perform automatic code splitting per route, which is convenient -- you get route-level chunks without manual `lazy` wrappers or bundler configuration. However, frameworks typically do not create separate vendor chunks by default. Third-party libraries end up in the same bundles as application code.

This means that when application code changes (even on a different page), the chunks containing vendor code also get new hashes, forcing returning visitors to re-download libraries that have not actually changed. In the study project, this caused repeated-visitor LCP of 628 ms with TanStack vs 545 ms with manual chunking -- an 83 ms penalty.

Since TanStack (and many other frameworks) uses Vite underneath, this is fixable with manual Vite chunk configuration:

```js
// vite.config.ts -- override the framework's default chunking
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: (id) => {
          if (id.includes('node_modules')) {
            return 'vendor';
          }
          return null;
        },
      },
    },
  },
});
```

Whether this optimization is worth the added configuration depends on the project. For apps with stable dependencies and frequent code changes, separating vendor chunks measurably helps returning visitors.

### Framework Preloading Strategy

With manual implementations, preloading requires creating and maintaining a map of paths to their corresponding dynamic imports, modifying the `Link` component, and importing chunks manually. Frameworks handle all of this automatically.

TanStack Start uses the "intent" preloading strategy by default: route chunks are not loaded on initial page render, but when the user hovers over a link leading to that route, the corresponding chunks appear in the network panel. This provides a good balance between network efficiency and navigation speed.

TanStack also supports two other strategies via a single prop change on `Link` or a global setting in config:

- **`"viewport"`** -- preload when the link becomes visible on screen. Good for pages where any visible link is a likely navigation target.
- **`"render"`** -- preload when the `Link` component mounts (equivalent to the manual `useEffect`-based approach). Most aggressive, preloads all linked routes immediately after render.

```jsx
// Changing the preloading strategy on a single Link
<Link to="/settings" preload="viewport">Settings</Link>

// Or setting it globally in the router configuration
```

The study project on TanStack showed slightly worse repeated-visitor LCP (628 ms vs 545 ms manual) because the framework does not separate vendor chunks by default. This is fixable with manual Vite configuration if needed.

## Lazy Loading + SSR

Lazy loading interacts with SSR in a specific way. The key measurements from the study project:

**CSR Baseline:** LCP = 915 ms, interactive at 915 ms (same time).

**SSR without lazy loading:** LCP = 600 ms, interactive at ~850 ms. The page is visible 315 ms earlier but has a 250 ms gap of being visible but non-interactive.

**SSR with lazy loading (LCP on critical path):** LCP = 600 ms (unchanged -- bottleneck is CSS, not JS), layout interactive at ~720 ms, lazy content interactive at ~1 s.

With SSR + lazy loading, the AppLayout (search, navigation) becomes interactive 130 ms earlier than without lazy loading. The lazy-loaded page content takes longer to become interactive, but the most important elements are functional sooner.

The LCP improvement from lazy loading primarily benefits CSR. For SSR, the LCP bottleneck is typically the CSS file download, which lazy loading does not affect. The benefit for SSR is in reducing the time-to-interactive for the critical path elements.

## Suspense Boundaries (Nesting Strategies)

Where you place `Suspense` boundaries determines the loading experience:

**One Suspense per component:** Independent loading, each component appears when ready.

```jsx
<Suspense fallback={<SidebarSkeleton />}>
  <Sidebar />
</Suspense>
<Suspense fallback={<ContentSkeleton />}>
  <MainContent />
</Suspense>
```

**Shared Suspense:** Components load together, appear at the same time.

```jsx
<Suspense fallback={<PageSkeleton />}>
  <Sidebar />
  <MainContent />
</Suspense>
```

**Nested Suspense:** Creates a data fetching waterfall -- the inner Suspense only starts loading after the outer one resolves. Use this intentionally when there is a true dependency.

```jsx
<Suspense>
  <ParentComponent>
    {/* This only starts loading after ParentComponent resolves */}
    <Suspense>
      <ChildComponent />
    </Suspense>
  </ParentComponent>
</Suspense>
```

The correct strategy depends on the visual design and the priority of each section. The LCP element should always be outside any Suspense boundary (on the critical path).

## Key Takeaways

- Lazy loading without Suspense makes performance **worse** by creating sequential downloads.
- The biggest LCP wins come from moving the LCP element **outside** Suspense boundaries onto the critical path.
- Manual preloading is simple but does not scale; frameworks provide automatic preloading via Link components.
- Conditional lazy loading (load on click) needs a meaningful fallback to avoid the appearance of a broken UI.
- For SSR, lazy loading primarily helps reduce time-to-interactive for critical path elements, not LCP itself.
- Route-based lazy loading alone does not help first-time visitors if the vendor chunk is the bottleneck.
- Always verify that lazy chunks are excluded from `index.html` -- check the generated HTML after any bundler configuration changes.
- Frameworks handle code splitting, preloading, and SSR integration automatically, but understanding the fundamentals lets you evaluate their default behavior and customize when needed.

## See Also

- [Bundle Size Optimization](bundle-size-optimization.md)
- [React Server Components](react-server-components.md)
- [Rendering Strategies](rendering-strategies.md)
- [Data Fetching Patterns](data-fetching-patterns.md)
- [Web Performance Metrics](web-performance-metrics.md)
