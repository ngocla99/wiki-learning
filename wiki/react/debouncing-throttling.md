# Debouncing, Throttling, and useLayoutEffect

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

Debouncing and throttling are techniques for controlling how often a function executes when triggered by rapid events like typing, scrolling, or resizing. Implementing them correctly in React is surprisingly challenging because component re-renders re-create functions on every render, destroying the internal timers these techniques rely on. This article covers why naive approaches fail, how to use Refs and the closure escape pattern to build a robust `useDebounce` hook, and how `useLayoutEffect` prevents visual flickering when adjusting UI based on DOM measurements.

## What Are Debouncing and Throttling?

**Debouncing** and **throttling** are techniques that allow us to skip function execution if that function is called too many times over a certain period.

### Debounce

Debounce waits until a specified interval has passed since the **last** call, then executes. If called again within the delay, the timer resets.

**Example use case**: An async search input where a user types and results are fetched from a backend. A skilled typist can produce ~6 keypresses per second — that is 6 requests per second without debouncing. With debounce, only one request is sent after the user pauses:

```jsx
const Input = () => {
  const onChange = (e) => {
    // will be triggered 500 ms after the user stopped typing
  };

  const debouncedOnChange = debounce(onChange, 500);

  return <input onChange={debouncedOnChange} />;
};
```

Before debouncing: typing "React" sends requests for "R", "Re", "Rea", "Reac", "React".
After debouncing: it waits 500ms after the user stops, then sends one request for "React".

### Throttle

Throttle guarantees execution at **regular intervals** — at most once per specified period. Unlike debounce (which constantly resets the timer), throttle fires periodically.

**Example use case**: An auto-save editing field. With debounce, if a user types continuously for 10 minutes, only one save fires at the end — if something breaks, the entire work is lost. With throttle, saves happen periodically, and only the last few milliseconds of work might be lost.

### How Debounce Works Internally

Underneath, `debounce` is a function that accepts a function and returns another function, with an internal timer that tracks whether the waiting interval has passed:

```js
const debounce = (callback, wait) => {
  // initialize the timer
  let timer;

  const debouncedFunc = () => {
    // checking whether the waiting time has passed
    if (shouldCallCallback(Date.now())) {
      callback();
    } else {
      // if time hasn't passed yet, restart the timer
      timer = startTimer(callback);
    }
  };

  return debouncedFunc;
};
```

The key insight: `debounce` **creates a new timer** and **returns a new function** every time it is called. The timer and the returned function must persist across invocations for the mechanism to work.

## Why Naive Approaches Fail in React

### The Re-render Problem

In real applications, inputs usually have state. As soon as state is involved, every keystroke causes a re-render:

```jsx
const Input = () => {
  const [value, setValue] = useState();

  const sendRequest = (value) => {
    // send value to the backend
  };

  // now sendRequest is debounced
  const debouncedSendRequest = debounce(sendRequest, 500);

  const onChange = (e) => {
    const value = e.target.value;
    setValue(value);           // triggers re-render
    debouncedSendRequest(value); // called on a NEW debounced function each render
  };

  return <input onChange={onChange} value={value} />;
};
```

This looks correct but **does not work**. Every keystroke triggers a re-render, which calls `debounce(sendRequest, 500)` again, creating:
1. A **new timer**
2. A **new returned function** with the callback in its arguments

The old function is never cleaned up — it sits in memory, waits for its timer to pass, fires the callback, then dies. We end up with a simple `delay` function, not a proper debounce.

### Fix Attempt 1: Move Outside the Component

The simplest fix is to call `debounce` only once by moving it outside the component:

```jsx
const sendRequest = (value) => {
  // send value to the backend
};
const debouncedSendRequest = debounce(sendRequest, 500);

const Input = () => {
  const [value, setValue] = useState();

  const onChange = (e) => {
    const value = e.target.value;
    setValue(value);
    debouncedSendRequest(value);
  };

  return <input onChange={onChange} value={value} />;
};
```

This works but **only if** the debounced function has no dependencies on anything inside the component (state, props, context).

### Fix Attempt 2: useMemo and useCallback

For functions that need to live inside the component, use memoization hooks:

```jsx
const Input = () => {
  const [value, setValue] = useState('initial');

  // memoize the callback with useCallback
  // we need it since it's a dependency in useMemo below
  const sendRequest = useCallback((value) => {
    console.log('Changed value:', value);
  }, []);

  // memoize the debounce call with useMemo
  const debouncedSendRequest = useMemo(() => {
    return debounce(sendRequest, 1000);
  }, [sendRequest]);

  const onChange = (e) => {
    const value = e.target.value;
    setValue(value);
    debouncedSendRequest(value);
  };

  return <input onChange={onChange} value={value} />;
};
```

This works correctly. The `sendRequest` is memoized, `debouncedSendRequest` is created once, and debounce behaves properly.

**Until we need state inside the callback.**

## Dealing with State Inside Debounced Callbacks

What if instead of passing the value as an argument, we want to use state directly? Maybe we have a chain of callbacks, or need access to multiple state variables:

```jsx
const Input = () => {
  const [value, setValue] = useState('initial');

  const sendRequest = useCallback(() => {
    // value is now coming from state
    console.log('Changed value:', value);

    // adding it to dependencies
  }, [value]);
};
```

Because `sendRequest` now depends on `value`, it changes with every keystroke. This means `debouncedSendRequest` (which has `sendRequest` as a dependency) also changes constantly:

```jsx
// this will now change on every state update
// because sendRequest has dependency on state
const debouncedSendRequest = useMemo(() => {
  return debounce(sendRequest, 1000);
}, [sendRequest]);
```

We are back to where we started: debounce has turned into just a delay.

### Naive Ref Approach (Also Broken)

Many articles recommend `useRef` for debouncing:

```jsx
const Input = () => {
  const [value, setValue] = useState();

  const ref = useRef(
    debounce(() => {
      // send request to the backend here
    }, 500),
  );

  const onChange = (e) => {
    const value = e.target.value;
    ref.current();
  };
};
```

This works **only if there is no state inside the callback**. The Ref's initial value is cached and never updated — it is frozen at mount time, creating a stale closure.

If we try to update the ref in `useEffect` to fix the stale closure:

```jsx
useEffect(() => {
  // updating ref when state changes
  ref.current = debounce(() => {
    // send request with access to latest state
  }, 500);
}, [value]);
```

This recreates the debounced function every time, resetting the timer each time. Debounce turns into delay again.

We could add a cleanup function to cancel the old debounce before reassigning:

```jsx
useEffect(() => {
  ref.current = debounce(() => {}, 500);

  // cancel the old debounce before re-assigning
  return () => ref.current.cancel();
}, [value]);
```

This works for debounce, but **not for throttle** — canceling throttle on every update prevents it from ever firing at the interval it is supposed to.

## The Ref Callback Pattern (The Correct Solution)

The closure escape pattern from the closures chapter provides a universal solution. The idea:

1. Store the **actual callback** (not the debounced version) in a Ref.
2. Update that Ref in `useEffect` to always have the latest closure.
3. Create the debounced function **once**, and inside it, call `ref.current()`.

Since refs are mutable objects and closures only freeze **references** to objects (not the objects themselves), the debounced function always calls the latest version of the callback.

```jsx
const Input = () => {
  const [value, setValue] = useState();

  const sendRequest = () => {
    // send request to the backend here
    // value is coming from state
    console.log(value);
  };

  // creating ref and initializing it with the sendRequest function
  const ref = useRef(sendRequest);

  useEffect(() => {
    // updating ref when state changes
    // now, ref.current will have the latest sendRequest
    // with access to the latest state
    ref.current = sendRequest;
  }, [value]);

  // creating debounced callback only once — on mount
  const debouncedCallback = useMemo(() => {
    // func will be created only once — on mount
    const func = () => {
      // ref is mutable! ref.current is a reference to the latest sendRequest
      ref.current?.();
    };
    // debounce the func that was created once
    // but has access to the latest sendRequest via ref
    return debounce(func, 1000);
    // no dependencies! never gets updated
  }, []);

  const onChange = (e) => {
    const value = e.target.value;
    setValue(value);
    debouncedCallback();
  };

  return <input onChange={onChange} value={value} />;
};
```

### Extracting into a useDebounce Hook

The closure escape logic can be extracted into a reusable hook:

```jsx
const useDebounce = (callback) => {
  const ref = useRef();

  useEffect(() => {
    ref.current = callback;
  }, [callback]);

  const debouncedCallback = useMemo(() => {
    const func = () => {
      ref.current?.();
    };

    return debounce(func, 1000);
  }, []);

  return debouncedCallback;
};
```

Then production code becomes clean and free of closure concerns:

```jsx
const Input = () => {
  const [value, setValue] = useState();

  const debouncedRequest = useDebounce(() => {
    // send request to the backend
    // access to the latest state here!
    console.log(value);
  });

  const onChange = (e) => {
    const value = e.target.value;
    setValue(value);
    debouncedRequest();
  };

  return <input onChange={onChange} value={value} />;
};
```

No chains of `useMemo` and `useCallback`, no worrying about dependencies, and full access to the latest state and props.

## useLayoutEffect: The Flickering Problem

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

React works the same way — it is an engine that splits our code into the smallest possible chunks that browsers can process within the ~13ms budget.

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

From the browser's perspective, this is one unbreakable task — exactly like the red-green-black border transition that could not be seen. The browser waits for the entire flow to complete, then paints only the final result. No flickering.

### When to Use Which

**Use `useLayoutEffect`** only when you need to eliminate visual glitches caused by adjusting UI based on real DOM measurements (sizes, positions).

**Use `useEffect`** for everything else. `useLayoutEffect` can hurt performance because it turns what would be multiple small tasks into one large blocking task. The last thing you want is for your entire app to become one giant synchronous task.

### Nuances of useEffect Timing

While the mental model of `useEffect` running inside `setTimeout` is convenient, it is not technically correct:

- React uses a `postMessage` combined with `requestAnimationFrame` trick internally.
- `useEffect` is not **guaranteed** to run asynchronously. If `useLayoutEffect` is somewhere in the chain of updates and triggers a state update, React must finish the current cycle before starting a new one. Since `useLayoutEffect` is synchronous, React has no choice but to run `useEffect` synchronously as well.

Each re-render cycle runs in order: **State update -> useLayoutEffect -> useEffect**. If any step triggers a new state update, a new cycle begins — but the current one finishes first.

## useLayoutEffect and SSR (Next.js)

In Server-Side Rendering frameworks like Next.js, `useLayoutEffect` **does not work** for the initial page load.

### Why SSR Breaks It

During SSR, the first rendering pass happens on the server via something like `React.renderToString(<App />)`. React calls all component functions and produces HTML, which is sent to the browser. The browser displays this HTML, then downloads and runs React on the client to "hydrate" the page with interactivity.

The problem: there is no browser during server rendering. No elements with dimensions exist — just strings. Since the whole purpose of `useLayoutEffect` is to access element sizes, React simply does not run it on the server.

As a result, what users see during the initial SSR load is the "first pass" — all navigation items visible. Only after client-side React runs does `useLayoutEffect` fire and hide the overflow items. The visual glitch is back.

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

While `typeof window === undefined` correctly detects the server environment, React requires the SSR HTML and the first client render HTML to **match exactly**. If they differ, the app will behave erratically — broken styles, mispositioned blocks, content in wrong places. Using state with `useEffect` ensures the first client render matches SSR, and only the subsequent update swaps in the real component.

## Key Takeaways

- **Debounce** waits until the user stops triggering, then fires once. **Throttle** fires periodically at regular intervals.
- Debounce/throttle functions must be **called only once** in a component's life (typically at mount). If called in the render function directly, the timer is re-created every render and they behave as simple delays.
- **useMemo/useCallback** can memoize debounced functions, but if the callback needs state, the dependencies change every render, breaking the memoization.
- **The Ref escape pattern** solves this: store the actual callback in a Ref (updated in `useEffect`), create the debounced function once (via `useMemo` with empty deps), and call `ref.current()` inside. The mutable ref always points to the latest closure.
- **useLayoutEffect** runs synchronously after DOM mutations but **before** the browser paints. It prevents visual flickering when adjusting UI based on DOM measurements.
- **useEffect** runs asynchronously (separate browser task), allowing a paint between the initial render and the effect — which causes the flicker.
- **Use `useLayoutEffect` sparingly** — only for visual glitch elimination. It blocks painting and can hurt performance.
- **useLayoutEffect and SSR**: React does not run `useLayoutEffect` on the server. Fix by conditionally rendering with a state flag toggled in `useEffect`.

## See Also

- [Closures in React](closures-in-react.md)
- [Refs and Imperative API](refs-and-imperative-api.md)
- [React Re-renders](react-re-renders.md)
- [Memoization in React](memoization.md)
- [Interaction Performance](interaction-performance.md)
- [Web Performance Metrics](web-performance-metrics.md)
