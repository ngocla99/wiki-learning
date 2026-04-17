# Creational Design Patterns

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Creational design patterns address problems related to **object creation**. In JavaScript and Node.js, their implementation differs significantly from classical OO languages due to dynamic typing, first-class functions, closures, and the prototype-based object model. What matters is not the exact class hierarchy, but the underlying idea each pattern solves.

This chapter covers five patterns:

| Pattern | Core Problem It Solves |
|---|---|
| **Factory** | Decouple object creation from implementation |
| **Builder** | Simplify construction of objects with many parameters |
| **Revealing Constructor** | Expose private internals only during creation |
| **Singleton** | Ensure a single shared instance across modules |
| **Dependency Injection** | Wire modules together with loose coupling |

---

## Factory

### Problem

Using `new` directly binds consumer code to a specific class. If the implementation changes (e.g., splitting one class into several subclasses), every call site must be updated. A factory function decouples creation from implementation, giving flexibility almost for free.

### How It Works

A factory is simply a function that returns a new object. It can:

- Return different types based on input or runtime conditions
- Enforce **encapsulation** via closures (private state inaccessible from outside)
- Expose a **smaller surface area** than a class (cannot be extended or monkey-patched)

### Code Example -- Decoupling Creation

```js
function createImage(name) {
  if (name.match(/\.jpe?g$/)) return new ImageJpeg(name)
  if (name.match(/\.gif$/))   return new ImageGif(name)
  if (name.match(/\.png$/))   return new ImagePng(name)
  throw new Error('Unsupported format')
}
```

The caller just writes `createImage('photo.jpeg')` -- it never needs to know which concrete class was instantiated.

### Code Example -- Encapsulation via Closures

```js
function createPerson(name) {
  const privateProperties = {}

  const person = {
    setName(name) {
      if (!name) throw new Error('A person must have a name')
      privateProperties.name = name
    },
    getName() {
      return privateProperties.name
    }
  }

  person.setName(name)
  return person
}
```

`privateProperties` is truly private -- no external code can read or write it directly.

### Code Example -- Code Profiler

A factory can return entirely different implementations based on runtime conditions:

```js
class Profiler {
  constructor(label) {
    this.label = label
    this.lastTime = null
  }
  start() { this.lastTime = process.hrtime() }
  end() {
    const diff = process.hrtime(this.lastTime)
    console.log(`Timer "${this.label}" took ${diff[0]}s and ${diff[1]}ns.`)
  }
}

const noopProfiler = { start() {}, end() {} }

export function createProfiler(label) {
  if (process.env.NODE_ENV === 'production') return noopProfiler
  return new Profiler(label)
}
```

In production, profiling is silently disabled. In development, full profiling runs. The consumer code is identical in both cases:

```js
const profiler = createProfiler('Finding factors')
profiler.start()
// ... work ...
profiler.end()
```

### In the Wild

- **Knex** (`nodejsdp.link/knex`) -- exports a factory function that selects the right SQL dialect based on configuration and returns a configured query builder instance.

> **Other encapsulation techniques** besides closures: private class fields (`#field`), WeakMaps, Symbols, or the underscore `_` convention (not truly private).

---

## Builder

### Problem

When a class constructor takes many parameters (especially optional/interdependent ones), calling it becomes unreadable and error-prone:

```js
const myBoat = new Boat(true, 2, 'Best Motor Co.', 'OM123', true, 1,
                        'fabric', 'white', 'blue', false)
```

Even a configuration-object approach (`new Boat({ hasMotor: true, ... })`) lacks guidance about which parameter groups belong together.

### How It Works

The Builder pattern provides a **fluent interface** with setter methods that:

1. Have descriptive names revealing what is being set
2. Group related parameters (e.g., `withMotors(count, brand, model)`)
3. Return `this` to enable method chaining
4. Finalize construction via a `build()` method that returns a validated, consistent instance

### Code Example -- URL Builder

```js
export class Url {
  constructor(protocol, username, password, hostname,
    port, pathname, search, hash) {
    this.protocol = protocol
    this.username = username
    this.password = password
    this.hostname = hostname
    this.port = port
    this.pathname = pathname
    this.search = search
    this.hash = hash
    this.validate()
  }

  validate() {
    if (!(this.protocol && this.hostname)) {
      throw new Error('Must specify at least a protocol and a hostname')
    }
  }

  toString() {
    let url = `${this.protocol}://`
    if (this.username && this.password) url += `${this.username}:${this.password}@`
    url += this.hostname
    if (this.port) url += this.port
    if (this.pathname) url += this.pathname
    if (this.search) url += `?${this.search}`
    if (this.hash) url += `#${this.hash}`
    return url
  }
}

