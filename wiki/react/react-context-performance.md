# React Context and Performance

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

React Context provides a way to pass data directly from one component to another deep in the render tree without prop drilling. When applied correctly, Context can significantly improve performance by eliminating unnecessary re-renders of intermediate components. However, Context has serious caveats: all consumers re-render when the provider value changes, this re-render cannot be stopped by memoization, and all consumers re-render even if they only use a portion of the value that did not change. Understanding these tradeoffs -- and the techniques to mitigate them -- is the same mental model used by external state management libraries like Redux.

## The Problem Context Solves

Consider a page with a collapsible sidebar and a main content area. The sidebar has an expand/collapse button, and somewhere deep in the main area, a component needs to know whether the sidebar is expanded to decide how many columns to render.

Without Context, the only option is to lift state up to their closest common parent and drill props down through every intermediate component:

```jsx
const Page = () => {
  const [isNavExpanded, setIsNavExpanded] = useState();

  return (
    <Layout>
      <Sidebar
        isNavExpanded={isNavExpanded}
        toggleNav={() => setIsNavExpanded(!isNavExpanded)}
      />
      <MainPart isNavExpanded={isNavExpanded} />
    </Layout>
  );
};
```

This approach has two problems:

1. **Bloated APIs**: `Sidebar` and `MainPart` accept props they do not use themselves -- they only forward them to children. Every component in the chain gets props it does not care about.
2. **Poor performance**: Every time the button is clicked, the state in `Page` changes. That state change causes `Page` and every component inside it to re-render -- including `VerySlowComponent` and `AnotherVerySlowComponent` inside `MainPart`, which have nothing to do with navigation state.

Composition techniques (children as props, component extraction) cannot help here because `Sidebar` and `MainPart` both directly depend on the state that triggers re-rendering. Memoizing every intermediate slow component is possible but makes the code even more bloated.

## How Context Improves Performance

Context allows you to escape the component tree and pass data directly from a provider at the top to consumers at the bottom. The key pattern combines Context with the "children as props" technique:

```jsx
const Context = React.createContext({
  isNavExpanded: true,
  toggle: () => {},
});

const NavigationController = ({ children }) => {
  const [isNavExpanded, setIsNavExpanded] = useState(true);
  const toggle = () => setIsNavExpanded(!isNavExpanded);
  const value = { isNavExpanded, toggle };

  return (
    <Context.Provider value={value}>
      {children}
    </Context.Provider>
  );
};
```

The `Page` component passes everything as children:

```jsx
const Page = () => {
  return (
    <NavigationController>
      <Layout>
        <Sidebar />
        <MainPart />
      </Layout>
    </NavigationController>
  );
};
```

Only the components that actually consume the Context re-render when the value changes:

```jsx
const ExpandButton = () => {
  const { isNavExpanded, toggle } = useNavigation();
  return (
    <button onClick={toggle}>
      {isNavExpanded ? 'Collapse' : 'Expand'}
    </button>
  );
};

const AdjustableColumnsBlock = () => {
  const { isNavExpanded } = useNavigation();
  return isNavExpanded ? <TwoColumns /> : <ThreeColumns />;
};
```

All other components inside `Sidebar` or `MainPart` that do not call `useNavigation()` are completely unaffected by the state change. The children passed through `NavigationController` are just props -- they are not re-created when the controller's internal state changes.

## The Three Critical Caveats

Context has a bad reputation for good reason. Three things must be understood:

1. **Context consumers re-render when the `value` on the Provider changes.**
2. **All consumers re-render, even if they only use a portion of the value that did not change.**
3. **Those re-renders cannot be prevented with memoization (easily).**

## Caveat 1: Preventing Unnecessary Value Changes

Every time the `NavigationController` re-renders -- for any reason, not just its own state change -- the `value` object is re-created. Because objects are compared by reference, a new object means the Context value has "changed," and every consumer re-renders.

This happens when the provider sits inside a component that has its own state. For example, if `Layout` adds scroll tracking:

```jsx
const Layout = ({ children }) => {
  const [scroll, setScroll] = useState();

  useEffect(() => {
    window.addEventListener('scroll', () => {
      setScroll(window.scrollY);
    });
  }, []);

  return (
    <NavigationController>
      <div className="layout">{children}</div>
    </NavigationController>
  );
};
```

Now every scroll event causes `Layout` to re-render, which re-renders `NavigationController`, which re-creates the `value` object, which triggers re-renders in every Context consumer. Half the app re-renders on every scroll.

### The Fix: Always Memoize the Provider Value

```jsx
const NavigationController = ({ children }) => {
  const [isNavExpanded, setIsNavExpanded] = useState(true);

  const toggle = useCallback(() => {
    setIsNavExpanded(!isNavExpanded);
  }, [isNavExpanded]);

  const value = useMemo(() => {
    return { isNavExpanded, toggle };
  }, [isNavExpanded, toggle]);

  return (
    <Context.Provider value={value}>
      {children}
    </Context.Provider>
  );
};
```

This is one of the few cases where memoization is not premature optimization. It prevents much bigger problems that will almost inevitably occur as the app grows.

## Caveat 2: Splitting Providers

Even with memoization, all consumers re-render when any part of the value changes. If you add `open` and `close` functions (with no dependencies) to the navigation API:

```jsx
const open = useCallback(() => setIsNavExpanded(true), []);
const close = useCallback(() => setIsNavExpanded(false), []);

const value = useMemo(() => {
  return { isNavExpanded, open, close };
}, [isNavExpanded, open, close]);
```

A component that only uses `open` will still re-render when `isNavExpanded` changes, because the entire Context value changed. This cannot be prevented by wrapping the consumer in `useCallback` or `useMemo`:

```jsx
// DOES NOT WORK - component still re-renders
const useNavOpen = () => {
  const { open } = useNavigation();
  return useCallback(open, []);
};
```

### The Fix: Split Into Multiple Contexts

Create separate Contexts for data that changes and API that does not:

```jsx
const ContextData = React.createContext({ isNavExpanded: false });
const ContextApi = React.createContext({ open: () => {}, close: () => {} });

const NavigationController = ({ children }) => {
  const [isNavExpanded, setIsNavExpanded] = useState(true);

  const open = useCallback(() => setIsNavExpanded(true), []);
  const close = useCallback(() => setIsNavExpanded(false), []);

  const data = useMemo(() => ({ isNavExpanded }), [isNavExpanded]);
  const api = useMemo(() => ({ open, close }), [open, close]);

  return (
    <ContextData.Provider value={data}>
      <ContextApi.Provider value={api}>
        {children}
      </ContextApi.Provider>
    </ContextData.Provider>
  );
};
```

Expose two hooks:

```jsx
const useNavigationData = () => useContext(ContextData);
const useNavigationApi = () => useContext(ContextApi);
```

Now a component that only needs `open` uses `useNavigationApi()` and never re-renders when navigation state changes, because the `api` value never changes.

### The Toggle Problem

The `toggle` function depends on the current state (`!isNavExpanded`), so it cannot go into the API context without making that context depend on state again. This means consumers must implement toggle logic themselves:

```jsx
const ExpandButton = () => {
  const { isNavExpanded } = useNavigationData();
  const { open, close } = useNavigationApi();

  return (
    <button onClick={isNavExpanded ? close : open}>
      {isNavExpanded ? 'Collapse' : 'Expand'}
    </button>
  );
};
```

This is not ideal. The solution is `useReducer`.

## Reducers and Split Providers

`useReducer` eliminates the dependency of action functions on state. Instead of manipulating state directly, functions dispatch named actions:

```jsx
const reducer = (state, action) => {
  switch (action.type) {
    case 'open-sidebar':
      return { ...state, isNavExpanded: true };
    case 'close-sidebar':
      return { ...state, isNavExpanded: false };
    case 'toggle-sidebar':
      return { ...state, isNavExpanded: !state.isNavExpanded };
  }
};

const NavigationController = ({ children }) => {
  const [state, dispatch] = useReducer(reducer, {
    isNavExpanded: true,
  });

  const api = useMemo(() => ({
    open: () => dispatch({ type: 'open-sidebar' }),
    close: () => dispatch({ type: 'close-sidebar' }),
    toggle: () => dispatch({ type: 'toggle-sidebar' }),
  }), []); // No dependencies! dispatch is stable

  const data = useMemo(() => ({
    isNavExpanded: state.isNavExpanded,
  }), [state.isNavExpanded]);

  return (
    <ContextData.Provider value={data}>
      <ContextApi.Provider value={api}>
        {children}
      </ContextApi.Provider>
    </ContextData.Provider>
  );
};
```

None of the action functions depend on state anymore -- they just call `dispatch`. The `api` value has an empty dependency array, so it never changes. Consumers of the API context never re-render on state changes.

The reducer pattern is especially powerful with multiple state variables and complex actions, but from the re-render perspective it works the same as `useState`: updating state through `dispatch` still forces the component to re-render.

## Context Selectors via Higher-Order Components

What if splitting providers is too heavy-handed for a one-off optimization? React does not natively support Context selectors (selecting a subset of the value). But we can imitate selectors using higher-order components and `React.memo`:

```jsx
const withNavigationOpen = (AnyComponent) => {
  const AnyComponentMemo = React.memo(AnyComponent);

  return (props) => {
    const { open } = useContext(Context);

    return <AnyComponentMemo {...props} openNav={open} />;
  };
};
```

How this works:

1. The HOC returns a wrapper component that consumes the Context.
2. When the Context value changes, the wrapper re-renders (unavoidable).
3. Inside the wrapper, the actual component is wrapped in `React.memo`.
4. The memoized component only re-renders if its props change.
5. The `open` function is memoized inside the provider, so it never changes.
6. Props from outside (`...props`) are not affected by context changes.

The result: `SomeHeavyComponent` wrapped in `withNavigationOpen` gets the `openNav` prop but does not re-render when the Context value changes:

```jsx
const SomeHeavyComponent = withNavigationOpen(
  ({ openNav }) => {
    return <button onClick={openNav}>Open nav</button>;
  },
);
```

This pattern works but requires careful attention: whatever is passed as props to the memoized component must not change between re-renders.

## When to Use Context vs External State Management

Context works well for smaller apps with a few isolated pieces of shared state. For larger, more complex applications, an external state management solution that supports memoized selectors (Redux, Zustand, Jotai) is preferable due to Context's inherent re-render behavior. However, the mental model is the same -- if you learn Context performance patterns, you can use any state management library optimally.

## Key Takeaways

- Context (or any context-like state management) passes data directly from one component to another deep in the render tree without prop drilling
- This can improve performance by avoiding re-renders of all intermediate components
- All components consuming a Context re-render when the provider value changes -- this cannot be stopped with standard memoization
- Always memoize the value passed to a Context provider with `useMemo`/`useCallback`
- Split providers into data (changes) and API (stable) to minimize re-renders
- `useReducer` makes split providers practical by making `dispatch` stable and independent of state
- Higher-order components with `React.memo` can imitate Context selectors for one-off optimizations

## See Also

- [React State Management](state-management.md) — when Context is sufficient vs. when to reach for Zustand or Redux
- [React Re-renders](react-re-renders.md)
- [Memoization in React](memoization.md)
- [Component Composition Patterns](component-composition-patterns.md)
- [Closures in React](closures-in-react.md)
