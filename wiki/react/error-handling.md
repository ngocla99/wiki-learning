# Universal Error Handling in React

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

Since React 16, any uncaught error thrown during the React lifecycle unmounts the entire application, leaving users with a blank white screen. This makes robust error handling non-negotiable for every React application. React provides two complementary mechanisms: standard JavaScript `try/catch` for imperative code (event handlers, async operations), and the `ErrorBoundary` class component API for declarative component tree errors. Neither mechanism alone covers all cases, but they can be combined through a clever `setState` trick that re-throws async errors into the render phase, enabling `ErrorBoundary` to catch them. This article covers the full spectrum of error handling strategies, their limitations, and how to combine them for comprehensive coverage.

## Why Error Handling Matters in React

Starting from version 16, an error thrown during the React lifecycle causes the entire app to unmount itself if not stopped. Before React 16, broken components would remain on screen even if malformed and misbehaving. Now, an unfortunate uncaught error in some insignificant part of the UI -- or even from some external library you have no control over -- can destroy the entire page and render an empty white screen for every user.

This behavioral change makes error handling a first-class concern. At minimum, a few `ErrorBoundary` components in strategic positions are essential to prevent a single failing widget from taking down the whole application.

## Catching Errors in JavaScript: try/catch Fundamentals

Before diving into React-specific patterns, recall how JavaScript handles errors natively. The `try/catch` statement is straightforward: try to do something, and if it fails, catch the mistake and handle it:

```js
try {
  // if we're doing something wrong, this might throw an error
  doSomething();
} catch (e) {
  // if error happened, catch it and do something with it
  // without stopping the app, like sending to a logging service
}
```

This works the same way with `async/await`:

```js
try {
  await fetch('/bla-bla');
} catch (e) {
  // oh no, the fetch failed! We should do something about it!
}
```

And for promise-based APIs, there is the `.catch()` method:

```js
fetch('/bla-bla')
  .then((result) => {
    // if a promise is successful, the result will be here
    // we can do something useful with it
  })
  .catch((e) => {
    // oh no, the fetch failed! We should do something about it!
  });
```

Both approaches accomplish the same thing; the rest of this article uses `try/catch` syntax for consistency.

## Simple try/catch in React: How To and Caveats

When an error is caught, the most user-friendly response is to render a fallback UI rather than leaving a blank screen. Since we can set state inside a `catch` block, a simple approach looks like this:

```jsx
const SomeComponent = () => {
  const [hasError, setHasError] = useState(false);

  useEffect(() => {
    try {
      // do something like fetching some data
    } catch (e) {
      // oh no! the fetch failed, we have no data to render!
      setHasError(true);
    }
  });

  // something happened during fetch, let's render a nice error screen
  if (hasError) return <SomeErrorScreen />;

  // all's good, data is here, let's render it
  return <SomeComponentContent {...data} />;
};
```

This works well for simple, predictable, narrow use cases like catching a failed `fetch` request. But trying to catch *all* errors in a component with `try/catch` alone reveals serious limitations.

### Limitation 1: You Cannot Wrap useEffect with try/catch

If you wrap `useEffect` with `try/catch`, it simply will not work:

```jsx
try {
  useEffect(() => {
    throw new Error('Hulk smash!');
  }, []);
} catch (e) {
  // useEffect throws, but this will never be called
}
```

This happens because `useEffect` is called **asynchronously after rendering**. From the `try/catch` perspective, everything went successfully -- the hook was registered without error. The actual effect callback runs later, long after the `try/catch` block has completed. This is the same story as with any Promise: if you do not `await` the result, JavaScript continues executing, and the `try/catch` block finishes before the async code runs.

To catch errors inside `useEffect`, the `try/catch` must be placed *inside* the effect:

```jsx
useEffect(() => {
  try {
    throw new Error('Hulk smash!');
  } catch (e) {
    // this one will be caught
  }
}, []);
```

This applies to any hook that uses `useEffect` internally or to anything asynchronous. Instead of one `try/catch` that wraps everything, you end up splitting into multiple blocks: one for each hook.

### Limitation 2: Cannot Catch Errors from Child Components

`try/catch` cannot catch anything that happens inside children components. Neither of these approaches works:

```jsx
const Component = () => {
  let child;

  try {
    child = <Child />;
  } catch (e) {
    // useless for catching errors inside Child component, won't be triggered
  }

  return child;
};
```

```jsx
const Component = () => {
  try {
    return <Child />;
  } catch (e) {
    // still useless for catching errors inside Child component, won't be triggered
  }
};
```

