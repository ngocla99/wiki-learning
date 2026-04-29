# Knowledge Base Index

## react

Advanced React patterns, performance optimization, and web performance fundamentals for React applications.

### Core Rendering Model

How React decides what to render, how re-renders propagate, and how to structure components to control that flow.

| Article | Summary | Updated |
|---------|---------|---------|
| [React Re-renders](react/react-re-renders.md) | How re-renders work, triggers, propagation, the big myth, and prevention patterns | 2026-04-13 |
| [Reconciliation and Diffing](react/reconciliation-and-diffing.md) | Virtual DOM diffing algorithm, position-based comparison, key attribute, state reset and force-reuse techniques | 2026-04-13 |
| [Component Composition Patterns](react/component-composition-patterns.md) | Elements as props, children, render props, HOCs, compound components, and flexible layout patterns | 2026-04-13 |

### Hooks & Patterns

Specific React APIs and common coding patterns for everyday development.

| Article | Summary | Updated |
|---------|---------|---------|
| [Memoization in React](react/memoization.md) | useMemo, useCallback, React.memo — when to use, pitfalls, children problem, progressive optimization case study, and anti-patterns | 2026-04-13 |
| [Refs and Imperative API](react/refs-and-imperative-api.md) | useRef, forwardRef, useImperativeHandle, callback refs, and DOM measurement patterns | 2026-04-13 |
| [Closures in React](react/closures-in-react.md) | Stale closure problem in hooks, event handlers, timers, and Refs; escape patterns and mental models | 2026-04-13 |
| [Debouncing and Throttling](react/debouncing-throttling.md) | Correct debounce/throttle with Refs, the closure escape pattern, and useDebounce hook extraction | 2026-04-13 |
| [useLayoutEffect](react/use-layout-effect.md) | Flicker prevention, browser rendering pipeline (tasks vs painting), useEffect vs useLayoutEffect timing, and SSR considerations | 2026-04-15 |
| [React Portals](react/react-portals.md) | Portals for escaping CSS stacking contexts, event bubbling vs DOM events, form submission caveat | 2026-04-13 |
| [Error Handling in React](react/error-handling.md) | ErrorBoundary, try/catch limitations, async errors, strategic placement, and recovery patterns | 2026-04-13 |
| [React Compiler](react/react-compiler.md) | Automated memoization via Babel plugin, code transformation examples, measured performance impact, limitations, and compatibility | 2026-04-13 |

### State & Data

Managing application state across different categories and fetching data effectively.

| Article | Summary | Updated |
|---------|---------|---------|
| [React State Management](react/state-management.md) | State categories (remote/URL/local/shared), TanStack Query + nuqs + Zustand stack, Context limitations, and library evaluation framework | 2026-04-14 |
| [React Context and Performance](react/react-context-performance.md) | Context for state distribution, re-render implications, splitting strategies, and higher-order component optimization | 2026-04-14 |
| [Data Fetching Patterns](react/data-fetching-patterns.md) | Client-side fetching, request waterfalls, browser limits, Suspense, SSR vs RSC strategies | 2026-04-14 |
| [Race Conditions in React](react/race-conditions.md) | Stale promise problem, cleanup flags, Ref comparison, AbortController, and forced re-mounting | 2026-04-15 |

### Performance

Measuring, diagnosing, and optimizing web performance in React applications.

| Article | Summary | Updated |
|---------|---------|---------|
| [Web Performance Metrics](react/web-performance-metrics.md) | Core Web Vitals (FCP, LCP, INP), CrUX, RUM, flame graphs, Lighthouse, and measurement methodology | 2026-04-13 |
| [Interaction Performance](react/interaction-performance.md) | INP measurement, SPA transitions, Long Tasks, yielding (and when NOT to), dev vs prod mode for debugging, React DevTools Profiler, and re-render-driven optimization | 2026-04-13 |
| [Bundle Size Optimization](react/bundle-size-optimization.md) | Code splitting, tree shaking (ESM vs CJS), bundle analysis, HTTP/1 vs HTTP/2, transitive deps, duplicate libraries, TTI+SSR gap, and compression trade-offs | 2026-04-13 |
| [Lazy Loading and Suspense](react/lazy-loading-and-suspense.md) | React.lazy, Suspense boundaries, critical path extraction, preloading strategies, framework evaluation checklist, and vendor chunk trade-offs | 2026-04-13 |

