# Data Fetching Patterns

> Sources: Nadia Makarevich, 2023; Nadia Makarevich, 2025
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md); [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)

## Overview

Data fetching in React encompasses initial data loading, on-demand fetching, handling browser limitations, avoiding request waterfalls, preventing race conditions, and choosing between client-side and server-side strategies. While a simple `fetch` in `useEffect` works for trivial cases, production apps benefit from libraries that handle caching, deduplication, error states, and cancellation. The evolution from client-side fetching to SSR to React Server Components reflects the community's ongoing effort to move data fetching closer to the data source, reduce round trips, and improve both performance and developer experience.

## Types of Data Fetching

- **Initial data**: Data needed when the page loads (dashboard content, user profile, navigation items). Typically triggered in `useEffect` or during rendering. This is the most critical category for perceived performance -- it forms the user's first impression of whether the app is "fast" or "slow."
- **On-demand data**: Data fetched after user interaction (autocomplete, search results, form submissions, dynamic forms). Triggered in event callbacks.

Although these seem totally different, the core principles and fundamental patterns are the same for both. However, initial data fetching is usually the most crucial: the first impression of your app as "slow as hell" or "blazing fast" forms during this stage.

## Do You Need a Library?

Short answer: no, and yes. It depends on your use case.

For a simple one-time fetch, plain `fetch` in `useEffect` is sufficient:

```jsx
const Component = () => {
  const [data, setData] = useState();

  useEffect(() => {
    const dataFetch = async () => {
      const data = await (await fetch('/api/data')).json();
      setData(data);
    };
    dataFetch();
  }, []);

  return <>...</>;
};
```

But as soon as your use case exceeds "fetch once and forget," you face tough questions: What about error handling? What if multiple components fetch from the same endpoint -- should you cache? For how long? What about race conditions? Should you cancel requests when a component unmounts? What about memory leaks?

**React-independent libraries** like Axios abstract the complexities of the `fetch` API itself (request cancellation, interceptors) but have no opinion on React patterns. You can replace all `fetch` calls with `axios.get` and the architectural patterns remain the same.

**React-integrated libraries** like SWR or TanStack Query additionally abstract `useEffect`, state management, error handling, caching, deduplication, and refetching. Instead of the `useEffect` boilerplate:

```jsx
const Comments = () => {
  const { data } = useSWR('/get-comments', fetcher);
  // the rest of comments code
};
```

Underneath, all of them use `useEffect` (or equivalent) and state to trigger re-renders. The choice of library does not change the fundamental data fetching and orchestration patterns -- it just makes some things easier at the cost of making other things harder.

## What Is a "Performant" App?

Performance with data fetching is not straightforward. Consider an issue tracker with sidebar, main issue content, and comments:

1. **App 1**: Shows loading until *all* data loads, then renders everything at once (~3s total). Fastest overall but shows nothing for 3 seconds.
2. **App 2**: Shows sidebar first (~1s), then the rest (~3s total). Fastest to show *something*, but slowest to show the main content.
3. **App 3**: Shows issue data first (~2s), then sidebar (~3s), then comments (~5s total). Shows the *important* content first, but the layout shift feels "junky."

Which is most performant? It depends entirely on the story you are telling your users. Think of yourself as a storyteller -- what is the most important piece? Can you tell the story in pieces, or should users see everything at once? The true power comes from understanding:

- When is it okay to **start** fetching data?
- What can we do **while** the data fetch is in progress?
- What should we do **when** the data fetch is done?

## React Lifecycle and Data Fetching

A critical principle: **only components that are actually returned (mounted) will trigger their `useEffect` and data fetching.**

```jsx
const Parent = () => {
  const [isLoading, setIsLoading] = useState(true);

  // Child's useEffect will NOT fire while isLoading is true
  if (isLoading) return 'loading';

  return <Child />;
};
```

Even creating the element before the condition does not help:

```jsx
const Parent = () => {
  const [isLoading, setIsLoading] = useState(true);
  const child = <Child />; // This does NOT render Child

  if (isLoading) return 'loading';
  return child;
};
```

