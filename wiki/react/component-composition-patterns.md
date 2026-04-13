# Component Composition Patterns

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

React provides several composition patterns for structuring components beyond simple parent-child nesting. These patterns -- elements as props, children as props, render props, and higher-order components -- solve both performance and flexibility concerns. Many of them prevent unnecessary re-renders without any memoization at all, simply by leveraging how React's reconciliation works under the hood. Understanding the distinction between Elements and Components is the foundation for understanding why these patterns work.

## Elements vs Components

A **Component** is a function that accepts props and returns Elements:

```jsx
const Parent = () => {
  return <Child />;
};
```

An **Element** is the object that JSX produces. The HTML-like syntax is syntactic sugar for `React.createElement`:

```jsx
// These two are equivalent:
<Child />
React.createElement(Child, null, null)
```

Both produce a plain object with a `type` property:

```js
{
  type: Child,
  props: {},
  // ... other internal React stuff
}
```

For DOM elements, the `type` is a string:

```js
// <h1>Some title</h1> becomes:
{
  type: "h1",
  // ... props and internal React stuff
}
```

This distinction matters because **re-rendering is just React calling the component function**. From the return value, React builds a tree of these objects (the "Virtual DOM" or Fiber Tree). It compares the tree before and after re-render using `Object.is` on each element object. If the object reference is the same before and after, React skips re-rendering that component and its children entirely.

When a component has state and re-renders, every element defined locally within that component is re-created (new object, new reference). So `Object.is` returns `false`, and React proceeds with re-rendering:

```jsx
const Parent = () => {
  const [state, setState] = useState();
  // <Child /> is re-created on every render -- new object each time
  return <Child />;
};
```

But if the element comes from **outside** the component (via props), it was created in a different scope. The reference does not change when the receiving component re-renders:

```jsx
const Parent = ({ child }) => {
  const [state, setState] = useState();
  // "child" is the same object reference before and after re-render
  return child;
};

// Somewhere else:
<Parent child={<Child />} />;
```

Here, `<Child />` is created in the parent of `Parent`. When `Parent` re-renders due to `setState`, the `child` prop still points to the exact same object. `Object.is` returns `true`, and React skips re-rendering `Child`.

## Children as Props: Preventing Re-renders

### The Problem

Consider a scrollable area with a scroll-tracking feature and heavy child components:

```jsx
const MainScrollableArea = () => {
  const [position, setPosition] = useState(300);

  const onScroll = (e) => {
    const calculated = getPosition(e.target.scrollTop);
    setPosition(calculated);
  };

  return (
    <div className="scrollable-block" onScroll={onScroll}>
      <MovingBlock position={position} />
      <VerySlowComponent />
      <BunchOfStuff />
      <OtherStuffAlsoComplicated />
    </div>
  );
};
```

Every scroll event triggers `setPosition`, which re-renders `MainScrollableArea` and all its children -- including the slow components. The scrolling is laggy.

We cannot just "move state down" because the `onScroll` handler is attached to the div that wraps everything.

### The Solution: Extract State and Pass Children

Extract the state and scroll logic into its own component, then pass the slow children as props:

```jsx
const ScrollableWithMovingBlock = ({ children }) => {
  const [position, setPosition] = useState(300);

  const onScroll = (e) => {
    const calculated = getPosition(e.target.scrollTop);
    setPosition(calculated);
  };

  return (
    <div className="scrollable-block" onScroll={onScroll}>
      <MovingBlock position={position} />
      {children}
    </div>
  );
};

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

When `setPosition` triggers a re-render of `ScrollableWithMovingBlock`, React compares the `children` prop before and after. The children were created in `App`, not in `ScrollableWithMovingBlock`, so their reference has not changed. React skips re-rendering them. Only `MovingBlock` re-renders (because it is created inside `ScrollableWithMovingBlock`).

### Why `children` Is Just a Prop

The JSX nesting syntax is nothing more than sugar for the `children` prop:

```jsx
<Parent>
  <Child />
</Parent>

