# Backend & System Concepts — Interview Questions

> Sources: Node.js Design Patterns (Mammino & Casciaro), MDN, MongoDB docs, personal experience
> Updated: 2026-04-21

## Overview

Interview questions covering Node.js fundamentals, WebSockets, MongoDB query patterns, and Node.js clustering for horizontal scaling.

---

## Node.js Fundamentals

### What is Node.js and how does it work?

Node.js is a JavaScript runtime built on V8 (Chrome's JS engine) and **libuv** — a cross-platform C library providing the non-blocking I/O and event loop implementation.

**Core architecture:**

```
┌─────────────────────────────────┐
│         Your Application        │
├─────────────────────────────────┤
│          Node.js Core           │
│  (http, fs, crypto, streams...) │
├──────────────┬──────────────────┤
│     V8       │      libuv       │
│ (JS engine)  │ (event loop,     │
│              │  thread pool,    │
│              │  async I/O)      │
└──────────────┴──────────────────┘
```

**Key characteristics:**
- **Single-threaded** — application JS runs on one thread
- **Non-blocking I/O** — I/O operations (file reads, network, DB) are delegated to the OS/thread pool; the main thread continues
- **Event-driven** — results arrive as events and are handled by callbacks/promises
- **Reactor pattern** — the event loop polls for I/O completion and dispatches handlers

### Why is Node.js good for I/O-heavy workloads but not CPU-heavy?

```
CPU-heavy request (e.g., image resize):
  Main thread: |------ processing -------|
  Other req:                              |--- queued ---|

I/O-heavy request (e.g., DB query):
  Main thread: |---→ (delegates to OS) ---→ |handler|
  Other req:         |---→ (delegates) ----→ |handler|
```

For I/O, the thread delegates and is free immediately. For CPU-heavy work, the thread blocks. Solution for CPU work: **Worker Threads** or offload to a child process.

### What is the Node.js event loop?

The event loop (implemented in libuv) processes tasks in phases:

```
   ┌───────────────┐
   │    timers     │  ← setTimeout, setInterval callbacks
   └───────┬───────┘
   ┌───────┴───────┐
   │ pending cbs   │  ← I/O errors deferred from last iteration
   └───────┬───────┘
   ┌───────┴───────┐
   │  idle/prepare │  ← internal use
   └───────┬───────┘
   ┌───────┴───────┐
   │     poll      │  ← retrieve new I/O events; execute I/O callbacks
   └───────┬───────┘
   ┌───────┴───────┐
   │     check     │  ← setImmediate callbacks
   └───────┬───────┘
   ┌───────┴───────┐
   │ close cbs     │  ← socket.on('close', ...)
   └───────────────┘
```

Between each phase, Node.js drains the **microtask queue** (`process.nextTick` first, then Promises).

### What is the difference between `process.nextTick` and `setImmediate`?

```js
setImmediate(() => console.log('setImmediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));

// Output: nextTick → promise → setImmediate
```

- `process.nextTick` — fires before the next event loop iteration (highest priority, can starve I/O if called recursively)
- Promises — microtask queue, after `nextTick`
- `setImmediate` — check phase of the next event loop iteration

### What are Streams in Node.js?

Streams process data in **chunks** rather than loading everything into memory — essential for large files, video, or network data.

```js
import { createReadStream, createWriteStream } from 'fs';
import { createGzip } from 'zlib';

// Compress a large file without loading it all into memory
createReadStream('large-file.txt')
  .pipe(createGzip())
  .pipe(createWriteStream('large-file.txt.gz'));
```

**Four types:** Readable, Writable, Duplex (both), Transform (duplex + data transformation).

**Backpressure** — streams communicate flow control automatically. If the writable destination is slower than the readable source, the readable pauses. This prevents memory overflow.

---

## WebSockets

### What are WebSockets and when should you use them?

WebSockets provide a **persistent, full-duplex** TCP connection between client and server. Unlike HTTP (request-response), either side can send data at any time without waiting for the other.

**HTTP vs WebSocket:**

| | HTTP | WebSocket |
|---|------|-----------|
| Connection | New connection per request | Single persistent connection |
| Direction | Client initiates always | Bidirectional |
| Overhead | Headers on every request | Small frame headers after handshake |
| Use case | REST APIs, page loads | Chat, live data, gaming, collaboration |

**Handshake process** — WebSocket upgrades an HTTP connection:

```
Client → Server:  GET /ws HTTP/1.1
                  Upgrade: websocket
                  Connection: Upgrade
                  Sec-WebSocket-Key: dGhlIHNhbXBsZQ==

Server → Client:  HTTP/1.1 101 Switching Protocols
                  Upgrade: websocket
                  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

After the 101 response, the connection is a WebSocket — HTTP is no longer used.

### How do you implement WebSockets in Node.js?

```js
import { WebSocketServer } from 'ws';

const wss = new WebSocketServer({ port: 8080 });

wss.on('connection', (ws, req) => {
  console.log('Client connected');

  ws.on('message', (data) => {
    // Broadcast to all connected clients
    wss.clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(data.toString());
      }
    });
  });

  ws.on('close', () => console.log('Client disconnected'));
  ws.on('error', (err) => console.error(err));

  ws.send(JSON.stringify({ type: 'welcome', message: 'Connected!' }));
});
```

**Client side:**

```js
const ws = new WebSocket('ws://localhost:8080');

