# Interaction Performance

> Sources: Nadia Makarevich, 2025
> Raw: [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)

## Overview

Interaction performance measures how quickly an application responds to user actions after the initial load: clicks, keypresses, toggling controls, navigating between SPA pages. While initial load performance captures the first impression, interaction performance determines every subsequent impression. The primary metric is **INP (Interaction to Next Paint)**, and the primary cause of poor interaction performance is **Long Tasks** blocking the main thread. In React applications specifically, the overwhelming majority of slow interactions trace back to unnecessary re-renders that force the browser to execute too much JavaScript in a single uninterrupted task. Fixing interaction performance requires a combination of profiling tools (Chrome DevTools, React DevTools Profiler), understanding the Long Task model, knowing how to yield to the main thread, and applying React-specific patterns to minimize re-render scope.

## SPA Transitions vs. Traditional Navigation

Single-Page Applications handle routing entirely on the client side. When a user clicks a link in an SPA, the default browser navigation is prevented. Instead, JavaScript updates the URL via `window.history.pushState` and dispatches a `popstate` event. A listener sets state with the new pathname, and React renders the appropriate page component:

```jsx
// Inside a Link component
<a
  onClick={(e) => {
    e.preventDefault();
    navigate(href);
  }}
>
  {children}
</a>

// Inside navigate()
window.history.pushState({}, '', newPath);
dispatchEvent(new PopStateEvent('popstate', { state: {} }));

// Somewhere else: listening and rendering
window.addEventListener('popstate', () => {
  setPath(window.location.pathname);
});

switch (path) {
  case "/login":
    return <LoginPage />;
  default:
    return <DashboardPage />;
}
```

All major frameworks (Next.js, Remix, TanStack Router) implement this pattern with varying degrees of abstraction.

### Performance Difference

The performance gap between SPA transitions and traditional navigation is dramatic. In the Performance panel with 6x CPU slowdown and Average 3G network:

- **SPA transition** (Dashboards to Login): ~60ms. The Network panel is empty (no server requests). The Main section shows a click event, JavaScript execution, style recalculation, layout, and painting.
- **Traditional navigation** (Dashboards to Settings via `<a>` tag): ~1 second. The full cycle returns: server request, CSS/JS download, HTML parsing. FCP/LCP metrics reappear.

The SPA transition is more than **10x faster** than a traditional page load. This speed is what makes SPAs attractive despite their initial load tradeoffs.

### No FCP/LCP for SPA Transitions

SPA transitions do not generate FCP or LCP metrics because no new HTML document is loaded. This is where **INP** becomes the relevant metric.

## Measuring INP (Interaction to Next Paint)

INP was introduced in 2023 as a Core Web Vital. It measures how fast the app responds to user interactions -- how "snappy" interactions feel. Think of LCP as the first impression and INP as the ongoing relationship with users.

INP records the **worst (highest) interaction latency** on a page. In SPAs, this score persists across SPA navigations but resets on traditional full-page navigations.

### Google's INP Thresholds

| Rating | INP Value |
|--------|-----------|
| Good | < 200ms |
| Needs Improvement | 200ms - 500ms |
| Poor | > 500ms |

These thresholds are context-dependent. A 150ms INP for page navigation feels fine, but the same value would make an animation look janky.

### Where to Find INP in Chrome DevTools

**Lighthouse Timespan Report**: Open Lighthouse, select "Timespan", start recording, interact with the app, and stop. This produces an overall INP score -- the highest value among all interactions during the recording. Useful for tracking the general health of interactions over time, but lacking detail on root causes.

**Performance Panel -- Live View**: Without recording, the bottom of the Performance panel shows a live list of interactions with their durations. The INP block (third of three summary blocks) updates to the highest value observed. This is a fast way to identify problematic interactions before diving deeper.

**Performance Panel -- Recording**: Start a recording, perform the interaction, and stop. The recording shows:

- **Interactions section**: Each interaction (pointer or keyboard), how long the browser thinks it took from input to screen update. Red diagonal lines indicate poor performance.
- **Main section**: The familiar flame graph of JavaScript execution, style recalculation, layout, and paint.