// is exactly the same as:
<Parent children={<Child />} />
```

And the resulting object looks like:

```js
{
  type: Parent,
  props: {
    children: {
      type: Child,
      // ...
    },
  },
}
```

This means `children` has exactly the same performance benefits as any other element prop -- whatever is passed through props is not re-created when the receiving component re-renders.

## Elements as Props: Flexible Configuration

### The Problem with Config Props

Imagine a Button component that needs to support icons. You start with `isLoading`, then add `iconName`, `iconColor`, `iconSize`, then support for left/right icons and avatars. The props list explodes:

```jsx
const Button = ({
  isLoading,
  iconLeftName,
  iconLeftColor,
  iconLeftSize,
  isIconLeftAvatar,
  // ... more and more
}) => {
  // no one knows what's happening here
  return ...
};
```

### The Solution: Pass Elements Directly

Pass the icon as a complete element and let the consumer configure it:

```jsx
const Button = ({ icon }) => {
  return <button>Submit {icon}</button>;
};

// Consumers control everything:
<Button icon={<Loading />} />
<Button icon={<Error color="red" />} />
<Button icon={<Warning color="yellow" size="large" />} />
<Button icon={<Avatar />} />
```

This pattern scales to complex components like modal dialogs:

```jsx
const ModalDialog = ({ children, footer }) => {
  return (
    <div className="modal-dialog">
      <div className="content">{children}</div>
      <div className="footer">{footer}</div>
    </div>
  );
};

// One button in footer
<ModalDialog footer={<SubmitButton />}>
  <SomeFormHere />
</ModalDialog>

// Two buttons in footer
<ModalDialog footer={<><SubmitButton /><CancelButton /></>}>
  <SomeFormHere />
</ModalDialog>
```

Or layout components that can render anything:

```jsx
<ThreeColumnsLayout
  leftColumn={<Something />}
  middleColumn={<OtherThing />}
  rightColumn={<SomethingElse />}
/>
```

The component says: "Give me whatever you want, I will put it in the right place."

### Conditional Rendering Is Safe

A common concern: if you create an element before a conditional check, is it "rendered" even when the condition is false?

```jsx
const App = () => {
  const [isDialogOpen, setIsDialogOpen] = useState(false);

  const footer = <Footer />;  // Is this rendered even when dialog is closed?

  return isDialogOpen ? (
    <ModalDialog footer={footer} />
  ) : null;
};
```

No. Creating an element (`<Footer />`) only creates an object in memory. It is cheap. The component only actually renders when it appears in the return value of a component that React processes. In this case, `Footer` renders only when `ModalDialog` renders, which is only when `isDialogOpen` is true.

This is the same principle that makes routing patterns safe:

```jsx
<Route path="/some/path" element={<Page />} />
<Route path="/other/path" element={<OtherPage />} />
```

Both elements are created, but only the matching route's element is actually rendered.

### Default Values with cloneElement

Sometimes the parent component needs to set default props on the elements it receives. For example, a Button should set icon size and color based on the button's own appearance:

```jsx
const Button = ({ appearance, size, icon }) => {
  const defaultIconProps = {
    size: size === 'large' ? 'large' : 'medium',
    color: appearance === 'primary' ? 'white' : 'black',
  };

  const newProps = {
    ...defaultIconProps,
    // Consumer's props override defaults
    ...icon.props,
  };

  const clonedIcon = React.cloneElement(icon, newProps);

  return <button>Submit {clonedIcon}</button>;
};

// Now icons get correct defaults automatically:
<Button appearance="primary" icon={<Loading />} />   // white icon
<Button appearance="secondary" icon={<Loading />} />  // black icon
<Button size="large" icon={<Loading />} />            // large icon

// Consumer can still override:
<Button appearance="secondary" icon={<Loading color="red" />} />
```

**Warning**: Always spread the original icon's props after the defaults (`...icon.props`). If you only pass `defaultIconProps` to `cloneElement`, you silently destroy the consumer's ability to customize the icon. This pattern is fragile -- use it only for simple cases.

## Render Props

### Converting Elements to Functions

When a component needs to pass data or state back to the elements it renders, elements as props are not enough. Render props solve this by replacing element props with functions that return elements:

```jsx
const Button = ({ renderIcon }) => {
  // Call the function where the icon should go
  return <button>Submit {renderIcon()}</button>;
};

<Button renderIcon={() => <HomeIcon />} />
```

The power comes from passing data to the function. Instead of `cloneElement`, pass an object:

```jsx
const Button = ({ appearance, size, renderIcon }) => {
  const defaultIconProps = {
    size: size === 'large' ? 'large' : 'medium',
    color: appearance === 'primary' ? 'white' : 'black',
  };

  return (
    <button>Submit {renderIcon(defaultIconProps)}</button>
  );
};

// Spread defaults, override what you need:
<Button renderIcon={(props) => <HomeIcon {...props} />} />

