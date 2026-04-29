# JavaScript Fundamentals — Interview Questions

> Sources: MDN, YDKJS, personal experience
> Updated: 2026-04-21

## Overview

Core JavaScript interview questions covering the event loop, closures, variable declarations, function context methods, type coercion, timers, and async patterns. Includes output-based questions commonly used in technical screenings.

---

## Event Loop

### How does the JavaScript event loop work?

JavaScript is single-threaded — it can only do one thing at a time. The event loop is the mechanism that allows it to handle asynchronous operations without blocking.

**Key components:**

| Component | Description |
| --------- | ----------- |
| **Call Stack** | LIFO stack of execution contexts. When a function is called, a frame is pushed; when it returns, the frame is popped |
| **Web APIs** | Browser-provided APIs (setTimeout, fetch, DOM events) that run outside the call stack |
| **Callback Queue (Macro-task)** | Holds callbacks from Web APIs (setTimeout, setInterval, I/O) |
| **Microtask Queue** | Holds Promise callbacks (`.then`, `.catch`) and `queueMicrotask()` |
| **Event Loop** | Continuously checks: if the call stack is empty, drain the microtask queue first, then take one macro-task |

**Priority order:** Call Stack → Microtask Queue (all) → Macro-task Queue (one at a time)

```js
console.log('1');

setTimeout(() => console.log('2'), 0);

Promise.resolve().then(() => console.log('3'));

console.log('4');

// Output: 1, 4, 3, 2
// Reason: sync runs first (1, 4), then microtask (3), then macro-task (2)
```

## Closures and Memory Management

### What is a closure?

A closure is a function that retains access to its **lexical scope** (the variables of its outer function) even after the outer function has finished executing.

```js
function makeCounter() {
  let count = 0; // persists in the closure
  return {
    increment() { count++; },
    decrement() { count--; },
    value() { return count; },
  };
}

const counter = makeCounter();
counter.increment();
counter.increment();
console.log(counter.value()); // 2
```

**Common use cases:** data encapsulation, factory functions, memoization, event handlers with captured state.

### How can closures cause memory leaks?

A closure keeps its outer scope alive. If the closure is long-lived (attached to a DOM node, stored globally, set in a timer), it prevents garbage collection of those variables.

```js
// LEAK: listener holds reference to largeData forever
function setup() {
  const largeData = new Array(1_000_000).fill('x');
  document.getElementById('btn').addEventListener('click', () => {
    console.log(largeData.length); // largeData is captured
  });
}

// FIX: remove listener when no longer needed
function setup() {
  const largeData = new Array(1_000_000).fill('x');
  const handler = () => console.log(largeData.length);
  const btn = document.getElementById('btn');
  btn.addEventListener('click', handler);
  return () => btn.removeEventListener('click', handler); // cleanup fn
}
```

**Detection:** Chrome DevTools → Memory tab → Heap Snapshot. Compare snapshots and look for detached DOM trees or growing retained sizes.

---

## let vs var vs const

| Feature | `var` | `let` | `const` |
| ------- | ----- | ----- | ------- |
| Scope | Function | Block | Block |
| Hoisting | Hoisted + initialized to `undefined` | Hoisted but **not** initialized (TDZ) | Hoisted but **not** initialized (TDZ) |
| Re-declaration | Allowed | Not allowed | Not allowed |
| Re-assignment | Allowed | Allowed | Not allowed |
| Global property | Creates `window.x` | Does not | Does not |

**Temporal Dead Zone (TDZ):** the period between entering a block scope and the variable's declaration being reached. Accessing `let`/`const` in TDZ throws `ReferenceError`.

```js
console.log(a); // undefined (hoisted)
var a = 1;

console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 2;
```

**Rule of thumb:** Default to `const`. Use `let` when the variable needs to be reassigned. Avoid `var`.

---

## call, bind, and apply

All three explicitly set the `this` context for a function. The difference is *when* the function is invoked and *how* arguments are passed.

| Method | Invokes immediately? | Arguments |
| ------ | ------------------- | --------- |
| `call` | Yes | Comma-separated: `fn.call(ctx, a, b)` |
| `apply` | Yes | Array: `fn.apply(ctx, [a, b])` |
| `bind` | No — returns new function | Comma-separated (partial application) |

```js
const user = { name: 'Ngoc' };

function greet(greeting, punctuation) {
  return `${greeting}, ${this.name}${punctuation}`;
}

greet.call(user, 'Hello', '!');       // "Hello, Ngoc!"
greet.apply(user, ['Hello', '!']);    // "Hello, Ngoc!"

const boundGreet = greet.bind(user, 'Hi');
boundGreet('.');                       // "Hi, Ngoc."
```

**Practical uses:**

- `call` — borrowing methods from other objects
- `apply` — spreading arrays into variadic functions (`Math.max.apply(null, arr)`)
- `bind` — fixing `this` in callbacks, partial application

---

## The `this` Keyword

### How does `this` work in JavaScript?

`this` refers to the **execution context** — the object that is currently calling the function. Its value is determined at **call time**, not at definition time (except for arrow functions, which capture `this` lexically).

**5 binding rules (in priority order):**

| Rule                 | How it's triggered                   | Value of `this`                              |
| -------------------- | ------------------------------------ | -------------------------------------------- |
| **new binding**      | `new Fn()`                           | Newly created object                         |
| **Explicit binding** | `.call()`, `.apply()`, `.bind()`     | First argument passed                        |
| **Implicit binding** | Method call: `obj.fn()`              | The object left of the dot (`obj`)           |
| **Default binding**  | Plain function call: `fn()`          | `globalThis` (or `undefined` in strict mode) |
| **Event handler**    | `el.addEventListener('click', fn)`   | The DOM element (`el`)                       |
| **Lexical (arrow)**  | Arrow function                       | Inherited from enclosing scope               |

```js
// 1. new binding
function Person(name) { this.name = name; }
const p = new Person('Ngoc');
console.log(p.name); // "Ngoc" — this = newly created object

// 2. Explicit binding
function greet() { return this.name; }
greet.call({ name: 'Ngoc' }); // "Ngoc"

// 3. Implicit binding
const user = {
  name: 'Ngoc',
  greet() { return this.name; },
};
user.greet(); // "Ngoc" — this = user

// 4. Default binding
function standalone() { return this; }
standalone(); // globalThis (window in browser) — or undefined in strict mode

// 5. Arrow function — lexical this
const obj = {
  name: 'Ngoc',
  greet: () => this.name, // this = enclosing scope (module/global), NOT obj
  greetFixed() {
    const inner = () => this.name; // this = obj (inherited from greetFixed)
    return inner();
  },
};
obj.greet();       // undefined
obj.greetFixed();  // "Ngoc"
```

## async/await and Handling Asynchronous Operations

`async/await` is syntactic sugar over Promises that makes asynchronous code look synchronous.
