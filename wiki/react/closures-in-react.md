# Closures in React

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

Closures are formed every time a function is created inside another function in JavaScript. Since React components are just functions, every callback, hook, and inline function inside a component forms a closure over the component's current scope. This creates one of the most subtle and feared categories of bugs in React: the **stale closure problem**. Understanding closures deeply is essential for working correctly with `useCallback`, `useMemo`, Refs, and `React.memo`.

## JavaScript Scope and Closures

### Local Scope

When you declare a function in JavaScript (via declaration or arrow function), you create a **local scope** — variables declared inside are invisible from the outside:

```js
const something = () => {
  const value = 'text';
};

console.log(value); // ReferenceError: value is not defined
```

Functions created inside other functions also have their own local scope, invisible to the outer function:

```js
const something = () => {
  const inside = () => {
    const value = 'text';
  };

  console.log(value); // won't work, "value" is local to "inside"
};
```

### The Open Road Inward

In the opposite direction, it is an open road. The innermost function can "see" all variables declared outside:

```js
const something = () => {
  const value = 'text';

  const inside = () => {
    // perfectly fine, value is available here
    console.log(value);
  };
};
```

This works because the `inside` function **closes over** all the data from the outside. A closure is essentially a snapshot of all "outside" data frozen in time and stored separately in memory.

### Closures as Frozen Snapshots

If we pass `value` as an argument and return the inner function, we can observe how closures freeze values:

```js
const something = (value) => {
  const inside = () => {
    console.log(value);
  };

  return inside;
};

const first = something('first');
const second = something('second');

first();  // logs "first"
second(); // logs "second"
```

Each call to `something` creates a new closure. The returned function has permanent access to the value that was passed when it was created. It is like taking a photograph of a dynamic scene — as soon as you press the button, the entire scene is "frozen" in the picture forever.

This applies to **any** variable declared locally inside the outer function:

```js
const something = (value) => {
  const r = Math.random();

  const inside = () => {
    console.log(value, r);
  };

  return inside;
};

const first = something('first');
const second = something('second');

first();  // logs "first" and some random number
second(); // logs "second" and a different random number
```

### Closures in React Components

In React, we create closures all the time without realizing it. Every callback declared inside a component is a closure:

```jsx
const Component = () => {
  const onClick = () => {
    // closure!
  };

  return <button onClick={onClick} />;
};
```

Everything in `useEffect` or `useCallback` is a closure:

```jsx
const Component = () => {
  const [state, setState] = useState();

  const onClick = useCallback(() => {
    // closure! has access to state
    console.log(state);
  });

  useEffect(() => {
    // closure! has access to state
    console.log(state);
  });
};
```

Every single function inside a component is a closure, since a component itself is just a function.

## The Stale Closure Problem

Closures live for as long as a reference to the function that caused them exists. That reference is just a value that can be assigned to anything. The problem emerges when we **cache** a closure and prevent it from being re-created.

### A Pure JavaScript Example

Consider what happens if we try to cache the inner function to avoid re-creating it:

```js
const cache = {};

const something = (value) => {
  if (!cache.current) {
    cache.current = () => {
      console.log(value);
    };
  }

  return cache.current;
};
```

On the surface, this seems harmless — just an external cache. But observe what happens:

```js
const first = something('first');
const second = something('second');
const third = something('third');

first();  // logs "first"
second(); // logs "first"  <-- stale!
third();  // logs "first"  <-- stale!
```

No matter how many times we call `something` with different arguments, the logged value is always "first". This is a **stale closure**. The closure was frozen at the point when it was first created, saved into the cache, and never refreshed.

### Fixing with Dependency Tracking

To fix this, we need to re-create the function and its closure every time the value changes — essentially implementing dependency tracking:

```js
const cache = {};
let prevValue;

const something = (value) => {
  // check whether the value has changed
  if (!cache.current || value !== prevValue) {
    cache.current = () => {
      console.log(value);
    };
  }

  // refresh it
  prevValue = value;
  return cache.current;
};

const first = something('first');
const anotherFirst = something('first');
const second = something('second');

first();  // logs "first"
second(); // logs "second"
console.log(first === anotherFirst); // true — same cached reference
```

This is essentially what React's `useCallback` hook does for us.

## Stale Closures in useCallback

Every time we use `useCallback`, we create a closure, and the function we pass to it is cached:

