# React State Management in 2025: What You Actually Need

> Source: https://www.developerway.com/posts/react-state-management-2025
> Collected: 2026-04-14
> Published: 2025-09-25

## Why Do You Want to Make a State Management Decision?

There's "no such thing as 'the best' anything" — it always depends on context.

Several scenarios are addressed:

**For job seekers:** Learn the most popular libraries from surveys like "State of React," or better yet, master core React concepts like re-renders and reconciliation. Understanding those fundamentals makes all state management libraries feel like variations on the same theme.

**For teams unhappy with Redux on legacy projects:** The priority should be understanding *why* the current solution causes frustration before replacing it. Swapping Redux for something like XState to solve a "too complicated" problem risks creating new complexity while requiring significant team upskilling.

**For those blaming performance on the library:** The author urges measurement with actual numbers, betting that the library's own impact is negligible compared to unoptimized re-render patterns or slow critical-path calculations.

**For greenfield projects:** The remainder of the article is targeted at this audience — choosing a tech stack free from legacy constraints.

A key insight: investigating current code problems often reveals that ~80% of Redux-related code handles concerns better served by specialized libraries. The best migration path may involve replacing one library with three purpose-built ones.

## State Concerns That Don't Need a State Management Library

State is defined as data influencing system behavior. In React terms: modal open/closed status, data fetching status, component lifecycle status, or anything affecting rendering and user interaction responses.

### Remote State

Remote data (from databases, REST endpoints, etc.) is described as one of the most complicated state management use cases. Even basic "fetch and render" requires handling multiple states:

- No data yet
- Loading
- Data loaded successfully
- Error occurred

Beyond that, applications need: request deduplication, caching across pages, cache invalidation, avoiding request waterfalls, handling race conditions from user-interaction-triggered fetches, and optimistic updates with data integrity.

The author estimates that "~80% of your Redux-related code handles everything above" in legacy applications.

**Recommended solution: TanStack Query** (formerly React Query)

Basic usage example:

```tsx
function Component() {
  const { isPending, error, data } = useQuery({
    queryKey: ['my-data'],
    queryFn: () => fetch('https://my-url/data').then((res) => res.json()),
  });

  if (isPending) return 'Loading...';

  if (error) return 'Oops, something went wrong';

  return ... // render whatever here based on the data
}
```

Key capabilities highlighted:
- Automatic request deduplication via shared `queryKey`
- Prefetching via `queryClient.prefetchQuery`
- Built-in paginated queries, optimistic updates, and configurable retries

**Alternative: SWR** — described as equally capable with a slightly different API. SWR is maintained by Vercel, while TanStack Query has independent maintainers. The choice comes down to trust and API preference.

### URL State

URL query parameters represent state that should stay synchronized with the UI. Example URL: `/somepath?search="test"&tab=1&sidebar=open&onboarding=step1`

React Router provides `useSearchParams` for this:

```tsx
export function SomeComponent() {
  const [searchParams, setSearchParams] = useSearchParams();
  // ...
}
```

However, Next.js and other routers make URL-state synchronization painful. Manual syncing produces "misery and weird bugs."

**Recommended solution: nuqs**

Onboarding state synced with URL:

```tsx
export function MyApp() {
  const [step, setStep] = useQueryState('onboarding');

  return (
    <>
      <button onClick={() => setStep('step2')}>Next Step</button>
    </>
  );
}
```

Tab state with type parsing and defaults:

```tsx
export function MyApp() {
  const [tab, setTab] = useQueryState('tab', parseAsInteger.withDefault(1));

  return (
    <>
      <button onClick={() => setTab(2)}>Second tab</button>
    </>
  );
}
```

### Local State

State that exists within a single component's logical boundaries needs no external library. Examples: dropdown open/closed, tooltip visibility, component mount status.

```tsx
// the isOpen state is localized and doesn't need sharing
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

The author notes that older Redux-heavy apps often managed *all* state through the library, but new projects have "zero reason" to do this for local concerns.

## Shared State

The interesting challenge arises when state must be shared across loosely related components. The sidebar example illustrates this: a collapsible panel controllable via a hover button, drag interaction, top-bar icon, keyboard shortcut, and full-screen mode toggle — components that don't know about each other.

### Shared State and Props Drilling

The "lifting state up" approach moves state to the nearest common ancestor and passes values/setters through props:

```tsx
export function MyApp() {
  const [isSidebarOpen, setIsSidebarOpen] = useState(false);
  return (
    <>
      <TopBar
        isSidebarOpen={isSidebarOpen}
        setIsSidebarOpen={setIsSidebarOpen}
      />
      <div className="content">
        <Sidebar
          isOpen={isSidebarOpen}
          setIsOpen={setIsSidebarOpen}
        />
        <div className="main">
          <MainArea
            isSidebarOpen={isSidebarOpen}
            setIsSidebarOpen={setIsSidebarOpen}
          />
        </div>
      </div>
      <KeyboardShortcutsController
        isSidebarOpen={isSidebarOpen}
        setIsSidebarOpen={setIsSidebarOpen}
      />
    </>
  );
}
```

Problems identified:
- State references scattered everywhere, reducing readability
- Refactoring requires changing every component in the chain
- Every component in the hierarchy re-renders on state change with no way to prevent it — memoization won't help since every prop changes

The pattern is viable for shallow parent/child sharing, but problematic beyond three hierarchy levels or when used more than two or three times across an app.

### Shared State and Context

Context bypasses hierarchy levels by allowing direct data access. Implementation involves three steps:

**1. Create Context with defaults:**

```tsx
const SidebarContext = React.createContext({
  isSidebarOpen: false,
  setIsSidebarOpen: (isOpen: boolean) => {},
});