// Or override specific props:
<Button renderIcon={(props) => (
  <HomeIcon {...props} size="large" color="red" />
)} />

// Or map to a different prop API entirely:
<Button renderIcon={(props) => (
  <HomeIcon fontSize={props.size} style={{ color: props.color }} />
)} />
```

Sharing internal state is straightforward -- merge it into the object:

```jsx
const Button = ({ appearance, size, renderIcon }) => {
  const [isHovered, setIsHovered] = useState(false);

  const iconParams = {
    size: size === 'large' ? 'large' : 'medium',
    color: appearance === 'primary' ? 'white' : 'black',
    isHovered,
  };

  return (
    <button
      onMouseOver={() => setIsHovered(true)}
      onMouseOut={() => setIsHovered(false)}
    >
      Submit {renderIcon(iconParams)}
    </button>
  );
};

// Consumer decides what to do with the hovered state:
<Button renderIcon={(props) =>
  props.isHovered ? <HomeIconHovered {...props} /> : <HomeIcon {...props} />
} />
```

Everything is explicit. No hidden magic overriding props. The data flow is visible and traceable.

### Children as Render Props

Since `children` is just a prop, it can be a function too:

```jsx
const Parent = ({ children }) => {
  return children(someData);
};

// Pretty nesting syntax works:
<Parent>{(data) => <Child data={data} />}</Parent>
```

A practical example: a resize detector that shares window width:

```jsx
const ResizeDetector = ({ children }) => {
  const [width, setWidth] = useState();

  useEffect(() => {
    const listener = () => setWidth(window.innerWidth);
    window.addEventListener("resize", listener);
    return () => window.removeEventListener("resize", listener);
  }, []);

  return children(width);
};

// Usage -- no state management needed in the consumer:
const Layout = () => {
  return (
    <ResizeDetector>
      {(windowWidth) => {
        return windowWidth > 600 ? <WideLayout /> : <NarrowLayout />;
      }}
    </ResizeDetector>
  );
};
```

### Why Hooks Replaced Render Props (But Not Completely)

The resize detector above can be rewritten as a hook:

```jsx
const useResizeDetector = () => {
  const [width, setWidth] = useState();

  useEffect(() => {
    const listener = () => setWidth(window.innerWidth);
    window.addEventListener("resize", listener);
    return () => window.removeEventListener("resize", listener);
  }, []);

  return width;
};

// Much simpler:
const Layout = () => {
  const windowWidth = useResizeDetector();
  return windowWidth > 600 ? <WideLayout /> : <NarrowLayout />;
};
```

Hooks replaced render props for sharing stateful logic in about 99% of cases. But render props are still useful when:

1. **Configuration and flexibility** -- the render prop pattern for passing default props and state to configurable elements (like the Button icon example) remains viable.
2. **Legacy codebases** -- projects started before hooks are full of render props, especially for form validation logic.
3. **Logic tied to a DOM element** -- when you need to attach listeners to a specific DOM element, render props can be cleaner than hooks with refs:

```jsx
const ScrollDetector = ({ children }) => {
  const [scroll, setScroll] = useState();

  return (
    <div onScroll={(e) => setScroll(e.currentTarget.scrollTop)}>
      {children(scroll)}
    </div>
  );
};

const Layout = () => {
  return (
    <ScrollDetector>
      {(scroll) => <>{scroll > 30 ? <SomeBlock /> : null}</>}
    </ScrollDetector>
  );
};
```

Extracting this to a hook would require introducing a ref and passing it around, which is not as straightforward.

## Higher-Order Components

### What Is a Higher-Order Component?

A higher-order component (HOC) is a function that accepts a component as an argument and returns a new component that renders the original with enhanced behavior:

```jsx
const withSomeLogic = (Component) => {
  // return a new component that renders the argument component
  return (props) => <Component {...props} />;
};

// Usage:
const Button = ({ onClick }) => (
  <button onClick={onClick}>Button</button>
);

const ButtonWithSomeLogic = withSomeLogic(Button);
```

The simplest use case is injecting props:

```jsx
const withTheme = (Component) => {
  const theme = isDark ? 'dark' : 'light';
  return (props) => <Component {...props} theme={theme} />;
};

const Button = ({ theme }) => {
  return <button className={theme}>Button</button>;
};