The interaction duration includes not just JavaScript execution but also style recalculation (purple blocks), layout (purple), and painting/compositing (green blocks). In a typical SPA transition, JavaScript might account for only half the total interaction time.

### INP and SPA Score Persistence

A critical subtlety: INP scores persist across SPA navigations but reset on full-page navigations. In SPAs, the page navigation is typically the interaction with the largest INP, which means other interactions can be overlooked. Hyper-focusing on the overall INP score can mask poor performance on smaller but frequent interactions like typing or toggling controls.

## Chrome DevTools for Interaction Debugging

### Production vs. Dev Mode

For initial load profiling, production builds are essential (tree shaking, minification affect bundle size). For interaction profiling, the situation is **reversed**:

- **Production builds** are useful only for detecting that an interaction is slow. The minified code turns function names into unreadable symbols, so you cannot determine *what* is happening.
- **Dev mode** is necessary for understanding *why* an interaction is slow. Component names appear in the flame graph, and React DevTools become available.

The workflow: identify the slow interaction in production, then switch to dev mode to diagnose the cause.

This is the exact opposite of initial load debugging, where production builds are essential and dev mode is avoided. For initial load, React code can be treated as a generic blob where only total size matters -- production builds with tree shaking and minification provide all the information needed. For interactions, you need to understand *what specific components* are executing and *why* -- and the production profiler shows nothing beyond minified variable names that are impossible to trace back to source code. A generic performance profile of a production build gives you little more than "React does something" with no way to determine what.

### CPU Throttling

Always set CPU throttling when profiling interactions. High-end developer laptops mask performance issues that real users experience on mobile devices. Recommended settings:

- **6x slowdown**: Simulates a mid-range mobile device
- **20x slowdown**: Stress test -- if interactions are green at 20x, they are green for everyone

Without throttling, even badly written interactions can appear fast on a modern MacBook.

### Reading the Flame Graph for Interactions

In dev mode, the flame graph reveals named function calls that correspond to React components. Seeing many component function calls stacked inside the JavaScript task for a single keypress is a strong signal of **unnecessary re-renders** -- React is calling many component functions to process a state update that should affect only a small part of the tree.

## The Long Tasks Problem

A **Long Task** is any JavaScript execution that takes more than **50ms**, blocking the main thread. During a Long Task, the browser cannot:

- Respond to user input
- Update the screen
- Run animations

Long Tasks are the primary cause of poor INP scores. In the Performance panel, they appear with a **red corner** on the task bar.

### How Tasks Work

All synchronous JavaScript runs as a single uninterrupted task:

```js
const sleep = (ms) => {
  const start = Date.now();
  while (Date.now() - start < ms) {
    // busy wait
  }
};

sleep(5000); // One single 5-second task -- browser is completely frozen
```

Tasks also originate from callbacks and asynchronous operations (event handlers, Promises). The browser queues these and processes them after the current task completes.

In React, each keypress in a search field triggers an `onChange` callback containing `setState`. If that state update causes the entire component tree to re-render, the resulting JavaScript execution becomes one long uninterrupted task:

```jsx
<InputWithIconsNormalToLarge
  value={search}
  onChange={(e) => setSearch(e.target.value)}
/>
```

If `setSearch` lives at the root of the app, every component re-renders synchronously, producing a task that can easily exceed 500ms on a throttled CPU.

### Two Categories of Solutions

Solutions fall into two broad categories:

1. **Split** the task into smaller chunks (yield to main thread)
2. **Shorten** the task by reducing the work (eliminate unnecessary re-renders)

## Yielding to the Main Thread

The "yield to main" pattern breaks a long synchronous task into smaller chunks, giving the browser opportunities to handle user events and paint between chunks.

### The Problem Setup

Multiple synchronous operations in sequence produce a single blocking task:

```js
// 10 operations x 10ms each = one 100ms blocking task
for (let i = 0; i < 10; i++) {
  sleep(10);
}
```