```jsx
// the inline function is cached
const onClick = useCallback(() => {}, []);
```

If we need access to state or props inside this function, we must add them to the dependencies array:

```jsx
const Component = () => {
  const [state, setState] = useState();

  const onClick = useCallback(() => {
    // access to state inside
    console.log(state);

    // need to add this to the dependencies array
  }, [state]);
};
```

The dependencies array is what makes React refresh the cached closure, exactly like the `value !== prevValue` comparison in the manual example. If we **forget** that array, the closure becomes stale:

```jsx
const Component = () => {
  const [state, setState] = useState();

  const onClick = useCallback(() => {
    // state will always be the initial state value here
    // the closure is never refreshed
    console.log(state);

    // forgot about dependencies!
  }, []);
};
```

Every time we trigger that callback, all that gets logged is `undefined` — the initial state value frozen in the very first closure.

## Stale Closures in Refs

The second most common way to introduce the stale closure problem, after `useCallback`/`useMemo`, is Refs.

Some articles recommend using Refs instead of `useCallback` to memoize props. On the surface, it looks simpler — just pass a function to `useRef` and access it through `ref.current`:

```jsx
const Component = () => {
  const ref = useRef(() => {
    // click handler
  });

  // ref.current stores the function and is stable between re-renders
  return <HeavyComponent onClick={ref.current} />;
};
```

However, every function inside a component forms a closure, including the function passed to `useRef`. A Ref is initialized only once and never updated by itself. It is basically the caching logic from the beginning — only the first closure is ever stored:

```js
const ref = {};

const useRef = (callback) => {
  if (!ref.current) {
    ref.current = callback;
  }

  return ref.current;
};
```

So the closure formed at mount time is preserved forever. When we try to access state or props inside that function, we only get their initial values:

```jsx
const Component = ({ someProp }) => {
  const [state, setState] = useState();

  const ref = useRef(() => {
    // both will be stale and will never change
    console.log(someProp);
    console.log(state);
  });
};
```

### Fixing Stale Refs

To fix this, we need to update the ref value every time something it depends on changes — essentially reimplementing the dependencies array functionality:

```jsx
const Component = ({ someProp }) => {
  const [state, setState] = useState();

  // initialize ref — creates closure!
  const ref = useRef(() => {
    console.log(someProp);
    console.log(state);
  });

  useEffect(() => {
    // update the closure when state or props change
    ref.current = () => {
      console.log(someProp);
      console.log(state);
    };
  }, [state, someProp]);
};
```

## Stale Closures in React.memo

This is the scenario that makes the stale closure problem most painful. Consider a form with a heavy memoized component:

```jsx
const HeavyComponentMemo = React.memo(
  HeavyComponent,
  (before, after) => {
    return before.title === after.title;
  },
);

const Form = () => {
  const [value, setValue] = useState();

  const onClick = () => {
    // submit our form data here
    console.log(value);
  };

  return (
    <>
      <input
        type="text"
        value={value}
        onChange={(e) => setValue(e.target.value)}
      />
      <HeavyComponentMemo
        title="Welcome to the form"
        onClick={onClick}
      />
    </>
  );
};
```

The heavy component does not re-render when typing — great for performance. But clicking the button always logs `undefined`. Why?

**Step-by-step:**

1. When the component first renders, `onClick` is created with a closure capturing the default state value (`undefined`).
2. This closure is passed to `HeavyComponentMemo` along with the `title` prop.
3. Inside the comparison function, only `title` is compared. It never changes (it is a string literal), so the comparison always returns `true`.
4. `HeavyComponent` is never updated, so it holds a reference to the **very first** `onClick` closure — the one with `undefined` frozen in it.

### The Dilemma

Including `onClick` in the comparison function means reimplementing what `React.memo` does by default:

```jsx
(before, after) => {
  return (
    before.title === after.title &&
    before.onClick === after.onClick
  );
};
```

But then we would need to wrap `onClick` in `useCallback`. And since it depends on `value` (the state), it changes with every keystroke — defeating the entire purpose of memoization. We are back to square one.

## Escaping the Closure Trap with Refs

This pattern is the key technique for having a **stable callback** that also has access to the **latest state**. Here is how it works, step by step.

### Step 1: Start with the Basics

Start with a clean component with state and a memoized heavy component:

```jsx
const HeavyComponentMemo = React.memo(HeavyComponent);

const Form = () => {
  const [value, setValue] = useState();

  return (
    <>
      <input
        type="text"
        value={value}
        onChange={(e) => setValue(e.target.value)}
      />
      <HeavyComponentMemo title="Welcome to the form" onClick={...} />
    </>
  );
};
```

### Step 2: Create an Empty Ref

We need an `onClick` that is stable between re-renders but also has access to the latest state. Add an empty ref:

```jsx
const Form = () => {
  const [value, setValue] = useState();

  // adding an empty ref
  const ref = useRef();
};
```

### Step 3: Update the Ref in useEffect

For the function to have access to the latest state, it needs to be re-created with every re-render — that is the nature of closures. We update the ref in `useEffect` (without a dependency array, so it runs on every render):

```jsx
const Form = () => {
  const [value, setValue] = useState();
  const ref = useRef();

  useEffect(() => {
    // our callback that we want to trigger, with latest state
    ref.current = () => {
      console.log(value);
    };
    // no dependencies array! runs every render
  });
};
```

Now `ref.current` always holds a closure with the latest state.

### Step 4: Create a Stable Wrapper with useCallback

We cannot pass `ref.current` directly to the memoized component — that value changes every render, breaking memoization. Instead, create a stable wrapper:

```jsx
const onClick = useCallback(() => {
  // empty dependency! will never change
}, []);
```

### Step 5: The Magic — Call ref.current Inside the Stable Callback

```jsx
useEffect(() => {
  ref.current = () => {
    console.log(value);
  };
});

const onClick = useCallback(() => {
  // call the ref here
  ref.current?.();

  // still empty dependencies array!
}, []);
```

Notice that `ref` is **not** in the dependencies of `useCallback`. It does not need to be. `ref` by itself never changes — it is just a reference to a mutable object returned by `useRef`.

### Why This Works

When a closure freezes everything around it, it does **not** make objects immutable or frozen. Objects are stored in a different part of memory, and multiple variables can contain references to the same object:

```js
const a = { value: 'one' };
const b = a; // b references the same object

a.value = 'two';
console.log(b.value); // "two"
```

In our case, we have exactly the same `ref` reference inside `useCallback` and inside `useEffect`. When we mutate `ref.current` inside `useEffect`, we can access that same property inside our `useCallback`. That property happens to be a closure that captured the **latest** state data.

### The Complete Solution

```jsx
const HeavyComponentMemo = React.memo(HeavyComponent);

const Form = () => {
  const [value, setValue] = useState();
  const ref = useRef();

  useEffect(() => {
    ref.current = () => {
      // will be latest
      console.log(value);
    };
  });

  const onClick = useCallback(() => {
    // will be latest
    ref.current?.();
  }, []);

  return (
    <>
      <input
        type="text"
        value={value}
        onChange={(e) => setValue(e.target.value)}
      />
      <HeavyComponentMemo
        title="Welcome closures"
        onClick={onClick}
      />
    </>
  );
};
```

Now we have the best of both worlds:
- The heavy component is properly memoized and **does not** re-render with every state change.
- The `onClick` callback has access to the **latest** data in the component without ruining memoization.

## Key Takeaways

- **Closures form every time** a function is created inside another function. Since React components are just functions, every function created inside forms a closure, including hooks like `useCallback` and `useRef`.
- **Frozen snapshot**: When a function that forms a closure is called, all the data around it is "frozen" like a snapshot at that point in time.
- **Refreshing closures**: To update that data, we need to re-create the "closed" function. This is what the dependencies array of hooks like `useCallback` allows us to do.
- **Stale closures occur** when we miss a dependency, or fail to refresh a closed function assigned to `ref.current`.
- **The Ref escape pattern**: We can escape the stale closure trap by taking advantage of the fact that Refs are mutable objects. We mutate `ref.current` outside of the stale closure (in `useEffect`), then access it inside. The value will always be the latest data.
- **Objects are not frozen by closures**: Closures freeze references to objects, not the objects themselves. Mutating a property on a mutable object is visible to all closures that hold a reference to that object.

## See Also

- [Refs and Imperative API](refs-and-imperative-api.md)
- [Debouncing and Throttling](debouncing-throttling.md)
- [Memoization in React](memoization.md)
- [Component Composition Patterns](component-composition-patterns.md)
- [React Re-renders](react-re-renders.md)
