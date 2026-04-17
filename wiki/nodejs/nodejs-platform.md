# The Node.js Platform

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Node.js is more than just a runtime -- it embodies a distinct philosophy that shapes how applications are designed, built, and maintained. At its core, Node.js uses a single-threaded, event-driven architecture built on the reactor pattern and powered by libuv and V8. This article covers the foundational principles of the "Node way," the internal mechanics that make Node.js efficient for I/O-heavy workloads, the differences between server-side and browser JavaScript, and how TypeScript fits into the Node.js ecosystem.

## The Node.js Philosophy

Every platform has a set of guiding principles. In Node.js, these principles influence everything from core design decisions to how the community builds and shares modules.

### Small Core

The Node.js core (runtime + built-in modules) has historically been kept minimal, pushing most functionality to "userland" -- the ecosystem of third-party modules. This encourages community experimentation and rapid innovation rather than relying on a slow-moving core.

In recent years this principle has relaxed somewhat. Mature, stable interfaces have been absorbed into core: command-line argument parsing, WebSockets, a built-in test runner, file watching, file globbing, and the `fetch` API, among others. The philosophy hasn't changed -- it has simply evolved as common patterns stabilized.

### Small Modules

Node.js treats the module as the fundamental building block, inheriting two Unix precepts:

- **"Small is beautiful."**
- **"Make each program do one thing well."**

Package managers (npm, pnpm, yarn) solve the dependency hell problem by allowing multiple versions of the same package to coexist in nested `node_modules` trees:

```
node_modules/
  depA@1.0.0/
    node_modules/
      depC@1.0.0
  depB@1.0.0/
    node_modules/
      depC@2.0.0
```

Benefits of small modules:

- Easier to understand and use
- Simpler to test and maintain
- Lightweight -- important for browser bundles and serverless environments (e.g., AWS Lambda) that require fast startup

> **Supply chain caution:** The rise in supply chain vulnerabilities means you should carefully evaluate whether a third-party dependency is well-maintained and truly necessary. More dependencies means a larger attack surface.

### Small Surface Area

Good Node.js modules expose a minimal API. The common pattern is to export a single function or class, providing one clear entry point. Modules are designed to be *used*, not *extended* -- locking down internals simplifies implementation, eases maintenance, and improves usability. In practice, this means preferring exported functions over classes and keeping internals private.

### Simplicity and Pragmatism

The KISS principle ("Keep It Simple, Stupid") and Richard P. Gabriel's "worse is better" philosophy run deep in Node.js:

> "The design must be simple, both in implementation and interface. It is more important for the implementation to be simple than the interface."

Practical implications:

- Simple functions and closures over complex class hierarchies
- Straightforward implementations over mathematically "perfect" abstractions
- Ship fast, adapt easily, maintain with less effort
- Traditional design patterns (Singleton, Decorator, etc.) are implemented in their simplest viable form

## How Node.js Works

### I/O Is the Bottleneck

I/O is the slowest fundamental operation on a computer. The speed gap between memory and disk/network is enormous:

| Operation | Latency Order | Bandwidth Order |
|-----------|--------------|-----------------|
| RAM access | nanoseconds (10^-9 s) | GB/s |
| Disk / Network access | milliseconds (10^-3 s) | MB/s to GB/s |

Add the human factor (mouse clicks, keystrokes) and I/O latency can be orders of magnitude worse. Node.js is designed to handle this bottleneck efficiently through its non-blocking, event-driven architecture.

### Blocking I/O

In traditional blocking I/O, a function call halts the thread until the operation completes:

```
// blocks the thread until the data is available
data = socket.read()
// data is available
print(data)
```

A web server using blocking I/O cannot handle multiple connections on a single thread -- each I/O call blocks everything else. The classic solution is **one thread per connection**, but threads are expensive: they consume memory and cause context switches. Most of the time, each thread sits idle waiting for I/O.

### Non-blocking I/O

Modern operating systems support non-blocking I/O where system calls return immediately. If no data is available, the call returns a predefined constant (e.g., `EAGAIN` on Unix).

The naive approach is **busy-waiting** -- polling resources in a tight loop:

```
resources = [socketA, socketB, fileA]
while (!resources.isEmpty()) {
  for (resource of resources) {
    data = resource.read()
    if (data === NO_DATA_AVAILABLE) continue
    if (data === RESOURCE_CLOSED) resources.remove(resource)
    else consumeData(data)
  }
}
```

This handles multiple resources on one thread but wastes CPU cycles spinning over unavailable resources.

### Event Demultiplexing

The efficient solution is the **synchronous event demultiplexer** (event notification interface), provided natively by operating systems. It monitors multiple resources and blocks until at least one is ready, then returns a set of events:

```
watchList.add(socketA, FOR_READ)
watchList.add(fileB, FOR_READ)

while (events = demultiplexer.watch(watchList)) {
  // event loop
  for (event of events) {
    data = event.resource.read()  // guaranteed not to block
    if (data === RESOURCE_CLOSED) {
      demultiplexer.unwatch(event.resource)
    } else {
      consumeData(data)
    }
  }
}
```

Key insight: tasks are spread over **time** on a single thread, rather than spread across **multiple threads**. This minimizes idle time and eliminates in-process race conditions, enabling much simpler concurrency strategies.