### scheduler.yield()

The latest API for yielding is `scheduler.yield()`. After each chunk of work, you `await` it to let the browser process any pending events:

```jsx
<InputWithIconsNormalToLarge
  onChange={async (e) => {
    for (let i = 0; i < 10; i++) {
      await scheduler.yield();
      sleep(10);
    }
  }}
/>
```

The result in the Performance panel: instead of one unbroken 100ms task, you see a series of small microtasks. The page stays responsive regardless of how fast you type. Everything in the Interactions block turns **green**, even with heavy CPU throttling.

### setTimeout Fallback

Since `scheduler.yield()` is not supported in all browsers (especially older mobile browsers where yielding matters most), a fallback using `setTimeout` works everywhere:

```js
const yieldToMainThread = () => {
  return new Promise((resolve) => {
    setTimeout(resolve, 0);
  });
};
```

Use it identically to `scheduler.yield()`:

```jsx
<InputWithIconsNormalToLarge
  onChange={async (e) => {
    for (let i = 0; i < 10; i++) {
      await yieldToMainThread();
      sleep(10);
    }
  }}
/>
```

The result is identical: the single task is broken into queued microtasks, and the browser remains responsive.

### When to Use Yielding in React

In practice, explicit yielding is **very rarely needed** in standard React UI applications. Most heavy computation is hidden behind state updates and delegated to React itself. You do not typically have explicit loops to yield from -- just an `onChange` callback with `setState` inside.

Yielding becomes useful when you have significant **non-React computation**, especially:

- Iterating over DOM nodes manually
- Custom browser-based games or animation frameworks
- Heavy data processing pipelines that are not delegated to React's rendering

In practice, the source author reports never having explicitly used `scheduler.yield()` in any standard React application. The technique exists for situations where *you* control the loop -- but in React, you rarely do. Your interaction handler calls `setState`, and React takes over from there. There is simply nothing in the typical React interaction handler to yield *between*.

For the vast majority of React interaction performance issues, the solution is not yielding but **reducing unnecessary re-renders**.

## React DevTools for Interactions

The Chrome Performance panel shows that "React does something," but cannot tell you what or why. React DevTools fills this gap.

### Highlight Updates on Render

Install the React DevTools Chrome extension (works only in dev mode). Open the **Profiler** tab, click the cog icon, and enable **"Highlight updates when components render"** in the General tab.

With this enabled, every re-rendering component is highlighted on every interaction. Typing in a search field that triggers root-level state updates will light up **the entire page like a Christmas tree**. By contrast, interacting with a well-scoped component (like a collapsible sidebar section) highlights only the affected components.

This visual immediately reveals the difference between "only legitimately necessary re-renders" and "the entire page re-renders unnecessarily."

### React Profiler Recording

The React Profiler has its own recording function (top-left Record button). After recording an interaction, the **Flamegraph section** shows:

- All components in their hierarchical order
- Color coding: colored components re-rendered during the interaction; grey components did not
- Relative timing: how long each component's render took compared to others
- Whether any single component is a bottleneck

For the search field problem, the Profiler confirms the entire tree re-renders. For a well-scoped interaction like a collapsible sidebar, only the relevant subtree is colored.

## Re-renders and Interaction Performance

Approximately **90% of all slow interactions** in React are caused by either too many re-renders in an interval, re-renders of too much at the same time, or both.

### Re-render Mechanics

A re-render is triggered by a **state update** (`useState`, `useReducer`, `useSyncExternalStore`, or external state libraries). When a component's state changes, React re-renders that component and then **every component rendered inside it**, recursively, until reaching the leaves. Props do not matter for this default behavior -- child components re-render regardless of whether their props changed.

The exception: **memoized components** (`React.memo` or React Compiler). When React encounters a memoized component, it checks whether props changed. If no prop changed, the re-render is skipped for that component and its subtree.

For more detail on re-render rules and patterns, see [React Re-renders](react-re-renders.md).

### The Search Field Problem

A typical problematic pattern: search state lives in the root `App` component, passes through multiple layers, and reaches the search field at the bottom of the tree.

