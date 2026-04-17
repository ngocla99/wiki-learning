# Messaging and Integration Patterns

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Messaging patterns are the backbone of distributed systems. They define how independent services communicate, share data, and coordinate work across network boundaries. This article covers the fundamental messaging concepts and three key exchange patterns -- **Publish/Subscribe**, **Task Distribution**, and **Request/Reply** -- implemented with three contrasting technologies: **Redis** (simple broker), **RabbitMQ/AMQP** (enterprise broker), and **ZeroMQ** (peer-to-peer).

See [Callbacks and Events](callbacks-and-events.md) for the Observer pattern foundation that Pub/Sub extends into the distributed world.

---

## 1. Fundamentals of Messaging Systems

### Communication Direction

| Pattern | Description | Example |
|---------|-------------|---------|
| **One-way** | Message flows from source to destination with no response expected | Email, server-sent events, task distribution to workers |
| **Request/Reply** | Each request is matched by a correlated response | HTTP calls, database queries |

Request/Reply becomes complex in multi-node async systems where the request and reply may travel through different nodes. The key differentiator from a bare one-way loop is the **correlation** between request and reply maintained by the initiator.

### Message Types

- **Command message** -- Triggers an action on the receiver (e.g., REST verbs: GET, POST, PUT, DELETE). Contains the operation name and arguments. Used for RPC and distributed computation.
- **Event message** -- Notifies that something has occurred. Contains event type and context (who, where, when). Enables loose coupling between components. Example: a chat message event broadcast to all connected users.
- **Document message** -- Transfers data with no instructions attached. Often the reply to a command message (e.g., database query results).

### Push vs Pull Delivery

| Aspect | Pull (consumer-initiated) | Push (producer-initiated) |
|--------|--------------------------|--------------------------|
| **Control** | Consumer decides when to fetch | Producer sends proactively |
| **Latency** | Higher (polling interval) | Lower (real-time) |
| **Complexity** | Simpler, easier to debug | Requires connection management, retries, backpressure |
| **Examples** | HTTP requests, SQL queries, AWS SQS | Webhooks, RabbitMQ push, WebSocket feeds |

**Hybrid approaches** are common: long polling for near real-time, or push notifications to signal changes followed by pull for full data.

### Asynchronous Messaging, Queues, and Streams

**Synchronous** = phone call (both parties must be connected simultaneously).
**Asynchronous** = SMS (sender and receiver decoupled in time).

Two key data structures enable async messaging:

**Message Queue (MQ)**
- Temporarily holds messages until the consumer is ready
- Messages are removed once consumed
- Typically point-to-point: each message delivered to a single consumer
- Ideal for task distribution, advanced routing, prioritization

**Stream (Log)**
- Append-only, durable sequence of records
- Messages remain after consumption (not deleted)
- Multi-subscriber friendly: many consumers read independently from different positions
- Ideal for high-volume sequential data, replay, and rebuilding state

### Peer-to-Peer vs Broker-Based Messaging

**Broker-based** (e.g., RabbitMQ, Redis):
- Decouples sender from receiver completely
- Provides persistent queues, routing, message transformations, monitoring
- Can bridge different protocols (AMQP, MQTT, STOMP)
- Trade-off: single point of failure, must be scaled separately, adds latency

**Peer-to-peer** (e.g., ZeroMQ):
- No single point of failure
- Only need to scale individual application nodes
- Lower latency (no intermediary)
- Trade-off: nodes must know each other's addresses, more implementation effort

---

## 2. Publish/Subscribe Pattern

Pub/Sub is a distributed Observer pattern. Publishers produce messages; subscribers register interest in specific categories. The publisher does not know who the recipients are -- subscribers opt in.

See [Callbacks and Events](callbacks-and-events.md) for the local Observer pattern that Pub/Sub extends.

### Example: Real-Time Chat Application

A minimal chat app demonstrates Pub/Sub. A WebSocket server broadcasts every incoming message to all connected clients:

```js
import { createServer } from 'node:http'
import { WebSocketServer } from 'ws'
import staticHandler from 'serve-handler'

const server = createServer((req, res) =>
  staticHandler(req, res, { public: 'web' })
)
const wss = new WebSocketServer({ server })

wss.on('connection', client => {
  client.on('message', msg => broadcast(msg))
})

function broadcast(msg) {
  for (const client of wss.clients) {
    if (client.readyState === WebSocket.OPEN) {
      client.send(msg)
    }
  }
}

server.listen(8080)
```

