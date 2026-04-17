# Behavioral Design Patterns

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Behavioral design patterns focus on how objects interact, communicate, and distribute responsibilities. They address questions like "How can we modify parts of an algorithm at runtime?", "How can an object change its behavior based on its state?", and "How can we iterate over a collection without knowing its underlying implementation?". In Node.js and JavaScript, these patterns often look radically different from their classical OOP counterparts due to the dynamic nature of the language, first-class functions, and protocols-over-inheritance approach.

This chapter covers six patterns: Strategy, State, Template, Iterator (including generators and async iterators), Middleware, and Command. The Observer pattern (another key behavioral pattern) is covered in [Callbacks and Events](callbacks-and-events.md).

## Strategy

The Strategy pattern allows a **context** object to adapt its behavior by delegating specific parts of its logic to interchangeable objects called **strategies**. The context defines the common structure of an algorithm, while each strategy encapsulates a variation.

**When to use:** When supporting variations in behavior requires complex conditional logic (lots of `if...else` or `switch`), or when mixing different components of the same family.

### Example: Multi-Format Configuration

```ts
// Define the strategy interface
type FormatStrategy = {
  deserialize: (data: string) => ConfigData
  serialize: (data: ConfigData) => string
}

// The context class
class Config {
  data?: ConfigData
  formatStrategy: FormatStrategy

  constructor(formatStrategy: FormatStrategy) {
    this.formatStrategy = formatStrategy
  }

  async load(filePath: string): Promise<void> {
    this.data = this.formatStrategy.deserialize(
      await readFile(filePath, 'utf-8')
    )
  }

  async save(filePath: string): Promise<void> {
    await writeFile(filePath, this.formatStrategy.serialize(this.data!))
  }
}
```

Concrete strategies for JSON, YAML, TOML:

```ts
export const jsonStrategy: FormatStrategy = {
  deserialize: (data) => JSON.parse(data),
  serialize: (data) => JSON.stringify(data, null, 2),
}

export const yamlStrategy: FormatStrategy = {
  deserialize: (data) => YAML.parse(data),
  serialize: (data) => YAML.stringify(data, { indent: 2 }),
}
```

Usage — same context, different strategies:

```ts
const jsonConfig = new Config(jsonStrategy)
await jsonConfig.load('config.json')

const yamlConfig = new Config(yamlStrategy)
await yamlConfig.load('config.yaml')
```

**In the wild:** Passport.js uses Strategy for 500+ authentication methods (OAuth, local, JWT, etc.).

> Strategy vs Adapter: An adapter does not add behavior — it just exposes functionality under another interface. In Strategy, both the context and the strategy implement parts of the algorithm.

## State

The State pattern is a specialization of Strategy where the **strategy changes depending on the state** of the context. The strategy (called the "state") is dynamic and can change during the lifetime of the context.

### Example: Failsafe Socket

A TCP client that queues data when offline and flushes it when the connection is reestablished:

```js
class FailsafeSocket {
  constructor(options) {
    this.options = options
    this.queue = []
    this.currentState = null
    this.states = {
      offline: new OfflineState(this),
      online: new OnlineState(this),
    }
    this.changeState('offline')
  }

  changeState(state) {
    this.currentState = this.states[state]
    this.currentState.activate()
  }

  send(data) {
    this.currentState.send(data) // delegates to current state
  }
}
```

- **OfflineState**: `send()` queues data; `activate()` retries connection every 1s, transitions to `online` on success.
- **OnlineState**: `send()` queues + flushes immediately; `activate()` flushes queued messages; transitions to `offline` on write failure.

The state transition can be initiated by the context, the client, or the state objects themselves. Having states control their own transitions provides the best decoupling.

**In the wild:** XState — a popular library for creating state machines in JavaScript/TypeScript. Used by Mastra (AI agent framework) to model workflow lifecycle states.

## Template

The Template pattern defines an abstract class with the skeleton of a component, leaving some steps (called **template methods**) undefined. Subclasses fill in the gaps.

**Key difference from Strategy:** Strategy selects behavior dynamically at runtime; Template "bakes" the behavior into the class at definition time via inheritance.

```js
class ConfigTemplate {
  async load(file) {
    this.data = this._deserialize(await readFile(file, 'utf-8'))
  }

  async save(file) {
    await writeFile(file, this._serialize(this.data))
  }

  _serialize() { throw new Error('Must be implemented') }
  _deserialize() { throw new Error('Must be implemented') }
}

class JsonConfig extends ConfigTemplate {
  _deserialize(data) { return JSON.parse(data) }
  _serialize(data) { return JSON.stringify(data, null, 2) }
}
```

