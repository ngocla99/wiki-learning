# REST API Design

> Sources: ByteByteGo (Alex Xu), 2023-05-17
> Raw: [Design Effective and Secure REST APIs](../../raw/system-design/2023-05-17-design-effective-secure-rest-apis.md)

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

## API Versioning

As business requirements evolve, APIs need redesign. Two strategies for backward compatibility:

| Strategy | Example | Notes |
|----------|---------|-------|
| **URL versioning** | `POST /v1/users` | Explicit, easy to route |
| **Header versioning** | `Accept-version: v1` | Cleaner URLs, but less discoverable |

## Idempotency

POST requests are **not idempotent** — calling the same POST N times creates N resources. This is critical for financial operations (e.g., trading, payments).

**Solution**: Carry a **request ID** (idempotency key) in the HTTP header. The service rejects duplicate request IDs. The key can be composed from operation-specific attributes (order type, quantity, price, etc.).

> For sign-up, backend email uniqueness checks may suffice. For trading/payment APIs, explicit idempotency keys are essential.

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

## See Also

- [API Design](api-design.md)
- [Authentication Methods](authentication-methods.md)
