# React Hooks — Interview Questions

> Sources: GreatFrontEnd, Unknown
> Raw: [React Interview Questions — Extended](../../raw/react/2026-04-14-react-interview-questions-extended.md)

## Overview

Interview questions covering React Hooks from first principles: why hooks exist, the rules that govern them, and how each built-in hook works and when to reach for it. These questions are high-frequency in frontend interviews at all levels.

---

## What are the benefits of using hooks in React?

Hooks (introduced in React 16.8) let functional components access state and other React features without class components.

| Benefit | Description |
|---------|-------------|
| Simplified state | `useState` adds state without converting to a class |
| Reusable logic | Custom hooks extract shared stateful logic across components |
| Better lifecycle | `useEffect` replaces `componentDidMount/Update/WillUnmount` in one place |
| Separation of concerns | Group related logic together rather than splitting across lifecycle methods |
| Easier testing | Functional components and hooks are simpler to test in isolation |

```jsx
// Reusable custom hook pattern
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);
  useEffect(() => {
    const handler = () => setWidth(window.innerWidth);
    window.addEventListener('resize', handler);
    return () => window.removeEventListener('resize', handler);
  }, []);
  return width;
}
```

---

## What are the rules of React hooks?

Two mandatory rules that React enforces to maintain consistent hook call order across renders.

### Rule 1 — Call hooks at the top level

Only call hooks at the outermost level of a function component — never inside loops, conditions, or nested functions.

```jsx
// CORRECT
function MyComponent() {
  const [count, setCount] = useState(0); // always called
  if (count > 0) { /* logic here, but no hooks */ }
}

// INCORRECT — breaks hook ordering
function MyComponent() {
  if (someCondition) {
    const [count, setCount] = useState(0); // conditional = violates rules
  }
}
```

### Rule 2 — Call hooks only from React functions

Valid contexts: React function components and custom hooks.
Invalid contexts: regular JS functions, class components, event handlers.

### Enforcement

```bash
npm install eslint-plugin-react-hooks --save-dev
```
```json
{
  "rules": {
    "react-hooks/rules-of-hooks": "error",
    "react-hooks/exhaustive-deps": "warn"
  }
}
```

**Why these rules exist**: React identifies hooks by their call order. If that order changes between renders (e.g., a hook skipped inside a condition), React mismatches state to the wrong hook.

---

## What is the purpose of the callback function argument of setState and when should it be used?

Because `setState` (and its hooks equivalent) is asynchronous and React may batch multiple updates, directly referencing `this.state` or a captured state variable can read a stale value.

The functional update form receives the **actual latest state** as its argument, bypassing stale closure issues.

```jsx
// WRONG — may use stale state in batched updates
setCount(count + 1);

// CORRECT — always increments from the real latest value
setCount((prev) => prev + 1);
```

### Class component equivalent
```javascript
this.setState((prevState, props) => ({
  counter: prevState.counter + props.step,
}));
```

### When to use it
- Any time new state depends on the previous value
- Inside async callbacks (setTimeout, promises) where closures might be stale
- When multiple state updates are batched together

---

## What does the dependency array of useEffect affect?

The dependency array is the second argument to `useEffect`. It controls **when** the effect re-runs.

| Dependency Array | Behavior |
|-----------------|----------|
| Omitted | Runs after **every** render |
| `[]` (empty) | Runs once after initial render only |
| `[a, b]` | Runs after initial render, then whenever `a` or `b` changes |

```jsx
useEffect(() => { /* runs every render */ });

useEffect(() => { /* runs once — like componentDidMount */ }, []);

useEffect(() => {
  fetchUser(userId);
}, [userId]); // re-fetches when userId changes
```

### Common pitfalls
- **Stale closures**: Omitting a dependency that the effect reads → effect uses an outdated value
- **Infinite loops**: Including an object/array created inline → new reference every render triggers the effect every render
- **Functions as deps**: Functions created inline recreate each render; wrap with `useCallback` if they must be listed

---

## What is the difference between useEffect and useLayoutEffect?

Both run after a render, but at different points in the browser pipeline.

```
Render → Commit DOM mutations → [useLayoutEffect] → Browser paint → [useEffect]
```

| | `useEffect` | `useLayoutEffect` |
|---|---|---|
| Timing | After browser paint | Before browser paint (synchronous) |
| Blocking | Non-blocking | Blocks paint until complete |
| Default choice | Yes | Only when needed |

### When to use useLayoutEffect

Use it when you need to **read from or write to the DOM before the user sees the frame** — otherwise you'll get a visible flicker:

```jsx
// Measuring element size to reposition a tooltip
useLayoutEffect(() => {
  const rect = tooltipRef.current.getBoundingClientRect();
  setPosition({ top: rect.top - 20 });
}, []);
```

Using `useEffect` here would let the browser paint the wrong position first, then jump — visible as a flash.

**Caution**: `useLayoutEffect` is synchronous and blocking. Heavy work here delays paint and hurts perceived performance. Prefer `useEffect` unless you observe a flicker.

> See also: [useLayoutEffect](use-layout-effect.md)

---

## What is the useRef hook and when should it be used?

`useRef` returns a mutable object `{ current: initialValue }` that persists across renders. Mutating `.current` does **not** trigger a re-render.

### Use case 1 — DOM access

```jsx
function AutoFocusInput() {
  const inputRef = useRef(null);
  useEffect(() => { inputRef.current.focus(); }, []);
  return <input ref={inputRef} />;
}
```

