# REST API Design

> Sources: ByteByteGo (Alex Xu), 2023-05-17; ByteByteGo, 2024-05-30; ByteByteGo, 2025-04-03
> Raw: [Design Effective and Secure REST APIs](../../raw/system-design/2023-05-17-design-effective-secure-rest-apis.md)
> Raw: [A Crash Course on REST APIs](../../raw/system-design/2024-05-30-crash-course-rest-apis-bytebytego.md)
> Raw: [The Art of REST API Design — Idempotency, Pagination, Security](../../raw/system-design/2025-04-03-art-of-rest-api-design-idempotency-pagination-security.md)

## Overview

REST (Representational State Transfer) remains the most popular API style (~89% adoption), but its popularity doesn't imply simplicity. REST only defines resources and HTTP methods — crafting effective REST APIs requires following specific guidelines around maturity, verb usage, status codes, and detecting design issues early.

## Detecting API Design Issues

APIs can give off "bad smells" when they need redesign.

### Consumer-Side Red Flags

- Documentation studied thoroughly, but functionality still unclear — constant clarification needed from API owners.
- Parameters and results are vaguely defined, leading to confusion and errors.
- Frontend and backend teams must collaborate extensively just to test and validate API behavior.

### Owner-Side Red Flags

- Constantly fielding usage queries — "wearing a part-time customer service hat."
- Influx of requests for minor enhancements — suggests the API isn't robust or flexible enough.

## Richardson Maturity Model (RMM)

Introduced by Leonard Richardson in 2008. Assesses how closely an API conforms to REST principles. The main factors are URI design, HTTP methods, and HATEOAS.
![[Pasted image 20260419195011.png]]

| Level | Name | Description |
|-------|------|-------------|
| **0** | The Swamp of POX | Single URI, single HTTP method (POST). RPC-style, no resource concept. Martin Fowler coined the name ("Plain Old XML") |
| **1** | Resources | Each resource has a unique URI, but still uses only POST |
| **2** | HTTP Verbs | Unique URIs + proper HTTP methods (GET, POST, PUT, DELETE). **Most popular level** |
| **3** | HATEOAS | Self-descriptive APIs — responses include hypermedia links to related resources and possible actions |

> Level 2 is a prerequisite for REST. Level 3 (HATEOAS) adds discoverability — e.g., querying account 12345 returns the balance AND links like `/account/12345/deposit`.

## Resource-Based Architecture

In REST, every piece of information that can be named and accessed through a URL is a **resource** — a user, a product, an order, or a collection. Resources are represented using standard formats (JSON or XML) and manipulated via HTTP methods sent to their unique URL endpoints.

### Resource Naming Conventions

| Convention | Good | Bad |
|------------|------|-----|
| **Nouns, not verbs** | `/users` | `/getUsers`, `/createUser` |
| **Plural for collections** | `/products` | `/product` |
| **Hierarchical paths** | `/orders/123/items` | `/orderItems?orderId=123` |
| **Hyphens for multi-word names** | `/product-categories` | `/productCategories` |
| **Lowercase** | `/user-profiles` | `/UserProfiles` |
![[Pasted image 20260429140348.png]]
## HTTP Verbs → CRUD Mapping

| HTTP Verb | CRUD Operation | Notes |
|-----------|---------------|-------|
| **GET** | Read | Retrieve single resource or list |
| **POST** | Create | Create new resource |
| **PUT** | Update (full) | Replace entire resource |
| **PATCH** | Update (partial) | Modify specific fields only |
| **DELETE** | Delete | Remove resource |

The mapping is not always one-to-one. For custom operations:

- **Map to standard verbs** — e.g., search → GET. Requires clear documentation.
- **Define custom methods** — e.g., `RESET` for game match resets. Use sparingly.
![[Pasted image 20260419195104.png]]
## HTTP Status Codes

| Range | Category | Examples |
|-------|----------|----------|
| **2xx** | Success | 200 OK, 201 Created, 204 No Content |
| **3xx** | Redirection | 301 Moved, 304 Not Modified |
| **4xx** | Client Error | 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found |
| **5xx** | Server Error | 500 Internal, 502 Bad Gateway, 503 Service Unavailable |
![[Pasted image 20260419195053.png]]
## Internal Error Codes

HTTP status codes aren't exhaustive for all possible outcomes. Establish **service-specific error code ranges** maintained in a shared library across repositories:

