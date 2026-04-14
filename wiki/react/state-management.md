# React State Management

> Sources: Nadia Makarevich, 2025-09-25
> Raw: [React State Management in 2025](../../raw/react/2025-09-25-react-state-management-2025.md)

## Overview

Most React applications do not need a general-purpose state management library. By categorizing state into distinct concerns — remote state, URL state, local state, and shared state — each concern maps to a specialized tool that handles it better than any single "do-everything" library. In legacy Redux-heavy codebases, roughly 80% of Redux code typically handles remote data concerns (caching, deduplication, invalidation) that libraries like TanStack Query solve out of the box. Another ~10% handles URL synchronization that tools like nuqs solve. Only the remaining ~10% — truly shared client-side state — warrants a state management library, and even then only when React Context becomes insufficient.

## State Categories

### Remote State (~80% of "state management" code)

Remote data from APIs, databases, or external services is the most complex state concern. A basic "fetch and render" already requires handling four states: no data yet, loading, success, and error. Production applications additionally need:

- Request deduplication (multiple components fetching the same endpoint)
- Caching across page navigations
- Cache invalidation
- Avoiding request waterfalls
- Race condition handling from user-triggered fetches
- Optimistic updates with data integrity

**Recommended: TanStack Query** (formerly React Query):

```tsx
function Component() {
  const { isPending, error, data } = useQuery({
    queryKey: ['my-data'],
    queryFn: () => fetch('https://my-url/data').then((res) => res.json()),
  });

  if (isPending) return 'Loading...';
  if (error) return 'Oops, something went wrong';

  return <div>{/* render based on data */}</div>;
}
```

Key capabilities: automatic request deduplication via shared `queryKey`, prefetching via `queryClient.prefetchQuery`, built-in paginated queries, optimistic updates, and configurable retries.

**Alternative: SWR** — equally capable with a slightly different API. SWR is maintained by Vercel; TanStack Query has independent maintainers.

### URL State (~10%)

URL query parameters that must stay synchronized with the UI (search terms, active tabs, sidebar state, onboarding steps). React Router's `useSearchParams` works for simple cases, but Next.js and other frameworks make URL-state synchronization painful with manual syncing producing "misery and weird bugs."

**Recommended: nuqs**

```tsx
export function MyApp() {
  const [tab, setTab] = useQueryState('tab', parseAsInteger.withDefault(1));

  return <button onClick={() => setTab(2)}>Second tab</button>;
}
```

Type-safe, supports parsing and defaults, works across frameworks.

### Local State

State within a single component's logical boundaries — dropdown open/closed, tooltip visibility, component mount status. Use React's built-in `useState` or `useReducer`. Older Redux-heavy apps managed *all* state through the library, but new projects have zero reason to externalize local concerns.

```tsx
export function CreateIssueComponent() {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <button onClick={() => setIsOpen(true)}>Open Create Issue Dialog</button>
      {isOpen && <CreateIssueDialog onClose={() => setIsOpen(false)} />}
    </>
  );
}
```

## Shared State: The Progression

Shared state is the genuinely interesting challenge — state consumed by loosely related components that don't know about each other. The sidebar example illustrates this: a collapsible panel controllable via a hover button, drag interaction, top-bar icon, keyboard shortcut, and full-screen toggle.

### Level 1: Props Drilling

Lift state to the nearest common ancestor and pass values/setters through props:

```tsx
export function MyApp() {
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  return (
    <>
      <TopBar isSidebarOpen={isSidebarOpen} setIsSidebarOpen={setIsSidebarOpen} />
      <Sidebar isOpen={isSidebarOpen} setIsOpen={setIsSidebarOpen} />
      <MainArea isSidebarOpen={isSidebarOpen} setIsSidebarOpen={setIsSidebarOpen} />
      <KeyboardShortcutsController isSidebarOpen={isSidebarOpen} setIsSidebarOpen={setIsSidebarOpen} />
    </>
  );
}
```

**Problems:**
- State references scattered everywhere, reducing readability
- Refactoring requires changing every component in the chain
- Every component in the hierarchy re-renders — memoization won't help since every prop changes

Viable for shallow parent/child sharing. Problematic beyond three hierarchy levels or when used more than two or three times across the app.

### Level 2: React Context

Context bypasses hierarchy levels by allowing direct data access:

```tsx
const SidebarContext = React.createContext({
  isSidebarOpen: false,
  setIsSidebarOpen: (isOpen: boolean) => {},
});

const useSidebarContext = () => React.useContext(SidebarContext);

const SidebarProvider = ({ children }) => {
  const [isSidebarOpen, setIsSidebarOpen] = React.useState(false);

  const value = React.useMemo(
    () => ({
      isSidebarOpen,
      setIsSidebarOpen,
      toggleSidebar: () => setIsSidebarOpen((prev) => !prev),
    }),
    [isSidebarOpen],
  );

  return (
    <SidebarContext.Provider value={value}>{children}</SidebarContext.Provider>
  );
};
```

