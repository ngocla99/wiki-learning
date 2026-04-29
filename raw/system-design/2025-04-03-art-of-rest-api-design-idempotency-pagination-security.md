# The Art of REST API Design: Idempotency, Pagination, and Security

> Source: https://blog.bytebytego.com/p/the-art-of-rest-api-design-idempotency
> Collected: 2026-04-29
> Published: 2025-04-03
> Author: ByteByteGo

APIs are the front doors to most systems. They expose functionality, enable integrations, and define how teams, services, and users interact. But while it's easy to get an API working, it's much harder to design one that survives change, handles failure gracefully, and remains a joy to work with six months later.

Poorly designed APIs don't just annoy consumers. They slow teams down, leak data, cause outages, and break integrations. One inconsistent response structure can turn into dozens of custom client parsers. One missing idempotency check can result in duplicate charges. One weak authorization path can cause a security breach.

A good API design is defensive — it anticipates growth, chances of misuse, and failures. Integration points are long-lived and every decision has an impact down the line.

## Principles of Good API Design

A well-designed API should not be a puzzle. They should behave predictably, use consistent structures, and make few assumptions about clients. Good API design prioritizes clarity above all else.

### Consistency Reduces Friction

Uniformity across endpoints simplifies development. When resource naming follows clear conventions and response formats adhere to a shared structure, teams can reuse client code and debugging tools.

Inconsistent APIs introduce silent traps. For example, an API that returns dates as ISO strings in some endpoints and Unix timestamps in others. Or one where error responses sometimes use error codes and other times return raw strings. Each exception forces a workaround.

### Nouns Represent Resources, Not Actions

REST works best when endpoints map to domain concepts (orders, users, payments) rather than commands. `POST /users` conveys intent more clearly than `POST /createUser`, which blends transport semantics with business logic.

Resource-oriented URLs support logical nesting (`/users/123/orders`) and permission modeling. Overusing verbs leads to RPC-style design that breaks tooling, limits caching, and makes the API harder to evolve.

### Method Semantics Guide Expectations

HTTP methods carry distinct semantics:
- **GET** — retrieves resources without side effects. Safe and cacheable.
- **POST** — creates new resources. Not idempotent by default.
- **PUT** — replaces resources. Guarantees idempotency.
- **PATCH** — applies partial updates.
- **DELETE** — removes resources. Expects idempotency.

A GET that triggers writes breaks caching. A retry on POST without safeguards creates duplicates. APIs that align with method semantics integrate more reliably with tooling.

### Idempotency Protects Against Retries

Clients retry requests when connections drop or timeouts occur. Without safeguards, those retries can create duplicate orders, inconsistent states, or unintended charges.

Idempotency solves this. Systems use idempotency keys (typically passed in headers) to detect repeated requests and return the original result. This works especially well for operations tied to money, provisioning, or resource creation.

### Versioning for Sustainable Change

Every API evolves. Without versioning, even small shifts risk breaking clients. URI versioning (`/v1/users`) offers the clearest and most discoverable path. Alternatives (query parameter, header, media-type) provide cleaner URLs but increase tooling complexity. Versioning works best when applied early and managed intentionally.

### Protocol Choice (REST vs gRPC)

- **REST** — human-readable interfaces, broad compatibility, tooling support. Suits public APIs, mobile clients, and browser integrations.
- **gRPC** — better performance, schema enforcement, bi-directional streaming. Built on HTTP/2 and Protobuf. Fits internal service communication where latency and structure matter more than accessibility.

Some systems adopt both: REST externally and gRPC internally.

### Predictable Responses Simplify Integration

A consistent format that wraps successful results in a `data` object and failures in an `error` block allows shared parsers, better observability, and more stable error handling.

Without consistency, client logic branches across endpoints. Some responses return plain values, others embed metadata, a few use nested fields for no clear reason. This inconsistency spreads across logging, testing, and monitoring layers.

### Schema-First Design Prevents Drift

Using OpenAPI (REST) or Protobuf (gRPC) enforces structure, supports contract validation, and enables code generation. Schemas serve as documentation, source of truth, and gatekeeper for change.