**Scaling problem**: Running multiple server instances means clients on different servers cannot communicate -- messages are only broadcast locally. This is where a messaging system comes in.

### Using Redis as a Message Broker

Redis Pub/Sub provides the simplest broker integration. Each server instance publishes to and subscribes from a Redis channel:

```js
import Redis from 'ioredis'

const redisPub = new Redis()
const redisSub = new Redis() // separate connection required for subscriber mode

// When a WebSocket message arrives, publish to Redis
client.on('message', msg => {
  redisPub.publish('chat_messages', msg)
})

// Subscribe to receive messages from all server instances
redisSub.subscribe('chat_messages')
redisSub.on('message', (channel, msg) => {
  broadcast(Buffer.from(msg))
})
```

**Key points**:
- Two Redis connections are needed (subscriber mode blocks other commands)
- Redis supports glob-style pattern subscriptions (e.g., `chat.*`)
- Fire-and-forget: no persistence, messages lost if subscriber is offline

### Peer-to-Peer Pub/Sub with ZeroMQ

ZeroMQ removes the broker entirely. Each instance uses **PUB** and **SUB** sockets:

- `PUB` socket binds to a port and broadcasts messages to connected subscribers
- `SUB` socket connects to remote PUB sockets and filters by topic

```js
import zmq from 'zeromq'

// Publisher socket
const pubSocket = new zmq.Publisher()
await pubSocket.bind(`tcp://127.0.0.1:${args.pub}`)

// Subscriber socket -- connect to other instances' PUB ports
const subSocket = new zmq.Subscriber()
for (const port of args.sub) {
  await subSocket.connect(`tcp://127.0.0.1:${port}`)
}
subSocket.subscribe('chat_messages')

// Receive from other servers
async function receiveMessages() {
  for await (const [_topic, msg] of subSocket) {
    broadcast(Buffer.from(msg))
  }
}
receiveMessages()

// Publish when a client sends a message
client.on('message', msg => {
  broadcast(msg) // local broadcast
  pubSocket.send(['chat_messages', msg]) // publish to peers
})
```

**Running 3 instances**:
```bash
node index.js --http 8080 --pub 5000 --sub 5001 --sub 5002
node index.js --http 8081 --pub 5001 --sub 5000 --sub 5002
node index.js --http 8082 --pub 5002 --sub 5000 --sub 5001
```

**ZeroMQ resilience**: Does not throw errors if a peer is unavailable -- it retries connections automatically.

### Technology Comparison for Pub/Sub

| Feature | Redis | ZeroMQ | RabbitMQ (AMQP) |
|---------|-------|--------|-----------------|
| Architecture | Broker-based | Peer-to-peer | Broker-based |
| Setup complexity | Low | Medium (manual wiring) | Medium (exchange/queue setup) |
| Durability | None (fire-and-forget) | None (fire-and-forget) | Configurable (durable queues) |
| Latency | Low | Lowest | Moderate |
| Scalability | Scale the broker | Scale individual nodes | Scale the broker |

---

## 3. Reliable Message Delivery with Queues

### Delivery Semantics

| Semantic | Behavior | Trade-off |
|----------|----------|-----------|
| **At most once** | Fire-and-forget. No persistence or acknowledgment. Messages may be lost. | Simplest, fastest |
| **At least once** | Message guaranteed delivered but duplicates possible (e.g., crash after processing but before ACK). | Requires persistence + ACK |
| **Exactly once** | Delivered once and only once. | Slowest, most complex |

A **durable subscriber** reliably receives all messages, even those sent while disconnected. Requires "at least once" or "exactly once" semantics backed by a message queue.

### AMQP (Advanced Message Queuing Protocol)

AMQP is an open standard with three core components:

**Queue** -- Stores messages. Can be:
- *Durable*: survives broker restart (but only persistent messages are preserved)
- *Exclusive*: bound to one connection, destroyed when it closes
- *Auto-delete*: deleted when last subscriber disconnects

**Exchange** -- Receives published messages and routes them to queues:
- *Direct*: routes by exact routing key match (e.g., `chat.msg`)
- *Topic*: routes by glob pattern (e.g., `chat.#` matches `chat.anything`)
- *Fan-out*: broadcasts to all bound queues (ignores routing key)

**Binding** -- Links exchanges to queues with optional routing key/pattern filter.

### Durable Subscribers with RabbitMQ

Example: A **history service** that persists chat messages to a database. It must never lose messages (durable subscriber), while the chat servers themselves only need fire-and-forget (exclusive queues).