ws.onopen = () => ws.send(JSON.stringify({ type: 'hello' }));
ws.onmessage = (event) => console.log(JSON.parse(event.data));
ws.onclose = () => console.log('Disconnected');
```

### WebSockets vs Server-Sent Events (SSE) vs Long Polling

| | WebSocket | SSE | Long Polling |
|---|-----------|-----|-------------|
| Direction | Bidirectional | Server → Client only | Client polls |
| Protocol | WS/WSS | HTTP | HTTP |
| Browser support | All | All (no IE) | All |
| Complexity | Higher | Low | Low |
| Use case | Chat, gaming | Live feeds, notifications | Simple updates |

**Rule of thumb:** Use SSE when you only need server-to-client updates (simpler, works over HTTP/2). Use WebSockets when the client also needs to send frequent messages.

---

## MongoDB Queries

### What are the essential MongoDB query patterns?

**Basic CRUD:**

```js
// Insert
await db.collection('users').insertOne({ name: 'Ngoc', age: 27 });
await db.collection('users').insertMany([{ name: 'A' }, { name: 'B' }]);

// Find
await db.collection('users').findOne({ name: 'Ngoc' });
await db.collection('users').find({ age: { $gte: 25 } }).toArray();

// Update
await db.collection('users').updateOne(
  { name: 'Ngoc' },
  { $set: { age: 28 }, $inc: { loginCount: 1 } }
);

// Delete
await db.collection('users').deleteOne({ _id: id });
```

**Query operators:**

| Operator | Meaning | Example |
|----------|---------|---------|
| `$eq` / `$ne` | Equal / Not equal | `{ age: { $ne: 0 } }` |
| `$gt` / `$gte` / `$lt` / `$lte` | Comparisons | `{ age: { $gte: 18 } }` |
| `$in` / `$nin` | In / Not in array | `{ status: { $in: ['active', 'pending'] } }` |
| `$and` / `$or` / `$nor` | Logical | `{ $or: [{ a: 1 }, { b: 2 }] }` |
| `$exists` | Field exists | `{ email: { $exists: true } }` |
| `$regex` | Pattern match | `{ name: { $regex: /^Ngoc/i } }` |

### How do you write aggregation pipelines in MongoDB?

The aggregation pipeline processes documents through sequential stages:

```js
const result = await db.collection('orders').aggregate([
  // Stage 1: Filter
  { $match: { status: 'completed', createdAt: { $gte: new Date('2026-01-01') } } },

  // Stage 2: Group and compute
  {
    $group: {
      _id: '$customerId',
      totalSpent: { $sum: '$amount' },
      orderCount: { $count: {} },
      avgOrder: { $avg: '$amount' },
    },
  },

  // Stage 3: Add computed field
  { $addFields: { tier: { $cond: [{ $gte: ['$totalSpent', 1000] }, 'gold', 'standard'] } } },

  // Stage 4: Sort
  { $sort: { totalSpent: -1 } },

  // Stage 5: Limit
  { $limit: 10 },

  // Stage 6: Lookup (JOIN)
  {
    $lookup: {
      from: 'customers',
      localField: '_id',
      foreignField: '_id',
      as: 'customer',
    },
  },

  // Stage 7: Unwind array from lookup
  { $unwind: '$customer' },

  // Stage 8: Shape the output
  {
    $project: {
      _id: 0,
      customerId: '$_id',
      name: '$customer.name',
      totalSpent: 1,
      orderCount: 1,
      tier: 1,
    },
  },
]).toArray();
```

### How do indexes improve MongoDB performance?

Without an index, MongoDB performs a **collection scan** — reading every document. Indexes store a sorted subset of fields, allowing fast lookup.

```js
// Single field index
await db.collection('users').createIndex({ email: 1 }); // 1 = ascending

