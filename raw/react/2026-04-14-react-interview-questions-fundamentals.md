# React Interview Questions — Fundamentals

> Source: https://www.greatfrontend.com/questions/quiz/react-interview-questions
> Collected: 2026-04-14
> Published: Unknown

## What is React? Describe the benefits of React

**Importance:** High | **Difficulty:** Medium

### What is React?

React is an open-source JavaScript library developed by Facebook for building user interfaces, particularly effective for single-page applications. It emphasizes component-based architecture where developers construct encapsulated components managing their own state.

### Key Characteristics

- **Declarative:** You specify desired UI states based on data; React handles DOM updates efficiently.
- **Component-based:** Reusable, modular UI elements managing their own logic.
- **Virtual DOM:** Lightweight in-memory representation enabling selective, efficient updates.
- **JSX:** Optional syntax extension allowing HTML-like code within JavaScript.

### Benefits of React

1. **Component-based architecture:** Encourages modular, reusable components that are maintainable, testable, and reduce unintended side effects from isolated changes.

2. **Virtual DOM and efficient updates:** React uses a diffing algorithm comparing virtual DOM versions to identify minimal necessary changes, significantly improving performance through reduced DOM manipulation.

3. **Large active community:** Extensive documentation, abundant third-party libraries and tools, plus strong community support through forums and meetups.

4. **Learn once, write anywhere:** React Native enables leveraging React knowledge for iOS/Android development, sharing code across web and mobile while managing a single codebase.

---

## What is the difference between React Node, React Element, and a React Component?

A **React Node** represents any renderable unit (element, string, number, or null). A **React Element** is an immutable object describing what to render, created via JSX or `React.createElement()`. A **React Component** is a function or class returning React Elements, enabling reusable UI pieces.

### React Node

A React Node is the most fundamental rendering unit in React's system. It encompasses any value that can be rendered, including:

- React elements
- Strings and numbers
- Booleans
- Null values

```javascript
const stringNode = 'Hello, world!';
const numberNode = 123;
const booleanNode = true;
const nullNode = null;
const elementNode = <div>Hello, world!</div>;
```

### React Element

A React Element is an immutable, plain object representing desired UI output. It contains the element type (HTML tag string or component), props, and children. Elements are created using JSX syntax or the `React.createElement()` method.

```javascript
const element = <div className="greeting">Hello, world!</div>;
// Equivalent to:
const element = React.createElement(
  'div',
  { className: 'greeting' },
  'Hello, world!',
);
```

### React Component

A React Component is a reusable UI piece accepting inputs (props) and returning React elements. Two types exist:

**Function Components** (simpler, modern approach):

```javascript
function Welcome(props) {
  return <h1>Hello, {props.name}</h1>;
}
```

**Class Components** (ES6 classes with `render()` method):

```javascript
class Welcome extends React.Component {
  render() {
    return <h1>Hello, {this.props.name}</h1>;
  }
}
```

### Key Distinctions

- **Scope**: Nodes are renderable values; Elements are objects; Components are reusable functions/classes.
- **Immutability**: Elements are immutable; Components manage state and logic.
- **Purpose**: Nodes describe what renders; Elements specify UI structure; Components enable modularity.

---

## What is JSX and how does it work?

JSX stands for JavaScript XML. It is a syntax extension for JavaScript that allows you to write HTML-like code within JavaScript.

### Transformation Process

JSX requires compilation before browser execution. Tools like Babel convert JSX syntax into regular JavaScript function calls. For example:

```jsx
<h1>Hello, world!</h1>
```

transforms into:

```javascript
React.createElement('h1', null, 'Hello, world!')
```

### Key Characteristics

- **Declarative syntax**: Write HTML-resembling structures directly in JavaScript code.
- **Expression embedding**: Use curly braces `{}` to incorporate JavaScript expressions.
- **Attribute handling**: Support both string literals (with quotes) and embedded expressions (with braces).
- **Injection prevention**: React automatically escapes embedded values before rendering, protecting against injection attacks.

### Core Benefit

JSX makes UI development more intuitive by allowing developers to write what appears as HTML directly within JavaScript files, streamlining component creation workflows.
