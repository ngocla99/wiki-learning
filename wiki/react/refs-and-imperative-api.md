# Refs and Imperative API

> Sources: Nadia Makarevich, 2023
> Raw: [Advanced React](../../raw/react/advanced-react-nadia-makarevich.md)

## Overview

Refs in React are mutable containers that persist across re-renders without triggering them. They serve two primary purposes: storing arbitrary data that should not cause visual updates, and accessing real DOM elements directly. When combined with `useImperativeHandle` or manual Ref mutation, they enable imperative APIs that let parent components trigger actions on children (focus, scroll, animate) without resorting to awkward prop-based workarounds. Understanding Refs deeply is a prerequisite for working with closures, debouncing, and advanced DOM measurement patterns.

## What Is a Ref

A Ref is a mutable object created by `useRef()` with a `.current` property. The initial value passed to `useRef` is cached -- it is set once when the component mounts and never re-initialized:

```jsx
const Component = () => {
  const ref = useRef({ id: 'test' });

  useEffect(() => {
    console.log(ref.current); // { id: 'test' }
  });
};
```

The initial value is preserved across re-renders. If you compare `ref.current` between re-renders, the reference is the same -- equivalent to `useMemo` on that object.

Once created, you can assign anything to `ref.current` inside `useEffect` or event handlers:

```jsx
const Component = () => {
  const ref = useRef(null);

  useEffect(() => {
    ref.current = { id: url };
  }, [url]);
};
```

## Ref vs State: Key Differences

| Aspect | Ref | State |
|--------|-----|-------|
| Triggers re-render | No | Yes |
| Update timing | Synchronous (immediate) | Batched/async (scheduled) |
| Mutability | Mutable (`ref.current = x`) | Immutable (via setter function) |
| Persistence | Across re-renders | Across re-renders |
| Accessible in render | Yes, but value may be stale | Yes, always current |

### Ref Updates Do Not Trigger Re-renders

This is the biggest practical difference. If you store a value in a Ref and try to display it in JSX, the display will not update when the Ref changes:

```jsx
const Form = () => {
  const ref = useRef();
  const numberOfLetters = ref.current?.length ?? 0;

  return (
    <>
      <input type="text" onChange={(e) => { ref.current = e.target.value; }} />
      {/* Always shows 0 -- Ref changes don't trigger re-render */}
      Characters count: {numberOfLetters}
    </>
  );
};
```

The count stays at 0 because the component never re-renders. If something unrelated causes a re-render (like opening a modal dialog with its own state), the count will suddenly jump to the current value -- because the Ref did store the latest data, it just never triggered a UI update.

Even more subtly, if you pass `ref.current` as a primitive prop to a child component, the child will never see the updated value:

```jsx
const Form = () => {
  const ref = useRef();
  const onChange = (e) => { ref.current = e.target.value; };

  return (
    <>
      <input type="text" onChange={onChange} />
      {/* SearchResults never receives updated search text */}
      <SearchResults search={ref.current} />
    </>
  );
};
```

Even triggering a re-render inside `SearchResults` (e.g., clicking a button) will not update `search` -- the prop was set during the parent's render, and the parent never re-rendered.

### Ref Updates Are Synchronous and Mutable

State updates are batched and asynchronous. When you call `setState`, the new value is not immediately available:

```jsx
const onChange = (e) => {
  console.log('before', value);  // e.g., "hello"
  setValue(e.target.value);       // schedule update to "hello!"
  console.log('after', value);   // still "hello" -- not updated yet
};
```

Ref updates take effect immediately because you are just mutating a plain JavaScript object:

```jsx
const onChange = (e) => {
  console.log('before', ref.current);  // "hello"
  ref.current = e.target.value;         // mutated immediately
  console.log('after', ref.current);   // "hello!" -- already changed
};
```

## When to Use Ref vs State

Ask two questions about the value you want to store:

1. Is this value used for rendering components, now or in the future?
2. Is this value passed as props to other components, now or in the future?

If the answer to both is "no," then Ref is appropriate. Common use cases:

- **Counting renders** (for debugging):
  ```jsx
  useEffect(() => {
    ref.current = ref.current + 1;
    console.log('Render number', ref.current);
  });
  ```

- **Storing previous state values**:
  ```jsx
  const usePrevious = (value) => {
    const ref = useRef();
    useEffect(() => {
      ref.current = value;
    }, [value]);
    return ref.current;
  };
  ```

- **Timer/interval IDs**: storing `setTimeout` or `setInterval` return values for cleanup
- **DOM element references**: the most common and important use case

## Assigning DOM Elements to Refs

Pass a Ref to a DOM element via the `ref` attribute. After React renders the element and creates the real DOM node, it assigns that node to `ref.current`:

```jsx
const Component = () => {
  const ref = useRef(null);

  useEffect(() => {
    // ref.current is the actual DOM element
    // same as document.getElementById('...')
    console.log(ref.current);
  });

  return <input ref={ref} />;
};
```

**Important**: `ref.current` is only available after the element renders. You must read it inside `useEffect` or event handlers, not during render:

```jsx
const Component = () => {
  const ref = useRef(null);

  // BAD: ref.current is null during first render
  // This blocks the input from ever rendering
  if (!ref.current) return null;

  return <input ref={ref} />;
};
```

### Practical Example: Focusing on Submit

```jsx
const Form = () => {
  const [name, setName] = useState('');
  const inputRef = useRef(null);

  const onSubmitClick = () => {
    if (!name) {
      inputRef.current.focus(); // Focus empty field
    } else {
      // Submit data
    }
  };

  return (
    <>
      <input onChange={(e) => setName(e.target.value)} ref={inputRef} />
      <button onClick={onSubmitClick}>Submit the form!</button>
    </>
  );
};
```