// Compound index — order matters (follows ESR rule: Equality, Sort, Range)
await db.collection('orders').createIndex({ customerId: 1, createdAt: -1 });

// Text index for full-text search
await db.collection('posts').createIndex({ title: 'text', body: 'text' });

// Unique index
await db.collection('users').createIndex({ email: 1 }, { unique: true });

// Sparse index — only indexes documents where the field exists
await db.collection('users').createIndex({ phone: 1 }, { sparse: true });
```

**Use `explain()` to verify index usage:**

```js
const plan = await db.collection('users')
  .find({ email: 'ngoc@example.com' })
  .explain('executionStats');

console.log(plan.queryPlanner.winningPlan.inputStage.stage); // "IXSCAN" = index used
// vs "COLLSCAN" = no index, full scan
```

---

## Clustering Basics

### How does Node.js clustering work?

Node.js runs on a single CPU core by default. The `cluster` module lets you fork multiple worker processes (one per CPU core) that all share the same server port. The OS kernel distributes incoming connections across workers.

```js
import cluster from 'node:cluster';
import { cpus } from 'node:os';
import { createServer } from 'node:http';

if (cluster.isPrimary) {
  const numCPUs = cpus().length;
  console.log(`Primary ${process.pid} spawning ${numCPUs} workers`);

  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code) => {
    console.log(`Worker ${worker.process.pid} died (code ${code}). Restarting...`);
    cluster.fork(); // auto-restart
  });
} else {
  createServer((req, res) => {
    res.end(`Handled by worker ${process.pid}`);
  }).listen(3000);

  console.log(`Worker ${process.pid} started`);
}
```

**What this gives you:**
- Utilizes all CPU cores (a 4-core machine runs 4 Node.js processes)
- Each worker is isolated — a crash in one worker doesn't kill the others
- The primary process monitors and restarts crashed workers

### How does load balancing work in the cluster module?

By default (`SCHED_RR` on most systems), the primary process uses **round-robin** — connections are distributed evenly across workers. On Windows, the default is `SCHED_NONE` (the OS handles distribution).

```js
import cluster from 'node:cluster';
cluster.schedulingPolicy = cluster.SCHED_RR; // explicitly set round-robin
```

### Cluster vs Worker Threads vs PM2

| | `cluster` | `worker_threads` | PM2 |
|---|-----------|-----------------|-----|
| Purpose | Multiple processes, shared port | CPU-intensive tasks in same process | Process manager (production) |
| Isolation | Full process isolation | Shared memory possible | Full process isolation |
| Communication | IPC messages | SharedArrayBuffer + messages | IPC |
| Use case | Scale I/O server across CPUs | CPU-bound work (parsing, crypto) | Production deployment |

**PM2 in production:**

```bash
pm2 start app.js -i max    # cluster mode: one process per CPU
pm2 start app.js -i 4      # exactly 4 processes
pm2 monit                  # real-time monitoring
pm2 logs                   # aggregate logs from all instances
```

PM2 adds: zero-downtime reload, automatic restart on crash, log management, and a web dashboard — making it the standard choice for production Node.js deployments.

### What is the difference between horizontal and vertical scaling?

**Vertical scaling** — upgrade the hardware (more RAM, faster CPU). Simple but has a ceiling and is expensive.

**Horizontal scaling** — run more instances (more processes or more machines). Cheaper, no ceiling, but requires load balancing and stateless application design.

For Node.js:
- **Single machine** — `cluster` or PM2 to use all cores
- **Multiple machines** — containerize with Docker, orchestrate with Kubernetes, use a reverse proxy (Nginx, HAProxy) or cloud load balancer to distribute traffic

**Stateless requirement:** For horizontal scaling to work, sessions and shared state must not live in process memory. Use Redis for sessions, a database for shared state. This way any instance can handle any request.
