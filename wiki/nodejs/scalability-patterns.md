# Scalability and Architectural Patterns

> Sources: Luciano Mammino & Mario Casciaro, 2024
> Raw: [Node.js Design Patterns](../../raw/nodejs/design-patterns.md)

## Overview

Scalability is the ability of a system to grow and adapt to changing conditions -- not only technical capacity but also the growth of teams and organizations. Node.js runs application code on a single thread by default, which is not a limitation but a push toward thinking about scalability from the start. With its non-blocking I/O model, a single Node.js process can handle hundreds or thousands of concurrent I/O-heavy requests, but to go further you need to scale across multiple processes and machines.

This chapter covers two primary scaling strategies: **cloning** (X-axis) and **decomposition by service** (Y-axis), along with the tools and architectural patterns that support them.

## The Scale Cube -- Three Dimensions of Scalability

The **Scale Cube** model (from *The Art of Scalability* by Abbott & Fisher) categorizes load distribution across three dimensions:

```
                    Y-axis
                    Decompose by service
                    ^
                   /
                  /
                 /
                +-------------------> X-axis
               /|                    Clone
              / |
             /  |
            v   |
           Z-axis
           Split by data partition
```

| Axis | Strategy | Description |
|------|----------|-------------|
| **X-axis** | Cloning | Duplicate the same application across multiple instances. Each handles a portion of the total workload. Simplest to implement. |
| **Y-axis** | Decompose by service | Split by functionality/use case into independent services (microservices). Each service has its own codebase and possibly its own database. |
| **Z-axis** | Split by data partition | Each instance handles only a portion of total data. Uses list, range, or hash partitioning. Requires a lookup mechanism. |

The bottom-left corner of the cube represents a monolithic single-instance application. As growth demands, you move along one or more axes. These dimensions are **not mutually exclusive** -- most real-world systems scale along multiple axes over time.

**Common journey:** Monolith (early stage) -> X-axis cloning (handle more traffic) -> Y-axis decomposition (reduce complexity, enable team autonomy) -> Z-axis partitioning (massive data volumes).

## Cloning and Load Balancing

### Vertical vs. Horizontal Scaling

| Type | Approach | Analogy |
|------|----------|---------|
| **Vertical** | Upgrade hardware (CPU, memory, disk) | Replacing a van with a truck |
| **Horizontal** | Run multiple instances across processes/machines | Using a fleet of vehicles |

Node.js naturally favors horizontal scaling because a single process uses only one CPU core. Running multiple processes is the standard way to utilize all available cores.

### The cluster Module

The simplest way to distribute load across multiple processes on a single machine. The primary process forks workers, and incoming connections are distributed across them automatically.

```js
import { createServer } from 'node:http'
import { cpus } from 'node:os'
import cluster from 'node:cluster'

if (cluster.isPrimary) {
  const availableCpus = cpus()
  console.log(`Clustering to ${availableCpus.length} processes`)
  for (const _ of availableCpus) {
    cluster.fork()
  }
} else {
  const server = createServer((_req, res) => {
    let i = 1e7; while (i > 0) { i-- } // simulate CPU work
    res.end(`Hello from ${process.pid}\n`)
  })
  server.listen(8080, () => console.log(`Started at ${process.pid}`))
}
```

**Key pattern:**

```js
if (cluster.isPrimary) {
  // fork workers
} else {
  // do actual work
}
```

Each worker is a **separate Node.js process** with its own event loop, memory space, and loaded modules. Under the hood, `cluster.fork()` uses `child_process.fork()`, so a communication channel is available between primary and workers.

**Behavior notes:**

- Most systems use **round-robin** load balancing (default on all platforms except Windows). Configurable via `cluster.schedulingPolicy`.
- `server.listen()` in workers is **delegated** to the primary process.
- `server.listen({fd: N})` -- file descriptors are per-process; create the FD in the primary and pass it to workers.
- `server.listen(handle)` -- uses the handle directly, bypassing primary delegation.
- `server.listen(0)` -- the "random" port is only random the first time; subsequent calls get the same port.

**Performance:** On an 8-core machine, clustering yields ~5.3x throughput (1,600 req/sec vs 300 req/sec for a single process).

### Resiliency and Availability

Multiple instances create a redundant system. If one crashes, others continue serving. Auto-restart crashed workers:

```js
if (cluster.isPrimary) {
  // ... fork workers ...
  cluster.on('exit', (worker, code) => {
    if (code !== 0 && !worker.exitedAfterDisconnect) {
      console.log(`Worker ${worker.process.pid} crashed. Starting a new one`)
      cluster.fork()
    }
  })
}
```

