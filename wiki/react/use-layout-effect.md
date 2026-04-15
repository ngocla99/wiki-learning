# useLayoutEffect

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

`useLayoutEffect` is a React hook that runs synchronously after DOM mutations but before the browser paints. Its primary use case is eliminating visual flickering when adjusting UI based on DOM measurements -- for example, responsive navigations that measure element widths and hide overflow items behind a "more" button. Understanding when to use `useLayoutEffect` vs `useEffect` requires understanding how browsers schedule tasks and painting. Using it incorrectly can block rendering and hurt performance.

## The Flickering Problem

### The Scenario

Consider building a responsive navigation that shows as many links as fit, then hides the rest behind a "more" button. The only way to know how many items fit is to render them, measure their widths via `getBoundingClientRect`, then remove the ones that do not fit.

```jsx
const Component = ({ items }) => {
  const ref = useRef(null);
  const [lastVisibleMenuItem, setLastVisibleMenuItem] = useState(-1);

  useEffect(() => {
    const itemIndex = getLastVisibleItem(ref.current);
    setLastVisibleMenuItem(itemIndex);
  }, [ref]);

  if (lastVisibleMenuItem === -1) {
    // first pass: render all items to measure them
    return (
      <div className="navigation" ref={ref}>
        {items.map((item) => (
          <a href={item.href}>{item.name}</a>
        ))}
        <button id="more">...</button>
      </div>
    );
  }

  const isMoreVisible = lastVisibleMenuItem < items.length - 1;
  const filteredItems = items.filter(
    (item, index) => index <= lastVisibleMenuItem
  );

  return (
    <div className="navigation" ref={ref}>
      {filteredItems.map((item) => (
        <a href={item.href}>{item.name}</a>
      ))}
      {isMoreVisible && <button id="more">...</button>}
    </div>
  );
};
```

This works but has a **major flaw**: a noticeable flash of content. Users see the initial render with all items and the "more" button before the unnecessary items are removed.

### Why useEffect Causes Flickering

To understand why, we need to understand how browsers render.

## How Rendering and Painting Work in Browsers

Browsers do not continuously update everything on screen in real time. Instead, they work like a slideshow: modern browsers maintain a 60 FPS rate, meaning one "slide" changes to the next roughly every **13 milliseconds**. This screen update is what React calls "painting."

The information that updates these slides is split into **tasks**. Tasks are put in a queue. The browser grabs a task from the queue, executes it, grabs the next one, and so on until no more time is left in the ~13ms gap. Then it refreshes the screen and continues.

### Synchronous Code = One Task

Everything executed synchronously is a single task. The browser paints only the final result:

```js
const app = document.getElementById('app');
const child = document.createElement('div');
child.innerHTML = '<h1>Heyo!</h1>';
app.appendChild(child);

child.style = 'border: 10px solid red';
child.style = 'border: 20px solid green';
child.style = 'border: 30px solid black';
```

The entire thing is one task. The browser executes every line, then draws the final result: a div with a black border. You never see the red or green borders.

Even with synchronous delays between the style changes, the browser still cannot paint until the task completes:

```js
const waitSync = (ms) => {
  let start = Date.now(), now = start;
  while (now - start < ms) {
    now = Date.now();
  }
};

child.style = 'border: 10px solid red';
waitSync(1000);
child.style = 'border: 20px solid green';
waitSync(1000);
child.style = 'border: 30px solid black';
waitSync(1000);
```

You stare at a blank screen for 3 seconds, then see the black border. This is **blocking render** (blocking painting) code.

### Async Code = Multiple Tasks

Wrapping operations in `setTimeout` (even with 0 delay) creates separate tasks with screen re-painting between them:

```js
setTimeout(() => {
  child.style = 'border: 10px solid red';
  waitSync(1000);
  setTimeout(() => {
    child.style = 'border: 20px solid green';
    waitSync(1000);
    setTimeout(() => {
      child.style = 'border: 30px solid black';
      waitSync(1000);
    }, 0);
  }, 0);
}, 0);
```

Now each timeout is a new task. The browser can repaint between them, and you see the slow transition from red to green to black.

React works the same way -- it is an engine that splits our code into the smallest possible chunks that browsers can process within the ~13ms budget.

## useEffect vs useLayoutEffect

### useEffect = Two Tasks (With Paint Between)

`useEffect` runs **asynchronously**. From the browser's perspective, the initial render and the effect are two separate tasks with a screen repaint in between:

1. **Task 1**: Render the initial pass (all navigation items visible)
2. **Browser paints** the intermediate state (flash!)
3. **Task 2**: `useEffect` runs, calculates sizes, updates state, re-renders with fewer items

### useLayoutEffect = One Task (No Paint Between)

