# Callbacks and Events

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Callbacks and the EventEmitter are the two pillars of Node.js asynchronous infrastructure. Callbacks embody the Reactor pattern's handlers, while the EventEmitter implements the Observer pattern. Understanding both -- their conventions, pitfalls, and when to combine them -- is essential before moving to higher-level abstractions like Promises and Async/Await (which are built on top of callbacks).

---

## The Callback Pattern

A **callback** is a function passed as an argument to another function, invoked with the result of an operation when it completes. In functional programming this is called **Continuation-Passing Style (CPS)** -- the result is propagated by passing it to another function rather than returning it directly.

### Synchronous CPS

The callback is invoked immediately, before the function returns:

```js
function addCps(a, b, callback) {
  callback(a + b)
}

console.log('before')
addCps(1, 2, result => console.log(`Result: ${result}`))
console.log('after')
// Output: before → Result: 3 → after
```

The result is delivered through the callback, but execution is still sequential -- nothing asynchronous happens.

### Asynchronous CPS

The callback is deferred to a future event loop cycle:

```js
function addAsync(a, b, callback) {
  setTimeout(() => callback(a + b), 100)
}

console.log('before')
addAsync(1, 2, result => console.log(`Result: ${result}`))
console.log('after')
// Output: before → after → Result: 3
```

`setTimeout` returns immediately, giving control back to the event loop. The callback runs later with a **fresh call stack**. Closures preserve the original context so variables remain accessible when the callback eventually executes.

### Non-CPS Callbacks

Not every function that takes a callback is CPS. Array methods like `map`, `filter`, and `forEach` use callbacks for iteration, not for propagating a result:

```js
const result = [1, 5, 7].map(element => element - 1)
// result is returned synchronously via direct style
```

The presence of a callback alone does not make a function asynchronous or CPS.

---

## Synchronous vs Asynchronous Consistency

**A function must be either always synchronous or always asynchronous -- never both.** Mixing the two creates subtle, hard-to-reproduce bugs.

### The Problem: "Unleashing Zalgo"

An inconsistent function that is asynchronous on first call but synchronous on cache hit:

```js
import { readFile } from 'node:fs'
const cache = new Map()

function inconsistentRead(filename, cb) {
  if (cache.has(filename)) {
    cb(cache.get(filename))           // synchronous!
  } else {
    readFile(filename, 'utf8', (_err, data) => {
      cache.set(filename, data)
      cb(data)                        // asynchronous
    })
  }
}
```

This breaks any code that registers listeners after calling the function. On the first (async) call, listeners register in time. On the second (sync) call, the callback fires immediately -- before listeners are registered -- so they are never invoked.

### Fix 1: Use Synchronous APIs

Make the function purely synchronous with direct style:

```js
import { readFileSync } from 'node:fs'
const cache = new Map()

function consistentReadSync(filename) {
  if (cache.has(filename)) {
    return cache.get(filename)
  }
  const data = readFileSync(filename, 'utf8')
  cache.set(filename, data)
  return data
}
```

> **Pattern:** Always choose direct style for purely synchronous functions.

**Caveat:** Synchronous I/O blocks the event loop. Acceptable for startup/config loading or CLI tools, but avoid in request-handling code that needs concurrency.

### Fix 2: Deferred Execution

Make the function purely asynchronous by deferring the synchronous path:

```js
import { readFile } from 'node:fs'
const cache = new Map()

function consistentReadAsync(filename, callback) {
  if (cache.has(filename)) {
    process.nextTick(() => callback(cache.get(filename)))
  } else {
    readFile(filename, 'utf8', (_err, data) => {
      cache.set(filename, data)
      callback(data)
    })
  }
}
```

> **Pattern:** Guarantee asynchronous callback invocation using `process.nextTick()`.

### Deferred Execution Priority

```js
setImmediate(() => console.log('setImmediate'))
setTimeout(() => console.log('setTimeout'), 0)
process.nextTick(() => console.log('nextTick'))
console.log('sync')

// Output: sync → nextTick → setTimeout → setImmediate
```