Under stress testing with frequent random crashes, ~99.7% of requests still complete successfully. Most failures are in-flight requests interrupted by abrupt process termination.

> **Idempotency matters:** When clients retry failed requests, operations that modify data (payments, orders) must be designed so retries are safe. Use techniques such as idempotent handling, request identifiers, transactions, or the Saga pattern for distributed transactions.

### Zero-Downtime Restart

Restart workers one at a time so the remaining ones continue serving requests during deployment:

```js
import { once } from 'node:events'

if (cluster.isPrimary) {
  // ... fork workers ...
  process.on('SIGUSR2', async () => {
    const workers = Object.values(cluster.workers)
    for (const worker of workers) {
      console.log(`Stopping worker: ${worker.process.pid}`)
      worker.disconnect()           // graceful stop (finish in-flight requests)
      await once(worker, 'exit')
      if (!worker.exitedAfterDisconnect) continue
      const newWorker = cluster.fork()
      await once(newWorker, 'listening') // wait until ready before next
    }
  })
}
```

Trigger with `kill -SIGUSR2 <primary-PID>`. The process is sequential: stop one worker, wait for it to exit, start a replacement, wait for it to be ready, then move to the next.

> **pm2** (`npm install pm2 -g`) is a popular utility built on `node:cluster` that provides load balancing, process monitoring, and zero-downtime restarts out of the box.

### Stateful Communications

The cluster module (and any stateless load balancer) does not work well with stateful sessions stored in memory, because different requests from the same user may hit different instances.

```
User logs in -> Instance A stores session in memory
User's next request -> routed to Instance B -> no session found -> rejected!
```

**Solution 1: Shared state** (preferred)

Store session data in a shared datastore accessible by all instances:

```
┌──────────┐     ┌──────────┐     ┌──────────┐
│Instance A│     │Instance B│     │Instance C│
└────┬─────┘     └────┬─────┘     └────┬─────┘
     │                │                │
     └────────┬───────┘────────┬───────┘
              │                │
         ┌────▼────────────────▼────┐
         │   Redis / PostgreSQL /   │
         │   Memcached / MongoDB    │
         └──────────────────────────┘
```

**Solution 2: Sticky load balancing** (less recommended)

Route all requests from the same session to the same instance using a session cookie or IP hash. Available via the `sticky-session` npm package.

**Drawbacks of sticky sessions:**
- Reduces redundancy benefits (session tied to one instance)
- Makes failover harder
- Not supported by `node:cluster` by default

**Best approach:** Design stateless applications. Use **JWTs** or include all necessary info in each request so the server needs no session memory.

## Scaling with a Reverse Proxy

A reverse proxy sits between clients and application instances, distributing traffic across them. Unlike `node:cluster`, it can balance across **multiple machines**, not just processes.

```
                    ┌──────────┐
                    │  Client  │
                    └────┬─────┘
                         │
                ┌────────▼────────┐
                │  Reverse Proxy  │
                │  (Load Balancer)│
                └──┬─────┬─────┬──┘
                   │     │     │
          ┌────────▼┐ ┌──▼───┐ ┌▼────────┐
          │ App :8081│ │:8082 │ │ :8083   │
          │ Machine 1│ │Mach 1│ │Machine 2│
          └──────────┘ └──────┘ └─────────┘
```

**Advantages over `node:cluster`:**

- Distribute across multiple machines (not just processes)
- Sticky load balancing out of the box
- Route to any server regardless of language/platform
- More powerful load-balancing algorithms
- URL rewrites, caching, SSL termination, DoS protection, serving static files

**Popular reverse proxy options:**

| Solution | Description |
|----------|-------------|
| **Nginx** | Web server/reverse proxy/load balancer, non-blocking I/O |
| **HAProxy** | Fast TCP/HTTP load balancer |
| **Caddy** | Web server in Go with automatic HTTPS |
| **Node.js proxies** | Custom reverse proxies (e.g., `httpxy`) |
| **Cloud-based** | AWS ALB/ELB, GCP Load Balancer, etc. |

> You can combine both: use `node:cluster` to scale vertically within a machine, and a reverse proxy to scale horizontally across machines.

### Nginx Load Balancing

Start multiple app instances on different ports (supervised by pm2):

```bash
pm2 start --name 'app1' app.js -- 8081
pm2 start --name 'app2' app.js -- 8082
pm2 start --name 'app3' app.js -- 8083
pm2 start --name 'app4' app.js -- 8084
```

Minimal `nginx.conf`:

```nginx
daemon off;
error_log /dev/stderr info;

events {
  worker_connections 2048;
}

http {
  access_log /dev/stdout;

  upstream my-load-balanced-app {
    server 127.0.0.1:8081;
    server 127.0.0.1:8082;
    server 127.0.0.1:8083;
    server 127.0.0.1:8084;
  }

  server {
    listen 8080;
    location / {
      proxy_pass http://my-load-balanced-app;
    }
  }
}
```

Start Nginx: `nginx -c ${PWD}/nginx.conf`

Nginx uses round-robin by default; other algorithms are configurable.

## Dynamic Horizontal Scaling

In cloud environments, application capacity adjusts in real time based on traffic. New instances are provisioned or decommissioned via API. The challenge: the load balancer must always know which servers are currently available.

### Service Registry

A central repository tracking which services are available, how many instances are running, and their network addresses. Each instance **registers** on startup and **unregisters** on shutdown.

```
                  ┌──────────────────┐
                  │  Service Registry│
                  │   (e.g., Consul) │
                  └───────┬──────────┘
          ┌───────────────┼───────────────┐
          │ register/     │ query         │ register/
          │ deregister    │               │ deregister
     ┌────▼────┐    ┌─────▼─────┐   ┌────▼────┐
     │ API svc │    │   Load    │   │ WebApp  │
     │ inst 1  │    │ Balancer  │   │ inst 1  │
     └─────────┘    └───────────┘   └─────────┘
```

### Dynamic Load Balancer with Consul

Service registration (each app instance):

```js
import { createServer } from 'node:http'
import { randomUUID } from 'node:crypto'
import portfinder from 'portfinder'
import { ConsulClient } from './consul.js'

const serviceType = process.argv[2]
const consulClient = new ConsulClient()
const port = await portfinder.getPort()
const serviceId = randomUUID()

// Register on startup
async function registerService() {
  await consulClient.registerService({
    id: serviceId, name: serviceType,
    address: 'localhost', port, tags: [serviceType],
  })
}

// Deregister on shutdown
async function unregisterService(err) {
  err && console.error(err)
  await consulClient.deregisterService(serviceId)
  process.exit(err ? 1 : 0)
}

process.on('uncaughtException', unregisterService)
process.on('SIGINT', unregisterService)

const server = createServer((_req, res) => {
  res.end(`${serviceType} response from ${process.pid}\n`)
})

server.listen(port, 'localhost', () => registerService())
```

Load balancer queries Consul for the current service list and routes with round-robin:

```js
import { createServer } from 'node:http'
import { createProxyServer } from 'httpxy'
import { ConsulClient } from './consul.js'

const routing = [
  { path: '/api', service: 'api-service', index: 0 },
  { path: '/',    service: 'webapp-service', index: 0 },
]

const consulClient = new ConsulClient()
const proxy = createProxyServer()

const server = createServer(async (req, res) => {
  const route = routing.find(r => req.url.startsWith(r.path))
  try {
    const services = await consulClient.getAllServices()
    const servers = Object.values(services)
      .filter(s => s.Tags.includes(route.service))
    if (servers.length > 0) {
      route.index = (route.index + 1) % servers.length
      const target = `http://${servers[route.index].Address}:${servers[route.index].Port}`
      return proxy.web(req, res, { target })
    }
  } catch (err) { console.error(err) }
  res.writeHead(502)
  res.end('Bad gateway')
})

server.listen(8080)
```

> **Optimization:** Cache the service list and refresh periodically (e.g., every 10 seconds) instead of querying Consul on every request. Use Consul's built-in health checks to auto-remove unhealthy services.

## Peer-to-Peer Load Balancing

Instead of routing through a central load balancer, the **client** distributes requests across available server instances directly. Also called **client-side load balancing**.

```
Centralized:                    Peer-to-peer:

  Service A                       Service A
      │                          /    |    \
      ▼                         ▼     ▼     ▼
  Load Balancer              Svc B  Svc B  Svc B
   /    |    \              inst 1  inst 2  inst 3
  ▼     ▼     ▼
Svc B  Svc B  Svc B
```

**Advantages:**
- Removes a network component (no load balancer bottleneck or single point of failure)
- Faster communication (direct to destination)
- Better scalability (not limited by central load balancer performance)

**Trade-offs:**
- Client must implement load-balancing logic
- Client must maintain an up-to-date view of available instances

### Example: Client-Side Balanced HTTP Requests

```js
const servers = [
  { host: 'localhost', port: 8081 },
  { host: 'localhost', port: 8082 },
]
let i = 0

export function balancedRequest(url, fetchOptions = {}) {
  i = (i + 1) % servers.length
  const server = servers[i]
  const rewrittenUrl = new URL(url, `http://${server.host}:${server.port}`)
  rewrittenUrl.host = `${server.host}:${server.port}`
  return fetch(rewrittenUrl.toString(), fetchOptions)
}
```

Usage:

```js
import { balancedRequest } from './balancedRequest.js'