### The Reactor Pattern

The reactor pattern is the heart of Node.js. It associates a **handler** (callback function) with each I/O operation, and the event loop invokes that handler when the operation completes.

**How it works:**

1. The application submits an I/O request to the **Event Demultiplexer** along with a handler (callback). This is non-blocking and returns immediately.
2. When I/O operations complete, the Event Demultiplexer pushes corresponding events into the **Event Queue**.
3. The **Event Loop** iterates over items in the Event Queue.
4. For each event, the associated handler is invoked.
5. The handler executes application code and returns control to the Event Loop. During execution, it may request new async operations, which feed back into the demultiplexer (step 1).
6. When the queue is empty, the Event Loop blocks on the demultiplexer until new events arrive.

**Definition:** The reactor pattern handles I/O by blocking until new events are available from a set of observed resources, then reacts by dispatching each event to an associated handler.

A Node.js process exits when the event loop has no pending operations left -- no active handles (timers, open sockets, filesystem operations) remain. Resources like an HTTP server keep the event loop alive by continuously registering events.

> **Reactor vs. Proactor:** In the reactor pattern, the application controls when and how I/O is performed (responding to "I/O ready" signals). The proactor pattern abstracts the entire I/O process, notifying the application only after completion. Node.js uses the reactor pattern.

### libuv -- The I/O Engine

Each OS has its own event demultiplexer API: `epoll` (Linux), `kqueue` (macOS), IOCP (Windows). Even within one OS, different resource types behave differently (e.g., regular files on Unix don't support non-blocking operations and require a separate thread).

**libuv** is the native C library that abstracts all these inconsistencies. It:

- Normalizes non-blocking behavior across operating systems and resource types
- Implements the reactor pattern
- Provides APIs for creating event loops, managing event queues, running async I/O, and queuing tasks

### The Complete Node.js Architecture

The full Node.js platform consists of:

| Component | Role |
|-----------|------|
| **libuv** | Low-level I/O engine, event loop, cross-platform abstraction |
| **V8** | JavaScript engine (originally developed by Google for Chrome) -- fast execution and efficient memory management |
| **Bindings** | Wrap and expose libuv and other low-level C/C++ functionality to JavaScript |
| **Core JS library** | High-level Node.js API (fs, http, path, etc.) |

## JavaScript in Node.js

Server-side JavaScript differs from browser JavaScript in important ways due to fundamentally different execution environments.

### Key Differences from Browser JavaScript

- **No DOM** -- no `window` or `document` objects
- **Full OS access** -- filesystem (`fs`), TCP/UDP sockets (`net`, `dgram`), HTTP servers (`http`, `https`), cryptography (`crypto`), child processes (`child_process`), process info (`process.env`, `process.argv`)
- **Known runtime** -- you control which Node.js version runs in production, so you can use the latest ES features without transpilers or polyfills

### The Module System

Node.js has two module systems:

- **CommonJS** (`require`) -- the original module system, still found in older codebases
- **ES modules** (`import`) -- now the primary module system, adopted from the ECMAScript standard (though the underlying implementation differs from the browser's)

See [Module System](module-system.md) for a deep dive.

### Running Native Code

Node.js can bind to native compiled code (C, C++, Rust) via the **Node-API** interface. This enables:

- Reusing existing C/C++ libraries and legacy code
- Accessing low-level hardware (USB, serial ports) -- popular in IoT and robotics
- Offloading CPU-intensive work for better performance than pure JavaScript

**WebAssembly (Wasm)** offers a complementary approach: compile C++, Rust, etc. into a format executable by JavaScript VMs, gaining similar performance benefits without directly interfacing with native code.

## Node.js and TypeScript

TypeScript adds a static type system to JavaScript -- types for function arguments, return values, and object structures are checked before runtime, catching bugs early.

### Key Concepts

- TypeScript cannot run directly on Node.js -- it must be **transpiled** to JavaScript first
- The extra compilation step is worth it for larger projects due to early error detection and better tooling

### Running TypeScript with Node.js

**Option 1: Official TypeScript compiler**

```bash
npm install --save-dev typescript
npx tsc example.ts    # type-checks and outputs example.js
node example.js
```

**Option 2: On-the-fly transpilers** (convenient for development)

```bash
npm install --save-dev ts-node
# or
npm install --save-dev tsx

npx ts-node example.ts
# or
npx tsx example.ts
```

`tsx` can also be used as a Node.js loader:

```bash
node --import=tsx example.ts
```

> **Production note:** On-the-fly transpilation incurs overhead on every run. Pre-transpile your code before deploying to production.

**Option 3: Built-in type stripping** (Node.js 24+)

As of Node.js 24, you can execute TypeScript files directly with the `node` CLI via built-in type stripping. Only erasable TypeScript syntax is supported and `tsconfig.json` is ignored, so a dedicated runner is still best for full feature support.

### The @types/node Package

Since Node.js is written in JavaScript, TypeScript needs separate type definitions for Node.js APIs:

```bash
npm install --save-dev @types/node
```

This package provides type definitions for the entire Node.js API (`fs`, `http`, `path`, `process`, `Buffer`, etc.), enabling strong typing, autocompletion, and static type checking in your editor.