### Use case 2 — Mutable value without re-render

```jsx
function Stopwatch() {
  const intervalId = useRef(null);
  const start = () => { intervalId.current = setInterval(tick, 100); };
  const stop  = () => clearInterval(intervalId.current);
  // ...
}
```

### Use case 3 — Capturing previous state/prop

```jsx
function UsePrevious({ value }) {
  const prev = useRef();
  useEffect(() => { prev.current = value; }, [value]);
  return <span>Was: {prev.current}</span>;
}
```

### State vs Ref

| | `useState` | `useRef` |
|---|---|---|
| Triggers re-render | Yes | No |
| Persists across renders | Yes | Yes |
| Accessed via | `value` | `.current` |

> See also: [Refs and Imperative API](refs-and-imperative-api.md)

---

## What is the useCallback hook and when should it be used?

`useCallback` returns a **memoized function reference** that only changes when its dependencies change.

```jsx
const handleClick = useCallback(() => {
  doSomething(a, b);
}, [a, b]);
```

### When it matters

A function defined inside a component is recreated on every render. If that function is passed as a prop to a child wrapped in `React.memo`, the child re-renders every time because it receives a "new" function reference even though the logic hasn't changed.

```jsx
const Parent = () => {
  const [count, setCount] = useState(0);

  // Without useCallback: new reference every render → ChildComponent always re-renders
  // With useCallback + []: stable reference → ChildComponent skips re-render
  const handleClick = useCallback(() => {
    console.log('clicked');
  }, []);

  return <ChildComponent onClick={handleClick} />;
};

const ChildComponent = React.memo(({ onClick }) => {
  console.log('child rendered');
  return <button onClick={onClick}>Click</button>;
});
```

### Caveats
- Only beneficial when the child is wrapped in `React.memo` (or similar)
- Adds overhead itself — don't apply it everywhere by default
- Wrong/missing dependencies lead to stale closures

> See also: [Memoization in React](memoization.md)

---

## What is the useMemo hook and when should it be used?

`useMemo` caches the **return value** of a function and recomputes it only when dependencies change.

```jsx
const memoizedValue = useMemo(() => expensiveCompute(a, b), [a, b]);
```

### When it matters

1. **Expensive calculations** that would run on every render:
```jsx
const sortedList = useMemo(
  () => [...items].sort((a, b) => b.score - a.score),
  [items]
);
```

2. **Stable object/array references** passed to `React.memo` children:
```jsx
// Without useMemo: new array reference every render → child always re-renders
const config = useMemo(() => ({ theme, lang }), [theme, lang]);
```

### useMemo vs useCallback

| | `useMemo` | `useCallback` |
|---|---|---|
| Memoizes | A **value** | A **function** |
| Equivalent | `useMemo(() => fn, deps)` | `useCallback(fn, deps)` |

### Caveats
- Don't memoize cheap operations — the cache has overhead too
- React may discard the cache (e.g., during development or off-screen work in concurrent mode) — never rely on it for correctness

> See also: [Memoization in React](memoization.md)

---

## What is the useReducer hook and when should it be used?

`useReducer` is an alternative to `useState` for managing **complex state logic**. It follows the reducer pattern (pure function, action → next state) familiar from Redux.

```jsx
const [state, dispatch] = useReducer(reducer, initialState);
```

### Anatomy

```jsx
function reducer(state, action) {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + 1 };
    case 'decrement': return { ...state, count: state.count - 1 };
    case 'reset':     return initialState;
    default: throw new Error(`Unknown action: ${action.type}`);
  }
}

const initialState = { count: 0 };

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);
  return (
    <>
      <p>{state.count}</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </>
  );
}
```

### When to prefer useReducer over useState

| Scenario | Use |
|----------|-----|
| Simple boolean/string/number | `useState` |
| Multiple related values updated together | `useReducer` |
| Next state depends on previous in complex ways | `useReducer` |
| Many event types that affect the same state | `useReducer` |
| Want to extract state logic for testing | `useReducer` |

> See also: [React State Management](state-management.md)

---

## What is the useId hook and when should it be used?

`useId` (React 18+) generates a unique, stable identifier per component instance. It's designed specifically for accessibility attributes that require matching IDs across the DOM.

```jsx
import { useId } from 'react';

function FormField({ label }) {
  const id = useId();
  return (
    <div>
      <label htmlFor={id}>{label}</label>
      <input id={id} type="text" />
    </div>
  );
}

// Safe to render multiple times — each instance gets a unique ID
<FormField label="First name" />
<FormField label="Last name" />
```

### Why not just hardcode an ID?

Hardcoded IDs break when a component renders multiple times (the same ID appears twice in the DOM, which is invalid HTML and breaks screen-reader associations). `useId` guarantees uniqueness across the entire React tree, including SSR (server and client generate the same IDs, preventing hydration mismatches).

### What to avoid
- **Don't use for list keys** — `useId` is for DOM attributes, not list reconciliation. Use data-derived IDs for keys.

## See Also

- [Memoization in React](memoization.md)
- [Refs and Imperative API](refs-and-imperative-api.md)
- [Debouncing and Throttling](debouncing-throttling.md)
- [useLayoutEffect](use-layout-effect.md)
- [Closures in React](closures-in-react.md)
- [React Re-renders](react-re-renders.md)
- [React Hooks — Components & State](react-components-interview.md)