for (let i = 0; i < 10; i++) {
  const response = await balancedRequest(`/?request=${i}`)
  console.log(await response.text()) // alternates between servers
}
```

> An improvement is to integrate a service registry directly into the client to obtain the server list dynamically.

> Peer-to-peer load balancing is used extensively in the ZeroMQ library. See [Messaging Patterns](messaging-patterns.md) for more.

## Containers and Orchestration

### Docker Basics

A **container** packages code and all dependencies so the application runs identically everywhere. Containers run with minimal overhead (nearly native performance) and are highly portable.

**Simple Dockerfile for a Node.js app:**

```dockerfile
FROM node:24-slim
EXPOSE 8080
COPY app.js package.json /app/
WORKDIR /app
CMD ["npm", "start"]
```

```bash
docker build -t hello-web:v1 .          # Build the image
docker run -it -p 8080:8080 hello-web:v1 # Run the container
```

Each container runs in a sandboxed environment, isolated from the host OS by default.

### Kubernetes

A **container orchestration** platform that manages containers across a cluster of machines. Originally developed and open-sourced by Google (2014).

**What Kubernetes provides:**

| Capability | Description |
|------------|-------------|
| **Cluster management** | Combine servers (nodes) into a single managed cluster |
| **High availability** | Auto-restart crashed containers; reschedule if a node goes down |
| **Service discovery** | Built-in service discovery and load balancing |
| **Persistent storage** | Manage storage across restarts and rescheduling |
| **Automated rollouts** | Deploy updates safely with zero downtime |
| **Secrets management** | Inject env vars and sensitive data securely |

Kubernetes uses a **declarative** model: you describe the desired state, and Kubernetes continuously ensures reality matches intent. Key objects include **Deployments** (app specifications) and **Pods** (the basic unit -- a set of containers that run together sharing network/storage).

### Deploying and Scaling on Kubernetes

```bash
# Create a deployment (1 instance)
kubectl create deployment hello-web --image=hello-web:v1

# Expose it via a LoadBalancer
kubectl expose deployment hello-web --type=LoadBalancer --port=8080

# Scale to 5 replicas
kubectl scale --replicas=5 deployment hello-web

# Check status
kubectl get deployments
# NAME        READY   UP-TO-DATE   AVAILABLE
# hello-web   5/5     5            5