This happens because when you write `<Child />`, you are **not actually rendering** the component. You are creating a component Element -- just an object containing the component type and props that describes what should be rendered. React itself triggers the actual render of that component *later*, after the `try/catch` block has already executed successfully. This is the same timing issue as with `useEffect` and Promises.

### Limitation 3: Setting State During Render Causes Infinite Loops

If you try to catch errors outside of `useEffect` and callbacks (i.e., during the component's render), dealing with them via state updates is problematic. Setting state during render is not allowed, and code like this causes an infinite loop of re-renders:

```jsx
const Component = () => {
  const [hasError, setHasError] = useState(false);

  try {
    doSomethingComplicated();
  } catch (e) {
    // don't do that! will cause infinite loop in case of an error
    setHasError(true);
  }
};
```

You could return the error screen directly instead of setting state:

```jsx
const Component = () => {
  try {
    doSomethingComplicated();
  } catch (e) {
    // this is allowed
    return <SomeErrorScreen />;
  }
};
```

But this forces inconsistent error handling within a single component: state for `useEffect` and callbacks, direct return for everything else. The result is a mess:

```jsx
// while it will work, it's super cumbersome and hard to maintain, don't do that
const SomeComponent = () => {
  const [hasError, setHasError] = useState(false);

  useEffect(() => {
    try {
      // do something like fetching some data
    } catch (e) {
      // can't just return in case of errors in useEffect or callbacks
      // so have to use state
      setHasError(true);
    }
  });

  try {
    // do something during render
  } catch (e) {
    // but here we can't use state, so have to return directly in case of an error
    return <SomeErrorScreen />;
  }

  // and still have to return in case of error state here
  if (hasError) return <SomeErrorScreen />;

  return <SomeComponentContent {...data} />;
};
```

**Bottom line**: relying solely on `try/catch` in React means you either miss most errors or turn every component into an incomprehensible mess of code that will probably cause errors by itself.

## React ErrorBoundary Component

To mitigate the limitations above, React provides "Error Boundaries" -- a special API that turns a regular component into a `try/catch` equivalent, but for React's declarative code.

### Basic Usage

```jsx
const Component = () => {
  return (
    <ErrorBoundary>
      <SomeChildComponent />
      <AnotherChildComponent />
    </ErrorBoundary>
  );
};
```

If something goes wrong in any of those components or their children during render, the error will be caught and handled.

### Implementation

React does not give you the `ErrorBoundary` component directly -- it gives you the API to build one. The simplest implementation:

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    // initialize the error state
    this.state = { hasError: false };
  }

  // if an error happened, set the state to true
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  render() {
    // if error happened, return a fallback component
    if (this.state.hasError) {
      return <>Oh no! Epic fail!</>;
    }

    return this.props.children;
  }
}
```

This must be a **class component** -- there is no hooks equivalent for error boundaries. The `getDerivedStateFromError` static method is what turns the component into a proper error boundary.

### componentDidCatch for Error Logging

Another important method is `componentDidCatch`, which lets you send error information to a logging or monitoring service:

```jsx
class ErrorBoundary extends React.Component {
  // everything else stays the same

  componentDidCatch(error, errorInfo) {
    // send error to somewhere here
    log(error, errorInfo);
  }
}
```

- `getDerivedStateFromError` -- used to update state and render the fallback UI
- `componentDidCatch` -- used for side effects like error logging; receives the error and an `errorInfo` object with the component stack trace

### Making It Reusable with a Fallback Prop

Since `ErrorBoundary` is a regular component, you can make it configurable. A common pattern is accepting a fallback UI as a prop:

```jsx
class ErrorBoundary extends React.Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }

  static getDerivedStateFromError(error) {
    return { hasError: true };
  }

  componentDidCatch(error, errorInfo) {
    log(error, errorInfo);
  }

  render() {
    // if error happened, return the fallback component from props
    if (this.state.hasError) {
      return this.props.fallback;
    }

    return this.props.children;
  }
}
```

Usage:

```jsx
const Component = () => {
  return (
    <ErrorBoundary fallback={<>Oh no! Do something!</>}>
      <SomeChildComponent />
      <AnotherChildComponent />
    </ErrorBoundary>
  );
};
```

You can extend this further: reset state on button click, differentiate between error types, push the error into a Context, and so on.

## ErrorBoundary Limitations

Error boundaries only catch errors that happen **during the React lifecycle**. Things that happen outside of it disappear if not dealt with explicitly:

- **Event handlers** (e.g., `onClick`, `onChange`)
- **Resolved promises and async code** (`setTimeout`, `fetch().then()`)
- **Server-side rendering**

```jsx
const Component = () => {
  useEffect(() => {
    // this one WILL be caught by ErrorBoundary component
    throw new Error('Destroy everything!');
  }, []);

  const onClick = () => {
    // this error will just disappear into the void
    throw new Error('Hulk smash!');
  };

  useEffect(() => {
    // if this one fails, the error will also disappear
    fetch('/bla');
  }, []);

  return <button onClick={onClick}>click me</button>;
};