### Architecture

Application-level rendering strategies and server-side patterns.

| Article | Summary | Updated |
|---------|---------|---------|
| [Rendering Strategies](react/rendering-strategies.md) | CSR, SSR, SSG, hydration mechanics, streaming, and framework trade-offs | 2026-04-13 |
| [React Server Components](react/react-server-components.md) | RSC architecture, serialized element trees, streaming with Suspense, async components, "use client" boundary rules, and performance comparison vs CSR/SSR | 2026-04-13 |

### Interview Questions

Curated question-and-answer sets for React interview preparation, organized by topic area.

| Article | Summary | Updated |
|---------|---------|---------|
| [React Fundamentals](react/interview/react-fundamentals-interview.md) | What is React and its benefits, React Node vs Element vs Component, JSX transformation and key characteristics | 2026-04-14 |
| [React Hooks](react/interview/react-hooks-interview.md) | Benefits of hooks, rules of hooks, useState callback, useEffect dependency array, useEffect vs useLayoutEffect, useRef, useCallback, useMemo, useReducer, useId | 2026-04-14 |
| [React Components & State](react/interview/react-components-interview.md) | State vs props, key prop purpose, index keys pitfall, controlled vs uncontrolled, fragments, resetting state, immutability, error boundaries | 2026-04-14 |
| [React Architecture & Performance](react/interview/react-architecture-interview.md) | Re-rendering, Virtual DOM, reconciliation, Context pitfalls/optimization, HOCs, composition, code splitting, Suspense, hydration, SSR, SSG, portals, testing, state manager decision guide | 2026-04-14 |

---

## nodejs

Node.js design patterns, asynchronous programming, streams, scalability, and distributed systems from "Node.js Design Patterns, 4th Edition."

### Platform & Modules

Core runtime fundamentals and the module system.

| Article | Summary | Updated |
|---------|---------|---------|
| [The Node.js Platform](nodejs/nodejs-platform.md) | Node.js philosophy (small core/modules/surface area), reactor pattern, event demultiplexing, libuv, V8, and TypeScript integration | 2026-04-16 |
| [The Module System](nodejs/module-system.md) | ES modules syntax, CommonJS, ESM vs CJS differences, circular dependencies, monkey patching, TypeScript module configuration | 2026-04-16 |

### Asynchronous Patterns

Callbacks, events, promises, async/await, and streams — the core async building blocks.

| Article | Summary | Updated |
|---------|---------|---------|
| [Callbacks and Events](nodejs/callbacks-and-events.md) | CPS (sync/async), "Unleashing Zalgo", Node.js callback conventions, EventEmitter, Observer pattern, memory leaks | 2026-04-16 |
| [Asynchronous Control Flow](nodejs/async-control-flow.md) | Sequential/concurrent/limited-concurrent execution with callbacks, promises, and async/await; TaskQueue, race conditions, promisification | 2026-04-16 |
| [Coding with Streams](nodejs/streams.md) | Readable/Writable/Duplex/Transform streams, backpressure, pipeline(), piping patterns (combine/fork/merge/mux-demux), Web Streams | 2026-04-16 |

### Design Patterns

Classic GoF patterns adapted for JavaScript and Node.js.

| Article | Summary | Updated |
|---------|---------|---------|
| [Creational Design Patterns](nodejs/creational-design-patterns.md) | Factory, Builder, Revealing Constructor, Singleton, Dependency Injection and wiring modules | 2026-04-16 |
| [Structural Design Patterns](nodejs/structural-design-patterns.md) | Proxy (composition/augmentation/built-in Proxy), Decorator, Adapter, and the line between Proxy and Decorator | 2026-04-16 |
| [Behavioral Design Patterns](nodejs/behavioral-design-patterns.md) | Strategy, State, Template, Iterator (protocols, generators, async iterators), Middleware, Command pattern | 2026-04-17 |

### Testing & Advanced

Testing patterns, advanced async recipes, and production techniques.

| Article | Summary | Updated |
|---------|---------|---------|
| [Testing Patterns](nodejs/testing-patterns.md) | Testing pyramid, Node.js test runner, AAA pattern, mocking (mock.fn, Undici, module mocking), integration tests, E2E with Playwright | 2026-04-17 |
| [Advanced Recipes](nodejs/advanced-recipes.md) | Async initialization (queues, State), request batching/caching, canceling with AbortController, CPU-bound tasks (worker threads, child processes) | 2026-04-17 |