**History service** (durable queue, explicit ACK):
```js
import amqp from 'amqplib'
import { Level } from 'level'

const connection = await amqp.connect('amqp://localhost')
const channel = await connection.createChannel()

await channel.assertExchange('chat', 'fanout')
const { queue } = channel.assertQueue('chat_history') // durable by default
await channel.bindQueue(queue, 'chat')

channel.consume(queue, async msg => {
  const data = JSON.parse(msg.content.toString())
  await db.put(ulid(), data)
  channel.ack(msg) // ACK only after successful persistence
})
```

**Chat server** (exclusive queue, no ACK needed):
```js
const { queue } = await channel.assertQueue(`chat_srv_${httpPort}`, {
  exclusive: true // destroyed when connection closes
})
await channel.bindQueue(queue, 'chat')

channel.consume(queue, msg => {
  broadcast(Buffer.from(msg.content.toString()))
}, { noAck: true })

// Publish messages to the fan-out exchange
channel.publish('chat', '', Buffer.from(JSON.stringify({
  text: msg.toString(),
  timestamp: Date.now()
})))
```

**Microservice resilience**: If the history service goes down, messages accumulate in its durable queue. When it restarts, it processes all missed messages. Meanwhile, the chat servers continue operating (reduced functionality, but no downtime).

---

## 4. Reliable Messaging with Streams

### Streams vs Queues

| Aspect | Message Queue | Stream |
|--------|--------------|--------|
| Consumption | Messages removed after delivery | Messages retained (append-only log) |
| Subscribers | Typically single consumer per message | Multiple consumers read independently |
| Delivery | Push-based | Pull-based (consumers control pace) |
| History | No replay | Replay from any point |
| Best for | Task distribution, complex routing | High-volume sequential data, audit logs, event sourcing |

### Redis Streams

Redis Streams implement an append-only log. Key commands:

- **`XADD`** -- Append a record. Use `*` as ID for auto-generation (monotonic, time-based).
- **`XRANGE`** -- Query records by ID range. `-` = lowest, `+` = highest.
- **`XREAD`** -- Read new records, optionally blocking. `$` = start after the latest current record.

```js
import Redis from 'ioredis'

const redisClient = new Redis()
const redisClientXread = new Redis()

// Publish a message to the stream
redisClient.xadd('chat_stream', '*', 'message', JSON.stringify({
  text: msg.toString(),
  timestamp: Date.now()
}))

// Load chat history (all past messages)
const logs = await redisClient.xrange('chat_stream', '-', '+')
for (const [, [, message]] of logs) {
  client.send(Buffer.from(message))
}

// Listen for new messages (blocking)
let lastRecordId = '$'
async function processStreamMessages() {
  while (true) {
    const [[, records]] = await redisClientXread.xread(
      'BLOCK', '0', 'STREAMS', 'chat_stream', lastRecordId
    )
    for (const [recordId, [, message]] of records) {
      broadcast(Buffer.from(message))
      lastRecordId = recordId
    }
  }
}
```

**Key advantage over the AMQP approach**: No separate history service needed. The stream itself is the history -- just query past records with `XRANGE`.

Records can be trimmed with `XDEL`, `XTRIM`, or the `MAXLEN` option of `XADD`.

---

## 5. Task Distribution Patterns

Task distribution fans out work to multiple remote workers for parallel processing. Unlike Pub/Sub (where every subscriber gets every message), each task goes to exactly one worker.

See [Scalability Patterns](scalability-patterns.md) for microservice architecture and horizontal scaling strategies.

### ZeroMQ Fan-Out/Fan-In (PUSH/PULL Sockets)

**PUSH/PULL socket properties**:
- Messages always travel from PUSH to PULL (one-way)
- Either side can bind or connect (bind = durable node, connect = transient node)
- Multiple PULL sockets connected to one PUSH socket = **automatic load balancing** (round-robin)
- Messages sent to a PUSH socket with no connected PULL sockets are queued (not lost)
- Built-in **backpressure**: `send()` blocks when the internal high water mark is reached

**Architecture**: Ventilator (PUSH) -> Workers (PULL/PUSH) -> Sink (PULL)

```js
// producer.js -- Ventilator
const ventilator = new zmq.Push()
await ventilator.bind('tcp://*:5016')

for (const task of generateTasks(searchHash, ALPHABET, maxLength, BATCH_SIZE)) {
  await ventilator.send(task) // backpressure-aware
}

// worker.js
const fromVentilator = new zmq.Pull()
const toSink = new zmq.Push()
fromVentilator.connect('tcp://localhost:5016') // connect (transient)
toSink.connect('tcp://localhost:5017')

for await (const rawMessage of fromVentilator) {
  const found = processTask(JSON.parse(rawMessage.toString()))
  if (found) {
    await toSink.send(`Found: ${found}`)
    break
  }
}

// collector.js -- Sink
const sink = new zmq.Pull()
await sink.bind('tcp://*:5017') // bind (durable)

for await (const rawMessage of sink) {
  console.log('Result:', rawMessage.toString())
}
```

