# React Components & State — Interview Questions

> Sources: GreatFrontEnd, Unknown
> Raw: [React Interview Questions — Extended](../../../raw/react/2026-04-14-react-interview-questions-extended.md)

## Overview

Interview questions covering React's core component and state concepts: props vs state, list keys, controlled vs uncontrolled inputs, fragments, state reset, immutability, and error boundaries.

---

## What is the difference between state and props in React?

State and props are both plain JavaScript objects that influence rendered output, but they serve different roles.

| Aspect | State | Props |
|--------|-------|-------|
| Owner | The component itself | Parent component |
| Mutability | Mutable (via setter) | Read-only in the child |
| Purpose | Internal, dynamic data | External configuration / data passing |
| Re-renders | Triggers re-render when updated | Child re-renders when new props arrive |

### State — owned internally

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

### Props — passed from parent

```jsx
function Greeting({ name }) {
  return <p>Hello, {name}!</p>; // can't change name here
}

function App() {
  return <Greeting name="Alice" />;
}
```

### Passing callbacks as props

A common pattern for child-to-parent communication: the parent passes a state setter (or wrapped function) down as a prop.

```jsx
function Parent() {
  const [value, setValue] = useState('');
  return <Child onChange={setValue} />;
}
function Child({ onChange }) {
  return <input onChange={(e) => onChange(e.target.value)} />;
}
```

---

## What is the purpose of the key prop in React?

`key` is a special prop that helps React identify which items in a list have changed, been added, or been removed. React uses it to match old and new elements during reconciliation.

```jsx
{items.map((item) => (
  <ListItem key={item.id} value={item.value} />
))}
```

### Why it matters

Without stable keys:
- React cannot distinguish between reordered items and new items
- Unnecessary re-renders occur even when data didn't change
- Component state may be incorrectly preserved or lost

### Rules for good keys
- **Unique** among siblings in a list
- **Stable** — same data should always produce the same key
- **Not index** — see next question

> See also: [Reconciliation and Diffing](../reconciliation-and-diffing.md)

---

## What is the consequence of using array indices as key values?

Using index as key breaks when the list is reordered, filtered, or items are inserted/removed. React uses the key to preserve component instances and state — if the key changes what item it refers to, the wrong state gets preserved.

### Concrete bug scenario

```jsx
const todos = ['Buy milk', 'Do laundry', 'Call Alice'];

// User adds a new item at the TOP — now indices shift
// Index 0 now maps to a different item than before
// React incorrectly reuses the old component instance, keeping stale state
{todos.map((todo, index) => (
  <TodoItem key={index} text={todo} />  // BUG: index key
))}
```

**The symptom**: input fields, checkboxes, or component state visually "jumps" to the wrong item after an insertion or reorder.

### When index keys are safe

Index keys are acceptable only when **all three** conditions hold:
1. The list never reorders
2. The list never filters
3. Items have no stateful children (pure display only)

```jsx
// SAFE: static list, display only, never mutated
{['Terms', 'Privacy', 'About'].map((page, i) => (
  <NavLink key={i} href={`/${page}`}>{page}</NavLink>
))}

// CORRECT pattern for dynamic lists
{todos.map((todo) => (
  <TodoItem key={todo.id} text={todo.text} />
))}
```

---

## What is the difference between controlled and uncontrolled React components?

The distinction is about **who owns the form element's state**: React component state (controlled) or the DOM itself (uncontrolled).

### Controlled component

React state is the single source of truth. Every keystroke flows through an event handler and updates state, which then drives the input's displayed value.

```jsx
function ControlledInput() {
  const [value, setValue] = useState('');
  return (
    <input
      value={value}
      onChange={(e) => setValue(e.target.value)}
    />
  );
}
```

**Advantages**: instant validation, conditional enables/disables, formatted input, easy to test.

### Uncontrolled component

The DOM stores the value. React reads it only when needed (e.g., on submit) via a ref.

```jsx
function UncontrolledInput() {
  const inputRef = useRef(null);
  const handleSubmit = () => console.log(inputRef.current.value);
  return <input ref={inputRef} defaultValue="" />;
}
```

**Advantages**: less boilerplate, closer to plain HTML, fine for simple read-on-submit forms.

### When to use which

| Situation | Approach |
|-----------|----------|
| Real-time validation | Controlled |
| Conditional field enabling | Controlled |
| Dynamic form with many fields | Controlled |
| Simple one-field form, read on submit | Either |
| Integrating with non-React DOM libraries | Uncontrolled (ref) |

---

## What are React Fragments used for?

Fragments let you return multiple sibling elements without adding an extra DOM node.