**Benefits over props drilling:**
- Clean code without unnecessary props through intermediate components
- Refactoring only affects actual consumers
- Encapsulated API (e.g., `toggleSidebar`) hides implementation details
- Middle-of-hierarchy components stop re-rendering

**Where Context breaks down:**

1. **Providers Hell** — Each shared state concern needs its own provider, creating deep nesting. Providers proliferate, develop interdependencies, and get misplaced during refactoring. Works for one or two concerns (theming, auth status). Degrades quickly beyond that.

2. **The Merged-State Trap** — Combining all shared state into one provider to avoid nesting seems appealing but has a fatal flaw: when *any* part of the Context value changes, *every* consumer re-renders regardless of which slice they actually use. A sidebar toggle button re-renders when the theme changes. At scale, every interaction triggers app-wide re-renders.

Splitting the merged provider into smaller pieces is possible but amounts to "re-inventing Zustand."

### Level 3: External State Management Library

Appropriate only when shared state concerns exceed what Context handles cleanly. After adopting specialized tools for remote (~80%), URL (~10%), and local state, only ~10% of state needs a library.

## Library Evaluation Criteria

1. **Simplicity** — The remaining 10% doesn't justify heavy cognitive overhead. Intuitive without studying documentation.
2. **Zero or one provider** — Must avoid Context's nesting problem.
3. **Selective re-renders** — Components only update when their specific consumed state slice changes.
4. **React compatibility** — Support for SSR, RSC, hooks, unidirectional data flow, declarative paradigms. Signals and observables fail this criterion.
5. **Open source sustainability** — High popularity or reputable maintainers.
6. **Switchability** — Simplicity enables easier migration if needed.

## Library Comparison

| Library | Simplicity | Providers | Selective Re-renders | React Compat | OS Health |
|---------|-----------|-----------|---------------------|-------------|-----------|
| Redux | Notorious boilerplate | None needed | Via selectors | — | Healthy |
| Redux Toolkit | Still many concepts | None needed | Via selectors | Needs investigation | Healthy |
| **Zustand** | **Winner — two lines to start** | **None needed** | **Out-of-the-box** | **Confirmed** | **Healthy** |
| MobX | Signals/observables | None needed | Likely | Not React way | Healthy |
| Jotai | Abstract "atoms" concept | None needed | Likely | Same author as Zustand | Healthy |
| XState | State machines/actors | None needed | Likely | Event-driven, not declarative | Healthy |

**Zustand wins across all criteria** for the default case.

### Alternative Winners Under Different Priorities

- **"Structured, opinionated"** → **Redux Toolkit**. Enforces consistency — valuable in large organizations where "do whatever" flexibility becomes a liability.
- **"Advanced debugging"** → **Redux Toolkit**. Redux DevTools is unmatched.
- **"State machines"** for Figma-level complexity → **XState**. Event-driven architecture with formal state machine semantics.

## Recommended Stack for New Projects

| Concern | Tool | % of State |
|---------|------|-----------|
| Remote/server state | TanStack Query (or SWR) | ~80% |
| URL query parameters | nuqs | ~10% |
| Shared client state | Zustand | ~10% |
| Local state | React useState/useReducer | N/A |

Following this breakdown makes "~90% of your state management problems simply disappear."

## Key Takeaways

- There is no "best" state management library — it always depends on context and priorities
- Break state into categories (remote, URL, local, shared) and use specialized tools for each
- ~80% of Redux code in legacy apps handles remote data concerns better served by TanStack Query
- React Context works well for 1-2 shared state concerns; it degrades past that due to providers hell and the merged-state trap
- Only reach for a state management library when shared state exceeds Context's comfort zone
- Zustand is the recommended default for shared client state: simple, no providers, selective re-renders, React-compatible
- Redux Toolkit wins when consistency enforcement or advanced debugging is the priority
- Investigating *why* your current solution causes frustration is more important than picking a new library

## See Also

- [React Context and Performance](react-context-performance.md) — deep dive into Context re-render behavior, splitting patterns, and optimization techniques
- [Data Fetching Patterns](data-fetching-patterns.md) — waterfalls, race conditions, caching, and the evolution from client-side to server components
- [React Re-renders](react-re-renders.md) — how re-renders are triggered and propagate, the foundation for understanding state management performance
- [Component Composition Patterns](component-composition-patterns.md) — children as props, render props, and HOCs as alternatives to props drilling
- [Memoization in React](memoization.md) — useMemo, useCallback, React.memo and their relationship to state management performance
- [React Server Components](react-server-components.md) — server-side data fetching that eliminates client-side state for server data