`useLayoutEffect` runs **synchronously** during component updates. React guarantees that the render and the effect are one single task:

```jsx
const Component = ({ items }) => {
  // everything is exactly the same, only the hook name is different
  useLayoutEffect(() => {
    const itemIndex = getLastVisibleItem(ref.current);
    setLastVisibleMenuItem(itemIndex);
  }, [ref]);
};
```

From the browser's perspective, this is one unbreakable task -- exactly like the red-green-black border transition that could not be seen. The browser waits for the entire flow to complete, then paints only the final result. No flickering.

### When to Use Which

**Use `useLayoutEffect`** only when you need to eliminate visual glitches caused by adjusting UI based on real DOM measurements (sizes, positions).

**Use `useEffect`** for everything else. `useLayoutEffect` can hurt performance because it turns what would be multiple small tasks into one large blocking task. The last thing you want is for your entire app to become one giant synchronous task.

### Nuances of useEffect Timing

While the mental model of `useEffect` running inside `setTimeout` is convenient, it is not technically correct:

- React uses a `postMessage` combined with `requestAnimationFrame` trick internally.
- `useEffect` is not **guaranteed** to run asynchronously. If `useLayoutEffect` is somewhere in the chain of updates and triggers a state update, React must finish the current cycle before starting a new one. Since `useLayoutEffect` is synchronous, React has no choice but to run `useEffect` synchronously as well.

Each re-render cycle runs in order: **State update -> useLayoutEffect -> useEffect**. If any step triggers a new state update, a new cycle begins -- but the current one finishes first.

## useLayoutEffect and SSR (Next.js)

In Server-Side Rendering frameworks like Next.js, `useLayoutEffect` **does not work** for the initial page load.

### Why SSR Breaks It

During SSR, the first rendering pass happens on the server via something like `React.renderToString(<App />)`. React calls all component functions and produces HTML, which is sent to the browser. The browser displays this HTML, then downloads and runs React on the client to "hydrate" the page with interactivity.

The problem: there is no browser during server rendering. No elements with dimensions exist -- just strings. Since the whole purpose of `useLayoutEffect` is to access element sizes, React simply does not run it on the server.

As a result, what users see during the initial SSR load is the "first pass" -- all navigation items visible. Only after client-side React runs does `useLayoutEffect` fire and hide the overflow items. The visual glitch is back.

### The Fix: Opt Out of SSR for That Feature

Introduce a "shouldRender" state variable that flips to `true` in `useEffect` (which only runs on the client):

```jsx
const Component = () => {
  const [shouldRender, setShouldRender] = useState(false);

  useEffect(() => {
    setShouldRender(true);
  }, []);

  if (!shouldRender) return <SomeNavigationSubstitute />;

  return <Navigation />;
};
```

The initial SSR pass shows the substitute component. Then client code kicks in, `useEffect` runs, state changes, and React renders the responsive navigation with `useLayoutEffect` working correctly.

### Incorrect SSR Detection

Do **not** try to detect SSR with `typeof window === undefined`:

```jsx
const Component = () => {
  // This seems like it would work, but it doesn't
  if (typeof window === undefined)
    return <SomeNavigationSubstitute />;

  return <Navigation />;
};
```

While `typeof window === undefined` correctly detects the server environment, React requires the SSR HTML and the first client render HTML to **match exactly**. If they differ, the app will behave erratically -- broken styles, mispositioned blocks, content in wrong places. Using state with `useEffect` ensures the first client render matches SSR, and only the subsequent update swaps in the real component.

## Key Takeaways

- **useLayoutEffect** runs synchronously after DOM mutations but **before** the browser paints. It prevents visual flickering when adjusting UI based on DOM measurements.
- **useEffect** runs asynchronously (separate browser task), allowing a paint between the initial render and the effect -- which causes the flicker.
- Browsers work like a slideshow at ~60 FPS (~13ms per frame). Synchronous code is one task (one frame); async code (`setTimeout`, `useEffect`) creates multiple tasks with paints between them.
- **Use `useLayoutEffect` sparingly** -- only for visual glitch elimination. It blocks painting and can hurt performance.
- **useLayoutEffect and SSR**: React does not run `useLayoutEffect` on the server. Fix by conditionally rendering with a state flag toggled in `useEffect`.
- Do **not** detect SSR with `typeof window === undefined` -- it causes hydration mismatches.

## See Also

- [Debouncing and Throttling](debouncing-throttling.md) -- the Ref patterns used alongside useLayoutEffect for DOM measurement
- [Refs and Imperative API](refs-and-imperative-api.md)
- [Rendering Strategies](rendering-strategies.md) -- SSR and hydration mechanics
- [Interaction Performance](interaction-performance.md)
- [React Re-renders](react-re-renders.md)