kubectl get pods
# NAME                         READY   STATUS    RESTARTS
# hello-web-65f47d9997-df7nr   1/1     Running   0
# hello-web-65f47d9997-g98jb   1/1     Running   0
# ... (5 total)
```

**Rolling update to a new version:**

```bash
docker build -t hello-web:v2 .
kubectl set image deployment/hello-web hello-web=hello-web:v2
# Kubernetes replaces containers one by one, gracefully stopping old ones
```

> **Key takeaway:** With Kubernetes, application code stays simpler -- scaling, rollouts, restarts, and load balancing are handled by the platform. The trade-off is the need to learn and manage the platform itself.

> Containers in Kubernetes are disposable and can be killed/restarted at any time. Design **stateless** applications; persist data in databases or persistent volumes.

**Cleanup:**

```bash
kubectl scale --replicas=0 deployment hello-web
kubectl delete -n default service hello-web
minikube stop
```

## Decomposing Complex Applications

### Monolithic Architecture

A monolith contains all functionality in a single codebase running as a single application. It can still be modular internally (e.g., Products, Cart, Checkout, Search, Auth modules), but:

- A failure in **any** component can bring down the **entire** system
- Tight coupling creeps in easily (e.g., Checkout directly querying Products' database)
- Over time, interdependencies accumulate and the codebase becomes a "Jenga tower"
- Changes ripple unpredictably through shared runtime

### The Microservice Architecture

Break the system into small, self-contained services, each responsible for a specific slice of functionality. Each service has its **own codebase, database, and deployment lifecycle**.

```
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Products   │  │    Cart     │  │  Checkout   │
│  Service    │  │  Service    │  │  Service    │
│  + own DB   │  │  + own DB   │  │  + own DB   │
└─────────────┘  └─────────────┘  └─────────────┘
┌─────────────┐  ┌─────────────┐
│   Search    │  │ Auth/Users  │
│  Service    │  │  Service    │
│  + own DB   │  │  + own DB   │
└─────────────┘  └─────────────┘
```

### Advantages and Disadvantages

| Advantages | Disadvantages |
|------------|---------------|
| **Expendable services** -- crashes don't propagate; one service can be rebuilt from scratch | **Higher integration complexity** -- services must communicate over the network |
| **Independent technology choices** -- each service can use its own language/database | **Deployment overhead** -- many more applications to deploy, scale, and monitor |
| **Reusability** -- services expose remote interfaces with high information hiding | **Code sharing challenges** -- common logic must be distributed as packages |
| **Independent scaling** -- each service scales based on its own traffic demands | **Data consistency** -- no shared database means more effort to keep data in sync |
| **Team autonomy** -- different teams own different services, move independently | **Operational complexity** -- service discovery, health checks, logging, tracing |

> "Do not build large-scale Node.js systems" -- instead, build many small ones.

### Integration Patterns

#### API Proxy (API Gateway)

A server that provides a **single access point** for multiple API endpoints. It performs structural integration: routing, load balancing, caching, authentication, and rate limiting, but no semantic logic.

```
         Client
           │
    ┌──────▼──────┐
    │  API Proxy  │
    └──┬──────┬───┘
       │      │
  ┌────▼──┐ ┌─▼────────┐
  │/api/* │ │ /* (else) │
  │-> API │ │-> WebApp  │
  │service│ │ service   │
  └───────┘ └───────────┘
```

The proxy abstracts the complexity of connecting to all services, which also allows restructuring services (splitting or merging) with no impact on upstream clients.

#### API Orchestration

An abstraction layer that **composes** generically-modeled services into more specific, application-level operations. Unlike a proxy, it performs **semantic** integration.

**Example: `completeCheckout()` operation**

```
Store frontend calls completeCheckout()
         │
    ┌────▼────────────────┐
    │  Orchestration Layer │
    └──┬──────┬──────┬────┘
       │      │      │
       ▼      ▼      ▼
   checkout  cart   products
   /pay     /delete /update
```

1. Invoke `checkoutService/pay` to process payment
2. On success, call `cartService/delete` to clear the cart
3. Call `productsService/update` to adjust product availability

The orchestrator can also perform **data aggregation** -- e.g., fetching product IDs from the Cart service and enriching them with details from the Products service.

> **Backend for Frontend (BFF):** A variation where each frontend (web, mobile, watch) gets its own dedicated backend that delivers exactly the data it needs, avoiding over-fetching or under-fetching.

#### Message Broker Integration

Distributes the responsibility of keeping services in sync using a **publish/subscribe** pattern. No god service needed -- each service handles its own part of the integration.

```
Store frontend -> checkoutService/pay
                       │
                  ┌────▼────┐
                  │ Message  │ publishes "purchased" event
                  │ Broker   │
                  └──┬────┬──┘
                     │    │
              ┌──────▼┐  ┌▼──────────┐
              │ Cart  │  │ Products  │
              │Service│  │ Service   │
              │deletes│  │ updates   │
              │ cart  │  │availability│
              └───────┘  └───────────┘
```

**Flow:**
1. Store frontend invokes `checkoutService/pay`
2. Checkout service publishes a `purchased` event (with cartId and product list) to the broker
3. Cart service (subscribed) receives the event and removes the cart
4. Products service (subscribed) receives the same event and updates availability

**Key benefits:** Services are decoupled -- no service needs to know about the others. New services can subscribe to events without modifying existing ones.

See [Messaging Patterns](messaging-patterns.md) for detailed coverage of publish/subscribe, task distribution, and request/reply messaging patterns.

## Comparison: Integration Patterns

| Pattern | Coupling | Complexity | Best For |
|---------|----------|------------|----------|
| **API Proxy** | Low (structural only) | Low | Single entry point, routing, caching |
| **API Orchestration** | Medium (semantic knowledge) | Medium | Composing workflows, data aggregation |
| **Message Broker** | Very low (fully decoupled) | Higher (eventual consistency) | Event-driven systems, async workflows |

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| **Scale Cube** | Three axes: cloning (X), decomposition (Y), data partitioning (Z) |
| **cluster module** | Simplest way to use all CPU cores; fork workers, auto-restart on crash, zero-downtime restart |
| **Reverse proxy** | Nginx/HAProxy for cross-machine load balancing with more features |
| **Service registry** | Dynamic discovery (Consul, etc.) for elastic scaling |
| **Peer-to-peer LB** | Client-side balancing removes bottleneck; best for internal services |
| **Containers + K8s** | Platform handles scaling, rollouts, restarts; keep app code simple and stateless |
| **Microservices** | Independent services with own DBs; integrate via proxy, orchestration, or message broker |

See [Advanced Recipes](advanced-recipes.md) for scaling CPU-bound work with worker threads and child processes.