Writing `<Child />` is just syntax sugar for creating a description object (a React Element). The component only renders when that description ends up in the actual visible render tree -- i.e., returned from the component. Until then, it does nothing.

## Browser Limitations

Browsers limit parallel HTTP connections per host. For HTTP/1.1 (still a large portion of the internet), Chrome allows only **6 parallel requests** to the same domain. Any additional requests queue and wait for an available slot.

This means that in a large app with many fetch requests, careless prefetching or analytics requests at startup can steal connection slots from critical data, slowing down the entire experience:

```jsx
// These 6 fire-and-forget requests block ALL other requests
// to the same domain until they complete
fetch('https://api.example.com/url1');
fetch('https://api.example.com/url2');
fetch('https://api.example.com/url3');
fetch('https://api.example.com/url4');
fetch('https://api.example.com/url5');
fetch('https://api.example.com/url6');

// This critical request now has to wait ~10 seconds
const App = () => {
  const { data } = useData('/fetch-critical-data');
  if (!data) return 'loading...';
  return <div>I'm an app</div>;
};
```

Even if the app's own fetch takes only ~50ms, adding six slow requests before it causes the entire app to wait. Strategies to mitigate: use HTTP/2 (multiplexing), CDN subdomains, and prioritize critical requests.

## Request Waterfalls

Waterfalls occur when requests are made sequentially rather than in parallel, dramatically increasing total loading time.

### How Waterfalls Appear

The most natural way to write data fetching in React -- co-locating fetches with components -- creates waterfalls:

```jsx
const App = () => {
  const { data } = useData('/get-sidebar');
  if (!data) return 'loading';
  return (
    <>
      <Sidebar data={data} />
      <Issue />
    </>
  );
};

const Issue = () => {
  const { data } = useData('/get-issue');
  if (!data) return 'loading';
  return (
    <div>
      <h3>{data.title}</h3>
      <p>{data.description}</p>
      <Comments />
    </div>
  );
};

const Comments = () => {
  const { data } = useData('/get-comments');
  if (!data) return 'loading';
  return data.map((comment) => <div>{comment.title}</div>);
};
```

Each component returns a loading state while waiting for data. Only when data arrives does the next component mount, trigger *its* fetch, show *its* loading state, and so on. If sidebar takes 1s, issue 2s, and comments 3s, the total wait is 1+2+3 = **6 seconds** instead of the maximum of 3 seconds if fetched in parallel.

### Solution 1: Promise.all

Pull all fetches to the root component and fire them simultaneously:

```jsx
const useAllData = () => {
  const [sidebar, setSidebar] = useState();
  const [comments, setComments] = useState();
  const [issue, setIssue] = useState();

  useEffect(() => {
    const dataFetch = async () => {
      const result = (
        await Promise.all([
          fetch(sidebarUrl),
          fetch(issueUrl),
          fetch(commentsUrl),
        ])
      ).map((r) => r.json());

      const [sidebarResult, issueResult, commentsResult] =
        await Promise.all(result);

      setSidebar(sidebarResult);
      setIssue(issueResult);
      setComments(commentsResult);
    };
    dataFetch();
  }, []);

  return { sidebar, comments, issue };
};
```

All three requests fire in parallel and the total wait equals the slowest one (~3s). The tradeoff: nothing renders until *everything* finishes. This is equivalent to "show the whole story at once."

### Solution 2: Parallel Promises (Independent Resolution)

Fire all requests in parallel but resolve them independently using `.then()`:

```jsx
useEffect(() => {
  fetch('/get-sidebar')
    .then((data) => data.json())
    .then((data) => setSidebar(data));
  fetch('/get-issue')
    .then((data) => data.json())
    .then((data) => setIssue(data));
  fetch('/get-comments')
    .then((data) => data.json())
    .then((data) => setComments(data));
}, []);
```

Now each piece of the UI renders as soon as its data arrives. The overall wait drops from 6s to 3s while preserving incremental rendering. Note: this triggers three independent state updates, causing three re-renders of the parent component -- keep this in mind for large component trees.

### Solution 3: Data Providers with Context