## Passing Refs to Child Components

### As a Regular Prop

Refs are just mutable objects. You can pass them as any named prop:

```jsx
const InputField = ({ inputRef }) => {
  return <input ref={inputRef} />;
};

const Form = () => {
  const inputRef = useRef(null);

  useEffect(() => {
    // The actual <input> DOM element from InputField is here
    console.log(inputRef.current);
  }, []);

  return <InputField inputRef={inputRef} />;
};
```

When React renders the `<input>` inside `InputField`, it mutates the Ref object. Since the same object reference was created in `Form`, the parent now has access to the child's DOM element.

### With forwardRef

The `ref` prop name is reserved in React. On functional components, passing `ref` directly produces a console warning: "Function components cannot be given refs."

To use the `ref` prop specifically, wrap the component in `forwardRef`. This injects the Ref as the second argument to the component function:

```jsx
const InputField = forwardRef((props, ref) => {
  return <input ref={ref} />;
});

// Now the parent can use the standard ref prop
const Form = () => {
  const inputRef = useRef(null);
  return <InputField ref={inputRef} />;
};
```

Whether to use `forwardRef` or a custom prop name is a matter of taste -- the result is identical. `forwardRef` is the conventional approach for library components, while custom prop names (like `innerRef`) are simpler for internal code.

## Imperative API with useImperativeHandle

### The Problem with Exposing Raw DOM

Consider an `InputField` component that supports both focus and shake animations on validation error. The parent `Form` needs to trigger both actions. Exposing the raw DOM element via Ref leaks implementation details -- the Form should not need to know which DOM element is used internally or how shaking is implemented.

### Building a Custom API

`useImperativeHandle` attaches a custom object to a Ref's `.current` property. It takes three arguments:

1. The Ref to attach the API to (from props, `forwardRef`, or created internally)
2. A factory function returning the API object
3. A dependency array (like other hooks)

```jsx
const InputField = ({ apiRef }) => {
  const inputRef = useRef(null);
  const [shouldShake, setShouldShake] = useState(false);

  useImperativeHandle(apiRef, () => ({
    focus: () => {
      inputRef.current.focus();
    },
    shake: () => {
      setShouldShake(true);
    },
  }), []);

  const className = shouldShake ? 'shake-animation' : '';

  return (
    <input
      ref={inputRef}
      className={className}
      onAnimationEnd={() => setShouldShake(false)}
    />
  );
};
```

The parent creates a Ref, passes it to `InputField`, and calls the API:

```jsx
const Form = () => {
  const inputRef = useRef(null);
  const [name, setName] = useState('');

  const onSubmitClick = () => {
    if (!name) {
      inputRef.current.focus();
      inputRef.current.shake();
    } else {
      // Submit data
    }
  };

  return (
    <>
      <InputField label="name" onChange={setName} apiRef={inputRef} />
      <button onClick={onSubmitClick}>Submit the form!</button>
    </>
  );
};
```

The Form does not know (or care) that focus is implemented via a DOM `<input>` element or that shaking uses a CSS animation with state. The implementation is fully encapsulated.

## Imperative API Without useImperativeHandle

`useImperativeHandle` is essentially syntactic sugar. Since Refs are just mutable objects, you can achieve the same result by assigning the API directly in `useEffect`:

```jsx
const InputField = ({ apiRef }) => {
  const inputRef = useRef(null);
  const [shouldShake, setShouldShake] = useState(false);

  useEffect(() => {
    apiRef.current = {
      focus: () => inputRef.current.focus(),
      shake: () => setShouldShake(true),
    };
  }, [apiRef]);

  return (
    <input
      ref={inputRef}
      className={shouldShake ? 'shake-animation' : ''}
      onAnimationEnd={() => setShouldShake(false)}
    />
  );
};
```

This is almost exactly what `useImperativeHandle` does under the hood. Choose whichever feels clearer.

## Common Use Cases for DOM Refs

- **Manual focusing**: Input fields after render, after validation errors, after modal open
- **Click outside detection**: Comparing `event.target` against `ref.current` to detect clicks outside a popup
- **Scrolling to elements**: `ref.current.scrollIntoView()`
- **Size/position measurement**: `ref.current.getBoundingClientRect()` for tooltips, responsive layouts
- **Third-party library integration**: Passing DOM elements to non-React libraries (charts, maps, rich text editors)

## Anti-Patterns

- **Using Refs for values that should trigger re-renders**: If the UI needs to reflect the value, use state
- **Reading `ref.current` during render for conditional logic**: The value may not be assigned yet
- **Overusing imperative APIs**: React is declarative by design. Imperative APIs are an escape hatch -- use them only when the declarative model breaks down (focus, scroll, animation)

## Key Takeaways

- A Ref is a mutable object persisted across re-renders. Updates are synchronous and do not trigger re-renders
- Use Refs for values that need persistence but should not cause visual updates
- Assign Refs to DOM elements via the `ref` attribute to access the real DOM node
- Pass Refs as regular props or use `forwardRef` for the reserved `ref` prop name
- `useImperativeHandle` (or manual Ref mutation in `useEffect`) creates clean imperative APIs that encapsulate implementation details
- Only read/write `ref.current` inside `useEffect` or event handlers, never during render

## See Also

- [Closures in React](closures-in-react.md)
- [Debouncing and Throttling](debouncing-throttling.md)
- [Memoization in React](memoization.md)
- [React Re-renders](react-re-renders.md)