| Mechanism | Priority | Notes |
|---|---|---|
| `process.nextTick()` | Highest (microtask) | Runs before any I/O; recursive use can cause **I/O starvation** |
| `setTimeout(cb, 0)` | Timer phase | Runs after microtasks, before I/O callbacks |
| `setImmediate()` | Check phase | Runs after I/O callbacks |

---

## Node.js Callback Conventions

### Callback Last

The callback is always the **last argument**:

```js
readFile(filename, [options], callback)
```

### Error First

The first argument of the callback is reserved for an error (`null`/`undefined` on success). Actual results follow from the second argument onward:

```js
readFile('foo.txt', 'utf8', (err, data) => {
  if (err) {
    handleError(err)
  } else {
    processData(data)
  }
})
```

Errors must always be `Error` instances, never plain strings or numbers.

### Error Propagation

In asynchronous CPS, errors are propagated by passing them to the next callback -- **not** by throwing:

```js
import { readFile } from 'node:fs'

function readJson(filename, callback) {
  readFile(filename, 'utf8', (err, data) => {
    let parsed
    if (err) {
      return callback(err)       // propagate I/O error
    }
    try {
      parsed = JSON.parse(data)
    } catch (err) {
      return callback(err)       // propagate parse error
    }
    callback(null, parsed)       // success
  })
}
```

Key points:
- `return callback(err)` stops execution and propagates the error.
- Synchronous errors (like `JSON.parse`) must be caught with `try/catch` and forwarded via the callback.
- Never invoke the callback inside a `try` block -- it would catch errors thrown by the callback itself.

### Uncaught Exceptions

An error thrown inside an async callback travels up to the event loop, **not** to the caller. Wrapping the call in `try/catch` does not work because the callback runs on a different call stack.

```js
// This will NOT catch the error:
try {
  readJsonThrows('bad.json', (err) => console.error(err))
} catch (err) {
  console.log('Never reached')
}
```

As a last resort, listen for the `uncaughtException` event, then exit:

```js
process.on('uncaughtException', (err) => {
  console.error(`Uncaught: ${err.message}`)
  process.exit(1)  // fail-fast
})
```

An uncaught exception leaves the application in an inconsistent state. The recommended practice is **fail-fast**: log, clean up, exit, and let a supervisor restart the process.

---

## The Observer Pattern (EventEmitter)

The Observer pattern defines a **subject** that notifies multiple **observers** (listeners) when a state change occurs. Unlike callbacks (one result, one listener), an EventEmitter can notify many listeners for each event type.

### Core API

```js
import { EventEmitter } from 'node:events'
const emitter = new EventEmitter()
```

| Method | Description |
|---|---|
| `on(event, listener)` | Register a listener for an event type |
| `once(event, listener)` | Register a one-time listener (auto-removed after first emit) |
| `emit(event, [args...])` | Fire an event, passing args to listeners |
| `removeListener(event, listener)` / `off()` | Remove a specific listener |
| `removeAllListeners(event)` | Remove all listeners for an event |

All methods return the EventEmitter instance for chaining. Listener signature: `function([arg1], [...])` -- **no error-first convention** (unlike callbacks).

### Creating and Using EventEmitter

```js
import { EventEmitter } from 'node:events'
import { readFile } from 'node:fs'

function findRegex(files, regex) {
  const emitter = new EventEmitter()
  for (const file of files) {
    readFile(file, 'utf8', (err, content) => {
      if (err) return emitter.emit('error', err)
      emitter.emit('fileread', file)
      const match = content.match(regex)
      if (match) {
        for (const elem of match) {
          emitter.emit('found', file, elem)
        }
      }
    })
  }
  return emitter
}

findRegex(['fileA.txt', 'fileB.json'], /hello [\w.]+/)
  .on('fileread', file => console.log(`${file} was read`))
  .on('found', (file, match) => console.log(`Matched "${match}" in ${file}`))
  .on('error', err => console.error(`Error: ${err.message}`))
```

### Propagating Errors

The convention is to emit an `'error'` event with an `Error` object. The EventEmitter treats `'error'` specially: if emitted with no registered listener, it **throws an exception and crashes the process**. Always register an error listener.

### Making Any Object Observable

Extend `EventEmitter` to make a class observable:

```js
import { EventEmitter } from 'node:events'
import { readFile } from 'node:fs'

class FindRegex extends EventEmitter {
  constructor(regex) {
    super()
    this.regex = regex
    this.files = []
  }

  addFile(file) {
    this.files.push(file)
    return this
  }

  find() {
    for (const file of this.files) {
      readFile(file, 'utf8', (err, content) => {
        if (err) return this.emit('error', err)
        this.emit('fileread', file)
        const match = content.match(this.regex)
        if (match) {
          for (const elem of match) {
            this.emit('found', file, elem)
          }
        }
      })
    }
    return this
  }
}

new FindRegex(/hello [\w.]+/)
  .addFile('fileA.txt')
  .addFile('fileB.json')
  .find()
  .on('found', (file, match) => console.log(`Matched "${match}" in ${file}`))
  .on('error', err => console.error(err.message))
```

This pattern is pervasive in Node.js: `http.Server`, streams, and many core objects inherit from `EventEmitter`.

### Memory Leaks

Unreleased EventEmitter listeners are the **main source of memory leaks** in Node.js. Listeners hold references to variables in their closure scope -- those variables cannot be garbage collected while the listener is registered.

```js
// Register
emitter.on('an_event', listener)

// Release when no longer needed
emitter.removeListener('an_event', listener)
```

- The EventEmitter warns when >10 listeners are registered for a single event (adjustable via `setMaxListeners()`).
- `once()` auto-removes after the first emit, but if the event is **never** emitted, the listener is never released -- still a leak.

### Synchronous vs Asynchronous Events

The same consistency rule applies: never mix synchronous and asynchronous emission for the same event type.

**Asynchronous events** (the common case): listeners can be registered after the task is triggered, because events won't fire until the next event loop cycle.

```js
findRegexInstance
  .addFile('fileA.txt')
  .find()                    // triggers async I/O
  .on('found', ...)          // safe -- registered before events fire
```

**Synchronous events**: all listeners must be registered **before** the task runs, or they will miss events.

```js
findRegexSyncInstance
  .addFile('fileA.txt')
  .on('found', ...)          // must register BEFORE find()
  .find()
```

> Synchronous emission can be deferred with `process.nextTick()` to guarantee async behavior.

---

## EventEmitter vs Callbacks

| Use Case | Prefer |
|---|---|
| Return a single result asynchronously | **Callback** |
| Event may occur multiple times (or not at all) | **EventEmitter** |
| Multiple listeners need to react to the same event | **EventEmitter** |
| Multiple distinct event types | **EventEmitter** (cleaner than overloading a callback) |

---

## Combining Callbacks and Events

A powerful pattern: use a **callback** for the final result and an **EventEmitter** for progress/status during a long-running operation.

```js
import { EventEmitter } from 'node:events'
import { get } from 'node:https'

function download(url, cb) {
  const emitter = new EventEmitter()

  const req = get(url, resp => {
    const chunks = []
    let downloadedBytes = 0
    const fileSize = Number.parseInt(resp.headers['content-length'], 10)

    resp
      .on('error', err => cb(err))
      .on('data', chunk => {
        chunks.push(chunk)
        downloadedBytes += chunk.length
        emitter.emit('progress', downloadedBytes, fileSize)
      })
      .on('end', () => cb(null, Buffer.concat(chunks)))
  })

  req.on('error', err => cb(err))
  return emitter
}

// Usage:
download('https://example.com/file.zip', (err, data) => {
  if (err) return console.error(`Failed: ${err.message}`)
  console.log('Download complete', data.length, 'bytes')
}).on('progress', (downloaded, total) => {
  console.log(`${((downloaded / total) * 100).toFixed(1)}%`)
})
```

The `get()` function from `node:https` itself uses this same pattern: it accepts a callback for the response while returning an EventEmitter (the request object) for monitoring.

This pattern applies to file uploads, data processing pipelines, database migrations, and any long-running task where you need both completion handling and real-time progress.

> In modern Node.js, this pattern is gradually being replaced by combining **Promises** with **async iterators**. You can also return `{ promise, events }` for a more explicit API.

---

## See Also

- See [Async Control Flow](async-control-flow.md) for control flow patterns (sequential execution, parallel execution, error handling with callbacks)