### Scalability & Messaging

Scaling applications and distributed system communication patterns.

| Article | Summary | Updated |
|---------|---------|---------|
| [Scalability Patterns](nodejs/scalability-patterns.md) | Scale cube (X/Y/Z), cluster module, reverse proxy (Nginx), dynamic scaling, Docker/Kubernetes, monolith vs microservices, integration patterns | 2026-04-17 |
| [Messaging Patterns](nodejs/messaging-patterns.md) | Pub/Sub, reliable queues (AMQP/RabbitMQ), Redis Streams, task distribution (fan-out/fan-in), request/reply (correlation ID, return address) | 2026-04-17 |

---

## system-design

System design concepts, architecture patterns, and infrastructure fundamentals.

### Database Selection

| Article | Summary | Updated |
|---------|---------|---------|
| [Database Types](system-design/database-types.md) | Relational, NoSQL (document/column/key-value/graph), NewSQL, and time-series — strengths, weaknesses, examples, and comparison table | 2026-04-29 |
| [Database Selection Factors](system-design/database-selection-factors.md) | Seven factors for choosing a database — scalability, performance, consistency (CAP/PACELC), data model, security, cost, community/ecosystem | 2026-04-29 |
| [Database Selection Process](system-design/database-selection-process.md) | Five-step selection workflow plus three case studies (e-commerce hybrid PostgreSQL+Elasticsearch, social media DynamoDB adjacency lists, IoT InfluxDB) | 2026-04-29 |

### API Design

| Article | Summary | Updated |
|---------|---------|---------|
| [API Design](system-design/api-design.md) | API First principle and 7 architectural styles (REST, GraphQL, WebSocket, Webhook, gRPC, SOAP, Kafka), popularity data, and selection guidance | 2026-04-19 |
| [GraphQL](system-design/graphql.md) | GraphQL query language and runtime — declarative queries, SDL, mutations, subscriptions, resolvers, request lifecycle, REST/BFF comparison, federation (directives, vs stitching), adoption patterns, real-world use cases, and ecosystem tools | 2026-04-19 |
| [REST API Design](system-design/rest-api-design.md) | API smell detection, Richardson Maturity Model, resource naming, HTTP verbs/status codes, versioning, pagination strategies (offset/cursor/keyset), filtering, sorting, schema-first (OpenAPI/Protobuf), REST vs gRPC, predictable response envelopes, idempotency (keys, storage, gRPC notes), authentication/authorization (RBAC/ABAC/OAuth scopes, token types), scalability (stateless, horizontal scaling, caching, async/queues, monitoring), and security (rate limiting, input validation, error leakage, HTTPS/HSTS) | 2026-04-29 |

### Security & Authentication

| Article | Summary | Updated |
|---------|---------|---------|
| [Authentication Methods](system-design/authentication-methods.md) | Evolution from HTTP Basic Auth → Session-Cookie → Token → JWT; stateful vs stateless auth, CSRF/XSS mitigations, tiered token management, and JWT structure | 2026-04-16 |
| [SSO, OAuth, and Passwordless Auth](system-design/authentication-sso-oauth-passwordless.md) | OTP, SSO (CAS/SAML/OIDC), OAuth 2.0 grant types, biometrics, MFA/TOTP, and FIDO2 (WebAuthn + CTAP) | 2026-04-16 |

---

## aws

AWS cloud computing services, infrastructure, CI/CD integration, and interview preparation for fullstack developers.

| Article | Summary | Updated |
|---------|---------|---------|
| [AWS Services Interview Q&A](aws/aws-services-interview.md) | Practical Q&A covering EC2, RDS, IAM, S3, CloudWatch, VPC, CodeBuild — mapped to real project experience | 2026-04-17 |

---

## interview

Behavioral interview preparation, self-introduction, and STAR-method stories.

| Article | Summary | Updated |
|---------|---------|--------|
| [Behavioral Interview](interview/behavioral-interview.md) | Self-introduction, why leave, STAR story (legacy modernization: 98% startup improvement), common behavioral Q&A, and interview tips | 2026-04-17 |
