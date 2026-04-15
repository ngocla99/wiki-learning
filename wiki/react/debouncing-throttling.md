# Debouncing and Throttling

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

Debouncing and throttling are techniques for controlling how often a function executes when triggered by rapid events like typing, scrolling, or resizing. Implementing them correctly in React is surprisingly challenging because component re-renders re-create functions on every render, destroying the internal timers these techniques rely on. This article covers why naive approaches fail, how to use Refs and the closure escape pattern to build a robust `useDebounce` hook.

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

## Key Takeaways

- **Debounce** waits until the user stops triggering, then fires once. **Throttle** fires periodically at regular intervals.
- Debounce/throttle functions must be **called only once** in a component's life (typically at mount). If called in the render function directly, the timer is re-created every render and they behave as simple delays.
- **useMemo/useCallback** can memoize debounced functions, but if the callback needs state, the dependencies change every render, breaking the memoization.
- **The Ref escape pattern** solves this: store the actual callback in a Ref (updated in `useEffect`), create the debounced function once (via `useMemo` with empty deps), and call `ref.current()` inside. The mutable ref always points to the latest closure.

## See Also

- [useLayoutEffect](use-layout-effect.md) -- flicker prevention, browser rendering pipeline, SSR considerations
- [Closures in React](closures-in-react.md)
- [Refs and Imperative API](refs-and-imperative-api.md)
- [React Re-renders](react-re-renders.md)
- [Memoization in React](memoization.md)
- [Interaction Performance](interaction-performance.md)