| Service       | Code Range  |     |
| ------------- | ----------- | --- |
| User Service  | 50000–59999 |     |
| Shopping Cart | 60000–69999 |     |

Wrap these message details in the HTTP response body so customer service and clients can identify issues by code.

> **Golden rule**: Always return a code for every response, even on timeout. Predictable, consistent behavior is essential for quality service.

## Practical Design Lessons (Stripe Sign-up Example)

Real-world API design often deviates from pure REST. Stripe's registration endpoint uses `POST /register?email={email}` rather than `POST /users` — including the verb in the URL is more intuitive for a web page. Similarly, email verification uses GET (easiest to trigger from a clickable link in an email) rather than POST.

> **Key insight**: There is no one-size-fits-all solution in API design. When system designs change frequently, strict RESTful patterns may not be practical.

## API Design Best Practices

### Versioning

As business requirements evolve, APIs need redesign. Three strategies for backward compatibility:

| Strategy | Example | Tradeoff |
|----------|---------|----------|
| **URL versioning** | `GET /v1/users` | Explicit and easy to route, but URLs must be updated in client code on version change |
| **Query parameter** | `GET /users?version=1` | Clean base URL, easy switching; feels unnatural since query params are typically for filtering |
| **Custom header** | `X-API-Version: 1` | Clean URLs; requires extra header config in client code |

![[Pasted image 20260429140517.png]]
### Pagination

Limits the number of results returned in a single response — essential for large datasets. Without it, list endpoints cause performance bottlenecks (full-table scans), payload bloat, and unstable queries (results shift between requests).

Three strategies, each with distinct trade-offs:

| Strategy | Example | Strengths | Weaknesses | Use Case |
|----------|---------|-----------|------------|----------|
| **Offset-based** | `?limit=20&offset=40` | Simple SQL `LIMIT`/`OFFSET`; predictable for paging UI | Records shift on inserts/deletes; high offsets do full-table scans | Dashboards, admin panels, stable lists |
| **Cursor-based** | `?limit=20&after=1679212341` | Stable under inserts/deletes; no offset scans | Requires unique ordered field; client must store cursor | Activity feeds, mobile apps, event logs |
| **Keyset** | `WHERE created_at < '...' ORDER BY created_at DESC LIMIT 20` | Efficient on large datasets via indexed WHERE | Requires indexed sort field | Scroll-optimized feeds at scale |
![[Pasted image 20260429142006.png]]

![[Pasted image 20260429142021.png]]

**Response metadata** should include `items`, `has_next`, `next_cursor` (or `next_token`). `total_count` is optional — expensive on large datasets, sometimes computed asynchronously and returned as approximate.

**Pagination in gRPC** — no query parameters; modeled via request/response messages with `page_token` / `next_page_token`, mapping cleanly to token-based pagination. With gRPC streaming, pagination can be avoided entirely.

### Filtering

Lets clients narrow down records based on specific criteria, reducing data transferred over the network. Implemented via query parameters specifying the field and value.

```
GET /products?category=electronics&price_max=100
```

### Sorting

Lets clients control the order of results via query parameters specifying the sort field and direction (`asc`/`desc`).

```
GET /users?sort=name&order=asc
```

### Error Handling

Return meaningful error messages and appropriate HTTP status codes. Error responses should include:
- A clear human-readable message
- An error code or type
- Additional details if necessary

Consistent error handling across all endpoints lets clients handle errors predictably.

### Documentation

Comprehensive and up-to-date documentation is a prerequisite for API adoption. Should cover: endpoints, request/response formats, authentication, error handling, and code examples.

**Swagger / OpenAPI** — generates interactive documentation from API specifications. Many frameworks have native integration that auto-updates docs as the code changes.

### Schema-First Design

Define the contract before writing the implementation. Use **OpenAPI** (REST) or **Protobuf** (gRPC) as the source of truth.

Benefits:
- Contract diffs reveal breaking changes before deployment
- Consumer teams align on a shared interface earlier
- Code generation eliminates hand-written request/response types
- Automated tests validate assumptions before runtime

APIs without schemas tend to drift — fields change silently and docs lag behind actual behavior.

### Predictable Response Structure

Wrap successful results and failures in consistent envelopes so clients can share parsers, observability, and error handling:

```json
// success
{ "data": { ... } }
// failure
{ "error": { "code": "USER_NOT_FOUND", "message": "..." } }
```

