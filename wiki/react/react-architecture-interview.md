# React Architecture & Performance — Interview Questions

> Sources: GreatFrontEnd, Unknown
> Raw: [React Interview Questions — Extended](../../raw/react/2026-04-14-react-interview-questions-extended.md)

## Overview

Interview questions on React's architecture and performance topics: Virtual DOM, reconciliation, re-rendering, Context pitfalls and optimization, HOCs, composition, code splitting, Suspense, hydration, SSR, static generation, state management decision-making, testing, and portals.

---

## What is re-rendering in React?

Re-rendering is React calling a component's function again to produce an updated virtual DOM tree, after which React diffs it against the previous tree and patches only the changed DOM nodes.

### What triggers a re-render

1. **State update** (`setState` / state setter)
2. **New props** from a parent that re-rendered
3. **Parent re-renders** — by default, all children re-render even if their props haven't changed
4. **Context value change** — all consumers of that context re-render

### The re-render pipeline

```
Trigger → React re-runs the component function
         → Produces new virtual DOM (React elements)
         → Diffs new vs old virtual DOM (reconciliation)
         → Commits minimal real DOM changes
```

### Preventing unnecessary re-renders

| Tool | What it does |
|------|-------------|
| `React.memo` | Wraps a component; skips re-render if props are shallowly equal |
| `useMemo` | Memoizes a computed value; stable reference prevents child re-renders |
| `useCallback` | Memoizes a function; stable reference with `memo` children |
| `PureComponent` | Class component equivalent of `React.memo` |

> See also: [React Re-renders](react-re-renders.md), [Memoization in React](memoization.md)

---

## What is the Virtual DOM in React?

The Virtual DOM (VDOM) is a lightweight, in-memory JavaScript representation of the actual DOM tree. React keeps two copies — the current VDOM and the next VDOM — and compares them to compute the minimal set of real DOM changes needed.

### How it works

```
State change
  → React re-runs component render function
  → Produces new virtual DOM tree (plain JS objects)
  → Diffing algorithm compares new tree vs previous tree
  → Computes minimal patch (what changed, moved, added, removed)
  → Applies patch to real DOM
```

### Benefits

- **Batching**: Multiple state updates in one event handler → single DOM update
- **Minimal writes**: Real DOM operations are expensive; the VDOM diff minimizes them
- **Declarative**: You describe "what" the UI should look like; React figures out "how" to update the DOM

### Downsides

- **Memory overhead**: Maintaining two VDOM trees costs memory
- **Not always faster than direct DOM**: For very simple, infrequent updates, direct DOM access can be faster than the VDOM overhead
- **Initial render cost**: The first render still creates the full VDOM and real DOM

> See also: [Reconciliation and Diffing](reconciliation-and-diffing.md)

---

## What is reconciliation in React?

Reconciliation is the algorithm React uses to determine what changed between two virtual DOM trees and how to update the real DOM efficiently.

### Two key heuristics

1. **Type-based comparison**: Two elements of different types produce entirely different trees. React tears down the old subtree and builds a new one from scratch.

   ```jsx
   // Old: <div><Counter /></div>
   // New: <span><Counter /></span>
   // → div→span type change: Counter is unmounted and remounted, losing state
   ```

2. **Position-based comparison with keys**: Among siblings, React matches elements by position. If you provide `key` props, React uses those to match across positions — enabling efficient reordering without destroying state.

### Why this matters in practice

```jsx
// This DESTROYS Counter state on every toggle — types differ at same position
{isEditing ? <EditForm /> : <ViewForm />}

// This PRESERVES state (same type, same position)
{isEditing ? <Form editing={true} /> : <Form editing={false} />}
```

> See also: [Reconciliation and Diffing](reconciliation-and-diffing.md)

---

## What are some pitfalls about using Context in React?

### Pitfall 1 — Broad re-renders (the main one)

Any component that calls `useContext(MyContext)` re-renders whenever the context **value reference** changes — even if the part of the context the component uses didn't change.

```jsx
// Every render of Parent creates a new object → all consumers re-render
<MyContext.Provider value={{ user, theme }}>
```

**Fix**: Memoize the value:
```jsx
const ctxValue = useMemo(() => ({ user, theme }), [user, theme]);
<MyContext.Provider value={ctxValue}>
```

### Pitfall 2 — Overusing Context for frequently-updating state

Context is well-suited for low-frequency globals (auth, theme, locale). Using it for high-frequency data (mouse position, scroll offset, counter) triggers too many re-renders.

### Pitfall 3 — Debugging difficulty

Context changes can cascade re-renders across many unrelated components, making the source of a performance problem or stale-data bug hard to trace in DevTools.

### Pitfall 4 — No fine-grained subscriptions

