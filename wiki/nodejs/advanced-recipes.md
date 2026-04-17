# Advanced Recipes

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

This article covers four advanced patterns that address recurring challenges in real-world Node.js applications: safely handling components that require asynchronous initialization, batching and caching concurrent requests for dramatic performance gains, canceling in-flight async operations, and running CPU-bound tasks without blocking the event loop. Each recipe explains the problem, presents one or more solution patterns with code, and discusses trade-offs.

See [Async Control Flow](async-control-flow.md) for foundational promise and async/await patterns.
See [Behavioral Design Patterns](behavioral-design-patterns.md) for the State pattern.
See [Scalability Patterns](scalability-patterns.md) for scaling CPU-bound work across nodes.

---

## Asynchronously Initialized Components

### The Problem

Many components (database drivers, message queue clients) require an asynchronous initialization step -- a handshake, connection, or config fetch -- before their API can be used. Consumers can easily misuse such components by forgetting to `await` initialization or by calling methods before initialization completes:

```js
import { db } from './db.js'

// Bug: forgot to await connect()
db.connect()
const users = await db.query('SELECT * FROM users') // throws "Not connected yet"
```

### Solution 1: Local Initialization Check

The simplest fix -- guard every call site:

```js
async function getUsers() {
  if (!db.connected) {
    await db.connect()
  }
  await db.query('SELECT * FROM users')
}
```

This can also be moved inside the component itself (e.g., inside `query()`) so consumers never have to think about it.

**Trade-off:** Repetitive and error-prone if spread across many call sites. When moved inside the component, it adds an async check on every call, even after initialization.

### Solution 2: Delayed Startup

Create a factory that returns an already-initialized component:

```js
async function getConnectedDb() {
  await db.connect()
  return db
}

const connectedDb = await getConnectedDb()
await connectedDb.query('SELECT * FROM users')
```

Or delay the entire application startup until all async services are ready.

**Trade-off:** Requires knowing all dependencies ahead of time. Adds startup latency. Does not handle re-initialization (e.g., reconnection after a disconnect). Pre-initializes resources that may never be used.

### Solution 3: Pre-Initialization Queues

Queue method invocations while the component is still initializing, then flush them once initialization completes. This combines a queue with the Command pattern:

```js
class Database {
  connected = false
  commandsQueue = []

  async connect() {
    // ... establish connection ...
    this.connected = true
    // flush queued commands
    while (this.commandsQueue.length > 0) {
      const command = this.commandsQueue.shift()
      command()
    }
  }

  async query(queryString) {
    if (!this.connected) {
      console.log(`Request queued: ${queryString}`)
      return new Promise((resolve, reject) => {
        const command = () => {
          this.query(queryString).then(resolve, reject)
        }
        this.commandsQueue.push(command)
      })
    }
    // ... execute query normally ...
  }
}
```

Usage is transparent -- callers do not need to check state:

```js
db.connect()                       // no await needed
await db.query('SELECT * FROM users') // queued if not yet connected, runs when ready
```

**Trade-off:** If `connect()` is never called, queued queries return promises that never resolve. This pattern is used in production by libraries like **Mongoose** (MongoDB ORM) and **pg** (PostgreSQL client).

### Solution 4: State Pattern Approach

Encapsulate the two modes (queuing vs. initialized) into separate state objects. The component delegates all method calls to the current state. See [Behavioral Design Patterns](behavioral-design-patterns.md) for the State pattern.

```js
class InitializedState {
  constructor(db) { this.db = db }

  async query(queryString) {
    await setTimeout(100)
    console.log(`Query executed: ${queryString}`)
  }
}

const deactivate = Symbol('deactivate')

class QueuingState {
  constructor(db) {
    this.db = db
    this.commandsQueue = []
  }

  async query(queryString) {
    console.log(`Request queued: ${queryString}`)
    return new Promise((resolve, reject) => {
      const command = () => {
        this.db.query(queryString).then(resolve, reject)
      }
      this.commandsQueue.push(command)
    })
  }

  [deactivate]() {
    while (this.commandsQueue.length > 0) {
      this.commandsQueue.shift()()
    }
  }
}

class Database {
  constructor() {
    this.state = new QueuingState(this)
  }

  async query(queryString) {
    return this.state.query(queryString)   // delegate to current state
  }

  async connect() {
    // ... establish connection ...
    const oldState = this.state
    this.state = new InitializedState(this) // swap state
    oldState[deactivate]?.()                // flush pending commands
  }
}
```

**Trade-off:** More modular and eliminates repetitive `if (!connected)` checks inside every method. Requires control over the component's implementation -- if you cannot modify it, wrap it with a Proxy instead.

