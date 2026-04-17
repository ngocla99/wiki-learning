# Asynchronous Control Flow Patterns

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Asynchronous programming is central to Node.js, but managing the flow of async operations is non-trivial. This article covers the main control flow patterns -- sequential execution, concurrent execution, and limited concurrent execution -- progressing from callback-based approaches to the modern async/await style. Understanding all three layers (callbacks, promises, async/await) is essential because each builds on top of the previous one.

See [Callbacks and Events](callbacks-and-events.md) for callback fundamentals.
See [Streams](streams.md) for stream-based control flow.

---

## Callback Hell and Solutions

### The Problem

When using callback-style (CPS) async code, nesting callbacks inside each other quickly leads to deeply indented, hard-to-follow code known as **callback hell** (or the **pyramid of doom**):

```js
asyncFoo(err => {
  asyncBar(err => {
    asyncFooBar(err => {
      // ...
    })
  })
})
```

Problems with callback hell:

- **Poor readability** -- deep nesting makes it hard to tell where one function ends and another begins
- **Variable shadowing** -- overlapping variable names (e.g., `err`) across scopes introduce confusion and bugs
- **Memory overhead** -- closures retain references to their outer scope, preventing garbage collection and potentially causing memory leaks

### Callback Discipline

Three principles to tame callback-based code without external libraries:

1. **Exit early** -- use `return cb(err)` instead of `if/else` blocks to keep code shallow
2. **Use named functions** -- extract callbacks into named functions for clarity and better stack traces
3. **Modularize** -- break code into smaller, reusable functions

Before (callback hell):

```js
export function spider(url, cb) {
  const filename = urlToFilename(url)
  exists(filename, (err, alreadyExists) => {
    if (err) {
      cb(err)
    } else if (alreadyExists) {
      cb(null, filename, false)
    } else {
      get(url, (err, content) => {
        if (err) {
          cb(err)
        } else {
          recursiveMkdir(dirname(filename), err => {
            if (err) {
              cb(err)
            } else {
              writeFile(filename, content, err => {
                if (err) {
                  cb(err)
                } else {
                  cb(null, filename, true)
                }
              })
            }
          })
        }
      })
    }
  })
}
```

After (applying callback discipline):

```js
function saveFile(filename, content, cb) {
  recursiveMkdir(dirname(filename), err => {
    if (err) return cb(err)
    writeFile(filename, content, cb)
  })
}

function download(url, filename, cb) {
  console.log(`Downloading ${url} into ${filename}`)
  get(url, (err, content) => {
    if (err) return cb(err)
    saveFile(filename, content, err => {
      if (err) return cb(err)
      cb(null, content)
    })
  })
}

export function spider(url, cb) {
  const filename = urlToFilename(url)
  exists(filename, (err, alreadyExists) => {
    if (err) return cb(err)
    if (alreadyExists) return cb(null, filename, false)
    download(url, filename, err => {
      if (err) return cb(err)
      cb(null, filename, true)
    })
  })
}
```

> A common mistake is forgetting to `return` after invoking the callback. Without `return`, the rest of the function continues to execute, causing subtle bugs.

---

## Sequential Execution

Sequential execution means running tasks one at a time, one after the other, where the result of one task may affect the next. There are two main sub-patterns:

- Executing a **known set of tasks** in sequence
- **Iterating** over a collection, running an async task on each element sequentially

### Callback Approach

**Known tasks:**

```js
function task1(cb) {
  asyncOperation(() => { task2(cb) })
}
function task2(cb) {
  asyncOperation(() => { task3(cb) })
}
function task3(cb) {
  asyncOperation(() => { cb() })
}
task1(() => {
  console.log('tasks 1, 2, and 3 executed')
})
```

**Sequential iteration** (the pattern):

```js
function iterate(index) {
  if (index === tasks.length) {
    return finish()
  }
  const task = tasks[index]
  task(() => iterate(index + 1))
}
iterate(0)
```

This recursive-style pattern can be adapted for async map, reduce, early-exit (like `Array.some()`), or even iterating over infinite sequences.

> If `task()` is synchronous, this becomes truly recursive and risks hitting the maximum call stack size.

### Promise Approach

With promises, sequential iteration becomes a matter of dynamically building a promise chain:

```js
function spiderLinks(currentUrl, body, maxDepth) {
  let promise = Promise.resolve()
  if (maxDepth === 0) return promise

  const links = getPageLinks(currentUrl, body)
  for (const link of links) {
    promise = promise.then(() => spider(link, maxDepth - 1))
  }
  return promise
}
```

An alternative using `reduce()`:

```js
const promise = tasks.reduce((prev, task) => {
  return prev.then(() => task())
}, Promise.resolve())
```

### Async/Await Approach (Preferred)

With async/await, sequential execution looks like synchronous code:

```js
async function download(url, filename) {
  console.log(`Downloading ${url} into ${filename}`)
  const content = await get(url)
  await saveFile(filename, content)
  return content
}

async function spiderLinks(currentUrl, body, maxDepth) {
  if (maxDepth === 0) return

  const links = getPageLinks(currentUrl, body)
  for (const link of links) {
    await spider(link, maxDepth - 1)
  }
}

export async function spider(url, maxDepth) {
  const filename = urlToFilename(url)
  let content

  if (!(await exists(filename))) {
    content = await download(url, filename)
  }

  if (!filename.endsWith('.html')) return

  if (!content) {
    content = await readFile(filename)
  }

  return spiderLinks(url, content.toString('utf8'), maxDepth)
}
```

Notice how async/await makes conditional async logic trivial -- something that was clumsy with both callbacks and promise chains.

### Antipattern: forEach with async/await

Using `Array.forEach()` with an async callback does **not** execute sequentially:

```js
// WRONG -- all spider() calls fire concurrently, forEach ignores the returned promises
links.forEach(async (link) => {
  await spider(link, nesting - 1)
})
```

`forEach()` calls the async callback for every element but discards the returned Promise. All iterations start in the same event loop tick. Use a `for...of` loop instead.

---

## Concurrent Execution

When tasks have no data dependency or ordering requirement, run them concurrently to maximize throughput.

**Concurrency vs. parallelism:** In Node.js (single-threaded event loop), tasks run *concurrently* -- one thread interleaves between tasks during I/O waits. *Parallelism* means truly simultaneous execution on multiple CPU cores (achievable via worker threads).

### Callback Approach

Start all tasks at once; track completions with a counter:

```js
const tasks = [/* ... */]
let completed = 0

for (const task of tasks) {
  task(() => {
    if (++completed === tasks.length) {
      finish()
    }
  })
}
```

### Promise Approach

Use `Promise.all()`:

```js
function spiderLinks(currentUrl, body, maxDepth) {
  if (maxDepth === 0) return Promise.resolve()

  const links = getPageLinks(currentUrl, body)
  const promises = links.map(link => spider(link, maxDepth - 1))
  return Promise.all(promises)
}
```

`Promise.all()` rejects immediately if any promise rejects (the others continue running but their results are discarded). Use `Promise.allSettled()` if you want to tolerate individual failures and still get all results.

### Async/Await Approach (Preferred)

```js
async function spiderLinks(currentUrl, body, maxDepth) {
  if (maxDepth === 0) return

  const links = getPageLinks(currentUrl, body)
  const promises = links.map(link => spider(link, maxDepth - 1))
  return Promise.all(promises)
}
```

`Promise.all()` remains the workhorse even with async/await. The `await` keyword simply unwraps the aggregate promise.

**Other useful Promise combinators:**

| Method | Behavior |
|---|---|
| `Promise.all(iterable)` | Fulfills when all fulfill; rejects on first rejection |
| `Promise.allSettled(iterable)` | Waits for all to settle; returns status + value/reason for each |
| `Promise.race(iterable)` | Settles with the first promise that settles (fulfillment or rejection) |

---

## Race Conditions with Concurrent Tasks

Even in single-threaded Node.js, race conditions are common. They occur because of the delay between starting an async operation and receiving its result.

**Example:** Two concurrent spider tasks check if the same file exists. Both see "file does not exist" before either finishes downloading, so both start downloading the same URL.

**Fix:** Use a `Set` to track in-progress operations:

```js
const spidering = new Set()

function spider(url, maxDepth, queue) {
  if (spidering.has(url)) return
  spidering.add(url)
  // proceed with download...
}
```

This is a lightweight mutual exclusion mechanism that works because Node.js executes synchronous code atomically within a single event loop tick.

---

## Limited Concurrent Execution

Unlimited concurrency can exhaust system resources (file descriptors, network connections, memory). Limited concurrency is a hybrid of sequential and concurrent patterns.

### Callback Approach (Pattern)

```js
const concurrency = 2
let running = 0
let completed = 0
let nextTaskIndex = 0

function next() {
  while (running < concurrency && nextTaskIndex < tasks.length) {
    const task = tasks[nextTaskIndex++]
    task(() => {
      if (++completed === tasks.length) {
        return finish()
      }
      running--
      next()
    })
    running++
  }
}
next()
```

### TaskQueue (Global Concurrency Control)

For dynamic workloads (where tasks spawn more tasks), a queue-based approach provides global concurrency limiting:

```js
import { EventEmitter } from 'node:events'

export class TaskQueue extends EventEmitter {
  constructor(concurrency) {
    super()
    this.concurrency = concurrency
    this.running = 0
    this.queue = []
  }

  pushTask(task) {
    this.queue.push(task)
    process.nextTick(this.next.bind(this))
    return this
  }

  next() {
    if (this.running === 0 && this.queue.length === 0) {
      return this.emit('empty')
    }

    while (this.running < this.concurrency && this.queue.length > 0) {
      const task = this.queue.shift()
      task()
        .catch(err => this.emit('error', err))
        .finally(() => {
          this.running--
          this.next()
        })
      this.running++
    }
  }
}
```

**Usage with async/await:**

```js
import { once } from 'node:events'

const queue = new TaskQueue(concurrency)
queue.pushTask(() => spider(url, maxDepth, queue))
queue.on('taskError', console.error)
await once(queue, 'empty')
console.log('Download complete')
```

The `once()` function from `node:events` returns a promise that resolves when the specified event fires -- a clean bridge between events and async/await.

> In production, consider the [p-limit](https://github.com/sindresorhus/p-limit) or [p-map](https://github.com/sindresorhus/p-map) packages for ready-made concurrency limiting.

---

## Promises

### What Is a Promise?

A Promise is an object representing the eventual result (or error) of an async operation. A promise is in one of three states:

- **Pending** -- operation not yet complete
- **Fulfilled** -- operation completed successfully with a value
- **Rejected** -- operation failed with a reason (error)

Once fulfilled or rejected, a promise is **settled** and its state cannot change.

```js
promise.then(onFulfilled, onRejected)
```

Key properties of `then()`:

- Returns a **new Promise** synchronously (enabling chaining)
- If onFulfilled/onRejected returns a value, the new promise fulfills with it
- If onFulfilled/onRejected returns a promise, the new promise adopts its state
- If onFulfilled/onRejected throws, the new promise rejects with the thrown error
- Missing handlers cause the value/error to propagate to the next promise in the chain
- Callbacks are always invoked **asynchronously** (via the microtask queue), even on already-settled promises -- preventing Zalgo

### Promise API Summary

**Constructor:**

```js
new Promise((resolve, reject) => { /* executor -- runs synchronously */ })
```

**Static methods:**

| Method | Description |
|---|---|
| `Promise.resolve(value)` | Creates a fulfilled promise (or adopts if value is a promise) |
| `Promise.reject(err)` | Creates a rejected promise |
| `Promise.all(iterable)` | Fulfills when all fulfill; rejects on first rejection |
| `Promise.allSettled(iterable)` | Waits for all to settle; never short-circuits |
| `Promise.race(iterable)` | Settles with the first settled promise |
| `Promise.withResolvers()` | Returns `{ promise, resolve, reject }` for external control |

**Instance methods:**

| Method | Description |
|---|---|
| `.then(onFulfilled, onRejected)` | Core continuation method |
| `.catch(onRejected)` | Sugar for `.then(undefined, onRejected)` |
| `.finally(onFinally)` | Runs on settlement regardless of outcome; does not receive a value |

### Creating a Promise

```js
function delay(milliseconds) {
  return new Promise((resolve, _reject) => {
    setTimeout(() => resolve(Date.now()), milliseconds)
  })
}
```

The executor function runs synchronously when the Promise is constructed. For a simple timer, you can also use `setTimeout` from `node:timers/promises`:

```js
import { setTimeout } from 'node:timers/promises'
await setTimeout(1000)
```

### Promisification

Promisification converts a Node.js-style callback function into one that returns a Promise:

```js
function promisify(callbackBasedFn) {
  return function promisifiedFn(...args) {
    return new Promise((resolve, reject) => {
      callbackBasedFn(...args, (err, result) => {
        if (err) return reject(err)
        resolve(result)
      })
    })
  }
}
```

In practice, use `util.promisify()` from Node.js core, or import from `node:fs/promises`, `node:dns/promises`, `node:timers/promises`, etc.

---

## Async/Await

### Async Functions and the await Expression

An `async` function always returns a Promise. Inside it, `await` pauses execution until the awaited promise settles, then resumes with the fulfillment value:

```js
async function playingWithDelays() {
  console.log('Delaying...', Date.now())
  const timeAfterOneSecond = await delay(1000)
  console.log(timeAfterOneSecond)
  const timeAfterThreeSeconds = await delay(3000)
  console.log(timeAfterThreeSeconds)
  return 'done'
}
```

At each `await`, control returns to the event loop. The function resumes when the awaited promise fulfills. If it rejects, the rejection is thrown as an exception at the `await` site.

> `await` works with any value. Non-thenables are wrapped automatically via `Promise.resolve()`.

### Top-Level await

In ECMAScript Modules (ESM), `await` can be used at the module's top level, outside any async function:

```js
// No wrapper needed in ESM
const result = await playingWithDelays()
console.log(`After 4 seconds: ${result}`)
```

Useful for: database connections, fetching configuration, initializing remote dependencies at startup.

Before top-level await, you needed an async IIFE:

```js
(async () => {
  const result = await playingWithDelays()
  console.log(`After 4 seconds: ${result}`)
})()
```

### Error Handling with async/await

The `try...catch` block works uniformly for both synchronous throws and async rejections:

```js
async function playingWithErrors(throwSyncError) {
  try {
    if (throwSyncError) {
      throw new Error('This is a synchronous error')
    }
    await delayError(1000)
  } catch (err) {
    console.error(`We have an error: ${err.message}`)
  } finally {
    console.log('Done')
  }
}
```

Both synchronous and asynchronous errors are caught by the same `catch` block -- a major improvement over callbacks where error propagation required manual forwarding at every step.

### The `return` vs `return await` Trap

```js
async function errorNotCaught() {
  try {
    return delayError(1000)  // BUG: local catch never fires
  } catch (err) {
    console.error('Error caught by the async function: ' + err.message)
  }
}
```

Without `await`, the promise is returned directly to the caller, bypassing the local `catch`. The error surfaces at the **caller's** level instead.

**Fix:** Use `return await` when you need local error handling:

```js
async function errorCaught() {
  try {
    return await delayError(1000)  // local catch will fire on rejection
  } catch (err) {
    console.error('Error caught by the async function: ' + err.message)
  }
}
```

`return await` also preserves the async function's frame on the call stack, which aids debugging.

---

## Infinite Recursive Promise Chains (Memory Leak)

When a promise chain is built recursively and never settles, each link in the chain is retained in memory, causing a leak:

```js
// LEAKS MEMORY -- promise chain grows forever
function leakingLoop() {
  return delay(1)
    .then(() => {
      console.log(`Tick ${Date.now()}`)
      return leakingLoop()  // returned promise depends on the next, and so on
    })
}
```

**Solution 1:** Break the chain by not returning the recursive promise (loses error propagation):

```js
function nonLeakingLoop() {
  delay(1)
    .then(() => {
      console.log(`Tick ${Date.now()}`)
      nonLeakingLoop()
    })
}
```

**Solution 2:** Wrap in a Promise constructor to preserve error propagation without chaining:

```js
function nonLeakingLoopWithErrors() {
  return new Promise((_resolve, reject) => {
    (function internalLoop() {
      delay(1)
        .then(() => {
          console.log(`Tick ${Date.now()}`)
          internalLoop()
        })
        .catch(err => reject(err))
    })()
  })
}
```

**Solution 3 (Preferred):** Use async/await with a `while` loop:

```js
async function nonLeakingLoopAsync() {
  while (true) {
    await delay(1)
    console.log(`Tick ${Date.now()}`)
  }
}
```

> The async/await recursive version (`return leakingLoopAsync()` at the end of an async function) still leaks! Always prefer the `while` loop for infinite async iterations.

---

## Summary

| Pattern | Callback | Promise | Async/Await |
|---|---|---|---|
| Sequential (known tasks) | Chain via nested callbacks | `.then()` chains | `await` in sequence |
| Sequential iteration | Recursive `iterate(index)` | Dynamic promise chain in a loop | `for...of` with `await` |
| Concurrent | Counter + loop | `Promise.all()` | `Promise.all()` with `await` |
| Limited concurrent | `next()` with running counter | TaskQueue with `.finally()` | Same TaskQueue, `await once(queue, 'empty')` |

**Key takeaways:**

- **Async/await is the preferred style** for modern Node.js. It makes sequential code read like synchronous code and unifies error handling with `try/catch`.
- **Promises are the foundation** of async/await. `Promise.all()`, `Promise.allSettled()`, and `Promise.race()` remain essential utilities.
- **Callbacks still matter** for understanding internals, legacy APIs, and performance-critical paths.
- **Always guard against race conditions** when running tasks concurrently, even in single-threaded Node.js.
- **Use a TaskQueue** (or libraries like `p-limit`) for global concurrency control in workloads that dynamically spawn new tasks.
- **Watch for the forEach antipattern** -- use `for...of` for sequential async iteration.
- **Prefer `return await`** inside `try` blocks to ensure local error handling works.
- **Avoid infinite recursive promise chains** -- use `while (true)` with `await` instead.
