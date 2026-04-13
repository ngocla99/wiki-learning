# Memoization in React

> Sources: Nadia Makarevich, 2023; Nadia Makarevich, 2025
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md); [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)

## Overview

Memoization in React centers on three tools: `useMemo`, `useCallback`, and `React.memo`. Together they preserve references between re-renders so that React can skip unnecessary work. However, doing memoization correctly is much harder than it seems -- at least half the time it is applied, something in the chain is broken and the memoization provides no benefit. Composition patterns (moving state down, children as props, context) should be tried first; memoization should be a last resort. The React Compiler aims to automate the process, but has its own limitations.

## JavaScript Reference Comparison

The root cause behind memoization needs is how JavaScript compares non-primitive values. Primitives are compared by value:

```js
const a = 1;
const b = 1;
a === b; // true -- same value
```

Objects, arrays, and functions are compared by reference -- the memory address, not the content:

```js
const a = { id: 1 };
const b = { id: 1 };
a === b; // false -- different objects in memory

const c = a;
a === c; // true -- same reference
```

This matters because React uses `Object.is` (which behaves like `===` for these cases) to compare:

- Hook dependencies in `useEffect`, `useMemo`, `useCallback`
- Props on components wrapped in `React.memo`
- Element objects during reconciliation

Any locally declared object, array, or function inside a component gets a **new reference** on every re-render, because a re-render is just React calling the component function again. Every local variable is re-created, like any JavaScript function call:

```jsx
const Component = () => {
  const submit = () => {};  // new function reference every render

  useEffect(() => {
    submit();
  }, [submit]);  // fires on EVERY re-render -- submit is always "new"

  return ...
};
```

## useMemo and useCallback: How They Work

Both hooks preserve a reference between re-renders, returning the cached value as long as dependencies have not changed.

**`useCallback`** memoizes the function itself:

```jsx
const submit = useCallback(() => {
  // submit logic
}, []); // empty deps: reference never changes
```

**`useMemo`** memoizes the return value of the function:

```jsx
const submit = useMemo(() => {
  return () => {
    // submit logic -- this function is the memoized RESULT
  };
}, []);
```

### How They Work Internally

The inline function passed as the first argument to either hook is **always re-created** on every re-render. This is normal JavaScript -- calling a function with an inline callback always creates a new callback:

```js
const func = (callback) => { /* ... */ };
func(() => {});  // first call -- new function
func(() => {});  // second call -- new function again
```

For `useCallback`, React does something like this internally:

```js
let cachedCallback;

const useCallback = (callback, deps) => {
  if (dependenciesEqual(deps)) {
    return cachedCallback;  // return the old one
  }
  cachedCallback = callback;  // store the new one
  return callback;
};
```

For `useMemo`, the only difference is that React calls the function and caches the result:

```js
let cachedResult;

const useMemo = (callback, deps) => {
  if (dependenciesEqual(deps)) {
    return cachedResult;
  }
  cachedResult = callback();  // call it, cache result
  return cachedResult;
};
```

**Common misconception**: Some believe `useMemo` is "better for performance" than `useCallback` because `useCallback` re-creates the function on every render. As shown above, **both** re-create the inline function. The only difference is what gets cached (the function vs. its return value).

The one case where it matters: if you pass a function *call* as the first argument:

```jsx
const submit = useCallback(something(), []);
// something() is called every re-render, even though submit's reference is stable
```

Avoid expensive calculations inside those argument functions.

## The Anti-pattern: Memoizing Props Without React.memo

This is the most common memoization mistake:

```jsx
const Component = () => {
  const onClick = useCallback(() => {
    // do something
  }, []);

  return <button onClick={onClick}>click me</button>;
};
```

This `useCallback` is **completely useless**. There is a widespread belief that memoizing props prevents a component from re-rendering. It does not. As covered in the composition patterns, if a parent re-renders, every child component re-renders, regardless of whether their props changed. The `useCallback` here just makes React do extra work and makes the code harder to read.

When it is just one `useCallback`, it does not look bad. But in practice there is never just one. They start depending on each other, forming chains, and the logic gets buried under incomprehensible layers of memoization hooks.

### When Memoizing Props Actually Makes Sense

There are only two legitimate reasons to memoize a prop value:

1. **The prop is used as a dependency in a hook** in the downstream component:

```jsx
const Parent = () => {
  // Must be memoized -- Child uses it in useEffect
  const fetch = useCallback(() => { /* ... */ }, []);
  return <Child onMount={fetch} />;
};

const Child = ({ onMount }) => {
  useEffect(() => {
    onMount();
  }, [onMount]);  // if onMount changes every render, this fires every render
};
```

2. **The receiving component is wrapped in `React.memo`** (see next section).

3. The component passes those props down to other components that satisfy condition 1 or 2.

## React.memo: What It Does

`React.memo` is a higher-order component that wraps a component and adds a props check **before** re-rendering. When a re-render is triggered by a parent (and only then), React stops and compares every prop by reference. If nothing changed, the component is not re-rendered, and the cascade stops:

```jsx
const Child = ({ data, onChange }) => { /* ... */ };
const ChildMemo = React.memo(Child);

const Component = () => {
  const data = useMemo(() => ({ /* ... */ }), []);
  const onChange = useCallback(() => { /* ... */ }, []);

  // data and onChange have stable references
  // ChildMemo will NOT re-render when Component re-renders
  return <ChildMemo data={data} onChange={onChange} />;
};
```

If even **one** prop has a new reference, `React.memo` gives up and re-renders the component as usual:

```jsx
const Component = () => {
  // Inline object and function: new reference every render
  return <ChildMemo data={{ id: 1 }} onChange={() => {}} />;
  // ChildMemo re-renders every time -- memo is useless
};
```

## React.memo and Props from Props

The simplest way to break memoization is passing non-memoized values through intermediary components, especially when spreading props:

```jsx
const Child = () => {};
const ChildMemo = React.memo(Child);

const Component = (props) => {
  return <ChildMemo {...props} />;  // dangerous spread
};

const ComponentInBetween = (props) => {
  return <Component {...props} />;
};

const InitialComponent = (props) => {
  // data is created inline -- not memoized
  return <ComponentInBetween {...props} data={{ id: '1' }} />;
};
```

The inline `data` object propagates through spreads all the way to `ChildMemo`, breaking its memoization. Nobody adding a prop to `InitialComponent` is going to trace through every intermediate component to check for `React.memo` wrappers.

### Rules for Using React.memo Successfully

**Rule 1**: Never spread props from other components onto a memoized child. Be explicit:

```jsx
// Bad:
const Component = (props) => <ChildMemo {...props} />;

// Good:
const Component = (props) => (
  <ChildMemo some={props.some} other={props.other} />
);
```

**Rule 2**: Avoid passing non-primitive props that come from other components. Even with explicit prop passing, if the parent passes a non-memoized object, memoization breaks.

**Rule 3**: Avoid passing non-primitive values from custom hooks. Custom hooks hide whether their return values have stable references:

```jsx
const Component = () => {
  const { submit } = useForm();
  // Is submit memoized inside useForm? You can't tell from here.
  return <ChildMemo onChange={submit} />;
};

// Inside useForm:
const useForm = () => {
  const submit = () => { /* ... */ };  // new reference every render!
  return { submit };
};
// ChildMemo's memoization is broken.
```

## React.memo and Children

This is one of the most counterintuitive pitfalls. Consider:

```jsx
const ChildMemo = React.memo(Child);

const Component = () => {
  return (
    <ChildMemo>
      <div>Some text here</div>
    </ChildMemo>
  );
};
```

This looks innocent -- a memoized component with no props, rendering a static div. But memoization is **broken**. Remember: the nesting syntax is sugar for `children={<div>Some text here</div>}`. That JSX produces an object, and the object is created inline inside `Component`. New reference on every render. It is the same as:

```jsx
<ChildMemo children={{ type: "div", /* ... */ }} />
```

A memoized component with a non-memoized prop.

To fix it, memoize the children:

```jsx
const Component = () => {
  const content = useMemo(
    () => <div>Some text here</div>,
    [],
  );
  return <ChildMemo>{content}</ChildMemo>;
};
```

The same applies to children as render props:

```jsx
// Broken:
<ChildMemo>{() => <div>Some text here</div>}</ChildMemo>

// Fixed with useCallback:
const Component = () => {
  const content = useCallback(
    () => <div>Some text here</div>,
    [],
  );
  return <ChildMemo>{content}</ChildMemo>;
};
```

