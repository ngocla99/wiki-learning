---
tags:
  - system-design
  - security
---

# Authentication Methods

> Sources: Alex Xu (ByteByteGo), 2023
> Raw: [Authentication Explained Part 1](../../raw/system-design/2023-04-05-authentication-explained-bytebytego.md)

## Overview

Authentication is the process of verifying a user's identity before granting access to protected resources. This article traces the evolution of web authentication from basic passwords through session-cookies, tokens, and JWTs -- each method solving specific limitations of the previous one. Understanding these mechanisms is essential for designing secure systems: passwords establish identity, sessions track login state server-side, tokens enable stateless authentication, and JWTs carry structured claims within the token itself.

## Three Security Steps

Every application continuously performs three steps:

1. **Identity** -- who is the user?
2. **Authentication** -- prove that identity (password, token, biometric)
3. **Authorization** -- what is this authenticated user allowed to do?
![[auth-three-steps.webp]]
## Password Authentication

The most fundamental mechanism. Users submit a username + password; the server compares against stored credentials.

### HTTP Basic Access Authentication

The original HTTP-level password scheme. The browser encodes `username:password` as Base64 and sends it in every request via the `Authorization: Basic <encoded>` header.

**Flow:**

1. Client requests a protected resource
2. Server responds `401 Unauthorized` with `WWW-Authenticate: Basic`
3. Client prompts for credentials, encodes as Base64
4. Client resends request with `Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=`
5. Server decodes, validates against user database
6. Grants or denies access

**Limitations:**

- Base64 is encoding, not encryption -- trivially reversible
- Credentials sent with every request (relies entirely on TLS for security)
- Each website maintains its own username/password -- users struggle to remember
- Obsolete for modern applications
![[auth-http-basic-flow.webp]]
### Why Passwords Alone Are Insufficient

- Vulnerable to brute-force and dictionary attacks
- Users forget passwords or reuse them across sites
- No mechanism to track login state -- every request must re-authenticate

Modern systems complement passwords with session-cookie or token-based authentication for subsequent access.

## Session-Cookie Authentication

Solves the problem of tracking login state across requests. After initial password verification, the server creates a **session** and sends a **session ID** as a cookie. Subsequent requests carry this cookie instead of re-sending credentials.

### Flow

1. Client submits username + password
2. Server validates credentials, generates a unique **session ID**, stores session data server-side (memory, database, or dedicated session server)
3. Server sends session ID to client via `Set-Cookie` header
4. Client stores the cookie
5. Subsequent requests include cookie automatically
6. Server matches cookie's session ID against stored session data
7. On logout or expiration, server invalidates session, client deletes cookie
![[auth-session-cookie-flow.webp]]
### Session Storage Considerations

Session data can live in:

- **Server memory** -- fast but consumes RAM, lost on restart
- **Database** -- persistent but slower
- **Dedicated session server** (e.g., Redis) -- scalable and fast

Choice depends on scalability and reliability requirements. Monitor memory usage carefully if storing in-process.

### Limitations

| Vulnerability | Description | Mitigation |
|---|---|---|
| **Session hijacking** | Attacker steals session cookie (via XSS or insecure network) and impersonates user | Use HTTPS, `HttpOnly` + `Secure` cookie flags, short expiration |
| **CSRF** | Malicious site tricks browser into sending authenticated request | Anti-CSRF tokens, `SameSite` cookie attribute, re-authentication for sensitive actions |
| **Scalability** | Each session requires server-side storage; bottleneck at scale | External session store (Redis), or switch to token-based auth |
| **Mobile apps** | Cookies are browser-native; managing them in native apps is complex | Token-based auth is simpler for mobile |

## Token-Based Authentication

A modern, **stateless** approach. Instead of storing sessions server-side, the server issues a token containing the user's identity. The client stores the token (typically in `localStorage`) and sends it in the `Authorization` header.

### Flow

1. Client submits username + password
2. Server validates, issues a unique token
3. Token is sent back to the client
4. Client stores token in local storage
5. Subsequent requests include token in HTTP header (e.g., `Authorization: Bearer <token>`)
6. Server validates the token and grants access
7. The server grants access to the requested resource.
![[auth-token-based-flow.webp]]
### Session vs Token: Key Differences

| Aspect | Session-Cookie | Token-Based |
|---|---|---|
| State | **Stateful** -- server stores session data | **Stateless** -- token contains all needed info |
| Storage | Server-side (memory/DB/Redis) | Client-side (localStorage) |
| Transport | Cookie (automatic) | HTTP header (explicit) |
| Cross-domain | Restricted by cookie policies | Works freely across domains |
| CSRF vulnerability | Yes (cookies sent automatically) | No (token must be explicitly attached) |
| Mobile support | Complex cookie management | Native and straightforward |

### Limitations of Basic Tokens

- Vulnerable to theft if transmitted over insecure connections
- May lack built-in expiration or revocation mechanisms
- If compromised, attacker has access until token expires or is manually revoked

### Tiered Token Management

Production systems often implement multiple security levels. Stripe's API key model is a good example:

| Key Type        | Access Level                  | Use Case                                                |
| --------------- | ----------------------------- | ------------------------------------------------------- |
| **Publishable** | Read-only, public-safe        | Frontend display, read-only metrics                     |
| **Secret**      | Full access, confidential     | Server-side operations (check balances, create charges) |
| **Restricted**  | Granular per-endpoint control | Read+write payments, read-only charges                  |