Lifting data fetching up is good for performance but terrible for architecture -- it creates a giant component that fetches everything and massive props drilling. Data providers solve this:

```jsx
const Context = React.createContext();

export const CommentsDataProvider = ({ children }) => {
  const [comments, setComments] = useState();

  useEffect(() => {
    fetch('/get-comments')
      .then((data) => data.json())
      .then((data) => setComments(data));
  }, []);

  return (
    <Context.Provider value={comments}>
      {children}
    </Context.Provider>
  );
};

export const useComments = () => useContext(Context);
```

Wrap providers around the app root -- they fire fetches in parallel when mounted:

```jsx
export const VeryRootApp = () => (
  <SidebarDataProvider>
    <IssueDataProvider>
      <CommentsDataProvider>
        <App />
      </CommentsDataProvider>
    </IssueDataProvider>
  </SidebarDataProvider>
);
```

Deep components access data via hooks (`useComments()`) with no props drilling. This pattern works with any state management solution, not just Context.

### Solution 4: Pre-fetching Before React

Move the `fetch` call outside the component entirely:

```jsx
const commentsPromise = fetch('/get-comments');

const Comments = () => {
  useEffect(() => {
    const dataFetch = async () => {
      const data = await (await commentsPromise).json();
      setState(data);
    };
    dataFetch();
  }, []);
};
```

The fetch fires as soon as JavaScript is loaded, before any `useEffect` runs, before any component mounts. This effectively "escapes" the React lifecycle.

**Danger**: These uncontrolled fetches fire immediately, consuming browser connection slots. A component rendered "once in a blue moon" could steal precious connection slots from critical data. Only two legitimate use cases:

1. **Pre-fetching critical resources at the router level** -- data you know for sure is needed immediately.
2. **Pre-fetching data in lazy-loaded components** -- their JavaScript downloads only when they enter the render tree, so by definition after critical data is fetched.

### Solution 5: Inject Fetches in HTML (Extreme Pre-fetch)

For maximum speed, trigger fetches before *any* JavaScript loads by injecting them into the HTML:

```html
<script>
  window.__PREFETCH_PROMISES = {
    sidebar: fetch('http://localhost:5432/api/sidebar'),
    table: fetch('http://localhost:5432/api/statistics'),
  };
</script>
```

Then in React, pick them up from `window`:

```jsx
let sidebarCache = window.__PREFETCH_PROMISES?.sidebar;
```

The fetches start as soon as HTML is downloaded. When lazy components finally load, the data is already there -- almost no loading state. The cost: API calls steal bandwidth, potentially worsening LCP. Treat this as an exercise to understand fetch timing, not a production pattern (especially with SSR complications).

## Suspense and Data Fetching

Suspense is a clever way to replace manual loading state management. Instead of:

```jsx
const Comments = ({ comments }) => {
  if (!comments) return 'loading';
  // render comments
};
```

You lift the loading state up:

```jsx
const Issue = () => (
  <>
    {/* issue data */}
    <Suspense fallback="loading">
      <Comments />
    </Suspense>
  </>
);
```

Everything else -- browser limitations, React lifecycle, the nature of waterfalls -- stays the same. Suspense does not solve waterfalls by itself.

## Race Conditions

Race conditions occur when multiple asynchronous operations compete to update the same state, and the order of resolution does not match the order of initiation.

### The Problem

```jsx
const Page = ({ id }) => {
  const [data, setData] = useState({});

  useEffect(() => {
    fetch(`/some-url/${id}`)
      .then((r) => r.json())
      .then((r) => setData(r));
  }, [id]);

  return <h2>{data.title}</h2>;
};
```

If `id` changes rapidly (1 -> 2 -> 3), all three requests fire, but they may resolve in any order. If request 3 resolves first and request 1 resolves last, the UI shows data for id=1 even though the user navigated to id=3. The code looks completely innocent, but the app is broken.

### Why It Happens

Two factors combine:

1. **React lifecycle**: When `id` changes, the `Page` component *re-renders* (not re-mounts). The `useEffect` fires again with the new `id`, but the previous fetch is still in-flight. Both fetches have a reference to the same `setData`.
2. **Promise nature**: `fetch` is asynchronous. The old promise does not cancel when a new one starts. When it eventually resolves, it calls `setData` with stale data.

### Fix 1: Force Re-mounting with `key`

```jsx
<Page id={page} key={page} />
```

Changing the `key` forces React to unmount the old component and mount a new one. The old component's state and pending promises are destroyed. When the old promise resolves, `setData` targets a component that no longer exists -- the data disappears into the void.

**Caveat**: Not recommended as a general solution. Performance may suffer from re-mounting, and it can cause unexpected bugs with focus, state, and triggering `useEffect` down the render tree. It is more like sweeping the problem under the rug.

### Fix 2: Compare Results with a Ref

Use a ref to always hold the *latest* `id`, then compare before setting state:

```jsx
const Page = ({ id }) => {
  const ref = useRef(id);

  useEffect(() => {
    ref.current = id;

    fetch(`/some-data-url/${id}`)
      .then((r) => r.json())
      .then((r) => {
        // Only update if this result matches the current id
        if (ref.current === r.id) {
          setData(r);
        }
      });
  }, [id]);
};
```

If results do not include an identifying field, compare the URL instead using `result.url === ref.current`.

### Fix 3: Cleanup Flag (Closure-based)

Use the `useEffect` cleanup function to mark previous closures as stale:

```jsx
useEffect(() => {
  let isActive = true;

  fetch(`/some-data-url/${id}`)
    .then((r) => r.json())
    .then((r) => {
      if (isActive) {
        setData(r);
      }
    });

  return () => {
    isActive = false;
  };
}, [id]);
```

The cleanup function runs before every re-render with changed dependencies. The `isActive` variable in the *previous* closure gets set to `false`, so only the latest fetch can update state. This works because JavaScript closures give each `useEffect` run its own scope, and the `fetch` promise inside that closure can only access *that run's* local variables.

### Fix 4: AbortController

Cancel the actual HTTP request when the effect cleans up:

```jsx
useEffect(() => {
  const controller = new AbortController();

  fetch(url, { signal: controller.signal })
    .then((r) => r.json())
    .then((r) => setData(r))
    .catch((error) => {
      if (error.name === 'AbortError') {
        // Request was cancelled, do nothing
      } else {
        // Real error, handle it
      }
    });

  return () => {
    controller.abort();
  };
}, [url]);
```

On every re-render with changed dependencies, the in-progress request is cancelled and a new one starts. Aborting causes the promise to reject with an `AbortError`, which should be filtered out from real error handling.

### async/await Does Not Change Anything

`async/await` is just syntactic sugar for promises. The same race conditions occur, and the same solutions apply -- just with slightly different syntax.

## Client-Side vs Server-Side Data Fetching

### Client-Side Fetching

Client-side fetching in a SPA follows this sequence: download HTML -> download JS -> parse and execute JS -> render components -> trigger `useEffect` -> fetch data -> re-render with data. The user sees skeletons or loading states during the fetch.

### SSR with Client-Side Fetching

Adding basic SSR (using `renderToString`) without moving data fetching to the server barely changes the data fetch timeline. The critical path pre-renders, improving LCP, but `useEffect` is client-only -- so dynamic data still requires client-side fetches.

### SSR with Server-Side Data Fetching

Moving fetches to the server and passing data as props to `renderToString`:

```jsx
app.get("/*", async (c) => {
  const [sidebar, statistics] = await Promise.all([
    fetch('/api/sidebar').then((res) => res.json()),
    fetch('/api/statistics').then((res) => res.json()),
  ]);
  const reactHtml = renderToString(
    <App sidebar={sidebar} statistics={statistics} />
  );
  // ...
});
```

The server fetches data, renders HTML with it, and sends a complete page. No client-side loading states. But the cost is real: the server must **wait** for all data before sending any HTML. A 700ms data fetch delays the entire HTML response by 700ms, worsening LCP from ~480ms to ~1.45s.