### The Double-Memoization Trap

Even when both parent and child are wrapped in `React.memo`, the parent's memoization can break:

```jsx
const ChildMemo = React.memo(Child);
const ParentMemo = React.memo(Parent);

const Component = () => {
  return (
    <ParentMemo>
      <ChildMemo />
    </ParentMemo>
  );
};
```

`<ChildMemo />` is an element object (with `type: ChildMemo`). The object itself is not memoized -- it is created inline inside `Component`. So `ParentMemo` has a `children` prop with a non-memoized object. `ParentMemo` re-renders every time `Component` re-renders.

To fix:

```jsx
const Component = () => {
  const child = useMemo(() => <ChildMemo />, []);
  return <ParentMemo>{child}</ParentMemo>;
};
```

You might not even need `ChildMemo` at that point -- wrapping the element creation in `useMemo` already prevents it from being re-created:

```jsx
const Component = () => {
  const child = useMemo(() => <Child />, []);
  return <ParentMemo>{child}</ParentMemo>;
};
```

## useMemo for Expensive Calculations

Beyond reference preservation, `useMemo` can cache the result of expensive operations. But "expensive" is relative and must be measured:

- **On which device?** Sorting 300 items takes <2ms on a laptop but might take a second on an old Android phone.
- **In what context?** A 100ms regex is fine for a rare button click in settings, but unforgivable on every mouse move.
- **Compared to what?** Sorting 300 items took <2ms, but re-rendering the list elements from that array took >20ms. The re-renders are the bottleneck, not the calculation.

Rules of thumb:

1. **Measure first**. Do not guess. Profile on representative devices.
2. **Compare to rendering cost**. If memoizing saves 10ms on calculation but rendering still takes 200ms, the memoization is not the right fix.
3. **useMemo only helps on re-renders**. On the first mount, it does extra work (caching). If the component never re-renders, `useMemo` is pure overhead.
4. **Death by a thousand cuts**. One `useMemo` has negligible overhead. Hundreds scattered across a large app can measurably slow down the initial render.

## The Memoization Hierarchy

Before reaching for `useMemo`/`useCallback`/`React.memo`, try composition-based patterns first. They are simpler, less fragile, and often more effective. The hierarchy from least to most complex:

### 1. Move State Down

If the state update causes everything inside a component to re-render, move the state as close to where it is used as possible:

```jsx
// Before: state at root, entire page re-renders on every keystroke
export default function App() {
  const [search, setSearch] = useState("");
  return (
    <Layout search={search} setSearch={setSearch}>
      <DashboardPage />
      <div>Search results for {search}</div>
    </Layout>
  );
}

// After: state moved to the component that uses it
export const SearchBar = () => {
  const [search, setSearch] = useState('');
  return (
    <>
      <Input value={search} onChange={(e) => setSearch(e.target.value)} />
      <div>Search results for {search}</div>
    </>
  );
};
```

This alone can improve interaction performance by 10x (from ~500ms to ~50ms in a real example).

### 2. Children as Props Pattern

If state cannot be moved down far enough, encapsulate the stateful logic and pass expensive children as props:

```jsx
const LayoutWithSearch = ({ content }) => {
  const [search, setSearch] = useState('');
  return (
    <Layout search={search} setSearch={setSearch}>
      {content}
      <div>Search results for {search}</div>
    </Layout>
  );
};

export default function App() {
  return (
    <LayoutWithSearch content={<DashboardPage />} />
  );
}
```

Or equivalently with the `children` prop:

```jsx
export default function App() {
  return (
    <LayoutWithSearch>
      <DashboardPage />
    </LayoutWithSearch>
  );
}
```

`DashboardPage` is created in `App`, not in `LayoutWithSearch`. When search state changes, `DashboardPage` does not re-render.

### 3. Context / External State Management

If you need to avoid prop drilling entirely while keeping state accessible in distant components, use React Context or libraries like Zustand/Redux:

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
      <Layout>
        <DashboardPage />
      </Layout>
    </DataProvider>
  );
}
```

Only components that call `useSearch()` re-render when the search changes. Everything passed as `children` is unaffected.

### 4. Memoization (Last Resort)

When none of the above is feasible (often due to legacy code or complex prop dependencies), use `React.memo` with memoized props:

```jsx
const DashboardPageMemo = memo(DashboardPage);