**In the wild:** Node.js streams — when extending `Readable`, `Writable`, or `Transform`, you implement template methods like `_read()`, `_write()`, `_transform()`. See [Streams](streams.md).

## Iterator

The Iterator pattern defines a common interface for iterating over elements produced in sequence, regardless of the underlying data structure.

### The Iterator Protocol

An iterator is any object implementing a `next()` method that returns `{ done, value }`:

```js
function createAlphabetIterator() {
  let currCode = 'A'.charCodeAt(0)
  const Z = 'Z'.charCodeAt(0)
  return {
    next() {
      if (currCode > Z) return { done: true }
      return { value: String.fromCodePoint(currCode++), done: false }
    }
  }
}
```

### The Iterable Protocol

An iterable implements `[Symbol.iterator]()` returning an iterator. This unlocks `for...of`, spread `[...]`, destructuring, and built-in APIs like `Map()`, `Set()`, `Promise.all()`, `Array.from()`.

```js
class Matrix {
  constructor(data) { this.data = data }

  [Symbol.iterator]() {
    let nextRow = 0, nextCol = 0
    return {
      next: () => {
        if (nextRow === this.data.length) return { done: true }
        const val = this.data[nextRow][nextCol]
        if (nextCol === this.data[nextRow].length - 1) {
          nextRow++; nextCol = 0
        } else { nextCol++ }
        return { value: val }
      }
    }
  }
}

// Now works with for...of, spread, destructuring
for (const cell of new Matrix([[1, 2], [3, 4]])) { console.log(cell) }
```

**Tip:** Make iterators also iterable by adding `[Symbol.iterator]() { return this }`.

### Iterator Utilities (Lazy Evaluation)

Extending the `Iterator` prototype gives access to lazy helper methods — `map()`, `filter()`, `reduce()`, `toArray()`:

```js
class RangeIterator extends Iterator {
  #current; #start; #end; #step
  constructor(start, end, step = 1) {
    super(); this.#start = start; this.#end = end; this.#step = step
  }
  next() {
    this.#current = this.#current === undefined ? this.#start : this.#current + this.#step
    if (this.#step > 0 ? this.#current < this.#end : this.#current > this.#end) {
      return { done: false, value: this.#current }
    }
    return { done: true }
  }
}

// Lazy pipeline — no intermediate arrays
const result = new RangeIterator(0, 10)
  .filter(n => n % 2 === 0)
  .map(n => n * 2)
  .toArray()   // [0, 4, 8, 12, 16]
```

**Key difference from Array methods:** Iterator helpers are **lazy** — they build a processing pipeline and compute values only on demand. Array helpers are **eager** — they process all elements immediately and create intermediate arrays. For large datasets, lazy evaluation avoids memory exhaustion.

### Generators

Generator functions (`function*`) simplify iterator implementation. They are both iterators and iterables:

```js
class Matrix {
  // ... get/set methods
  *[Symbol.iterator]() {
    for (const row of this.data) {
      yield* row  // generator delegation
    }
  }
}
```

Two-way communication with generators:

```js
function* twoWayGenerator() {
  const who = yield null
  yield `Hello ${who}`
}

const gen = twoWayGenerator()
gen.next()                    // start generator
gen.next('world')             // { value: 'Hello world', done: false }
```

### Async Iterators

Async iterators return a promise from `next()`. They implement `[Symbol.asyncIterator]()` and are consumed with `for await...of`:

```js
class CheckUrls {
  constructor(urls) { this.urls = urls }

  async *[Symbol.asyncIterator]() {
    for (const url of this.urls) {
      try {
        const res = await fetch(url, {
          method: 'HEAD', signal: AbortSignal.timeout(5000)
        })
        yield res.ok
          ? `${url} is up, status: ${res.status}`
          : `${url} is down, error: ${res.status}`
      } catch (err) {
        yield `${url} is down, error: ${err.message}`
      }
    }
  }
}

for await (const status of new CheckUrls(['https://example.com'])) {
  console.log(status)
}
```

### Streams vs Async Iterators

| Aspect | Streams | Async Iterators |
|--------|---------|-----------------|
| Model | Push (data flows into buffers) | Pull (consumer requests each chunk) |
| Best for | Binary data, high-throughput, backpressure | Composability, paginated/lazy data |
| Interop | `Readable.from(asyncIterable)` | `for await (const chunk of stream)` |

Node.js `stream.Readable` implements `@@asyncIterator`, so you can use `for await...of` directly on streams. The `pipeline()` helper accepts a mix of streams and async iterables.

## Middleware

The Middleware pattern organizes processing units (functions) into an asynchronous pipeline. Each middleware function processes data and passes it to the next. This is the Node.js incarnation of the Chain of Responsibility / Intercepting Filter pattern.

