---
tags:
  - nodejs
  - design-patterns
---

# Structural Design Patterns

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Structural design patterns define clear relationships between components, enabling flexible and efficient architectures. They focus on how objects are composed and combined to form larger structures while keeping those structures adaptable and reusable.

Three key structural patterns in Node.js:

| Pattern       | Purpose                                           | Interface change         |
| ------------- | ------------------------------------------------- | ------------------------ |
| **Proxy**     | Controls access to an object by standing in for it | Same interface           |
| **Decorator** | Dynamically extends or modifies an object's behavior | Enhanced interface       |
| **Adapter**   | Bridges incompatible interfaces                   | Different interface      |

The critical distinction: **Proxy** provides the same interface as the wrapped object, **Decorator** provides an enhanced interface, and **Adapter** provides a different interface.

See [Creational Design Patterns](creational-design-patterns.md) | See [Behavioral Design Patterns](behavioral-design-patterns.md)

---

## Proxy Pattern

**Problem it solves:** You need to control access to an object -- adding validation, security, caching, lazy initialization, logging, or remote object capabilities -- without altering the object's interface.

A proxy wraps an actual **instance** of the subject (not the class) and preserves its internal state. The proxy and subject share an identical interface, so the client can swap one for the other transparently.

Common use cases:
- **Data validation** -- validate input before forwarding to the subject
- **Security** -- verify authorization before passing requests through
- **Caching** -- serve from an internal cache when data is already available
- **Lazy initialization** -- defer expensive object creation until actually needed
- **Observability** -- intercept and record method invocations
- **Remote objects** -- make a remote object appear local

### Implementation Techniques

#### 1. Object Composition

Create a new object with the same interface that holds a reference to the subject internally. Proxied methods add logic before/after delegating; other methods delegate directly.

```js
class SafeCalculator {
  constructor(calculator) {
    this.calculator = calculator
  }

  // Proxied method -- adds validation
  divide() {
    const divisor = this.calculator.peekValue()
    if (divisor === 0) {
      throw new Error('Division by 0')
    }
    return this.calculator.divide()
  }

  // Delegated methods
  putValue(value) { return this.calculator.putValue(value) }
  getValue()      { return this.calculator.getValue() }
  peekValue()     { return this.calculator.peekValue() }
  clear()         { return this.calculator.clear() }
  multiply()      { return this.calculator.multiply() }
}

const calculator = new StackCalculator()
const safeCalculator = new SafeCalculator(calculator)
```

**Pros:** Safe -- does not mutate the subject. Required when you need lazy initialization (defer creation of the subject until a method is called).

**Cons:** Must manually delegate every method you don't intercept. Tedious for complex classes.

**Tip:** Reduce delegation boilerplate with a loop:

```js
function createSafeCalculator(calculator) {
  const proxy = {
    divide() { /* validation + delegation */ }
  }
  for (const fn of ['putValue', 'getValue', 'peekValue', 'clear', 'multiply']) {
    proxy[fn] = calculator[fn].bind(calculator)
  }
  return proxy
}
```

#### 2. Object Augmentation (Monkey Patching)

Replace the method directly on the subject instance. No delegation needed for untouched methods.

```js
function patchToSafeCalculator(calculator) {
  const divideOrig = calculator.divide
  calculator.divide = () => {
    const divisor = calculator.peekValue()
    if (divisor === 0) {
      throw new Error('Division by 0')
    }
    return divideOrig.apply(calculator)
  }
  return calculator
}
```

**Pros:** Simplest approach -- no need to reimplement untouched methods.

**Cons:** Mutates the subject directly, which causes side effects if the subject is shared across the codebase. Use only when the subject exists in a controlled/private scope.

#### 3. Built-in `Proxy` Object (ES2015)

The native `Proxy` constructor accepts a `target` and a `handler` with **trap methods** (`get`, `set`, `has`, `apply`, `delete`, etc.) that intercept operations on the proxy instance.