const ButtonWithTheme = withTheme(Button);
```

Before hooks, HOCs were everywhere -- Redux's `connect`, React Router's `withRouter`, etc. Hooks replaced the majority of these use cases. But HOCs are still useful for specific patterns.

### Enhancing Callbacks

Imagine you want to fire logging events whenever a button is clicked, across your entire app. You could add logging inside each button, but that means copy-pasting the same logic to every clickable component.

A HOC encapsulates this cleanly:

```jsx
const withLoggingOnClick = (Component) => {
  return (props) => {
    const onClick = () => {
      console.log('Log on click something');
      props.onClick();  // don't forget to call the original!
    };

    return <Component {...props} onClick={onClick} />;
  };
};

// Apply to any component with an onClick:
const ButtonWithLogging = withLoggingOnClick(Button);
const ListItemWithLogging = withLoggingOnClick(ListItem);
```

No changes needed inside `Button` or `ListItem`. The HOC intercepts the callback and adds behavior.

#### Adding Data

Pass data as a second argument to the HOC function:

```jsx
const withLoggingOnClickWithParams = (Component, params) => {
  return (props) => {
    const onClick = () => {
      console.log('Log on click: ', params.text);
      props.onClick();
    };
    return <Component {...props} onClick={onClick} />;
  };
};

const ButtonWithLogging = withLoggingOnClickWithParams(Button, {
  text: 'button component',
});
```

Or extract data from the resulting component's props for per-instance configuration:

```jsx
const withLoggingOnClickWithProps = (Component) => {
  return ({ logText, ...props }) => {
    const onClick = () => {
      console.log('Log on click: ', logText);
      props.onClick();
    };
    return <Component {...props} onClick={onClick} />;
  };
};

// Usage:
<ButtonWithLogging onClick={callback} logText="this is Page button">
  Click me
</ButtonWithLogging>
```

### Enhancing Lifecycle Events

Since the returned component is a normal React component, you can use hooks inside it:

```jsx
const withLoggingOnMount = (Component) => {
  return (props) => {
    useEffect(() => {
      console.log('log on mount');
    }, []);

    return <Component {...props} />;
  };
};

const withLoggingOnReRender = (Component) => {
  return ({ id, ...props }) => {
    useEffect(() => {
      console.log('log on id change');
    }, [id]);

    return <Component {...props} />;
  };
};
```

### Intercepting DOM Events

HOCs can wrap components in a DOM element that intercepts events. For example, suppressing keyboard events to prevent global shortcuts while a modal is open:

```jsx
const withSuppressKeyPress = (Component) => {
  return (props) => {
    const onKeyPress = (event) => {
      event.stopPropagation();
    };

    return (
      <div onKeyPress={onKeyPress}>
        <Component {...props} />
      </div>
    );
  };
};

const ModalWithSuppressedKeyPress = withSuppressKeyPress(Modal);
const DropdownWithSuppressedKeyPress = withSuppressKeyPress(Dropdown);
```

Any key press event bubbles up through the DOM until it reaches the wrapping div, where it stops. The component developers do not need to know or care about this behavior.

### When HOCs Are Still Useful

While hooks handle most shared-logic use cases more clearly, HOCs remain relevant for:

- **Cross-cutting concerns** that intercept or enhance existing component interfaces (callbacks, DOM events) without modifying the component's code
- **Wrapping third-party components** where you cannot add hooks inside
- **Systematic behavior injection** (logging, analytics, permissions) across many diverse components

## Key Takeaways

- A **Component** is a function; an **Element** is the object it returns. Re-rendering is calling the function.
- Elements passed as props (including `children`) are not re-created when the receiving component re-renders, because they were created in a different scope. This is the cheapest way to prevent re-renders.
- The `children` nesting syntax (`<Parent><Child /></Parent>`) is just syntactic sugar for `<Parent children={<Child />} />`.
- Elements as props solve configuration explosion -- let the consumer compose whatever they need instead of adding config props.
- Creating an element (`<Footer />`) is cheap object creation. It only renders when it appears in a component's return value.
- `cloneElement` can set default props on element props, but the pattern is fragile. Always spread the original props after defaults.
- **Render props** convert element props to functions, enabling the parent to pass data and state back to the consumer. Hooks replaced most render-prop logic, but render props remain useful for DOM-attached logic and flexible configuration.
- **Higher-order components** accept a component and return an enhanced version. They are useful for intercepting callbacks, lifecycle events, and DOM events across multiple components without modifying their code.

## See Also

- [React Re-renders](react-re-renders.md)
- [Memoization in React](memoization.md)
- [Reconciliation and Diffing](reconciliation-and-diffing.md)
- [React Context and Performance](react-context-performance.md)