// Expose the Context via hook:
const useSidebarContext = () => React.useContext(SidebarContext);
```

**2. Create Provider component:**

```tsx
const SidebarProvider = ({ children }) => {
  const [isSidebarOpen, setIsSidebarOpen] = React.useState(false);

  const value = React.useMemo(
    () => ({ isSidebarOpen, setIsSidebarOpen }),
    [isSidebarOpen],
  );

  return (
    <SidebarContext.Provider value={value}>
      {children}
    </SidebarContext.Provider>
  );
};
```

**3. Wrap the app and consume directly:**

```tsx
export function MyApp() {
  return (
    <SidebarProvider>
      <TopBar />
      <div className="content">
        <Sidebar />
        <div className="main">
          <MainArea />
        </div>
      </div>
      <KeyboardShortcutsController />
    </SidebarProvider>
  );
}
```

The Provider can expose a richer API (like `toggleSidebar`) to encapsulate logic:

```tsx
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

  return ... // same as before
};
```

**Benefits over prop drilling:**
- Clean, readable code without unnecessary props
- Refactoring only affects components that actually consume the Context
- Encapsulated APIs mean provider internals can change without consumer changes
- Performance can *improve* since middle-of-hierarchy components no longer re-render

### Don't Use Context for Shared State

Despite its benefits, the author strongly advises against extensive Context use. The core problems:

**Providers Hell** — Each shared state concern requires its own provider, leading to deep nesting. Providers proliferate, develop interdependencies, get misplaced during refactoring, and cause confusion. Context works for one or two shared concerns (like Theming and UserAuthStatus) but degrades quickly beyond that.

**The merged-state trap** — Combining everything into one provider to avoid nesting:

```tsx
const SharedStateContext = React.createContext({
  sidebar: {
    isSidebarOpen: false,
    setIsSidebarOpen: (isOpen: boolean) => {},
    toggleSidebar: () => {},
  },
  theming: {
    isDarkMode: false,
    setIsDarkMode: (isDark: boolean) => {},
    toggleDarkMode: () => {},
  },
  user: {
    ... // user-auth-related state
  },
});
```

**The fatal flaw:** When Context value changes, *every* consumer re-renders regardless of which part of state actually changed. `SmallToggleSidebarButton` re-renders when theming changes even though it only uses sidebar state. At scale, this means every interaction triggers app-wide re-renders.

Splitting providers into smaller pieces is possible but amounts to "re-inventing Zustand."

## Shared State and External Libraries

External state management libraries become appropriate only when shared state concerns exceed what Context can handle cleanly. The pre-work of adopting specialized tools for remote state (~80%), URL state (~10%), and local state means only ~10% remains for a state management library.

### Evaluation Criteria

1. **Simplicity** — The remaining 10% doesn't justify cognitive overhead. The library should be intuitive without documentation study.
2. **One or fewer providers** — Must avoid Context's provider nesting problem.
3. **Selective re-renders** — Components should only update when their specific consumed state slice changes.
4. **React compatibility** — Must support latest React, SSR, RSC, hooks-based patterns, unidirectional data flow, and declarative paradigms. Signals and observables fail this criterion.
5. **Open source sustainability** — Needs either high popularity or reputable maintainers/organizations.
6. **Switchability** — Simplicity and React alignment enable easier migration.

### Library Evaluations

From the "State of React" survey:

| Library | Simplicity | Providers | Selective Re-renders | React Compat | OS Health |
|---------|-----------|-----------|---------------------|-------------|-----------|
| Redux | 👎 Notorious boilerplate | 🎉 | 🎉 Via selectors | — | 🎉 |
| Redux Toolkit | 😐 Still many concepts | 🎉 | 🎉 Via selectors | 🤨 | 🎉 |
| Zustand | 🎉 Winner | 🎉 | 🎉 Out-of-the-box | 🎉 | 🎉 |
| MobX | 👎 Signals/observables | 🎉 | Likely | 👎 Not React way | 🎉 |
| Jotai | 👎 Abstract atoms | 🎉 | Likely | 🎉 Same author as Zustand | 🎉 |
| XState | 👎 State machines | 🎉 | Likely | 👎 Event-driven | 🎉 |

### Result: Recommended Stack

- **TanStack Query** — remote/server state
- **nuqs** — URL query parameter state
- **Zustand** — shared client state

### Alternative Winners

- **"Structured, opinionated"** priority → **Redux Toolkit**. Enforces consistency, valuable in large organizations.
- **"Advanced debugging"** priority → **Redux Toolkit**. Redux DevTools plugin.
- **"State machines" pattern** for Figma-level complexity → **XState**.