export default function App() {
  const [search, setSearch] = useState('');
  const dataMemo = useMemo(() => [{ id: 1 }, { id: 2 }], []);
  const onClickMemo = useCallback(() => { console.log('click'); }, []);

  return (
    <Layout search={search} setSearch={setSearch}>
      <DashboardPageMemo data={dataMemo} onClick={onClickMemo} />
      <div>Search results for {search}</div>
    </Layout>
  );
}
```

Remember: if `DashboardPageMemo` has children, those must be memoized too:

```jsx
export default function App() {
  const memoChild = useMemo(() => <ChildComponent />, []);

  return (
    <DashboardPageMemo>{memoChild}</DashboardPageMemo>
  );
}
```

## The React Compiler: Automated Memoization

The React Compiler is a **Babel plugin** (not part of React itself) that automatically transforms component code to add memoization. It supports React 17+.

### What It Does

Given this code:

```jsx
const App = () => {
  return (
    <HeavyComponent
      arrayProp={[1, 2, 3]}
      callbackProp={() => { console.log('Callback happened'); }}
    >
      <ChildComponent />
    </HeavyComponent>
  );
};
```

Manual memoization would require changes in four places (component, array prop, callback prop, children). The Compiler transforms the code to behave equivalently, but using internal caching mechanisms rather than literal `useMemo`/`useCallback` wrappers. For example, it might move stable callbacks outside the component entirely and group related cached values together.

### Performance Impact

From measurements on a real application:

| Metric | Baseline | With Compiler |
|--------|----------|--------------|
| Home page LCP | 1.3s | 1.4s |
| Search INP | 650ms | 50ms |
| Navigation INP | 260-350ms | 260-355ms |

- **Interaction performance** (INP) can improve dramatically for re-render-heavy interactions (13x improvement for search typing).
- **Initial load** (LCP) may worsen slightly due to increased bundle size and JavaScript work.
- **Bundle size** increases modestly (20-40% per chunk in the example, though compressed impact is smaller).

### Limitations

The Compiler cannot catch everything:

- **Your code may be the problem**: If components re-render because they use a hook that triggers state updates (like a routing hook), the Compiler cannot fix that -- the state change is intentional.
- **External libraries are not compiled**: The Compiler only transforms your code. Third-party UI libraries (MUI, Radix, Ant) remain un-memoized unless they run the Compiler themselves.
- **Complex code may defeat it**: When components are large and mix state with conditional rendering, the Compiler may group everything into a single cached variable that depends on the state, effectively making the memoization useless. Simpler, cleaner components are easier for the Compiler to optimize.

In practice, the Compiler catches 40-80% of unnecessary re-renders, with varying degrees of improvement.

## Key Takeaways

- React compares objects, arrays, and functions by reference. Locally declared non-primitives get new references on every re-render.
- The inline function passed to `useMemo` or `useCallback` is always re-created. `useCallback` caches the function; `useMemo` caches its return value.
- **Memoizing props without `React.memo` is an anti-pattern** -- it does nothing to prevent re-renders.
- `React.memo` checks all props by reference before re-rendering. If even one prop has a new reference, the component re-renders normally.
- `children` is a non-primitive prop. Passing JSX children to a `React.memo` component breaks memoization unless the children are also memoized.
- Even `<ChildMemo />` passed as children to `<ParentMemo>` breaks `ParentMemo` -- the element object is not memoized, only the component type is.
- For "expensive calculations", measure first. Re-rendering components is usually an order of magnitude more expensive than the calculations themselves.
- **Prefer composition over memoization**: move state down, use children as props, use context. Only reach for `React.memo` when these fail.
- The React Compiler automates memoization as a Babel plugin but cannot fix architectural issues, external library re-renders, or overly complex components.

## See Also

- [React Re-renders](react-re-renders.md)
- [Component Composition Patterns](component-composition-patterns.md)
- [Reconciliation and Diffing](reconciliation-and-diffing.md)
- [React Context and Performance](react-context-performance.md)
- [React Compiler](react-compiler.md)