export class UrlBuilder {
  setProtocol(protocol) { this.protocol = protocol; return this }
  setAuthentication(username, password) {
    this.username = username
    this.password = password
    return this
  }
  setHostname(hostname) { this.hostname = hostname; return this }
  setPort(port) { this.port = port; return this }
  setPathname(pathname) { this.pathname = pathname; return this }
  setSearch(search) { this.search = search; return this }
  setHash(hash) { this.hash = hash; return this }

  build() {
    return new Url(this.protocol, this.username, this.password,
      this.hostname, this.port, this.pathname, this.search, this.hash)
  }
}
```

Usage becomes self-documenting:

```js
const url = new UrlBuilder()
  .setProtocol('https')
  .setAuthentication('user', 'pass')
  .setHostname('example.com')
  .build()

console.log(url.toString())
// => https://user:pass@example.com
```

**Key design decision**: keeping the Builder separate from the target class guarantees that every object returned by `build()` is in a **consistent, validated state**. If the builder were built into `Url` itself, calling `toString()` mid-construction might return invalid output.

### Builder Guidelines

- Break a complex constructor into multiple readable steps
- Group related parameters in a single setter method
- Deduce/default implicit parameters when possible
- Add validation in `build()` to guarantee consistency
- In JS, Builders can also wrap complex **function calls** (use `invoke()` instead of `build()`)

### In the Wild

- **superagent** -- HTTP client with a fluent builder API. The `then()` method acts as the "build" trigger (custom thenable), so `await superagent.post(url).send(data).set('accept', 'json')` both constructs and executes the request.
- **Drizzle ORM** -- SQL query builder: `await db.select().from(posts).leftJoin(comments, eq(posts.id, comments.post_id)).where(eq(posts.id, 10))`. Also uses a custom thenable to trigger execution on `await`.
- **commander.js** -- CLI argument parser using the Builder pattern. Construction finalizes on `parse()`:

```js
import { Command } from 'commander'

const program = new Command()
program
  .name('string-util')
  .description('CLI to some JavaScript string utilities')
  .version('0.8.0')

program
  .command('split')
  .description('Split a string into substrings')
  .argument('<string>', 'string to split')
  .option('-s, --separator <char>', 'separator character', ',')
  .action((str, options) => {
    console.log(str.split(options.separator))
  })

program.parse()
```

- **Knex** -- also provides a query builder alongside its factory.

---

## Revealing Constructor

### Problem

How do you allow an object's internals to be manipulated **only at creation time**, then lock them down? This enables:

- Objects that become **immutable** after construction
- Custom behavior that can only be defined during creation
- One-time initialization guarantees

### How It Works

The constructor takes an **executor function** as input. During construction, the executor receives access to private/internal members. Once the constructor returns, those members are no longer accessible:

```js
//              (1)                (2)         (3)
const obj = new SomeClass(function executor(revealedMembers) {
  // manipulation code -- only runs during construction
})
```

Three elements: **(1)** a constructor, **(2)** an executor function passed to it, **(3)** revealed members passed to the executor.

### Code Example -- Immutable Buffer

```js
const MODIFIER_NAMES = ['swap', 'write', 'fill']

export class ImmutableBuffer {
  constructor(size, executor) {
    const buffer = Buffer.alloc(size)
    const modifiers = {}

    for (const prop in buffer) {
      if (typeof buffer[prop] !== 'function') continue

      if (MODIFIER_NAMES.some(m => prop.startsWith(m))) {
        modifiers[prop] = buffer[prop].bind(buffer)  // revealed to executor
      } else {
        this[prop] = buffer[prop].bind(buffer)        // exposed publicly (read-only)
      }
    }

    executor(modifiers)  // executor can write; after this, modifiers are gone
  }
}
```

Usage:

```js
const hello = 'Hello!'
const immutable = new ImmutableBuffer(hello.length, ({ write }) => {
  write(hello)  // writing is only possible here, during construction
})

console.log(String.fromCharCode(immutable.readInt8(0))) // 'H' -- reading works
// immutable.write('Hello?')  // TypeError: immutable.write is not a function
```

After construction, only read methods are available. The write/fill/swap methods were only accessible inside the executor.

### In the Wild

- **Promise** -- the most well-known use of this pattern. The constructor receives an executor with `resolve` and `reject`. Once created, the Promise state can only be observed, not mutated:

```js
const p = new Promise((resolve, reject) => {
  // resolve/reject are "revealed" -- only available here
})
```

- **Domenic Denicola's revealing event emitter** -- event publishing behavior defined at construction time; cannot be changed afterward (`nodejsdp.link/revealing-ee`).

> The pattern was first identified and named by Domenic Denicola (`nodejsdp.link/domenic-revealing-constructor`).

---

## Singleton

### Problem

Enforce a **single shared instance** of a component across the application -- for sharing state, optimizing resources, or synchronizing access.

### How It Works in Node.js

In Node.js, the module system caches modules by their resolved file path. Exporting an instance from a module effectively creates a singleton within the current package:

```js
// dbInstance.js
import { Database } from './Database.js'