**Hydration problem**: The client also needs the data to avoid wiping out the server-rendered content. Solution: inject the data into the page via a `<script>` tag:

```html
<script>
  window.__SSR_DATA__ = { sidebar: ..., statistics: ... };
</script>
```

Then read it during `hydrateRoot`. Frameworks like Next.js (Pages Router) automate this with `getServerSideProps` and a `__NEXT_DATA__` script tag -- the exact same mechanism, just abstracted.

### React Server Components and Streaming

Server Components allow data fetching to be co-located with components while running entirely on the server:

```jsx
export default async function App() {
  const [sidebar, statistics] = await Promise.all([
    fetch('/api/sidebar').then((res) => res.json()),
    fetch('/api/statistics').then((res) => res.json()),
  ]);
  return (
    <>
      <Sidebar data={sidebar} />
      <Statistics data={statistics} />
    </>
  );
}
```

No client-side fetch, no `useEffect`, no loading states. The data just appears. Combined with **Streaming**, independent parts of the page can be sent as separate chunks. Wrapping components in `<Suspense>` marks them as independently streamable:

```jsx
export default async function App() {
  return (
    <div>
      <h1>Welcome to the App!</h1>
      <Suspense fallback={<SidebarSkeleton />}>
        <Sidebar />
      </Suspense>
      <Suspense fallback={<StatsSkeleton />}>
        <Statistics />
      </Suspense>
    </div>
  );
}
```

The `<h1>` renders immediately. `Sidebar` and `Statistics` stream independently as their data resolves. Each `<Suspense>` boundary defines one streaming chunk.

**Beware of server-side waterfalls**: Nesting async Server Components inside each other creates the same waterfall problem, just on the server:

```jsx
// This creates a waterfall: Statistics fetches first,
// Sidebar only starts fetching after Statistics resolves
const Statistics = async () => {
  const data = await fetch('/api/statistics').then(r => r.json());
  return (
    <>
      <Suspense><Sidebar /></Suspense>
      {/* statistics content */}
    </>
  );
};
```

### Performance Comparison

| Metric | Client-Side Fetch | Classic SSR | Server Components |
|---|---|---|---|
| Critical Path (LCP) | 740ms | 1.2s | 410ms |
| Sidebar Items | 1.4s | 1.2s | 550ms |
| Statistics Table | 1.8s | 1.2s | 1s |

Server Components with streaming outperform both alternatives, in some cases by 2x.

## Key Takeaways

- Data fetching separates into **initial** (critical for first impressions) and **on-demand** (triggered by user interaction).
- Plain `fetch` works for simple cases; beyond that, consider libraries for caching, error handling, and deduplication.
- "Performant" is subjective -- it depends on the story you want to tell. Prioritize what users need to see first.
- Browser connection limits (6 parallel for HTTP/1.1) mean careless prefetching can block critical requests.
- **Waterfalls** are the primary performance killer: they appear when fetches depend on the React component lifecycle (parent must finish before child starts).
- Solutions to waterfalls: `Promise.all`, parallel independent promises, data providers with Context, pre-fetching outside React, or injecting fetches in HTML.
- **Race conditions** appear when state is updated from stale promises. Fix with cleanup flags, refs, `AbortController`, or forced re-mounting.
- SSR moves fetching to the server but delays HTML delivery. Streaming with Server Components eliminates this tradeoff by sending chunks incrementally.
- Regardless of the library, framework, or pattern, the fundamentals of orchestration, browser limits, and the React lifecycle always apply.

## See Also

- [React State Management](state-management.md) — TanStack Query as the recommended tool for remote state (~80% of state management)
- [Rendering Strategies](rendering-strategies.md)
- [React Server Components](react-server-components.md)
- [Error Handling in React](error-handling.md)
- [Lazy Loading and Suspense](lazy-loading-and-suspense.md)
- [Bundle Size Optimization](bundle-size-optimization.md)
- [Web Performance Metrics](web-performance-metrics.md)
- [Component Composition Patterns](component-composition-patterns.md)
- [React Context and Performance](react-context-performance.md)
- [Closures in React](closures-in-react.md)