```js
const safeCalculatorHandler = {
  get: (target, property) => {
    if (property === 'divide') {
      return () => {
        const divisor = target.peekValue()
        if (divisor === 0) {
          throw new Error('Division by 0')
        }
        return target.divide()
      }
    }
    return target[property] // all other methods/props pass through
  }
}

const calculator = new StackCalculator()
const safeCalculator = new Proxy(calculator, safeCalculatorHandler)

safeCalculator instanceof StackCalculator // true (inherits prototype)
```

**Pros:** No mutation of the subject. Automatic delegation of everything you don't intercept via `target[property]`. Deep language integration -- can intercept property access, `in` checks, deletions, function calls, and more. Revocable proxies available.

**Cons:** Cannot be fully transpiled or polyfilled (trap behavior is runtime-level).

### Comparing the Techniques

| Criteria              | Composition      | Augmentation       | `Proxy` object       |
| --------------------- | ---------------- | ------------------ | -------------------- |
| Mutates subject       | No               | Yes                | No                   |
| Manual delegation     | Yes (all methods) | No                 | No                   |
| Lazy initialization   | Supported        | Not supported      | Not supported (needs existing target) |
| Dynamic interception  | No               | No                 | Yes (traps)          |
| `instanceof` preserved | No              | Yes                | Yes                  |

### Example: Logging Writable Stream

A practical proxy that intercepts `write()` on any Writable stream to log each chunk:

```js
export function createLoggingWritable(writable) {
  return new Proxy(writable, {
    get(target, propKey) {
      if (propKey === 'write') {
        return (...args) => {
          const [chunk] = args
          console.log('Writing', chunk)
          return writable.write(...args)
        }
      }
      return target[propKey]
    }
  })
}

// Usage
import { createWriteStream } from 'node:fs'
const writable = createWriteStream('test.txt')
const writableProxy = createLoggingWritable(writable)
writableProxy.write('First chunk')  // logs: Writing First chunk
writableProxy.write('Second chunk') // logs: Writing Second chunk
writable.write('This is not logged')
writableProxy.end()
```

### Example: Change Observer with Proxy

The Change Observer pattern detects property mutations on an object and notifies observers. The `set` trap is the key mechanism.

```js
export function createObservable(target, observer) {
  return new Proxy(target, {
    set(obj, prop, value) {
      if (value !== obj[prop]) {
        const prev = obj[prop]
        obj[prop] = value
        observer({ prop, prev, curr: value })
      }
      return true
    }
  })
}

// Usage -- auto-updating invoice total
const invoice = { subtotal: 100, discount: 10, tax: 20 }
let total = invoice.subtotal - invoice.discount + invoice.tax // 110

const obsInvoice = createObservable(invoice, ({ prop, prev, curr }) => {
  total = invoice.subtotal - invoice.discount + invoice.tax
  console.log(`TOTAL: ${total} (${prop}: ${prev} -> ${curr})`)
})

obsInvoice.subtotal = 200 // TOTAL: 210 (subtotal: 100 -> 200)
obsInvoice.discount = 20  // TOTAL: 200 (discount: 10 -> 20)
obsInvoice.discount = 20  // no change -- observer not called
obsInvoice.tax = 30       // TOTAL: 210 (tax: 20 -> 30)
```

> This simplified implementation handles single-level properties with one observer. Production implementations would support multiple observers, nested objects, array mutations, and deletions.

### Proxy Pattern in the Wild

- **LoopBack** -- uses Proxy to intercept and enhance method calls on controllers (validation, authentication)
- **Vue.js** -- reimplemented reactive observable properties using the `Proxy` object
- **MobX** -- reactive state management library using `Proxy` for observable state (often paired with React or Vue)

---

## Decorator Pattern

**Problem it solves:** You need to dynamically add **new functionality** to an existing object instance at runtime, without modifying all objects of the same class. Unlike classical inheritance, decoration targets specific instances.

The key difference from Proxy: a decorator **augments** the interface with new methods/capabilities, whereas a proxy only **controls access** through the existing interface.

### Implementation Techniques

The same three techniques used for Proxy apply to Decorator, but the intent differs -- you are adding new methods alongside intercepting existing ones.

#### 1. Composition