Advantages:
- Contract diffs reveal breaking changes
- Consumers align more easily
- Automated tests validate assumptions before runtime

APIs without schemas tend to drift — fields change silently, docs lag behind actual behavior.

## Understanding Idempotency

A client times out mid-request (placing an order, making a payment). The user clicks "Confirm" again, retry triggered. The system receives the same request twice. Without idempotency safeguards, both are processed — duplicate orders, double payments, conflicting states.

### What Makes an Operation Idempotent?

Multiple identical calls produce the same result as one call. The server may still do work on retry — but the outcome doesn't change after the first successful execution.

Examples:
- `DELETE /users/123` twice — both succeed.
- Sending the same payment request twice — should process once or return the result of the original attempt, not charge twice.

### Method Semantics: Where Idempotency is Expected

| Method | Idempotent? |
|--------|-------------|
| GET, HEAD | Safe and idempotent by nature |
| PUT, DELETE | Idempotent by definition |
| POST | **Not idempotent by default** — explicit safeguards required |
| PATCH | Generally not considered idempotent |

Critical POST endpoints (payments, account creation, provisioning) must enforce idempotency explicitly.

### When to Enforce Idempotency

Idempotency adds complexity. Skip it for endpoints without significant side effects. Enforce it for:

- Financial operations (payment processing, refunds)
- Provisioning (VM creation, account registration)
- Webhook receivers (retries are often automatic)
- Critical workflows (order submission, subscription setup)

### Idempotency Implementation

Most systems require clients to include an idempotency key — a unique string identifying the **logical operation**, not the HTTP request.

Common characteristics:
- Sent as a header (`Idempotency-Key: abc123`)
- Scoped to a resource or endpoint
- Stored server-side, often with a TTL
- Returns the original result if the same key is received again

Implementation pattern:
1. Client sends POST with a unique idempotency key.
2. Server checks storage (cache, DB, Redis) for an existing result tied to that key.
3. If found, returns the original response, skipping execution.
4. If not found, performs the operation, stores the result keyed by the token, returns the response.
5. Optionally, the key expires after a timeout window.

Storage decisions:
- **Where** — Redis offers low latency; relational DBs offer durability.
- **How long** — long enough to cover retries, short enough to avoid bloated state.
- **What** — full response, or canonical representation of the request result.

**Collisions** — if the same key arrives with a different payload, the server must reject it with a clear error to avoid ambiguity.

### Trade-offs and Limitations

- Token storage requires write coordination
- Stateless services must integrate temporary persistence
- Edge caches complicate validation
- Badly implemented logic can return stale or incorrect data

But failing to implement idempotency where needed leads to duplicate charges, resource over-provisioning, corrupted state, and tense debugging.

### gRPC and Idempotency

gRPC doesn't provide built-in idempotency semantics. Engineers must enforce it via:
- Embedding request IDs in the message
- Server-side deduplication logic
- Handling retries at the client layer with caution

## Pagination Techniques

Without pagination or with inefficient pagination, list endpoints become dangerous. Large datasets cause performance bottlenecks (full-table scans), payload bloat, and unstable queries (results shift between requests).

### Offset-Based Pagination

`?limit=20&offset=40` — skips a number of rows and returns the next batch.

Pros: Easy to implement with SQL `LIMIT`/`OFFSET`. Predictable for UI scroll and paging controls.

Cons: Breaks down with fast-changing data. Inserts/deletes between pages cause record shifts or duplicates. High offsets result in full-table scans.

Use case: dashboards, admin panels, stable lists. Falls apart in real-time feeds or distributed sync systems.

### Cursor-Based Pagination

Uses a cursor (typically a unique field like `created_at` or `id`) to fetch the next page relative to the last record received.

`?limit=20&after=1679212341`

Pros: More stable under inserts/deletes. Avoids expensive offset scans. Easy to resume from the last seen time. Plays well with time-ordered data.

Cons: Requires unique, ordered fields. Clients must store the last cursor.

Use case: activity feeds, mobile apps, event logs.

### Keyset Pagination

A refined cursor pagination using indexed fields for efficient WHERE clauses:

```sql
SELECT * FROM posts
WHERE created_at < '2023-03-24T15:00:00Z'
ORDER BY created_at DESC
LIMIT 20;
```

Performs well on large datasets, avoids skipping rows. Worthwhile for scroll-optimized feeds.

### Design Choices for Pagination

Paginated responses should include:
- `items` — current page of results
- `has_next` — boolean indicator of more data
- `next_cursor` or `next_token` — continuation pointer
- `total_count` — optional, useful for UI but expensive on large datasets

Some systems avoid `total_count` entirely; others compute it asynchronously and return approximate values.

### Pagination in gRPC

gRPC doesn't support query parameters. Pagination is modeled via request/response messages:

```protobuf
message ListRequest {
  int32 limit = 1;
  string page_token = 2;
}

message ListResponse {
  repeated Item items = 1;
  string next_page_token = 2;
}
```

Maps cleanly to token-based pagination. With gRPC streaming, pagination can be avoided entirely.

## API Security Considerations

Anything exposed will be probed by automation, curiosity, or intent. Security failures rarely look dramatic at first — a forgotten debug endpoint without auth, a public S3 bucket, a token that never expires.

### Authentication vs Authorization

- **Authentication** — Is this caller who they claim to be?
- **Authorization** — Can this caller access this specific resource, in this specific way?

A valid token doesn't mean the request is safe. Common mistakes:
- Reusing admin tokens across systems
- Skipping resource-level checks
- Over-scoping tokens (e.g., `read-write-admin` instead of `read:user:123`)

Each request must be scoped to what the caller is explicitly allowed to do.

### Token Types: API Keys, JWT, OAuth2

| Token Type | Description | Trade-offs |
|------------|-------------|------------|
| **API Keys** | Simple shared secrets | Easy to implement, hard to scope or revoke. Suits internal/read-only |
| **JWT** | Self-contained credentials with claims | Efficient, stateless. Prone to misuse if validation isn't strict. Expires poorly |
| **OAuth2** | Delegation protocol for limited access | Granular, but more implementation overhead. Common for third-party integrations |

### Rate Limiting and Abuse Protection

Even valid clients can behave badly. Rate limiting prevents credential stuffing, scraping, and DoS by:
- Capping request volume per client/IP/token
- Different thresholds for public vs internal clients
- Tracking requests over time windows (e.g., 1000/hour)

Tools: Nginx, Envoy, API gateways.

### Input Validation

Anything the API receives from outside (IDs, filters, payloads) must be treated as untrusted. Validation happens in layers:

- **Transport-level** — reject malformed JSON, oversized payloads
- **Schema-level** — enforce field presence, types, formats (OpenAPI schemas)
- **Business logic** — domain-specific rules (e.g., cannot transfer money to self)

Failing to validate leads to SQL injection, script injection, path traversal, type mismatches, crashes.

### Error Handling Should Not Reveal Everything

Detailed stack traces, SQL fragments, or internal identifiers in errors provide unnecessary insight to attackers.

Examples of oversharing:
- `Error: SELECT * FROM users WHERE id = 'abc'::uuid failed`
- `NullReferenceException at AuthService.Authenticate`

Use standard codes (400/401/403/500) with minimal, actionable messages. Log full errors internally, never in client responses.

### HTTPS and TLS

Every API must use HTTPS. Beyond TLS basics, security-conscious systems also:
- Enforce HSTS (HTTP Strict Transport Security)
- Pin certificates where appropriate
- Auto-redirect plaintext traffic

## Summary

- Well-designed APIs behave consistently, fail predictably, and grow without friction.
- Resource-oriented paths and proper HTTP verb usage align APIs with expectations.
- Schema-first design prevents drift, enables contract testing, improves collaboration.
- Most security and reliability issues surface during retries, failures, or edge-case usage.
- Idempotency keys allow safe deduplication of side-effect operations.
- Pagination strategies: offset (simple, fragile), cursor (stable), keyset (efficient).
- Authentication identifies the caller; authorization must be enforced per action and resource.
- JWTs/OAuth2/API keys offer different trade-offs in revocability, scope, complexity.