Keys can be **rolled** (refreshed) periodically to limit exposure window.
![[auth-tiered-tokens.webp]]
## JWT (JSON Web Token)

An enhanced token format that carries structured claims within the token itself. JWTs are self-contained -- the server can validate them without querying a database, because the token includes its own verification data (signature).

### Flow

1. Client submits credentials
2. Server validates, issues a JWT
3. Client stores JWT, includes it in `Authorization: Bearer <jwt>` header
4. Server validates JWT signature and claims, grants access
![[auth-jwt-flow.webp]]
### JWT Structure

A JWT has three Base64URL-encoded parts separated by dots: `header.payload.signature`

**Header:**
```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload (claims):**
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "iat": 1516239022,
  "exp": 1516242622,
  "role": "admin"
}
```

Seven registered claims (recommended but not mandatory): `iss` (issuer), `sub` (subject), `aud` (audience), `exp` (expiration), `nbf` (not before), `iat` (issued at), `jti` (JWT ID). Custom claims can be added as public or private claims.

**Signature:**
```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret
)
```

The signature ensures the token has not been tampered with. The server verifies it using the secret key (HMAC) or public key (RSA/ECDSA).

### JWT vs Basic Token

| Aspect | Basic Token | JWT |
|---|---|---|
| Content | Opaque string (e.g., random UUID) | Self-contained with claims |
| Validation | Requires server-side lookup | Signature verification only (no DB query) |
| Information | Just an ID -- server must look up details | Carries user info, roles, permissions |
| Size | Small | Larger (grows with claims) |
| Revocation | Easy (delete from DB) | Hard (valid until expiration unless blacklisted) |

### Limitations

- **Size**: Embedding too much data in the payload increases token size, adding bandwidth overhead to every request
- **Revocation difficulty**: Once issued, a JWT is valid until expiration -- no built-in way to invalidate a compromised token without maintaining a blacklist (which partially defeats the stateless benefit)
- **Security**: Weak signing keys or insecure transmission can expose tokens to attacks

## Evolution Summary

```
Password (HTTP Basic)
  └─ Problem: no login state tracking, credentials sent every request
      │
Session-Cookie
  └─ Problem: stateful server, scalability, CSRF, poor mobile support
      │
Token-Based
  └─ Problem: opaque tokens require server-side lookup
      │
JWT
  └─ Self-contained, stateless, carries claims
     But: size overhead, revocation difficulty
```

## Comparison of Authentication Methods

| | HTTP Basic Auth | Session-Cookie | Token-Based | JWT |
|---|---|---|---|---|
| **How it works** | Credentials (Base64) sent with every request | Server stores session; client carries session ID cookie | Server issues opaque token; client stores and sends in header | Server issues signed token with embedded claims |
| **State** | Stateless (no server tracking) | **Stateful** -- server stores session data | **Stateless** -- server only validates token | **Stateless** -- self-contained, no DB lookup needed |
| **Where stored** | Not stored (re-sent each time) | Server: memory/DB/Redis; Client: cookie | Client: localStorage | Client: localStorage |
| **Transport** | `Authorization: Basic` header | Cookie (automatic by browser) | `Authorization: Bearer` header (explicit) | `Authorization: Bearer` header (explicit) |
| **Login state tracking** | None -- re-authenticates every request | Yes -- session ID tracks state | Yes -- token validity = logged in | Yes -- token validity + claims |
| **Cross-domain** | Limited | Restricted by cookie policies | Works freely | Works freely |
| **Mobile support** | Poor | Complex (cookie management) | Native and straightforward | Native and straightforward |
| **CSRF risk** | No (no cookies) | **Yes** -- cookies sent automatically | No -- token attached explicitly | No -- token attached explicitly |
| **XSS risk** | Low (no stored credential) | Medium (steal cookie) | **High** (steal from localStorage) | **High** (steal from localStorage) |
| **Scalability** | Good (no server state) | Poor (server stores all sessions) | Good (no server state) | Good (no server state) |
| **Carries user info** | No | No (server looks up by session ID) | No (server looks up by token) | **Yes** -- claims embedded in payload |
| **Revocation** | N/A | Easy -- delete session server-side | Easy -- delete from DB | **Hard** -- valid until expiration (needs blacklist) |
| **Performance** | Low overhead | DB/Redis lookup per request | DB lookup per request | No lookup -- signature verification only |
| **Best for** | Internal/legacy tools | Traditional web apps | APIs + mobile apps | APIs + microservices + SSO |

## Key Takeaways

- **HTTP Basic Auth** is obsolete -- Base64 encoding is not encryption, credentials travel with every request
- **Session-cookie** tracks login state server-side but introduces CSRF risks, scalability concerns, and poor mobile support
- **Token-based auth** is stateless and cross-platform friendly, eliminating CSRF by requiring explicit token attachment
- **JWT** extends tokens with self-contained claims, enabling validation without database queries -- but at the cost of larger payloads and difficult revocation
- Each method evolved to solve specific limitations of the previous one; modern systems often combine multiple approaches (e.g., JWT for authentication + refresh tokens for revocation)
- Security fundamentals apply to all methods: use HTTPS, short expiration times, secure storage, and principle of least privilege

## See Also

- [SSO, OAuth, and Passwordless Authentication](authentication-sso-oauth-passwordless.md) -- Part 2: OTP, SSO, OAuth 2.0, OIDC, biometrics, MFA, FIDO2