Unlike state management libraries (Zustand, Jotai, MobX), Context has no built-in way to subscribe to only part of the context value.

> See also: [React Context and Performance](react-context-performance.md)

---

## How would you optimize the performance of React contexts to reduce re-renders?

### Strategy 1 — Split context by update frequency

Separate data that changes often from data that rarely changes:

```jsx
// Before: one context → any change re-renders all consumers
<AppContext.Provider value={{ user, notifications, theme }}>

// After: split by frequency
<ThemeContext.Provider value={theme}>        {/* rarely changes */}
  <UserContext.Provider value={user}>        {/* session-level */}
    <NotifContext.Provider value={notifications}> {/* frequent */}
```

### Strategy 2 — Memoize the provider value

```jsx
function UserProvider({ children }) {
  const [user, setUser] = useState(null);
  const value = useMemo(() => ({ user, setUser }), [user]);
  return <UserContext.Provider value={value}>{children}</UserContext.Provider>;
}
```

### Strategy 3 — Use React.memo on consumers

Wrap heavy consumers with `React.memo` so they skip re-renders when their own props haven't changed (they still re-render from context changes, but not from unrelated parent re-renders).

### Strategy 4 — Move context consumption down the tree

Don't consume context at a high level and pass it down as props; consume it as close to where it's used as possible to limit the blast radius.

### Strategy 5 — Consider an external library for complex cases

For high-frequency or large shared state, Zustand and Jotai offer selective subscriptions (components only re-render when the slice they subscribe to changes) — something Context can't provide natively.

> See also: [React Context and Performance](react-context-performance.md), [React State Management](state-management.md)

---

## What are higher-order components (HOCs) in React?

A higher-order component is a **function that takes a component and returns a new, enhanced component**. It's a pattern for reusing component logic.

```jsx
function withAuth(WrappedComponent) {
  return function AuthenticatedComponent(props) {
    const { isLoggedIn } = useAuth();
    if (!isLoggedIn) return <Redirect to="/login" />;
    return <WrappedComponent {...props} />;
  };
}

const ProtectedDashboard = withAuth(Dashboard);
```

### Common HOC use cases
- Authentication / authorization guards
- Logging and analytics
- Feature flags
- Data fetching wrappers (less common now that hooks exist)

### HOCs vs hooks

HOCs were the primary logic-reuse pattern before hooks (React 16.8). Custom hooks now handle most of these use cases with less indirection:

```jsx
// HOC approach
const Enhanced = withAuth(MyComponent);

// Hook approach (simpler, no wrapper component)
function MyComponent() {
  const { isLoggedIn } = useAuth();
  if (!isLoggedIn) return <Redirect to="/login" />;
  return <div>...</div>;
}
```

**Still reach for HOCs** when you need to wrap third-party components you don't control, or when the enhancement needs to work transparently with `ref` forwarding and display name conventions.

> See also: [Component Composition Patterns](component-composition-patterns.md)

---

## Explain the composition pattern in React

Composition is React's preferred approach for building flexible, reusable UIs. Instead of inheritance, components accept other components (or elements) as props — primarily through `children`, but also via named props.

### Children composition

```jsx
function Card({ children }) {
  return <div className="card">{children}</div>;
}

// Caller decides what goes inside
<Card>
  <h2>Title</h2>
  <p>Body text</p>
</Card>
```

### Slot-style composition (multiple named regions)

```jsx
function Layout({ sidebar, main, footer }) {
  return (
    <div className="layout">
      <aside>{sidebar}</aside>
      <main>{main}</main>
      <footer>{footer}</footer>
    </div>
  );
}

<Layout
  sidebar={<Nav />}
  main={<ArticleList />}
  footer={<Footer />}
/>
```

### Why composition over inheritance

- No prop drilling through intermediate components
- Parent retains full control over what goes in each "slot"
- Child components stay generic (no knowledge of their context)
- Easier to test each piece in isolation

> See also: [Component Composition Patterns](component-composition-patterns.md)

---

## How do you decide between React state, Context, and external state managers?

Choose based on **who needs the data** and **how often it changes**.

### Decision framework

```
Data only used in one component?
  → useState / useReducer (local state)

Data shared between a few nearby components?
  → Lift state to nearest common ancestor + props

Data needed across many levels but doesn't change often?
  → React Context (theme, locale, auth status)

Data needed across many components AND changes frequently?
  → External library (Zustand, Jotai, Redux Toolkit)

Server/async data (fetching, caching, synchronization)?
  → TanStack Query / SWR
```

### Comparison

| Solution | Best for | Re-render behavior |
|----------|----------|-------------------|
| `useState` / `useReducer` | Local, component-owned state | Only that component |
| Context | Infrequent global data | All consumers |
| Zustand / Jotai | Frequent shared state | Only subscribed components |
| Redux Toolkit | Large teams, complex state machines | Only subscribed selectors |
| TanStack Query | Remote/server state | Cache-aware, stale-while-revalidate |

