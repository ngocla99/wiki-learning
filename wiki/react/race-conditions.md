# Race Conditions in React

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

Race conditions occur when multiple asynchronous operations compete to update the same state, and the order of resolution does not match the order of initiation. In React, this commonly happens when a prop like `id` changes rapidly, triggering multiple fetches where a slower earlier request resolves after a faster later one -- silently showing stale data. The code looks completely innocent, but the app is broken. Four fix strategies exist: forced re-mounting with `key`, comparing results with a Ref, cleanup flags in `useEffect`, and `AbortController` to cancel in-flight requests.

## The Problem

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

## Why It Happens

Two factors combine:

1. **React lifecycle**: When `id` changes, the `Page` component *re-renders* (not re-mounts). The `useEffect` fires again with the new `id`, but the previous fetch is still in-flight. Both fetches have a reference to the same `setData`.
2. **Promise nature**: `fetch` is asynchronous. The old promise does not cancel when a new one starts. When it eventually resolves, it calls `setData` with stale data.

## Fix 1: Force Re-mounting with `key`

```jsx
<Page id={page} key={page} />
```

Changing the `key` forces React to unmount the old component and mount a new one. The old component's state and pending promises are destroyed. When the old promise resolves, `setData` targets a component that no longer exists -- the data disappears into the void.

**Caveat**: Not recommended as a general solution. Performance may suffer from re-mounting, and it can cause unexpected bugs with focus, state, and triggering `useEffect` down the render tree. It is more like sweeping the problem under the rug.

## Fix 2: Compare Results with a Ref

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

## Fix 3: Cleanup Flag (Closure-based)

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

## Fix 4: AbortController

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

## async/await Does Not Change Anything

`async/await` is just syntactic sugar for promises. The same race conditions occur, and the same solutions apply -- just with slightly different syntax.

## Key Takeaways

- Race conditions appear when state is updated from stale promises -- the UI silently shows data that does not match the current request.
- The root cause is that React re-renders (not re-mounts) on prop changes, so old `useEffect` fetches keep running alongside new ones.
- **Force re-mounting with `key`** destroys old state but hurts performance and can cause side effects -- use sparingly.
- **Ref comparison** checks whether the resolved data still matches the current prop before updating state.
- **Cleanup flag** (`isActive`) uses closure scoping to discard stale results -- simple and reliable.
- **AbortController** cancels the actual HTTP request, saving bandwidth and preventing stale responses entirely -- the most complete solution.

## See Also

- [Data Fetching Patterns](data-fetching-patterns.md) -- waterfalls, browser limits, and fetching strategies
- [Closures in React](closures-in-react.md) -- the closure mechanics behind cleanup flags and stale references
- [Refs and Imperative API](refs-and-imperative-api.md) -- Ref patterns used in Fix 2
- [React Re-renders](react-re-renders.md) -- why re-renders (not re-mounts) cause the problem
- [Error Handling in React](error-handling.md) -- handling AbortError and async errors
