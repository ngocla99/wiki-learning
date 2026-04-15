# React Interview Questions — Extended Set

> Source: https://www.greatfrontend.com/questions/quiz/react-interview-questions
> Collected: 2026-04-14
> Published: Unknown

---

## What is the difference between state and props in React?

**State** is component-specific data that can change over time, managed internally and triggering re-renders when modified. **Props** are read-only attributes passed from parent to child components that cannot be altered by recipients.

### State Characteristics
- Local to the component
- Mutable via `setState()` (class) or state setter (hooks)
- Asynchronous updates
- Triggers component re-renders when changed

### State in Class Components
```javascript
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.state = { count: 0 };
  }
  increment = () => {
    this.setState({ count: this.state.count + 1 });
  };
}
```

### State in Functional Components
```javascript
function MyComponent() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

### Props Characteristics
- Passed from parent to child
- Immutable within child component
- Enable component reusability
- Can include functions for parent-child communication

### Props Usage
```javascript
function ParentComponent() {
  return <ChildComponent message="Hello, World!" />;
}
function ChildComponent(props) {
  return <p>{props.message}</p>;
}
```

### Key Differences Summary

| Aspect | State | Props |
|--------|-------|-------|
| Management | Internal to component | External, from parent |
| Mutability | Changeable | Read-only |
| Purpose | Internal data storage | Data passing |
| Re-renders | Triggers when updated | Triggers in child when changed |

---

## What is the purpose of the key prop in React?

The `key` prop uniquely identifies elements in a list. It helps React optimize rendering by efficiently updating and reordering elements. Without a unique `key`, React may re-render elements unnecessarily, leading to performance issues and bugs.

### Why key is Important
1. **Efficient updates**: React tracks which items changed, were added, or removed.
2. **Avoiding bugs**: Without unique keys, React may re-render elements incorrectly.
3. **Performance optimization**: Unique keys minimize DOM operations.

### How to Use the key Prop
```javascript
const items = [
  { id: 1, value: 'Item 1' },
  { id: 2, value: 'Item 2' },
  { id: 3, value: 'Item 3' },
];

function ItemList() {
  return (
    <ul>
      {items.map((item) => (
        <ListItem key={item.id} value={item.value} />
      ))}
    </ul>
  );
}
```

### Common Mistakes
```javascript
// Bad practice: using array index as key
{items.map((item, index) => <ListItem key={index} value={item.value} />)}