> See also: [React State Management](state-management.md), [React Context and Performance](react-context-performance.md)

---

## What is code splitting in a React application?

Code splitting divides your JavaScript bundle into smaller chunks that load on demand, reducing initial bundle size and improving Time to Interactive (TTI).

### React's built-in mechanism

```jsx
import { lazy, Suspense } from 'react';

const HeavyChart = lazy(() => import('./HeavyChart'));

function Dashboard() {
  return (
    <Suspense fallback={<Spinner />}>
      <HeavyChart />
    </Suspense>
  );
}
```

`React.lazy()` + dynamic `import()` tells the bundler (webpack/Vite) to split `HeavyChart` into its own chunk. The chunk loads only when `Dashboard` renders `HeavyChart`.

### Route-level splitting (most impactful)

```jsx
const Home    = lazy(() => import('./pages/Home'));
const Profile = lazy(() => import('./pages/Profile'));

<Routes>
  <Route path="/"        element={<Suspense fallback={<Spinner />}><Home /></Suspense>} />
  <Route path="/profile" element={<Suspense fallback={<Spinner />}><Profile /></Suspense>} />
</Routes>
```

### What it doesn't do automatically

- Code splitting doesn't deduplicate shared modules (configure `optimization.splitChunks` in webpack for that)
- It doesn't prefetch future routes (add `<link rel="prefetch">` or use `React.lazy` with prefetch wrappers)

> See also: [Lazy Loading and Suspense](lazy-loading-and-suspense.md), [Bundle Size Optimization](bundle-size-optimization.md)

---

## What is React Suspense and what does it enable?

`Suspense` is a React component that lets you declaratively specify a loading state while its children are "waiting" for something — originally lazy-loaded code, now extended to data fetching in React 18+.

```jsx
<Suspense fallback={<LoadingSpinner />}>
  <LazyLoadedComponent />   {/* or a data-fetching component in React 18+ */}
</Suspense>
```

### What it enables

| Feature | Description |
|---------|-------------|
| Lazy loading | Works with `React.lazy()` — shows fallback while chunk loads |
| Data fetching (React 18+) | Libraries (TanStack Query, Relay) integrate to show fallback while data loads |
| Streaming SSR | Server sends HTML in chunks; Suspense boundaries define where to stream |
| Concurrent features | `useTransition` / `useDeferredValue` use Suspense for non-blocking UI updates |

### Nested boundaries

```jsx
// Outer boundary catches everything; inner boundary isolates sidebar
<Suspense fallback={<PageSkeleton />}>
  <MainContent />
  <Suspense fallback={<SidebarSkeleton />}>
    <Sidebar />
  </Suspense>
</Suspense>
```

React shows `<SidebarSkeleton>` while the sidebar loads and `<PageSkeleton>` only if the main content is also loading.

> See also: [Lazy Loading and Suspense](lazy-loading-and-suspense.md), [React Server Components](react-server-components.md)

---

## Explain what React hydration is

Hydration is the process by which client-side React takes over a server-rendered HTML page and makes it interactive — attaching event handlers and initializing component state without re-generating the DOM from scratch.

### The SSR + hydration flow

```
Server:  ReactDOMServer.renderToString(<App />) → static HTML → sent to browser
Browser: Displays HTML immediately (fast FCP)
         → React downloads JS bundle
         → ReactDOM.hydrateRoot(root, <App />) → React walks the existing DOM,
           attaches event listeners, initializes state
         → Page becomes interactive (TTI)
```

### Benefits
- Fast First Contentful Paint (content visible before JS loads)
- SEO-friendly (crawlers see pre-rendered HTML)
- Reduced client rendering work vs full CSR

### Key challenge — hydration mismatch

If the HTML React generates on the client differs from the server's HTML, React logs a warning and may discard the server HTML. Common causes: `Date.now()`, `Math.random()`, browser-only APIs called during render, timezone differences.

> See also: [Rendering Strategies](rendering-strategies.md)

---

## Explain server-side rendering (SSR) of React applications and its benefits

SSR generates the full HTML on the server for each request and sends it to the browser. The client then hydrates the HTML to add interactivity.

```
Browser requests /page
  → Server runs React render → full HTML string → response
  → Browser paints HTML immediately
  → Browser loads JS → React hydrates
  → Page is interactive
```

### Benefits

| Benefit | Why |
|---------|-----|
| Fast FCP/LCP | Content arrives pre-rendered; no waiting for JS to render |
| Better SEO | Search engines index fully-rendered HTML |
| Good for slow devices | Less client-side JS execution |

