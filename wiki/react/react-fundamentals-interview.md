# React Fundamentals — Interview Questions

> Sources: GreatFrontEnd, Unknown
> Raw: [React Interview Questions — Fundamentals](../../raw/react/2026-04-14-react-interview-questions-fundamentals.md)

## Overview

A collection of foundational React interview questions covering what React is and its benefits, the distinction between React Node, React Element, and React Component, and how JSX works under the hood. These questions target core conceptual understanding rather than advanced patterns.

## What is React? Describe the benefits of React

React is an open-source JavaScript library developed by Facebook (Meta) for building user interfaces, particularly single-page applications. It uses a declarative, component-based model where developers build encapsulated UI pieces that manage their own state, and React efficiently handles DOM updates.

### Key Characteristics

- **Declarative**: Describe the desired UI state based on data; React figures out what DOM changes to make.
- **Component-based**: Build UIs from small, reusable, self-contained components that manage their own logic and state.
- **Virtual DOM**: A lightweight in-memory representation of the real DOM that enables selective, efficient updates through a diffing algorithm.
- **JSX**: An optional syntax extension that lets you write HTML-like markup directly in JavaScript.

### Benefits

| Benefit | Why it matters |
|---------|---------------|
| Component-based architecture | Modular, reusable pieces that are easier to maintain, test, and reason about in isolation |
| Virtual DOM and efficient updates | Diffing algorithm identifies minimal DOM changes, reducing costly direct DOM manipulation |
| Large active community | Extensive docs, rich ecosystem of third-party libraries, strong community support |
| Learn once, write anywhere | React Native shares the same mental model for iOS/Android, enabling code reuse across platforms |

## What is the difference between React Node, React Element, and a React Component?

These three concepts form a hierarchy from the most general rendering unit to the most powerful building block.

### React Node

The broadest type — any value React can render:

```javascript
const stringNode = 'Hello, world!';
const numberNode = 123;
const booleanNode = true;   // renders nothing
const nullNode = null;       // renders nothing
const elementNode = <div>Hello</div>;
```

Strings, numbers, booleans, `null`, `undefined`, and React elements are all valid React Nodes.

### React Element

An immutable plain object that describes **what** to render. It carries a type (HTML tag string or component reference), props, and children. Created via JSX or `React.createElement()`:

```jsx
const element = <div className="greeting">Hello, world!</div>;

// compiles to:
const element = React.createElement(
  'div',
  { className: 'greeting' },
  'Hello, world!',
);
```

Elements are cheap to create and are thrown away on every render — React compares old and new element trees to determine what changed (see [Reconciliation and Diffing](reconciliation-and-diffing.md)).

### React Component

A function (or class) that accepts props and returns React elements. Components are reusable and can hold state and side effects.

```jsx
// Function component (modern, preferred)
function Welcome({ name }) {
  return <h1>Hello, {name}</h1>;
}

// Class component
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

### Quick Comparison

| | React Node | React Element | React Component |
|---|---|---|---|
| What is it? | Any renderable value | Immutable object describing UI | Function/class producing elements |
| Examples | `"text"`, `42`, `null`, `<div/>` | `<div className="x">` | `function App() {}` |
| Mutable? | N/A | No (immutable) | Yes (holds state, effects) |
| Reusable? | N/A | No (one-time description) | Yes (call with different props) |

## What is JSX and how does it work?

JSX (JavaScript XML) is a syntax extension that lets you write HTML-like markup inside JavaScript files. It is not valid JavaScript on its own — a compiler (Babel, SWC, or the TypeScript compiler) transforms it into `React.createElement()` calls before the browser sees it.

### Transformation

```jsx
// What you write:
<h1 className="title">Hello, {name}!</h1>

// What the compiler produces:
React.createElement('h1', { className: 'title' }, 'Hello, ', name, '!')
```

With the modern JSX transform (React 17+), the output uses `jsx()` from `react/jsx-runtime` instead, so you no longer need `import React` at the top of every file.

### Key Characteristics

- **Expression embedding**: Curly braces `{}` embed any JavaScript expression — variables, function calls, ternaries.
- **Attribute handling**: Use `className` instead of `class`, `htmlFor` instead of `for`. Pass strings with quotes or expressions with braces.
- **Injection prevention**: React escapes all embedded values before rendering, protecting against XSS attacks by default.
- **Single root rule**: A JSX expression must have one root element (or use `<Fragment>` / `<>`).

### Why JSX?

JSX co-locates markup with logic in a single file, making components easier to read and maintain compared to separating HTML templates from JavaScript. Since JSX is "just JavaScript," you get full IDE support — autocompletion, type checking, and refactoring tools work out of the box.

## See Also

- [Reconciliation and Diffing](reconciliation-and-diffing.md)
- [React Re-renders](react-re-renders.md)
- [Component Composition Patterns](component-composition-patterns.md)