**Caveat**: No built-in message acknowledgment. If a worker crashes, its in-flight tasks are lost.

### AMQP Pipelines and Competing Consumers

In AMQP, task distribution uses **point-to-point** communication (bypassing exchanges, sending directly to a queue). Multiple workers consuming from the same queue become **competing consumers** -- the broker load-balances messages between them.

```js
// producer.js
const channel = await connection.createConfirmChannel()
await channel.assertQueue('tasks_queue')

for (const task of generateTasks(...)) {
  await channel.sendToQueue('tasks_queue', Buffer.from(task))
}
await channel.waitForConfirms() // wait for broker to confirm reception

// worker.js
const { queue } = await channel.assertQueue('tasks_queue')
channel.prefetch(1) // process one message at a time

channel.consume(queue, async rawMessage => {
  const found = processTask(JSON.parse(rawMessage.content.toString()))
  await channel.ack(rawMessage) // ACK after processing
  if (found) {
    await channel.sendToQueue('results_queue', Buffer.from(`Found: ${found}`))
  }
})
```

**AMQP vs ZeroMQ trade-off**: AMQP runs slightly slower due to broker overhead, but provides automatic re-delivery if a worker crashes (unacknowledged messages return to the queue). ZeroMQ requires manual implementation of acknowledgment/retry logic.

### Redis Streams with Consumer Groups

**Consumer groups** implement the Competing Consumer pattern on top of Redis Streams:
- A named group of consumers reads from a stream in round-robin
- Each record must be explicitly acknowledged (`XACK`)
- Unacknowledged records stay in a **pending list** per consumer
- On restart, a consumer retrieves its pending records first before consuming new ones

```js
// Create consumer group (idempotent with catch)
await redisClient
  .xgroup('CREATE', 'tasks_stream', 'workers_group', '$', 'MKSTREAM')
  .catch(() => console.log('Consumer group already exists'))

// Step 1: Process pending records from a previous crash
const [[, pendingRecords]] = await redisClient.xreadgroup(
  'GROUP', 'workers_group', consumerName,
  'STREAMS', 'tasks_stream', '0' // '0' = all pending for this consumer
)
for (const [recordId, [, rawTask]] of pendingRecords) {
  await processAndAck(recordId, rawTask)
}

// Step 2: Consume new records
while (true) {
  const [[, records]] = await redisClient.xreadgroup(
    'GROUP', 'workers_group', consumerName,
    'BLOCK', '0', 'COUNT', '1',
    'STREAMS', 'tasks_stream', '>' // '>' = new, undelivered records only
  )
  for (const [recordId, [, rawTask]] of records) {
    await processAndAck(recordId, rawTask)
  }
}

async function processAndAck(recordId, rawTask) {
  const found = processTask(JSON.parse(rawTask))
  if (found) {
    await redisClient.xadd('results_stream', '*', 'result', `Found: ${found}`)
  }
  await redisClient.xack('tasks_stream', 'workers_group', recordId)
}
```

### Task Distribution Technology Comparison

| Feature | ZeroMQ (PUSH/PULL) | RabbitMQ (AMQP) | Redis (Consumer Groups) |
|---------|--------------------|-----------------|------------------------|
| Architecture | Peer-to-peer | Broker-based | Broker-based |
| Load balancing | Built-in round-robin | Competing consumers | Consumer group round-robin |
| Acknowledgment | None (manual impl.) | Built-in (`channel.ack`) | Built-in (`XACK`) |
| Crash recovery | Tasks lost | Automatic re-delivery | Pending list per consumer |
| Backpressure | Built-in (high water mark) | `prefetch` setting | Consumer controls read rate |
| Ordering | FIFO per socket | FIFO per queue | Preserved in stream |

---

## 6. Request/Reply Patterns

When only asynchronous one-way channels are available, request/reply must be built as an abstraction on top.

### Correlation Identifier Pattern

Each request is tagged with a unique ID. The replier copies this ID into the response. The requestor uses it to match replies to pending requests, even if responses arrive out of order.