### Downsides

| Downside | Why |
|---------|-----|
| Server load | Every request runs React render on the server |
| Higher TTFB | Server must complete render before first byte arrives |
| Hydration cost | Client still downloads and runs all JS to attach interactivity |

### Frameworks: Next.js (App Router), Remix

> See also: [Rendering Strategies](rendering-strategies.md)

---

## Explain static generation (SSG) of React applications and its benefits

Static generation pre-renders HTML at **build time** (not per-request). The resulting HTML files are cached and served by a CDN with no server-side rendering at runtime.

```
npm run build
  → React renders all pages → static HTML files
  → Deploy files to CDN

Browser requests /page
  → CDN returns pre-built HTML immediately (no server processing)
  → Browser hydrates
```

### Benefits

| Benefit | Why |
|---------|-----|
| Ultra-fast TTFB | CDN edge serves pre-built HTML with no server round-trip |
| Infinite scalability | No per-request compute |
| Zero server cost per request | HTML is a file, not a computation |
| Great SEO | Fully-rendered HTML |

### Limitations

| Limitation | Why |
|-----------|-----|
| Stale data | HTML is baked at build time; data changes require a rebuild |
| Long build times | Thousands of pages → slow builds |
| Not suitable for user-specific pages | Same HTML served to everyone |

### Hybrid approach (ISR — Incremental Static Regeneration)

Next.js supports revalidating individual pages on a time interval or on-demand, combining SSG speed with near-real-time data.

> See also: [Rendering Strategies](rendering-strategies.md)

---

## What are React portals used for?

Portals render a component's output into a different DOM node than the parent, while keeping it in the React component tree for event bubbling purposes.

```jsx
import { createPortal } from 'react-dom';

function Modal({ isOpen, children }) {
  if (!isOpen) return null;
  return createPortal(
    <div className="modal-overlay">
      <div className="modal-content">{children}</div>
    </div>,
    document.getElementById('modal-root'), // renders here in the real DOM
  );
}
```

### Use cases

| Use case | Why portal helps |
|----------|----------------|
| Modals | Escape `overflow: hidden` and `z-index` stacking on ancestor elements |
| Tooltips / popovers | Absolute positioning without being clipped by parent overflow |
| Toast notifications | Fixed-position, application-level, independent of trigger location |

### Key behavior: React event bubbling is preserved

Even though the portal renders outside the parent's DOM subtree, React events inside the portal still bubble up through the **React component tree** — not the DOM tree. A click inside a modal portal reaches the modal's React parent.

> See also: [React Portals](react-portals.md)

---

## How do you test React applications?

Testing React apps typically involves three layers:

### Unit tests — Jest + React Testing Library

Test individual components in isolation. The React Testing Library philosophy: test what the **user sees and does**, not implementation details (no testing internal state or component methods).

```jsx
import { render, screen, fireEvent } from '@testing-library/react';

test('increments count on click', () => {
  render(<Counter />);
  fireEvent.click(screen.getByRole('button', { name: /increment/i }));
  expect(screen.getByText(/count: 1/i)).toBeInTheDocument();
});
```

### Integration tests — React Testing Library

Render multiple components together to test their interactions. Same tools as unit tests, larger scope.

### End-to-end tests — Cypress or Playwright

Run the full app in a real browser. Test complete user flows (login → create item → logout).

```javascript
// Cypress example
cy.visit('/login');
cy.get('[data-testid="email"]').type('user@example.com');
cy.get('[data-testid="password"]').type('password');
cy.get('button[type="submit"]').click();
cy.url().should('include', '/dashboard');
```

### Snapshot tests — Jest

Capture rendered HTML output and compare to a saved snapshot to detect unintended changes. Low signal/noise ratio — use sparingly for components with stable output.

### Best practices
- Query by accessible roles (`getByRole`, `getByLabelText`) rather than CSS selectors
- Avoid testing implementation details (internal state, method calls)
- Keep E2E tests for critical paths; unit/integration for component logic

## See Also

- [React Re-renders](react-re-renders.md)
- [Reconciliation and Diffing](reconciliation-and-diffing.md)
- [React Context and Performance](react-context-performance.md)
- [React State Management](state-management.md)
- [Component Composition Patterns](component-composition-patterns.md)
- [Lazy Loading and Suspense](lazy-loading-and-suspense.md)
- [Bundle Size Optimization](bundle-size-optimization.md)
- [Rendering Strategies](rendering-strategies.md)
- [React Portals](react-portals.md)
- [Error Handling in React](error-handling.md)
- [React Fundamentals — Interview Questions](react-fundamentals-interview.md)
- [React Hooks — Interview Questions](react-hooks-interview.md)
- [React Components & State — Interview Questions](react-components-interview.md)