```js
class EnhancedCalculator {
  constructor(calculator) {
    this.calculator = calculator
  }

  // New method (decoration)
  add() {
    const addend2 = this.getValue()
    const addend1 = this.getValue()
    const result = addend1 + addend2
    this.putValue(result)
    return result
  }

  // Modified method (proxy-like interception)
  divide() {
    const divisor = this.calculator.peekValue()
    if (divisor === 0) throw new Error('Division by 0')
    return this.calculator.divide()
  }

  // Delegated methods
  putValue(value) { return this.calculator.putValue(value) }
  getValue()      { return this.calculator.getValue() }
  peekValue()     { return this.calculator.peekValue() }
  clear()         { return this.calculator.clear() }
  multiply()      { return this.calculator.multiply() }
}
```

#### 2. Object Decoration (Monkey Patching)

Attach new methods and replace existing ones directly on the instance:

```js
function patchCalculator(calculator) {
  // New method
  calculator.add = () => {
    const addend2 = calculator.getValue()
    const addend1 = calculator.getValue()
    const result = addend1 + addend2
    calculator.putValue(result)
    return result
  }

  // Modified method
  const divideOrig = calculator.divide
  calculator.divide = () => {
    const divisor = calculator.peekValue()
    if (divisor === 0) throw new Error('Division by 0')
    return divideOrig.apply(calculator)
  }

  return calculator
}
```

#### 3. Proxy-Based Decoration

Use the `Proxy` object's `get` trap to expose new methods that don't exist on the target:

```js
const enhancedCalculatorHandler = {
  get(target, property) {
    if (property === 'add') {
      return function add() {
        const addend2 = target.getValue()
        const addend1 = target.getValue()
        const result = addend1 + addend2
        target.putValue(result)
        return result
      }
    }
    if (property === 'divide') {
      return () => {
        const divisor = target.peekValue()
        if (divisor === 0) throw new Error('Division by 0')
        return target.divide()
      }
    }
    return target[property]
  }
}

const enhancedCalculator = new Proxy(calculator, enhancedCalculatorHandler)
```

> **Regular vs arrow functions in proxy handlers:** Regular functions bind `this` to the proxy target; arrow functions inherit `this` from the surrounding lexical scope (the trap itself). This rarely matters when you access the target via the trap parameter, but it can cause subtle bugs if you rely on `this` inside the returned function.

### Example: Level Database Plugin

Level is a modular key-value database ecosystem for Node.js (based on LevelDB from Google). The decorator pattern via **object augmentation** is the idiomatic approach for Level plugins.

A plugin that notifies subscribers when objects matching a pattern are written:

```js
export function levelSubscribe(db) {
  db.subscribe = (pattern, listener) => {
    db.on('write', docs => {
      for (const doc of docs) {
        const match = Object.keys(pattern).every(
          k => pattern[k] === doc.value[k]
        )
        if (match) {
          listener(doc.key, doc.value)
        }
      }
    })
  }
  return db
}

// Usage
import { Level } from 'level'
import { levelSubscribe } from './level-subscribe.js'

const db = new Level('./db', { valueEncoding: 'json' })
levelSubscribe(db) // decorates db with subscribe()

db.subscribe(
  { doctype: 'message', language: 'en' },
  (_k, val) => console.log(val)
)

await db.put('1', { doctype: 'message', text: 'Hi', language: 'en' }) // triggers
await db.put('2', { doctype: 'company', name: 'ACME Co.' })           // no match
```

### ECMAScript Decorator Proposal

The TC39 Decorators proposal (Stage 3) introduces a **declarative syntax** for modifying classes and their members at definition time:

```js
@defineElement("my-class")
class C extends HTMLElement {
  @reactive accessor clicked = false
}
```

This is conceptually different from the Decorator design pattern:
- **Design pattern** -- augments specific object *instances* at runtime
- **ECMAScript proposal** -- extends *class definitions* through class-level modifiers

Frameworks like Angular and NestJS already use this syntax via transpilation. TypeScript also supports it. However, the proposal has not yet reached Stage 4, so caution is recommended for production code -- future spec changes could introduce breaking changes. Node.js built-in type stripping is not sufficient; a full transpilation step is still required.

### Decorator Pattern in the Wild