const ComponentWithBoundary = () => {
  return (
    <ErrorBoundary>
      <Component />
    </ErrorBoundary>
  );
};
```

Note: errors thrown directly inside `useEffect` (synchronous throws) *are* caught because they occur during the React lifecycle. It is the *async* code within effects (resolved promises, `setTimeout` callbacks) that escapes.

### The Naive Combination Approach

The common recommendation is to use `try/catch` for async and event handler errors. This works, but leads to maintaining dual error-handling mechanisms:

```jsx
const Component = () => {
  const [hasError, setHasError] = useState(false);

  // most errors in this component and children will be caught by ErrorBoundary

  const onClick = () => {
    try {
      // this error will be caught by catch
      throw new Error('Hulk smash!');
    } catch (e) {
      setHasError(true);
    }
  };

  if (hasError) return 'something went wrong';

  return <button onClick={onClick}>click me</button>;
};

const ComponentWithBoundary = () => {
  return (
    <ErrorBoundary fallback={'Oh no! Something went wrong'}>
      <Component />
    </ErrorBoundary>
  );
};
```

This approach has problems: every component needs its own "error" state, and each must decide what to render on error. You could propagate errors up to a parent via props or Context:

```jsx
const Component = ({ onError }) => {
  const onClick = () => {
    try {
      throw new Error('Hulk smash!');
    } catch (e) {
      // just call a prop instead of maintaining state here
      onError();
    }
  };

  return <button onClick={onClick}>click me</button>;
};