export const dbInstance = new Database('my-app-db', {
  url: 'localhost:5432',
  username: 'user',
  password: 'password'
})
```

Every module that imports `dbInstance` gets the **same reference**:

```js
import { dbInstance } from './dbInstance.js'
```

### Caveat -- Not a True Singleton

The module cache key is the **full file path**. If two packages in `node_modules` each install their own copy of a dependency (e.g., different major versions), each gets a separate module cache entry and therefore a **separate instance**:

```
app/
└── node_modules
    ├── package-a
    │   └── node_modules/mydb   ← instance 1
    └── package-b
        └── node_modules/mydb   ← instance 2 (different!)
```

A true global singleton requires `global.dbInstance = ...`, but this is rarely needed. Compatible versions get hoisted to a single `node_modules` entry, sharing the same instance.

> **Best practice**: if you are authoring a package for third-party use, keep it **stateless** to avoid singleton-related issues.

> **Note**: even hiding the class inside the module does not fully prevent extra instantiation. JavaScript allows `new dbInstance.constructor()` to access the original constructor. The Singleton pattern in JS is more of a convention than an absolute enforcement.

---

## Wiring Modules

Two main strategies for connecting components that depend on each other.

### Singleton Dependencies (Hardcoded)

The simplest approach -- import a singleton directly:

```js
// db.js
export const db = await open({
  filename: join(import.meta.dirname, 'data.sqlite'),
  driver: sqlite3.Database,
})

// blog.js
import { db } from './db.js'

export class Blog {
  initialize() { return db.run('CREATE TABLE IF NOT EXISTS posts (...)') }
  createPost(id, title, content, createdAt) {
    return db.run('INSERT INTO posts VALUES (?, ?, ?, ?)', id, title, content, createdAt)
  }
  getAllPosts() {
    return db.all('SELECT * FROM posts ORDER BY created_at DESC')
  }
}
```

**Pros**: simple, readable, immediate.
**Cons**: tight coupling -- hard to mock in tests or swap implementations.

### Dependency Injection (DI)

Dependencies are **provided as input** by an external entity (the injector) rather than hardcoded:

```js
// db.js -- factory instead of singleton
export function createDb(filename) {
  return open({ filename, driver: sqlite3.Database })
}

// blog.js -- dependency received via constructor
export class Blog {
  constructor(db) { this.db = db }

  initialize() { return this.db.run('CREATE TABLE IF NOT EXISTS posts (...)') }
  createPost(id, title, content, createdAt) {
    return this.db.run('INSERT INTO posts VALUES (?, ?, ?, ?)', id, title, content, createdAt)
  }
  getAllPosts() {
    return this.db.all('SELECT * FROM posts ORDER BY created_at DESC')
  }
}

// index.js -- the injector
import { Blog } from './blog.js'
import { createDb } from './db.js'

const db = await createDb(join(import.meta.dirname, 'data.sqlite'))
const blog = new Blog(db)  // inject the dependency
```

**Key differences from singleton approach**:
- `blog.js` no longer imports `db.js` -- it accepts any object with `run()` and `all()` methods (duck typing)
- Easy to **mock** for testing or swap to a different database backend
- The injector (`index.js`) is responsible for creating and wiring dependencies in the correct order

**Injection styles**:
- **Constructor injection** -- pass via constructor (most common)
- **Function injection** -- pass when calling a method
- **Property injection** -- assign to a property after creation

### Trade-offs

| Singleton | Dependency Injection |
|---|---|
| Simple, readable | More flexible, testable |
| Tight coupling | Loose coupling |
| Hard to mock | Easy to mock |
| Dependencies resolved at import time | Dependencies resolved at runtime |

**DI downside**: you must manually build the dependency graph in the right order, which can become complex. Libraries like **inversify** and **awilix** provide IoC containers to automate this wiring.

> **Inversion of Control (IoC)**: shifts wiring responsibility to a third-party entity -- either a service locator (`serviceLocator.get('db')`) or a DI container that injects dependencies based on metadata/configuration.

---

## Summary

| Pattern | When to Use |
|---|---|
| **Factory** | Decouple creation from implementation; enforce encapsulation; return different types at runtime |
| **Builder** | Complex constructors with many (optional/interdependent) parameters; fluent API for step-by-step construction |
| **Revealing Constructor** | Lock down internals after creation; immutable objects; one-time initialization |
| **Singleton** | Share a single stateful instance across modules (simple cases) |
| **DI** | Testable, loosely coupled modules; swappable backends; complex dependency graphs |

---

## Cross-References

- See [Structural Design Patterns](structural-design-patterns.md) -- Proxy, Decorator, Adapter
- See [Behavioral Design Patterns](behavioral-design-patterns.md) -- Strategy, State, Template, Iterator, Observer, Command