// Good practice: using a unique identifier as key
{items.map((item) => <ListItem key={item.id} value={item.value} />)}
```

---

## What is the consequence of using array indices as the value for keys in React?

Using array indices as keys causes performance issues and bugs. React may fail to identify which items have changed when order shifts, resulting in unnecessary re-renders or incorrect component updates.

### Performance Issues
When array indices serve as keys, React cannot efficiently update the DOM. If the array order changes, React loses the ability to correctly identify which items were added, removed, or moved.

### Incorrect Component Updates
When item order changes, React may reuse component instances for different items, passing incorrect state and props.

### Example Problem
```javascript
const items = ['Item 1', 'Item 2', 'Item 3'];
const List = () => (
  <ul>
    {items.map((item, index) => (
      <li key={index}>{item}</li>  // BAD: index changes when items reorder
    ))}
  </ul>
);
```

### Better Approach
```javascript
const items = [
  { id: 1, name: 'Item 1' },
  { id: 2, name: 'Item 2' },
  { id: 3, name: 'Item 3' },
];
const List = () => (
  <ul>
    {items.map((item) => (
      <li key={item.id}>{item.name}</li>  // GOOD: stable unique ID
    ))}
  </ul>
);
```

---

## What is the difference between controlled and uncontrolled React components?

The primary distinction centers on state management approach. Controlled components manage form data through component state (single source of truth). Uncontrolled components store state internally in the DOM and use refs for value access.

### Controlled Components
- React state manages form data
- Changes flow through event handlers
- State acts as the single source of truth
- Better for validation and conditional interactions

```javascript
function ControlledInput() {
  const [value, setValue] = React.useState('');
  return (
    <input
      type="text"
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

### Uncontrolled Components
- DOM manages form data internally
- Refs provide direct element access
- Similar to traditional HTML forms
- Simpler for basic, read-on-submit use cases

```javascript
function UncontrolledInput() {
  const inputRef = React.createRef();
  const handleSubmit = () => alert(inputRef.current.value);
  return <input type="text" ref={inputRef} />;
}
```

**Recommendation**: Choose controlled components for complex interactions (validation, conditional logic); uncontrolled for straightforward scenarios.

---

## What are some pitfalls about using Context in React?

### 1. Performance Issues
When the context value changes, all components that consume the context will re-render, even if they don't use the part of the context that changed.

```javascript
const MyContext = React.createContext();

function ParentComponent() {
  const [value, setValue] = React.useState(0);
  return (
    <MyContext.Provider value={value}>
      <ChildComponent />
    </MyContext.Provider>
  );
}
// ChildComponent re-renders on EVERY value change, even if unrelated
```

### 2. Overusing Context
Context is best for global state that doesn't change frequently (theme, auth status). Overusing it for frequently-updating state leads to tangled, hard-to-maintain code.

### 3. Debugging Difficulties
Context updates can trigger re-renders in many components simultaneously, making it difficult to trace the source of bugs.

### 4. Lack of Fine-Grained Control
Context has no built-in mechanism to prevent re-renders of consumers that don't care about the changed value.

### 5. Alternatives
For complex state, consider Redux, Zustand, or Recoil, which offer fine-grained subscription control.

---

## What are the benefits of using hooks in React?

### Simplified State Management
`useState` adds state to functional components without class conversion:
```javascript
const [count, setCount] = useState(0);
```

### Reusable Logic via Custom Hooks
Extract and share stateful logic across components:
```javascript
function useCustomHook() {
  const [state, setState] = useState(initialState);
  // custom logic
  return [state, setState];
}
```

### Reduced Lifecycle Method Dependency
`useEffect` replaces `componentDidMount`, `componentDidUpdate`, and `componentWillUnmount`:
```javascript
useEffect(() => {
  // side effect
  return () => { /* cleanup */ };
}, [dependencies]);
```

### Enhanced Code Readability
Group related logic together rather than splitting it across lifecycle methods.

### Better Separation of Concerns
Each concern (data fetching, subscriptions, DOM manipulation) can live in its own hook.

### Enhanced Testing
Functional components with hooks are simpler to test; hooks can be tested in isolation.

---

## What are the rules of React hooks?

### Rule 1: Call Hooks at the Top Level
Only call hooks at the outermost scope of your component — never inside loops, conditions, or nested functions.

```javascript
// CORRECT
function MyComponent() {
  const [count, setCount] = useState(0);
  if (count > 0) { /* fine */ }
}

// INCORRECT
function MyComponent() {
  if (someCondition) {
    const [count, setCount] = useState(0); // VIOLATES rules
  }
}
```

### Rule 2: Call Hooks Only from React Functions
Hooks may only be called from React function components or custom hooks — not from regular JavaScript functions or class components.

### Enforcement Tool
Use `eslint-plugin-react-hooks` to automatically catch violations:
```bash
npm install eslint-plugin-react-hooks --save-dev
```
```json
{
  "plugins": ["react-hooks"],
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

---

## What is the difference between useEffect and useLayoutEffect in React?

The primary difference is execution timing:
- **`useEffect`**: Runs asynchronously **after** the DOM has been painted.
- **`useLayoutEffect`**: Runs synchronously **after** DOM mutations but **before** the browser paints.

| Aspect | useEffect | useLayoutEffect |
|--------|-----------|-----------------|
| Execution | Post-paint (async) | Pre-paint (sync) |
| Blocking | Non-blocking | Blocking |
| UI Impact | Doesn't block rendering | Can delay visual updates |

### Typical Use Cases

**useEffect**: Data fetching, subscriptions, logging, event listener setup.

**useLayoutEffect**: Measuring DOM elements (dimensions, positioning), adjusting layout-dependent DOM properties, resolving UI flicker/synchronization issues.

```javascript
useEffect(() => {
  console.log('After paint');
}, []);

useLayoutEffect(() => {
  console.log('Before paint, element width:', ref.current.offsetWidth);
}, []);
```

**Warning**: `useLayoutEffect` is blocking — use it sparingly to avoid degrading perceived performance.

---

## What is the purpose of the callback function argument format of setState in React and when should it be used?

Since React's `setState()` operates asynchronously and batches updates, using a function-based argument ensures you're working with the most recent state rather than a stale snapshot.

### Syntax
```javascript
this.setState((prevState, props) => ({
  counter: prevState.counter + props.increment,
}));
```

### When to Use
- When new state depends on previous state
- When multiple `setState()` calls execute in quick succession (batched)

### Practical Example
```javascript
class Counter extends React.Component {
  state = { counter: 0 };

  // WRONG: may use stale state when batched
  increment = () => this.setState({ counter: this.state.counter + 1 });

  // CORRECT: always uses latest state
  increment = () => this.setState((prevState) => ({
    counter: prevState.counter + 1,
  }));
}
```

In hooks, the same pattern applies:
```javascript
setCount((prev) => prev + 1); // safe even in batched/async contexts
```

---

## What does the dependency array of useEffect affect?

The dependency array determines when the effect re-runs after renders.

### Three Behaviors

**Empty array `[]`** — Runs once after initial render (mimics `componentDidMount`):
```javascript
useEffect(() => {
  // runs once
}, []);
```

**Array with values** — Runs after initial render and whenever listed dependencies change:
```javascript
useEffect(() => {
  // runs when dep1 or dep2 changes
}, [dep1, dep2]);
```

**No array** — Runs after every render (can cause performance issues):
```javascript
useEffect(() => {
  // runs every render
});
```

### Common Pitfalls
- **Stale closures**: Include all state/props the effect uses to avoid accessing outdated values.
- **Functions as dependencies**: Functions recreate on every render; memoize with `useCallback` if needed.

---

## What is the useRef hook in React and when should it be used?

`useRef` creates a mutable object persisting across renders. The returned object's `.current` property maintains its value throughout the component's lifetime.

### Primary Use Cases

**1. Accessing DOM elements directly:**
```javascript
function TextInputWithFocusButton() {
  const inputEl = useRef(null);
  useEffect(() => { inputEl.current.focus(); }, []);
  return <input ref={inputEl} type="text" />;
}
```

**2. Storing mutable values without triggering re-renders:**
```javascript
function Timer() {
  const count = useRef(0);
  const increment = () => {
    count.current += 1;
    console.log(count.current); // doesn't re-render
  };
  return <button onClick={increment}>Increment</button>;
}
```

**3. Tracking previous values:**
```javascript
function Example() {
  const [count, setCount] = useState(0);
  const prevCountRef = useRef();
  useEffect(() => { prevCountRef.current = count; }, [count]);
  return <div>Now: {count}, Before: {prevCountRef.current}</div>;
}
```

**Key distinction**: State updates trigger re-renders; ref updates do not.

---

## What is the useCallback hook in React and when should it be used?

`useCallback` returns a memoized version of a callback function that only changes when its dependencies change.

```javascript
const memoizedCallback = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

### When to Use

**Preventing unnecessary child re-renders** — when passing functions as props to components wrapped with `React.memo`:
```javascript
const ParentComponent = () => {
  const [count, setCount] = useState(0);
  const handleClick = useCallback(() => {
    setCount((c) => c + 1);
  }, []); // stable reference — ChildComponent won't re-render needlessly
  return <ChildComponent onClick={handleClick} />;
};
const ChildComponent = React.memo(({ onClick }) => (
  <button onClick={onClick}>Click me</button>
));
```

### Caveats
- Don't overuse: memoization adds overhead and complexity.
- Incorrect dependencies cause stale closures and bugs.

---

## What is the useMemo hook in React and when should it be used?

`useMemo` caches the result of a computation and recomputes it only when dependencies change.

```javascript
const memoizedValue = useMemo(() => computeExpensiveValue(a, b), [a, b]);
```

### When to Use

**Expensive calculations:**
```javascript
const MyComponent = ({ number }) => {
  const result = useMemo(() => expensiveCalculation(number), [number]);
  return <div>{result}</div>;
};
```

**Stable references for child props:**
```javascript
const MyComponent = ({ items }) => {
  const sortedItems = useMemo(() => [...items].sort((a, b) => a - b), [items]);
  return <ChildComponent sortedItems={sortedItems} />;
};
```

### Caveats
- Overuse creates complexity without guaranteed gains — measure before memoizing.
- Incorrect dependencies cause stale values or unnecessary recalculation.

---

## What is the useReducer hook in React and when should it be used?

`useReducer` manages state as an alternative to `useState`, useful when state logic is complex or the next state depends on the previous one.

```javascript
const [state, dispatch] = useReducer(reducer, initialState);
```

### The Reducer Function
```javascript
function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { count: state.count + 1 };
    case 'decrement': return { count: state.count - 1 };
    default: throw new Error('Unknown action');
  }
}
```

### Practical Example
```javascript
function Counter() {
  const [state, dispatch] = useReducer(reducer, { count: 0 });
  return (
    <div>
      <p>Count: {state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
    </div>
  );
}
```

### When to Use
- Complex state with multiple sub-values
- State updates depending on previous state
- Centralized, predictable state transition logic (Redux-like patterns)

---

## What is the useId hook in React and when should it be used?

`useId` (React 18+) generates a unique, stable identifier for a component instance. It prevents ID conflicts when components render multiple times.

```javascript
import { useId } from 'react';

function MyComponent() {
  const id = useId();
  return (
    <div>
      <label htmlFor={id}>Name:</label>
      <input id={id} type="text" />
    </div>
  );
}
```

### Primary Use Case
Accessibility — linking form labels to inputs with `htmlFor`/`id` pairs without risking duplicate IDs across multiple rendered instances.

### How it Works
The hook generates a unique string automatically, requires no arguments, and does not cause re-renders when used.

---

## What does re-rendering mean in React?

Re-rendering is the process where a component updates its output to the DOM in response to changes in state or props. It keeps the UI synchronized with the underlying data.

### When Re-rendering Occurs
- A component's state changes via `setState` / state setter
- A component receives new props from its parent
- The parent component re-renders (children re-render by default)

### The Re-rendering Process
1. State or props change → React schedules a re-render
2. React calls the component's render function → new virtual DOM tree
3. Diffing algorithm compares new vs old virtual DOM
4. React calculates minimal DOM changes and applies them

### Performance Optimization Tools
- `React.PureComponent` / `React.memo` — shallow comparison to skip unnecessary re-renders
- `useMemo` / `useCallback` — memoize values and functions to stabilize references

---

## What are React Fragments used for?

Fragments group multiple elements without adding extra DOM nodes. They allow returning multiple sibling elements from a component without wrapping them in an unnecessary `<div>`.

### Syntax Options
```jsx
// Shorthand (recommended)
return (
  <>
    <ChildComponent1 />
    <ChildComponent2 />
  </>
);

// Full syntax (required when using key prop in lists)
return (
  <React.Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.description}</dd>
  </React.Fragment>
);
```

### Benefits
- Avoids unnecessary DOM nodes (reduces memory usage and complexity)
- Cleaner, flatter DOM hierarchy
- Useful in flex/grid layouts where wrapper divs would break styling

---

## How do you reset a component's state in React?

Set state back to its initial value using the state setter.

### Functional Components
```javascript
const MyComponent = () => {
  const initialState = { count: 0, text: '' };
  const [state, setState] = useState(initialState);
  const resetState = () => setState(initialState);
  return <button onClick={resetState}>Reset</button>;
};
```

### Dynamic Initial State (props-dependent)
```javascript
const MyComponent = ({ initialCount }) => {
  const getInitialState = () => ({ count: initialCount });
  const [state, setState] = useState(getInitialState);
  const resetState = () => setState(getInitialState());
  return <button onClick={resetState}>Reset</button>;
};
```

### Alternative: Force Unmount/Remount via key
Changing a component's `key` prop forces React to unmount and remount it, resetting all state automatically:
```javascript
<MyComponent key={resetKey} />
// increment resetKey to fully reset the component
```

---

## Why does React recommend against mutating state?

React relies on state immutability to efficiently detect changes and schedule re-renders. Direct mutation breaks this contract.

### Core Problems

**1. Missed re-renders**: React compares state references. Mutating an object doesn't change its reference, so React doesn't detect a change and skips re-rendering.
```javascript
// WRONG: same object reference, React won't re-render
state.value = newValue;
setState(state);

// CORRECT: new object, React detects the change
setState({ ...state, value: newValue });
```

**2. Stale UI**: The DOM falls out of sync with the actual data.

**3. Debugging difficulties**: Tracing when/where state changed becomes hard with mutations spread throughout code.

**4. Unpredictable behavior**: Shared mutable state between components causes unexpected interactions.

---

## What are error boundaries in React for?

Error boundaries are class components that catch JavaScript errors anywhere in their child component tree, log them, and display a fallback UI instead of crashing the entire application.

```javascript
class ErrorBoundary extends Component {
  constructor(props) {
    super(props);
    this.state = { hasError: false };
  }
  static getDerivedStateFromError(error) {
    return { hasError: true };
  }
  componentDidCatch(error, errorInfo) {
    console.error('Error caught:', error, errorInfo);
  }
  render() {
    if (this.state.hasError) return <h1>Something went wrong.</h1>;
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <MyComponent />
</ErrorBoundary>
```

### Important Limitations
Error boundaries do NOT catch errors in:
- Event handlers (use try/catch)
- Asynchronous code
- Server-side rendering
- The boundary component itself

### Best Practices
- Wrap high-level route components
- Log errors to monitoring services (Sentry, etc.)
- Provide user-friendly fallback UI

---

## How do you test React applications?

### Unit Testing with Jest
Jest is the standard testing framework. Use it for isolated component and function tests with mocking support.

### Integration Testing with React Testing Library
Test component interactions from the user's perspective rather than implementation details. Render components, simulate user events (clicks, typing), and assert on DOM output.

### End-to-End Testing with Cypress
Validate full application workflows as real users experience them — browser automation against a running app.

### Snapshot Testing
Capture rendered output and compare against saved snapshots to detect unintended UI changes.

**Best practice**: Test behavior, not implementation. Query by accessible roles/labels rather than CSS classes or internal state.

---

## Explain what React hydration is

Hydration is the process by which server-rendered HTML becomes interactive on the client side. React reuses the existing HTML from SSR rather than re-rendering from scratch, then attaches event handlers and initializes component state.

### Steps
1. Server generates HTML and sends it to the client
2. Browser displays static content immediately (fast FCP/LCP)
3. React "hydrates" the DOM: attaches event listeners, initializes state, without replacing the HTML

```javascript
// Server: ReactDOMServer.renderToString(<App />)
// Client:
ReactDOM.hydrate(<App />, document.getElementById('root'));
// or in React 18+:
ReactDOM.hydrateRoot(document.getElementById('root'), <App />);
```

### Benefits
- Fast initial content display
- Better SEO (pre-rendered HTML is crawlable)
- Reduced client-side rendering work

### Challenges
- **Hydration mismatch**: Server and client render different HTML → React errors and discards server HTML
- **Resource intensive**: Large apps can take time to hydrate, leaving users with non-interactive content (TTI gap)

---

## What are React portals used for?

Portals render children into a DOM node that exists outside the parent component's DOM hierarchy, while preserving React's event bubbling behavior.

```javascript
import ReactDOM from 'react-dom';

const Modal = ({ isOpen, children }) => {
  if (!isOpen) return null;
  return ReactDOM.createPortal(
    <div className="modal">{children}</div>,
    document.getElementById('modal-root'),
  );
};
```

### Primary Use Cases
- **Modals** — escape parent overflow/z-index constraints
- **Tooltips and dropdowns** — correct absolute positioning without parent clipping
- **Notifications/toasts** — fixed-position UI independent of component tree

### Key Property
Even though portals render outside the parent DOM, React events still bubble up through the React component tree as normal. A click in a portal's child reaches the portal's React parent.