Without this, client logic branches across endpoints — some return plain values, others embed metadata, a few use nested fields for no clear reason. The inconsistency spreads into logging, testing, and monitoring layers.

### REST vs gRPC

| | REST | gRPC |
|---|------|------|
| **Format** | JSON over HTTP/1.1 (typically) | Protobuf over HTTP/2 |
| **Schema** | Optional (OpenAPI) | Required (`.proto`) |
| **Streaming** | Limited (SSE, chunked) | Bi-directional native |
| **Tooling** | Universal browser/curl support | Requires generated stubs |
| **Best for** | Public APIs, mobile, browser | Internal service-to-service, low-latency |

Many systems run a hybrid — REST externally, gRPC internally — letting each protocol do what it does best.

## Idempotency

An operation is idempotent when multiple identical calls produce the same result as one call. The server may still do work on retry — but the outcome doesn't change after the first successful execution.

| Method | Idempotent? |
|--------|-------------|
| GET, HEAD | Safe and idempotent by nature |
| PUT, DELETE | Idempotent by definition |
| POST | **Not idempotent by default** — explicit safeguards required |
| PATCH | Generally not considered idempotent |

### When to Enforce

Idempotency adds complexity. Skip it where there are no significant side effects. Enforce it for:

- **Financial operations** — payment processing, refunds
- **Provisioning** — VM creation, account registration
- **Webhook receivers** — retries are often automatic
- **Critical workflows** — order submission, subscription setup

### Implementation Pattern

Most systems require clients to include an idempotency key — a unique string identifying the **logical operation**, not the HTTP request itself.

1. Client sends POST with `Idempotency-Key: abc123` header.
2. Server checks storage (Redis, DB) for an existing result tied to that key.
3. **Hit** → return the original response, skip execution.
4. **Miss** → perform the operation, store the result keyed by the token, return the response.
5. Key expires after a TTL window covering reasonable retries.

**Collision rule**: if the same key arrives with a different payload, reject with a clear error to avoid ambiguity.
![[Pasted image 20260429141502.jpg]]
### Storage Decisions

- **Where** — Redis offers low latency; relational DBs offer durability.
- **How long** — long enough to cover retries (often hours to days), short enough to avoid bloated state.
- **What to store** — the full response, or a canonical representation of the request result.

### Trade-offs

Storing tokens isn't trivial: write coordination is required, stateless services need temporary persistence, and edge caches complicate validation. But failing to implement idempotency where needed leads to duplicate charges, over-provisioning, corrupted state, and tense debugging.

### gRPC and Idempotency

gRPC has no built-in idempotency semantics. Engineers must enforce it via embedded request IDs in messages, server-side deduplication, and cautious client-side retry logic.

## Authentication and Authorization

### Authentication

Authentication verifies the identity of the user or client — "Who are you?" Common mechanisms:

- Username and password
- API keys or tokens (e.g., JWTs)
- OAuth for delegated access

Important for: protecting sensitive data, auditing user actions, enabling personalization. See [Authentication Methods](authentication-methods.md) for JWT flow diagrams and session-cookie vs token comparison.

### Authorization

Authorization determines what an authenticated user is allowed to do — "What are you allowed to do?" Common mechanisms:

| Mechanism | Description |
|-----------|-------------|
| **RBAC** (Role-Based Access Control) | Users are assigned roles; permissions are granted based on those roles |
| **ABAC** (Attribute-Based Access Control) | Access granted based on attributes of the user, resource, or environment |
| **OAuth scopes** | Tokens include scopes defining the permissions and access levels granted to the client |

## Scalability and Performance

### Stateless Architecture

Design REST APIs to be stateless — each request must contain all information needed for the server to process it. Practical implications:

- Don't store session data on API server instances; it hinders horizontal scaling.
- Use stateless auth mechanisms (JWTs, API keys), or store session data in a dedicated storage layer (e.g., Redis), not on the API server.

### Horizontal Scaling

Stateless architecture enables horizontal scaling: add more server instances behind a load balancer to handle increased traffic, rather than vertically scaling a single machine.

### Caching

Caching reduces server load and improves response times. Apply at multiple levels:

| Level | Mechanism | Use Case |
|-------|-----------|----------|
| **Client-side** | `Cache-Control`, `ETag` HTTP headers | Avoid re-fetching unchanged resources |
| **Server-side** | Redis, Memcached | Results of expensive computations or hot data |
| **Edge** | CDN | Static or infrequently-changing responses close to the user |

**Cache-Aside pattern** — application checks cache first; on a miss, loads from the database and populates the cache before returning.

### Efficient Data Serialization

Choose efficient serialization formats (JSON, Protocol Buffers) to minimize payload size. Avoid sending unnecessary fields in responses.

### Asynchronous Processing and Message Queues

For resource-intensive or time-consuming operations, don't block the HTTP response:

1. Client sends a request (e.g., `POST /reports`).
2. API validates input, enqueues a job, immediately returns `202 Accepted` with a task ID.
3. Background workers consume the queue and execute the task.
4. Client polls (`GET /reports/{id}`) or receives a webhook notification when complete.

**Message queue** (e.g., RabbitMQ, SQS, Redis Streams) decouples the API from the workers and absorbs traffic spikes.

### Monitoring and Logging

Track API health with a centralized logging solution collecting metrics from all server instances:

- **Response times** — detect latency regressions
- **Error rates** — surface failure patterns
- **Resource utilization** — identify saturation before it causes outages

## API Security

### User ID Design

- **Never use sequential IDs** — exposes system information (total users, growth rate) to attackers and competitors.
- Use **unpredictable identifiers** — e.g., Stripe uses `acct_xxxRbrL6xxxxDQxx`.

### Password Storage

- **Never store plaintext** — always hash with a unique, random **salt** per user.
- Flow: `hash(password + salt)` → store `(salt, hash)`. On sign-in, re-hash and compare.
- Consider **passwordless authentication** as an alternative.

### Resource Protection

Classify every resource as **public** or **protected**:

| Type | Access | Example |
|------|--------|---------|
| **Public** | Anonymous users | Platform transaction metrics |
| **Protected** | Signed-in users only | User profiles |

For protected resources, redirect unauthenticated users to the login page.

### Token Types

| Token | Description | Trade-offs |
|-------|-------------|------------|
| **API Key** | Shared secret in a header | Easy to implement; hard to scope or revoke. Suits internal/read-only |
| **JWT** | Self-contained credential with claims | Stateless and efficient; expires poorly, prone to misuse if validation is weak |
| **OAuth2** | Delegation protocol for limited access | Granular scopes; significant implementation overhead. Common for third-party |

Common authorization mistakes regardless of token type: reusing admin tokens across systems, skipping resource-level checks, over-scoping (e.g., `read-write-admin` instead of `read:user:123`). A valid token only proves identity — it doesn't prove the action is allowed.

### Rate Limiting and Abuse Protection

Even valid clients can behave badly — credential stuffing, scraping, denial of service. Defenses:

- Cap request volume per client / IP / token
- Different thresholds for public vs internal clients
- Track over time windows (e.g., 1000/hour)

Enforced at the gateway layer (Nginx, Envoy, API gateway) rather than in application code.
![[Pasted image 20260429142334.jpg]]
### Input Validation

Treat anything from the outside (IDs, filters, search queries, payloads) as untrusted. Validate in layers:

- **Transport-level** — reject malformed JSON and oversized payloads
- **Schema-level** — enforce field presence, types, formats (OpenAPI schemas)
- **Business logic** — domain-specific rules (e.g., cannot transfer money to self)

Failure to validate leads to SQL injection, script injection, path traversal, type mismatches, and crashes.

### Error Handling Without Leaking Internals

Detailed stack traces, SQL fragments, or internal identifiers in error responses give attackers free reconnaissance. Examples of oversharing:

- `Error: SELECT * FROM users WHERE id = 'abc'::uuid failed`
- `NullReferenceException at AuthService.Authenticate`

Return standard codes (400/401/403/500) with minimal, actionable messages. Log full errors internally, never in client responses.

### HTTPS and TLS

Every API must use HTTPS. Beyond TLS basics:

- **HSTS** (HTTP Strict Transport Security) — force browsers to use HTTPS
- **Certificate pinning** — where appropriate, for mobile/internal clients
- **Auto-redirect** plaintext traffic to HTTPS

Anything less is a critical vulnerability — credentials leak, sessions get hijacked, request data becomes visible to intermediaries.

## See Also

- [API Design](api-design.md)
- [Authentication Methods](authentication-methods.md)