---

## Asynchronous Request Batching and Caching

### What Is Request Batching?

When multiple callers invoke the same async operation with the same input while a previous call is still in flight, they can **piggyback** on the existing promise instead of launching duplicate work:

```
Without batching:  Client A ---> [operation] ---> result A
                   Client B ---> [operation] ---> result B   (identical, wasted)

With batching:     Client A --|
                              +--> [operation] ---> result (shared)
                   Client B --|
```

### Batching with Promises

Promises are ideal for batching because multiple `.then()` listeners can attach to the same promise. The pattern stores in-flight promises in a Map keyed by request parameters:

```js
import { totalSales as totalSalesRaw } from './totalSales.js'

const runningRequests = new Map()

export function totalSales(product) {
  if (runningRequests.has(product)) {
    console.log('Batching')
    return runningRequests.get(product)      // piggyback on existing request
  }

  const resultPromise = totalSalesRaw(product)
  runningRequests.set(product, resultPromise)
  resultPromise.finally(() => {
    runningRequests.delete(product)           // clean up when done
  })

  return resultPromise
}
```

In benchmarks, this simple layer yielded roughly **6.5x** more requests per second versus no batching.

**When it shines:** High-load servers with slow operations and many concurrent identical requests. The more concurrency and the slower the operation, the bigger the gain.

### Optimal Caching (Batching + Cache)

Batching alone only helps while a request is in flight. For data that does not change frequently, keep the resolved promise in the map after it settles -- effectively turning it into a cache:

```js
import { totalSales as totalSalesRaw } from './totalSales.js'

const CACHE_TTL = 30_000 // 30 seconds
const cache = new Map()

export function totalSales(product) {
  if (cache.has(product)) {
    console.log('Cache hit')
    return cache.get(product)                 // return cached promise
  }

  const resultPromise = totalSalesRaw(product)
  cache.set(product, resultPromise)

  resultPromise.then(
    () => setTimeout(() => cache.delete(product), CACHE_TTL),
    () => cache.delete(product)               // evict immediately on error
  )

  return resultPromise
}
```

The two phases:

1. **Batching phase** -- while the cache is empty and the first request is in flight, all concurrent identical requests share the same promise.
2. **Caching phase** -- once resolved, subsequent requests are served instantly from the cached promise until TTL expires.

Because a resolved promise always returns asynchronously via `.then()`, we naturally avoid the Zalgo anti-pattern (see [Callbacks and Events](callbacks-and-events.md)).

In benchmarks, batching + caching achieved roughly **2,500x** more requests per second versus the raw implementation.

**Production considerations:**

- Use **LRU** or **FIFO** eviction policies to bound memory
- In distributed systems, use a shared store (Redis, Valkey, Memcached) for consistency across instances
- Consider event-driven invalidation when the underlying data changes

---

## Canceling Async Operations

### The Problem

Long-running async operations sometimes need to be stopped -- the user navigated away, a timeout fired, or the result is no longer needed. JavaScript's cooperative multitasking means the runtime will not interrupt an async function; cancellation must be checked explicitly at `await` points.

### Basic Cancelable Function

Pass a shared cancel object and check it after each async step:

```js
async function cancelable(cancelObj) {
  const resA = await asyncRoutine('A')
  console.log(resA)
  if (cancelObj.cancelRequested) throw new CancelError()

  const resB = await asyncRoutine('B')
  console.log(resB)
  if (cancelObj.cancelRequested) throw new CancelError()

  const resC = await asyncRoutine('C')
  console.log(resC)
}

// Usage
const cancelObj = { cancelRequested: false }
setTimeout(() => { cancelObj.cancelRequested = true }, 100)

try {
  await cancelable(cancelObj)
} catch (err) {
  if (err instanceof CancelError) console.log('Canceled')
}
```

**Trade-off:** Works but is verbose and not DRY.

### Wrapping Async Invocations

Reduce boilerplate by wrapping each async call through a guard function:

```js
export function createCancelWrapper() {
  let cancelRequested = false

  function cancel() { cancelRequested = true }

  function callIfNotCanceled(func, ...args) {
    if (cancelRequested) {
      return Promise.reject(new CancelError())
    }
    return func(...args)
  }

  return { callIfNotCanceled, cancel }
}
```

The cancelable function becomes much cleaner:

```js
async function cancelable(callIfNotCanceled) {
  const resA = await callIfNotCanceled(asyncRoutine, 'A')
  console.log(resA)
  const resB = await callIfNotCanceled(asyncRoutine, 'B')
  console.log(resB)
  const resC = await callIfNotCanceled(asyncRoutine, 'C')
  console.log(resC)
}

const { callIfNotCanceled, cancel } = createCancelWrapper()
setTimeout(cancel, 100)
await cancelable(callIfNotCanceled)
```

### AbortController (Preferred Approach)

`AbortController` is the **standard** cancellation mechanism in JavaScript (browsers and Node.js). It provides interoperability across libraries without custom protocols.

**Key concepts:**

| Part | Role | Owner |
|------|------|-------|
| `AbortController` | Triggers cancellation via `.abort()` | Caller / initiator |
| `AbortSignal` | Receives and reacts to cancellation | The async operation |

```js
async function cancelable(abortSignal) {
  abortSignal.throwIfAborted()
  const resA = await asyncRoutine('A')
  console.log(resA)

  abortSignal.throwIfAborted()
  const resB = await asyncRoutine('B')
  console.log(resB)

  abortSignal.throwIfAborted()
  const resC = await asyncRoutine('C')
  console.log(resC)
}

const ac = new AbortController()
setTimeout(() => ac.abort(), 100)

try {
  await cancelable(ac.signal)
} catch (err) {
  if (err.name === 'AbortError') {
    console.log('Function canceled')
  } else {
    console.error(err)
  }
}
```

**Ways to check the signal inside an async function:**

- `abortSignal.throwIfAborted()` -- throws immediately if aborted
- `abortSignal.aborted` -- boolean, for custom handling
- `abortSignal.addEventListener('abort', handler)` -- event-driven, for decoupled reactions

**Trade-off:** Slightly verbose (a check at every `await` point), but this is inherent to cooperative multitasking. You can write a small `callIfNotAborted(signal, fn, ...args)` helper to reduce repetition. Always prefer `AbortController` over custom solutions for interoperability.

---

## Running CPU-Bound Tasks

### The Problem

Node.js excels at I/O-heavy work because async operations unwind the stack back to the event loop. But a **synchronous, CPU-intensive** computation blocks the event loop entirely, making the server unresponsive to all other requests -- including health checks.

Example: a brute-force **subset sum** solver (checking all 2^n subsets) blocks the event loop for seconds:

```js
export class SubsetSum extends EventEmitter {
  constructor(sum, set) {
    super()
    this.sum = sum
    this.set = set
  }

  _combine(set, subset) {
    for (let i = 0; i < set.length; i++) {
      const newSubset = subset.concat(set[i])
      this._combine(set.slice(i + 1), newSubset)
      this._processSubset(newSubset)
    }
  }

  _processSubset(subset) {
    const res = subset.reduce((prev, item) => prev + item, 0)
    if (res === this.sum) this.emit('match', subset)
  }

  start() {
    this._combine(this.set, [])
    this.emit('end')
  }
}
```

While this runs, even a simple `GET /` health check hangs. An exposed endpoint like this is also a **DoS vector**.

### Solution 1: Interleaving with `setImmediate`

Break the computation into steps and schedule each step with `setImmediate()`, yielding the event loop between iterations:

```js
_combineInterleaved(set, subset) {
  this.runningCombine++
  setImmediate(() => {
    this._combine(set, subset)
    if (--this.runningCombine === 0) {
      this.emit('end')
    }
  })
}

_combine(set, subset) {
  for (let i = 0; i < set.length; i++) {
    const newSubset = subset.concat(set[i])
    this._combineInterleaved(set.slice(i + 1), newSubset) // deferred recursion
    this._processSubset(newSubset)
  }
}

start() {
  this.runningCombine = 0
  this._combineInterleaved(this.set, [])
}
```

We track running instances with a counter (`runningCombine`) and emit `end` only when all deferred calls complete -- similar to the async parallel execution pattern. See [Async Control Flow](async-control-flow.md).

**Trade-off:**

- Keeps the event loop responsive
- Each `setImmediate()` adds scheduling overhead, which can significantly slow the total computation
- Does not help if individual steps are themselves long-running
- Never use `process.nextTick()` for this -- it runs before I/O and will starve the event loop

**Best for:** Sporadic, short-lived CPU tasks where simplicity matters more than raw throughput.

### Solution 2: External Processes (child_process.fork)

Offload the computation to a child process so the main event loop stays completely free:

**Process Pool:**