- **levelgraph** -- adds graph database capabilities to Level
- **search-index** -- adds full-text search capabilities to Level
- **level-ttl** -- adds TTL-based auto-expiry to Level keys
- **json-socket** -- decorates `net.Socket` with JSON send/receive methods
- **Fastify** -- exposes `decorate()` API to enrich server instances with additional functionality accessible across the application

---

## The Line Between Proxy and Decorator

In classical definitions:
- **Proxy** controls access to an existing object -- the interface stays the same
- **Decorator** wraps an object to add new behavior -- the interface is enhanced

In strongly typed languages, the difference is enforced at compile time. In JavaScript, the boundary is blurry because:

1. Both patterns use the same implementation techniques (composition, augmentation, `Proxy` object)
2. JavaScript's dynamic typing does not enforce interface contracts
3. A single wrapper often both intercepts existing methods (proxy behavior) and adds new ones (decorator behavior)

**Practical advice:** Rather than getting caught up in nomenclature, treat proxy and decorator as complementary and sometimes interchangeable tools. Focus on the class of problems they solve as a whole.

---

## Adapter Pattern

**Problem it solves:** You have an object with a certain interface (the **adaptee**), but a consumer expects a **different** interface. The adapter wraps the adaptee and exposes the interface the consumer needs.

Real-world analogy: a USB Type-A to Type-C adapter. The underlying device is unchanged; only the connection point is transformed.

The most common implementation technique is **composition** -- the adapter holds a reference to the adaptee and translates method calls.

### Example: Level Through the Filesystem API

An adapter that makes a Level database accessible through the `node:fs/promises` interface (`readFile` / `writeFile`):

```js
import { resolve } from 'node:path'

export function createFsAdapter(db) {
  return {
    async readFile(filename, options = undefined) {
      const valueEncoding =
        typeof options === 'string' ? options : options?.encoding
      const opt = valueEncoding ? { valueEncoding } : undefined
      const value = await db.get(resolve(filename), opt)
      if (typeof value === 'undefined') {
        const e = new Error(
          `ENOENT: no such file or directory, open '${filename}'`
        )
        e.code = 'ENOENT'
        e.errno = 34
        e.path = filename
        throw e
      }
      return value
    },

    async writeFile(filename, contents, options = undefined) {
      const valueEncoding =
        typeof options === 'string' ? options : options?.encoding
      const opt = valueEncoding ? { valueEncoding } : undefined
      await db.put(resolve(filename), contents, opt)
    }
  }
}
```

Usage -- drop-in replacement for `fs`:

```js
import { Level } from 'level'
import { createFsAdapter } from './fs-adapter.js'

const db = new Level('./db', { valueEncoding: 'binary' })
const fs = createFsAdapter(db)

await fs.writeFile('file.txt', 'Hello!', 'utf8')
const res = await fs.readFile('file.txt', 'utf8')
console.log(res) // Hello!

await fs.readFile('missing.txt') // Error: ENOENT
```

This becomes especially powerful combined with `browser-level`, which lets Level run in the browser -- allowing code that uses `fs`-style operations to work in both Node.js and browser environments.

### Adapter Pattern in the Wild

- **Level storage backends** -- adapters replicate the internal Level API for different storage engines (LevelDB, IndexedDB in browsers, etc.)
- **Prisma ORM** -- supports PostgreSQL, MySQL, MongoDB, and more through adapter-based database drivers
- **level-filesystem** -- a more complete implementation of the `fs` API on top of Level

---

## Summary

| Pattern       | Intent                            | Interface      | Mutates subject? | Key technique          |
| ------------- | --------------------------------- | -------------- | ----------------- | ---------------------- |
| **Proxy**     | Control access to an object       | Same           | Depends on technique | `Proxy` object / composition |
| **Decorator** | Add new functionality to an object | Enhanced       | Depends on technique | Object augmentation    |
| **Adapter**   | Bridge incompatible interfaces    | Different      | No                | Composition            |

All three patterns wrap an existing object, but they differ in intent:
- Use **Proxy** when you need to intercept, validate, cache, or observe operations without changing what the object can do
- Use **Decorator** when you need to add new capabilities to an existing object at runtime
- Use **Adapter** when you need to make an existing object work where a different interface is expected