```jsx
export default function App() {
  const [search, setSearch] = useState("");
  return (
    <AppLayoutLazySidebar search={search} setSearch={setSearch}>
      <DashboardPage />
      <div>Search results for {search}</div>
    </AppLayoutLazySidebar>
  );
}
```

Every keystroke updates `search` state at the root, re-rendering `App`, which re-renders `AppLayoutLazySidebar`, `DashboardPage`, and everything inside them. On a throttled CPU, this produces INP of ~650ms (deep red).

## Solutions for Unnecessary Re-renders

### 1. Moving State Down

The simplest and most overlooked solution. Reduce the blast radius by moving state to the lowest component that actually needs it:

```jsx
export const TopbarForSidebarContentLayout = () => {
  const [search, setSearch] = useState('');
  return (
    <>
      <InputWithIconsNormalToLarge
        value={search}
        onChange={(e) => setSearch(e.target.value)}
        placeholder="Search..."
      />
      <div>Search results for {search}</div>
    </>
  );
};
```

Result: interaction improved **10x**, from ~500ms to ~50ms, moving from "insufferable and very red" to "sleek and always green."

A good indicator for this pattern: several layers of components that just pass state through props without using it.

In practice, you would also want to add [debouncing](debouncing-throttling.md) for the search requests sent to the backend.

### 2. Components as Children / Props

When state cannot be moved down (e.g., the search results drawer is tied to other App-level logic), you can still reduce blast radius by passing expensive components as props:

```jsx
const LayoutWithSearch = ({ content }) => {
  const [search, setSearch] = useState('');
  return (
    <AppLayoutLazySidebar search={search} setSearch={setSearch}>
      {content}
      <div>Search results for {search}</div>
    </AppLayoutLazySidebar>
  );
};

export default function App() {
  return <LayoutWithSearch content={<DashboardPage />} />;
}
```

The `DashboardPage` is created in `App` (where there is no state change), so it is unaffected by re-renders inside `LayoutWithSearch`. This works because **anything passed as props is not affected** by re-renders of the receiving component.

The `content` prop can be renamed to `children` for cleaner JSX syntax -- `<LayoutWithSearch><DashboardPage /></LayoutWithSearch>` is syntactic sugar for passing `children` as a prop.

### 3. Context / External State Management

To avoid re-rendering even the layout/sidebar while keeping state accessible throughout the app, use React Context or external state management (Zustand, Redux):

```jsx
const Context = createContext({ search: '', setSearch: () => {} });
export const useSearch = () => useContext(Context);

const DataProvider = ({ children }) => {
  const [search, setSearch] = useState('');
  const value = useMemo(() => ({ search, setSearch }), [search, setSearch]);
  return <Context.Provider value={value}>{children}</Context.Provider>;
};

export default function App() {
  return (
    <DataProvider>
      <AppLayoutLazySidebar>
        <DashboardPage />
      </AppLayoutLazySidebar>
    </DataProvider>
  );
}
```

Only components that call `useSearch()` re-render when search state changes. The `children` prop pattern ensures `AppLayoutLazySidebar` and `DashboardPage` are unaffected.

### 4. Memoization (React.memo + useMemo + useCallback)

When refactoring is not feasible, wrap expensive components in `React.memo` and memoize their props:

```jsx
const DashboardPageMemo = memo(DashboardPage);

export default function App() {
  const [search, setSearch] = useState('');
  const dataMemo = useMemo(() => [{ id: 1 }, { id: 2 }], []);
  const onClickMemo = useCallback(() => { console.log('clicked'); }, []);

  return (
    <AppLayoutLazySidebar search={search} setSearch={setSearch}>
      <DashboardPageMemo data={dataMemo} onClick={onClickMemo} />
      <div>Search results for {search}</div>
    </AppLayoutLazySidebar>
  );
}
```

**Critical pitfall with children**: `<DashboardPageMemo><ChildComponent /></DashboardPageMemo>` breaks memoization because `<ChildComponent />` is an Element (an object), not a component. The object reference is not stable between renders. To fix this, memoize the element with `useMemo`:

```jsx
const memoChild = useMemo(() => <ChildComponentMemo />, []);
return <DashboardPageMemo>{memoChild}</DashboardPageMemo>;
```

This is React's most counterintuitive pattern. Even senior engineers get caught by it. If a single prop is not memoized correctly, the entire memoization chain breaks and the component behaves as if `memo` was never applied.

For more on memoization patterns and pitfalls, see [Memoization in React](memoization.md).

### 5. React Compiler

React Compiler automates memoization at build time. It is a separate tool (not part of React itself) that transpiles component code to include automatic memoization.

**Measured impact** (20x CPU slowdown, Fast 4G):

| Metric | Baseline | With Compiler |
|--------|----------|---------------|
| Home page LCP | 1.3s | 1.4s |
| Search INP | 650ms | 50ms |
| Home INP from Login | 350ms | 355ms |
| Home INP from Settings | 260ms | 260ms |

The Compiler improved search INP by **more than 10x** -- identical to manual solutions. However, initial load (LCP) slightly worsened because the Compiler generates more code and increases JavaScript work during the initial rendering pass. Bundle sizes increase modestly (e.g., 4.23 kB to 5.97 kB for the index chunk).

The Compiler does not catch everything. Components that re-render due to path-dependent state (like navigation items that read the current URL) may still re-render even when memoized, because their props legitimately change. Always verify with "Highlight updates" and `console.log` in `useEffect` to confirm whether re-renders are real or false positives from DevTools.

## Debugging Workflow Summary

1. **Set CPU throttling** (6x minimum, 20x for stress testing)
2. **Check INP live** in the Performance panel bottom bar -- identify which interactions are red
3. **Record** the problematic interaction in the Performance panel -- look for Long Tasks with red corners
4. **Switch to dev mode** -- production flame graphs are unreadable due to minification
5. **Enable "Highlight updates"** in React DevTools Profiler -- see which components re-render
6. **Record in React Profiler** -- identify the re-render tree and relative costs
7. **Apply fixes** (move state down, children pattern, Context, memoization, or Compiler)
8. **Re-measure** to confirm improvement

## Key Takeaways

- SPA transitions are 10x+ faster than traditional page loads because they bypass server requests, HTML parsing, and the full load cycle
- INP is the Core Web Vital for interaction performance; Google considers < 200ms "good"
- Always use CPU throttling when profiling interactions; developer laptops hide real-world problems
- Use production builds to detect slow interactions, then switch to dev mode to diagnose them
- Long Tasks (> 50ms) block the main thread and are the direct cause of poor INP
- `scheduler.yield()` and the `setTimeout` fallback break long tasks into smaller chunks, but are rarely needed in typical React apps
- ~90% of slow React interactions are caused by unnecessary re-renders
- Solutions in order of preference: move state down, components as children/props, Context or external state management, memoization, React Compiler
- Memoization with `React.memo` is fragile -- a single unmemoized prop (including `children` elements) breaks the entire chain
- React Compiler automates memoization and can improve interaction INP by 10x, but may slightly worsen initial load performance due to increased bundle size and JavaScript work

## See Also

- [React Re-renders](react-re-renders.md) -- re-render rules, triggers, and the component tree model
- [Web Performance Metrics](web-performance-metrics.md) -- INP, LCP, FCP, and Core Web Vitals definitions
- [Memoization in React](memoization.md) -- `React.memo`, `useMemo`, `useCallback` patterns and pitfalls
- [Rendering Strategies](rendering-strategies.md) -- CSR, SSR, SSG, and their performance tradeoffs
- [Lazy Loading and Suspense](lazy-loading-and-suspense.md) -- code splitting to reduce initial bundle size
- [Bundle Size Optimization](bundle-size-optimization.md) -- tree shaking, chunk analysis, and the Compiler's bundle size impact
- [Debouncing, Throttling, and useLayoutEffect](debouncing-throttling.md) -- rate-limiting input handlers for search and other frequent interactions