```js
import { fork } from 'node:child_process'

export class ProcessPool {
  constructor(file, poolMax) {
    this.file = file
    this.poolMax = poolMax
    this.pool = []     // idle workers
    this.active = []   // busy workers
    this.waiting = []  // queued acquire() callers
  }

  acquire() {
    return new Promise((resolve, reject) => {
      if (this.pool.length > 0) {
        const worker = this.pool.pop()
        this.active.push(worker)
        return resolve(worker)
      }
      if (this.active.length >= this.poolMax) {
        return this.waiting.push({ resolve, reject })
      }
      const worker = fork(this.file)
      worker.once('message', msg => {
        if (msg === 'ready') {
          this.active.push(worker)
          return resolve(worker)
        }
        worker.kill()
        reject(new Error('Improper process start'))
      })
    })
  }

  release(worker) {
    if (this.waiting.length > 0) {
      const { resolve } = this.waiting.shift()
      return resolve(worker)
    }
    this.active = this.active.filter(w => w !== worker)
    this.pool.push(worker)
  }
}
```

**Worker (child process):**

```js
import { SubsetSum } from '../subsetSum.js'

process.on('message', msg => {
  const subsetSum = new SubsetSum(msg.sum, msg.set)
  subsetSum.on('match', data => process.send({ event: 'match', data }))
  subsetSum.on('end', data => process.send({ event: 'end', data }))
  subsetSum.start()
})

process.send('ready')
```

**Main-thread wrapper** (preserves the same EventEmitter API):

```js
export class SubsetSum extends EventEmitter {
  constructor(sum, set) { super(); this.sum = sum; this.set = set }

  async start() {
    const worker = await workers.acquire()
    worker.send({ sum: this.sum, set: this.set })
    const onMessage = msg => {
      if (msg.event === 'end') {
        worker.removeListener('message', onMessage)
        workers.release(worker)
      }
      this.emit(msg.event, msg.data)
    }
    worker.on('message', onMessage)
  }
}
```

**Trade-off:**

- Algorithm runs at full speed with no interleaving overhead
- Leverages multiple CPU cores -- each process can use a different core
- Pool limits concurrency, preventing DoS
- Higher memory overhead per process compared to threads
- Communication is message-based (serialization cost)
- Use `os.cpus().length` to determine a good default pool size

### Solution 3: Worker Threads

Worker threads (`node:worker_threads`) are a lighter-weight alternative to child processes. They share the same process but run in separate V8 instances with independent event loops.

The API is nearly identical to the child process approach, with these substitutions:

| Child Process | Worker Thread |
|---|---|
| `fork(file)` | `new Worker(file)` |
| `worker.send(msg)` | `worker.postMessage(msg)` |
| `process.send(msg)` | `parentPort.postMessage(msg)` |
| `process.on('message', ...)` | `parentPort.on('message', ...)` |
| `worker.once('message', ...)` (ready) | `worker.once('online', ...)` |

**Thread worker:**

```js
import { parentPort } from 'node:worker_threads'
import { SubsetSum } from '../subsetSum.js'

parentPort.on('message', msg => {
  const subsetSum = new SubsetSum(msg.sum, msg.set)
  subsetSum.on('match', data => parentPort.postMessage({ event: 'match', data }))
  subsetSum.on('end', data => parentPort.postMessage({ event: 'end', data }))
  subsetSum.start()
})
```

**Trade-off vs. child processes:**

- Lower memory footprint and faster startup
- Can share memory via `SharedArrayBuffer` and `Atomics` for advanced use cases
- Same isolation guarantees for typical message-passing patterns
- Still requires a pool to limit concurrency

### Production Considerations

The custom pools shown above omit error handling, timeouts, and crash recovery. For production, use battle-tested libraries:

- **[workerpool](https://github.com/josdejong/workerpool)** -- supports both processes and threads
- **[piscina](https://github.com/piscinajs/piscina)** -- high-performance worker thread pool

Additional hardening:

- Terminate idle workers after inactivity to free memory
- Restart crashed or unresponsive workers automatically
- If a single machine is not enough, scale computation across multiple nodes. See [Scalability Patterns](scalability-patterns.md).

---

## Summary

| Recipe | Core Idea | Best For |
|---|---|---|
| Pre-initialization queues | Queue calls until component is ready, then flush | Database clients, any async-init singleton |
| Request batching | Piggyback on in-flight promises | High-load APIs with slow identical requests |
| Batching + caching | Keep resolved promises as cache entries | Data that changes infrequently |
| Cancelable functions | Check cancel flag at each `await` point | User-initiated cancellation, timeouts |
| `AbortController` | Standard cancel signal; prefer over custom | All new cancelable APIs |
| `setImmediate` interleaving | Yield event loop between computation steps | Light, sporadic CPU tasks |
| Child processes / worker threads | Offload to separate execution context | Heavy CPU work in production |
