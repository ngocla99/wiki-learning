# React Re-renders

> Sources: Nadia Makarevich, 2023; Nadia Makarevich, 2025
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md); [Web Performance Fundamentals](../../raw/react/web-performance-fundamentals-nadia-makarevich.md)

## Overview

Re-rendering is the process by which React updates already-existing components with new data. It is the foundation of all interactivity in React applications. Probably 90% of all slow interactions in React are caused by either too many re-renders in an interval, or re-renders of too much at the same time, or both. Understanding how re-renders are triggered, how they propagate through the component tree, and what their performance implications are is essential for building performant React apps. State updates are the sole initial trigger for all re-renders.

## Component Lifecycle Stages

A component goes through three key lifecycle stages:

- **Mounting**: When a component first appears on the screen. React creates the component's instance for the first time, initializes its state, runs its hooks, creates or hydrates DOM nodes, and appends elements to the DOM. This is the most expensive operation.
- **Unmounting**: When React detects a component is no longer needed. It does final cleanup, destroys the component's instance and everything associated with it (state, DOM elements), and removes it from the document.
- **Re-rendering**: When React updates an already existing component with new information. Compared to mounting, re-rendering is lightweight -- React reuses the existing instance, re-runs hooks, performs calculations, and updates the existing DOM element with new attributes.

## How Re-renders Are Triggered

Every re-render starts with a state update. The one and only way to initially trigger a re-render in React is to update the state. This may take the form of many different APIs -- `setState` from `useState`, `useSyncExternalStore`, `useReducer`, or any external state management library. Underneath all of them, there is a state update.

```jsx
const App = () => {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <Button onClick={() => setIsOpen(true)}>
      Open dialog
    </Button>
  );
};
```

When the Button is clicked, `setIsOpen` triggers a state update. The `App` component that holds that state re-renders itself.

**Changing a local variable does not trigger a re-render.** If you try to update the UI by mutating a local variable without going through state, nothing happens:

```jsx
const App = () => {
  let isOpen = false; // local variable, NOT state

  return (
    <div>
      <Button onClick={() => (isOpen = true)}>Open dialog</Button>
      {/* Will never show up -- React lifecycle is not triggered */}
      {isOpen ? <ModalDialog /> : null}
    </div>
  );
};
```

## How Re-renders Propagate

After the component that triggered the state update is re-rendered, React needs to distribute the changes to all other components that might depend on the changed data. It does this automatically: it grabs all the components that the initial component renders inside, re-renders those, then re-renders components nested inside of them, and so on until it reaches the end of the chain.

If you imagine a typical React app as a tree, **everything down from where the state update was initiated will be re-rendered**:

```jsx
const App = () => {
  const [isOpen, setIsOpen] = useState(false);

  // ALL of these re-render when isOpen changes
  return (
    <div className="layout">
      <Button onClick={() => setIsOpen(true)}>Open dialog</Button>
      {isOpen ? <ModalDialog onClose={() => setIsOpen(false)} /> : null}
      <VerySlowComponent />
      <BunchOfStuff />
      <OtherStuffAlsoComplicated />
    </div>
  );
};
```

Key rules of re-render propagation:

- **State update is the only initial source of re-renders.**
- **React never goes "up" the render tree** -- only "down" from the state update origin.
- **All nested components re-render**, even those with no props.
- The only way for "bottom" components to affect "top" components is by explicitly calling a state update in the top components or by passing components as functions.
- **Anything that comes from props is "higher" in the hierarchy** and is not affected by the child's re-renders.

## The Big Re-renders Myth

A widespread misconception is that "a component re-renders when its props change." This is **not true** in normal React behavior.

Normal React behavior is: if a state update is triggered, React will re-render all nested components **regardless of their props**. And if a state update is _not_ triggered, then changing props will be just "swallowed" -- React doesn't monitor props independently.

Props only matter for re-render prevention in one case: when the component is wrapped in `React.memo`. Then and only then will React stop its natural chain of re-renders and first check the props. If none of the props change, re-renders stop there. If even one single prop changes, they continue as usual.

## Preventing Unnecessary Re-renders

### Moving State Down

When state is needed only by a small part of the component tree, extract that state and the components that depend on it into a separate child component. This confines re-renders to only the new smaller component:

```jsx
// BEFORE: entire App re-renders on dialog open/close
const App = () => {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <div className="layout">
      <Button onClick={() => setIsOpen(true)}>Open dialog</Button>
      {isOpen ? <ModalDialog onClose={() => setIsOpen(false)} /> : null}
      <VerySlowComponent />
      <BunchOfStuff />
      <OtherStuffAlsoComplicated />
    </div>
  );
};

// AFTER: only ButtonWithModalDialog re-renders
const ButtonWithModalDialog = () => {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <>
      <Button onClick={() => setIsOpen(true)}>Open dialog</Button>
      {isOpen ? <ModalDialog onClose={() => setIsOpen(false)} /> : null}
    </>
  );
};

const App = () => {
  return (
    <div className="layout">
      <ButtonWithModalDialog />
      <VerySlowComponent />
      <BunchOfStuff />
      <OtherStuffAlsoComplicated />
    </div>
  );
};
```

A good indication of where this pattern applies: a few layers of components that just pass state around rather than utilizing it. In real-world measurements, moving state down from a root component to the component that actually needs it can improve interaction performance by 10x (e.g., from ~500ms to ~50ms).

### Components as Children / Props

When state cannot be fully moved down because it is attached to a wrapping element (e.g., `onScroll` on a div), you can pass the slow components as `children` or props. Elements passed as props are created in the parent scope and are not affected by the child's re-renders:

```jsx
// The state and onScroll handler are isolated here
const ScrollableWithMovingBlock = ({ children }) => {
  const [position, setPosition] = useState(300);
  const onScroll = (e) => {
    const calculated = getPosition(e.target.scrollTop);
    setPosition(calculated);
  };

  return (
    <div className="scrollable-block" onScroll={onScroll}>
      <MovingBlock position={position} />
      {children} {/* These won't re-render on scroll! */}
    </div>
  );
};

// VerySlowComponent et al. won't re-render on scroll
const App = () => {
  return (
    <ScrollableWithMovingBlock>
      <VerySlowComponent />
      <BunchOfStuff />
      <OtherStuffAlsoComplicated />
    </ScrollableWithMovingBlock>
  );
};
```

This works because `children` is created in `App`'s scope. When `ScrollableWithMovingBlock` re-renders due to its own state update, React compares element objects before and after using `Object.is`. Since `children` is the same reference (created outside the re-rendering component), React skips re-rendering it.

### Avoiding Props Drilling with Context

When you need state at the root but want to avoid re-rendering the entire page, you can use React Context (or external state management like Zustand/Redux) to bypass layers of components:

```jsx
const Context = createContext({ search: '', setSearch: () => {} });
const useSearch = () => useContext(Context);

const DataProvider = ({ children }) => {
  const [search, setSearch] = useState('');
  const value = useMemo(() => ({ search, setSearch }), [search, setSearch]);
  return <Context.Provider value={value}>{children}</Context.Provider>;
};

export default function App() {
  return (
    <DataProvider>
      <AppLayout>
        <DashboardPage /> {/* Won't re-render on search state change */}
      </AppLayout>
    </DataProvider>
  );
}
```

The `children` prop is not affected by the state change inside `DataProvider`. Only the components that actually consume the context via `useSearch()` will re-render.

### Memoization with React.memo

`React.memo` is a higher-order component that wraps a component and skips re-rendering if its props haven't changed (by reference). This is the **only** mechanism that makes props relevant for re-render decisions. Use it as a last resort after composition patterns have been exhausted. See [Memoization in React](memoization.md) for comprehensive details.

## The Danger of Custom Hooks

State update in a hook triggers a re-render of the component using that hook, **even if the state itself is not used** in the component's render output. Hooks are just "pockets in your trousers" -- if you put a 10-kilogram dumbbell in your pocket instead of carrying it in your hands, it doesn't change the fact that it's still hard to run.

```jsx
const useModalDialog = () => {
  const [isOpen, setIsOpen] = useState(false);
  return { isOpen, open: () => setIsOpen(true), close: () => setIsOpen(false) };
};

const App = () => {
  // State is hidden in the hook, but App still re-renders when it changes!
  const { isOpen, open, close } = useModalDialog();
  return (
    <div className="layout">
      <Button onClick={open}>Open dialog</Button>
      {isOpen ? <ModalDialog onClose={close} /> : null}
      <VerySlowComponent /> {/* Still re-renders! */}
    </div>
  );
};
```

### Chains of Hooks

The danger compounds with hooks that use other hooks. Any state update within the chain triggers a re-render of the component using the outermost hook:

```jsx
const useResizeDetector = () => {
  const [width, setWidth] = useState(0);
  useEffect(() => {
    const listener = () => setWidth(window.innerWidth);
    window.addEventListener('resize', listener);
    return () => window.removeEventListener('resize', listener);
  }, []);
  return null; // Returns nothing!
};

const useModalDialog = () => {
  useResizeDetector(); // Called but not even used
  const [isOpen, setIsOpen] = useState(false);
  return { isOpen, open: () => setIsOpen(true), close: () => setIsOpen(false) };
};

const App = () => {
  // The entire App re-renders on every window resize!
  // Even though useResizeDetector returns null and is buried two hooks deep
  const { isOpen, open, close } = useModalDialog();
  return // ...same as above
};
```

The fix is always the same: extract the hook and the components that depend on its state into a separate component:

```jsx
const ButtonWithModalDialog = () => {
  const { isOpen, open, close } = useModalDialog();
  return (
    <>
      <Button onClick={open}>Open dialog</Button>
      {isOpen ? <ModalDialog onClose={close} /> : null}
    </>
  );
};
```

## Creating Components Inside Other Components

Never define components inside other components. This is one of the **biggest performance killers** in React:

```jsx
// BAD - Input is re-created as a new function on every render
const Component = () => {
  const Input = () => <input />;
  return <Input />;
};
```

Each render creates a new function reference for `Input`. React sees a different type at the same position and **unmounts/remounts the entire subtree**, destroying all state. Re-mounting takes at least twice as long as a normal re-render. You will see flickering effects, lost state, and lost focus.

## Key Takeaways

- **Re-rendering is how React updates components with new data.** Without re-renders, there is no interactivity.
- **State update is the initial source of all re-renders.** There is no other way.
- **If a component re-renders, all nested components re-render.** Props do not matter unless `React.memo` is used.
- **React never goes "up" the render tree** -- only "down" from the state update origin.
- **Moving state down** is the simplest and most effective technique to reduce unnecessary re-renders.
- **Components as children/props** let you isolate state while keeping slow components unaffected.
- **Context or external state management** can bypass props drilling while keeping the blast radius small.
- **Hooks are transparent to re-renders**: a state update anywhere in a chain of hooks will re-render the component using the outermost hook, even if that state is never used or returned.
- **Never create components inside other components** -- it causes re-mounting on every render.
- **Measure first** before reaching for optimization. Most performance problems come from unnecessary re-renders of too many components, not from slow calculations.

## See Also

- [Memoization in React](memoization.md)
- [Component Composition Patterns](component-composition-patterns.md)
- [Reconciliation and Diffing](reconciliation-and-diffing.md)