```jsx
// Without Fragment — adds unnecessary <div>
return (
  <div>
    <dt>Term</dt>
    <dd>Description</dd>
  </div>
);

// With Fragment — no extra DOM node
return (
  <>
    <dt>Term</dt>
    <dd>Description</dd>
  </>
);
```

### Why it matters

Some HTML structures require specific parent-child relationships (`<table>/<tr>/<td>`, flex/grid containers). An extra wrapper div breaks these layouts or adds styling noise.

### When to use full `<React.Fragment>` syntax

The shorthand `<>` doesn't accept props. Use `<React.Fragment key={...}>` when rendering a list of fragments that need a `key`:

```jsx
{items.map((item) => (
  <React.Fragment key={item.id}>
    <dt>{item.term}</dt>
    <dd>{item.description}</dd>
  </React.Fragment>
))}
```

---

## How do you reset a component's state in React?

### Option 1 — Set state back to initial value

```jsx
const INITIAL = { count: 0, text: '' };

function MyForm() {
  const [state, setState] = useState(INITIAL);
  return (
    <>
      <input value={state.text} onChange={(e) => setState({ ...state, text: e.target.value })} />
      <button onClick={() => setState(INITIAL)}>Reset</button>
    </>
  );
}
```

For props-dependent initial state, regenerate with a function:

```jsx
const getInitial = () => ({ count: props.start });
const [state, setState] = useState(getInitial);
const reset = () => setState(getInitial());
```

### Option 2 — Force remount via key (the nuclear option)

Changing a component's `key` forces React to unmount the old instance and mount a fresh one. **All state is destroyed**, including deeply nested state you might not control.

```jsx
function Parent() {
  const [key, setKey] = useState(0);
  return (
    <>
      <ComplexForm key={key} />
      <button onClick={() => setKey((k) => k + 1)}>Reset entire form</button>
    </>
  );
}
```

This is a clean escape hatch when option 1 is impractical (e.g., third-party components with internal state).

> See also: [Reconciliation and Diffing](../reconciliation-and-diffing.md)

---

## Why does React recommend against mutating state?

React's rendering engine detects state changes via **reference equality**. Mutating an object in place keeps the same reference, so React sees "same object" → no change detected → no re-render.

### What breaks

```jsx
// WRONG — same reference, React skips re-render
const newItems = items;
newItems.push(newItem);
setItems(newItems); // items === newItems → no update

// CORRECT — new reference, React detects the change
setItems([...items, newItem]);
```

### For nested objects

```jsx
// WRONG
state.user.name = 'Alice';
setState(state); // still same top-level reference

// CORRECT — spread at each level that changed
setState({ ...state, user: { ...state.user, name: 'Alice' } });
// Or use immer for deep updates without manual spreading
```

### Why immutability also enables optimizations

`React.memo`, `useMemo`, `PureComponent`, and `shouldComponentUpdate` all use reference equality checks. Immutable updates give these optimizations a correct signal; mutations silently break them.

---

## What are error boundaries in React for?

Error boundaries are React class components that act as `try/catch` for the component subtree. They catch JavaScript errors during rendering, in lifecycle methods, and in constructors of any child component.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false };

  static getDerivedStateFromError(error) {
    return { hasError: true }; // triggers fallback render
  }

  componentDidCatch(error, info) {
    logErrorToService(error, info.componentStack); // report to Sentry etc.
  }

  render() {
    if (this.state.hasError) {
      return <p>Something went wrong. <button onClick={() => this.setState({ hasError: false })}>Retry</button></p>;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <FeatureWidget />
</ErrorBoundary>
```

### What they do NOT catch

| Scenario | Why | Alternative |
|----------|-----|-------------|
| Event handlers | Not during React rendering | `try/catch` in handler |
| `async/await` / promises | Not in render cycle | `.catch()` / async error state |
| Server-side rendering | Different execution context | Framework-level error handling |
| Errors inside the boundary itself | Can't catch own errors | Wrap in outer boundary |

### Placement strategy

- Wrap each major route/page with its own boundary (isolates failures per page)
- Wrap optional widgets (sidebar, widget panels) so one broken widget doesn't kill the page

> See also: [Error Handling in React](../error-handling.md)

## See Also

- [Reconciliation and Diffing](../reconciliation-and-diffing.md)
- [React Re-renders](../react-re-renders.md)
- [Error Handling in React](../error-handling.md)
- [Refs and Imperative API](../refs-and-imperative-api.md)
- [React Fundamentals — Interview Questions](react-fundamentals-interview.md)
- [React Hooks — Interview Questions](react-hooks-interview.md)
- [React Architecture & Performance — Interview Questions](react-architecture-interview.md)