### Express Middleware

```js
function (req, res, next) { ... }
```

Express middleware handles: body parsing, compression, logging, sessions, CSRF protection, etc. New middleware is registered with `use()`.

### Building a Middleware Framework (ZeroMQ example)

```js
class ZmqMiddlewareManager {
  #socket
  #inboundMiddleware = []
  #outboundMiddleware = []

  constructor(socket) {
    this.#socket = socket
    this.#handleIncomingMessages()
  }

  async #handleIncomingMessages() {
    for await (const [message] of this.#socket) {
      await this.#executeMiddleware(this.#inboundMiddleware, message)
    }
  }

  async send(message) {
    const final = await this.#executeMiddleware(this.#outboundMiddleware, message)
    return this.#socket.send(final)
  }

  use(middleware) {
    if (middleware.inbound) this.#inboundMiddleware.push(middleware.inbound)
    if (middleware.outbound) this.#outboundMiddleware.unshift(middleware.outbound)
  }

  async #executeMiddleware(middlewares, initialMessage) {
    let message = initialMessage
    for await (const fn of middlewares) {
      message = await fn.call(this, message)
    }
    return message
  }
}
```

Key design: outbound middleware is inserted at the **beginning** (via `unshift`), so complementary inbound/outbound pairs execute in inverted order. E.g., inbound: decompress → deserialize; outbound: serialize → compress.

Middleware functions can be sync or async:

```js
// JSON middleware
export function jsonMiddleware() {
  return {
    inbound: (msg) => JSON.parse(msg.toString()),
    outbound: (msg) => Buffer.from(JSON.stringify(msg)),
  }
}
```

**In the wild:** Express, Koa, Middy (AWS Lambda), Hono (multi-runtime web framework with `combine` middleware).

## Command

The Command pattern encapsulates all information necessary to perform an action into an object. Instead of invoking directly, you create an object representing the **intent** to invoke.

### Four Components

| Component | Role |
|-----------|------|
| **Command** | Object encapsulating invocation info |
| **Client** | Creates the command |
| **Invoker** | Executes the command on the target |
| **Target** | The subject of the invocation |

### The Task Pattern (Simplest Form)

```js
function createTask(target, ...args) {
  return () => target(...args)
}
// Equivalent to: target.bind(null, ...args)
```

### Complex Command with Undo and Serialization

```js
function createPostStatusCmd(service, status) {
  let postId = null
  return {
    run() { postId = service.postUpdate(status) },
    undo() {
      if (postId) { service.destroyUpdate(postId); postId = null }
    },
    serialize() { return { type: 'status', action: 'post', status } },
  }
}
```

The Invoker manages command execution, history, delayed execution, undo, and remote dispatch:

```js
class Invoker {
  #history = []

  run(cmd)   { this.#history.push(cmd); cmd.run() }
  undo()     { const cmd = this.#history.pop(); cmd?.undo() }
  delay(cmd, ms) { setTimeout(() => this.run(cmd), ms) }

  async runRemotely(cmd) {
    await fetch('http://localhost:3000/cmd', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(cmd.serialize()),
    })
  }
}
```

**Use cases:** Scheduling, undo/redo, serialization over network (RPC), history tracking, data synchronization, operational transformation (real-time collaboration).

**In the wild:** AWS SDK v3 — every API call is a command object (`PutObjectCommand`, `GetObjectCommand`, etc.) sent via `client.send(command)`.

> Use the full Command pattern only when you need advanced features (undo, serialization, history). For simple deferred execution, the Task pattern (closure/bound function) suffices.

## Pattern Comparison

| Pattern | Purpose | Selection Time | Mechanism |
|---------|---------|---------------|-----------|
| **Strategy** | Swap algorithm parts | Runtime (once) | Composition/delegation |
| **State** | Adapt behavior to state | Runtime (dynamic) | Strategy that changes with state |
| **Template** | Reuse algorithm skeleton | Definition time | Inheritance |
| **Iterator** | Uniform traversal interface | N/A | Protocol (`next()`) |
| **Middleware** | Processing pipeline | Runtime (ordered) | Sequential async chain |
| **Command** | Encapsulate action intent | Runtime | Object with `run()`/`undo()` |

## See Also

- [Creational Design Patterns](creational-design-patterns.md) — Factory, Builder, Singleton, DI
- [Structural Design Patterns](structural-design-patterns.md) — Proxy, Decorator, Adapter
- [Callbacks and Events](callbacks-and-events.md) — Observer pattern (EventEmitter)
- [Streams](streams.md) — Template pattern in stream classes, async iterator interop