const ComponentWithBoundary = () => {
  const [hasError, setHasError] = useState();
  const fallback = 'Oh no! Something went wrong';

  if (hasError) return fallback;

  return (
    <ErrorBoundary fallback={fallback}>
      <Component onError={() => setHasError(true)} />
    </ErrorBoundary>
  );
};
```

But this is a lot of additional code for every child component, and you are essentially maintaining two error states -- one in the parent component and another inside `ErrorBoundary` itself. Since `ErrorBoundary` already has all the mechanisms to propagate errors up the tree, this is double work.

## Catching Async Errors with ErrorBoundary: The setState Trick

There is a clever trick to catch *all* errors with `ErrorBoundary`, including async errors and event handler errors.

The idea: catch errors with `try/catch` first, then inside the `catch` statement, trigger a normal React re-render and **re-throw** those errors back into the re-render lifecycle. That way, `ErrorBoundary` can catch them like any other render-phase error.

The key insight is that `setState` can accept an **updater function** as an argument. If that updater function throws, the error occurs *during React's state update (render phase)*, which `ErrorBoundary` can catch:

```jsx
const Component = () => {
  // create some random state that we'll use to throw errors
  const [state, setState] = useState();

  const onClick = () => {
    try {
      // something bad happened
    } catch (e) {
      // trigger state update, with updater function as an argument
      setState(() => {
        // re-throw this error within the updater function
        // it will be triggered during state update
        throw e;
      });
    }
  };
};
```

**Why this works**: When React processes the state update, it calls the updater function. Since the updater function throws, the error occurs during React's state reconciliation (render phase). `ErrorBoundary` catches any error during the render phase, so the async error is now caught.

### The useThrowAsyncError Hook

To avoid creating random state in every component, abstract the pattern into a reusable hook:

```jsx
const useThrowAsyncError = () => {
  const [state, setState] = useState();

  return (error) => {
    setState(() => {
      throw error;
    });
  };
};
```

Usage with async code:

```jsx
const Component = () => {
  const throwAsyncError = useThrowAsyncError();

  useEffect(() => {
    fetch('/bla')
      .then()
      .catch((e) => {
        // throw async error here!
        throwAsyncError(e);
      });
  });
};
```

### The useCallbackWithErrorHandling Hook

Another approach is a wrapper hook for callbacks that automatically catches and re-throws errors:

```jsx
const useCallbackWithErrorHandling = (callback) => {
  const [state, setState] = useState();

  return (...args) => {
    try {
      callback(...args);
    } catch (e) {
      setState(() => {
        throw e;
      });
    }
  };
};
```

Usage:

```jsx
const Component = () => {
  const onClick = () => {
    // do something dangerous here
  };

  const onClickWithErrorHandler =
    useCallbackWithErrorHandling(onClick);

  return (
    <button onClick={onClickWithErrorHandler}>
      click me!
    </button>
  );
};
```

With these hooks in place, no errors can escape -- async errors, event handler errors, and render errors are all funneled through `ErrorBoundary`.

## react-error-boundary Library

For those who prefer battle-tested libraries over hand-rolled solutions, the `react-error-boundary` library provides a production-ready `ErrorBoundary` component with several useful features:

- A functional component API (wrapping the class component internally)
- Built-in reset functionality (retry after error)
- `onError` callback for logging
- A similar re-throw mechanism for catching async errors
- `useErrorBoundary` hook for programmatic error throwing (same concept as `useThrowAsyncError` above)

Whether to use the library or implement your own is a matter of personal preference, coding style, and the specific needs of your application. The library operates on the same principles described in this article.

## Strategic Placement of ErrorBoundaries

Where you place `ErrorBoundary` components matters as much as using them at all. The goal is to isolate failures so that an error in one part of the UI does not bring down unrelated parts.

### At the App Root (Catch-All)

Every application should have at least one `ErrorBoundary` at the top level. This is the last line of defense -- if every other boundary misses an error, this one catches it and shows a generic error page instead of a blank white screen:

```jsx
const App = () => {
  return (
    <ErrorBoundary fallback={<GenericErrorPage />}>
      <Router>
        <AppContent />
      </Router>
    </ErrorBoundary>
  );
};
```

### Around Independent Features and Widgets

If your page has independent sections (sidebar, header, dashboard widgets), wrapping each in its own `ErrorBoundary` prevents a failure in one widget from taking down the others:

```jsx
const Dashboard = () => {
  return (
    <div>
      <ErrorBoundary fallback={<WidgetError />}>
        <WeatherWidget />
      </ErrorBoundary>
      <ErrorBoundary fallback={<WidgetError />}>
        <StockTicker />
      </ErrorBoundary>
      <ErrorBoundary fallback={<WidgetError />}>
        <NotificationsFeed />
      </ErrorBoundary>
    </div>
  );
};
```

### Around Third-Party Components

Third-party libraries are code you do not control. Wrapping them in an `ErrorBoundary` ensures that a bug in an external component does not crash your entire application:

```jsx
const Page = () => {
  return (
    <div>
      <ErrorBoundary fallback={<p>Chart failed to load</p>}>
        <ThirdPartyChartLibrary data={data} />
      </ErrorBoundary>
      <OurOwnContent />
    </div>
  );
};
```

### Around Route Components

For applications with routing, placing `ErrorBoundary` around each route component provides per-page error handling. If one page crashes, users can navigate to other pages that still work:

```jsx
const App = () => {
  return (
    <ErrorBoundary fallback={<GenericErrorPage />}>
      <Routes>
        <Route path="/dashboard" element={
          <ErrorBoundary fallback={<PageError />}>
            <DashboardPage />
          </ErrorBoundary>
        } />
        <Route path="/settings" element={
          <ErrorBoundary fallback={<PageError />}>
            <SettingsPage />
          </ErrorBoundary>
        } />
      </Routes>
    </ErrorBoundary>
  );
};
```

## Key Takeaways

- **Uncaught errors in React 16+ unmount the entire app**, showing a blank white screen. A few `ErrorBoundary` components in strategic places are non-negotiable.
- **`try/catch` catches errors in callbacks and promises** just fine, but it cannot catch errors from nested child components, and you cannot wrap `useEffect` or the component's return statement with it.
- **`ErrorBoundary` is the opposite**: it catches errors from any component down the render tree, but it skips promises, callbacks, and anything async.
- **The two can be merged** into a universal solution: catch async errors with `try/catch`, then re-throw them into the normal React lifecycle via a `setState` updater function that throws.
- **The `useThrowAsyncError` hook** (or the `useCallbackWithErrorHandling` wrapper) abstracts this pattern into a clean, reusable API.
- **The `react-error-boundary` library** provides a production-ready implementation that operates on these same principles.
- **Place ErrorBoundaries strategically**: at the app root as a catch-all, around independent features to isolate failures, around third-party components you do not control, and around route components for per-page error handling.

## See Also

- [Data Fetching Patterns](data-fetching-patterns.md) -- error handling during data fetching
- [Lazy Loading and Suspense](lazy-loading-and-suspense.md) -- Suspense fallbacks work alongside ErrorBoundary
- [Closures in React](closures-in-react.md) -- understanding why try/catch timing issues occur
- [Component Composition Patterns](component-composition-patterns.md) -- structuring components for better error isolation
- [React Re-renders](react-re-renders.md) -- understanding the render lifecycle that ErrorBoundary hooks into