```js
// createRequestChannel.js
import { nanoid } from 'nanoid'

export function createRequestChannel(channel) {
  const correlationMap = new Map()

  function sendRequest(data) {
    return new Promise((resolve, reject) => {
      const correlationId = nanoid()

      const replyTimeout = setTimeout(() => {
        correlationMap.delete(correlationId)
        reject(new Error('Request timeout'))
      }, 10000)

      correlationMap.set(correlationId, replyData => {
        correlationMap.delete(correlationId)
        clearTimeout(replyTimeout)
        resolve(replyData)
      })

      channel.send({ type: 'request', data, id: correlationId })
    })
  }

  channel.on('message', message => {
    const handler = correlationMap.get(message.inReplyTo)
    if (handler) handler(message.data)
  })

  return sendRequest
}

// createReplyChannel.js
export function createReplyChannel(channel) {
  return function registerHandler(handler) {
    channel.on('message', async message => {
      if (message.type !== 'request') return
      const replyData = await handler(message.data)
      channel.send({ type: 'response', data: replyData, inReplyTo: message.id })
    })
  }
}
```

This works over any duplex channel (WebSocket, `child_process.fork()` IPC, etc.).

### Return Address Pattern with AMQP

When multiple queues or requestors are involved, the replier needs to know **where** to send the response. Each requestor creates a **private, exclusive queue** as its return address.

**Request abstraction**:
```js
export class AmqpRequest {
  constructor() { this.correlationMap = new Map() }

  async initialize() {
    this.connection = await amqp.connect('amqp://localhost')
    this.channel = await this.connection.createChannel()

    // Create anonymous exclusive queue for replies
    const { queue } = await this.channel.assertQueue('', { exclusive: true })
    this.replyQueue = queue

    // Listen for replies and match by correlation ID
    this.channel.consume(this.replyQueue, msg => {
      const handler = this.correlationMap.get(msg.properties.correlationId)
      if (handler) handler(JSON.parse(msg.content.toString()))
    }, { noAck: true })
  }

  send(queue, message) {
    return new Promise((resolve, reject) => {
      const id = nanoid()
      const timeout = setTimeout(() => {
        this.correlationMap.delete(id)
        reject(new Error('Request timeout'))
      }, 10000)

      this.correlationMap.set(id, data => {
        this.correlationMap.delete(id)
        clearTimeout(timeout)
        resolve(data)
      })

      this.channel.sendToQueue(queue, Buffer.from(JSON.stringify(message)), {
        correlationId: id,
        replyTo: this.replyQueue // return address
      })
    })
  }
}
```

**Reply abstraction**:
```js
export class AmqpReply {
  async initialize() {
    const connection = await amqp.connect('amqp://localhost')
    this.channel = await connection.createChannel()
    const { queue } = await this.channel.assertQueue(this.requestsQueueName)
    this.queue = queue
  }

  handleRequests(handler) {
    this.channel.consume(this.queue, async msg => {
      const content = JSON.parse(msg.content.toString())
      const replyData = await handler(content)
      this.channel.sendToQueue(
        msg.properties.replyTo, // send to requestor's private queue
        Buffer.from(JSON.stringify(replyData)),
        { correlationId: msg.properties.correlationId }
      )
      this.channel.ack(msg)
    })
  }
}
```

**Scalability bonus**: Multiple repliers on the same requests queue become competing consumers -- the broker automatically load-balances requests across them.

---

## Summary: Choosing the Right Pattern and Technology

| Need | Pattern | Recommended Technology |
|------|---------|----------------------|
| Real-time broadcast to many subscribers | Pub/Sub | Redis (simple), ZeroMQ (low latency), RabbitMQ (durable) |
| Never lose messages while subscriber is offline | Durable Subscriber | RabbitMQ (AMQP queues), Redis Streams |
| Distribute tasks to parallel workers | Task Distribution / Competing Consumers | RabbitMQ (reliable), ZeroMQ (fast, peer-to-peer), Redis Consumer Groups |
| Request/Reply over async channels | Correlation Identifier + Return Address | AMQP (built-in properties), any duplex channel |
| Event replay / audit log | Stream consumption | Redis Streams, Apache Kafka |
| Lowest latency, no single point of failure | Peer-to-peer messaging | ZeroMQ |
| Complex routing, protocol bridging | Broker with exchange types | RabbitMQ (AMQP) |

**General guidance**: Use brokers (Redis, RabbitMQ) for simplicity and reliability. Use peer-to-peer (ZeroMQ) for maximum performance and resilience. For production streaming at scale, consider Apache Kafka or Amazon Kinesis.
