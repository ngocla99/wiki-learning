---
tags:
  - system-design
  - security
---

# SSO, OAuth, and Passwordless Authentication

> Sources: Alex Xu (ByteByteGo), 2023
> Raw: [Authentication Explained Part 2](../../raw/system-design/2023-04-12-authentication-explained-part2-bytebytego.md)

## Overview

Beyond passwords and tokens, modern authentication has evolved toward reducing or eliminating passwords entirely. This article covers three major areas: Single Sign-On (SSO) for unified access across applications, OAuth 2.0 with OpenID Connect for delegated authorization, and passwordless methods including OTP, biometrics, MFA, and the FIDO2 framework. Together they represent the current state of the art in web authentication -- moving from "something you know" toward "something you own" and "something you are."

## Three Authentication Factors

| Factor | Type | Examples |
|---|---|---|
| **Knowledge** | Something you know | Password, PIN, security question |
| **Ownership** | Something you own | Phone, hardware token, email account |
| **Inherence** | Something unique to you | Fingerprint, face, retina |

Passwords rely solely on knowledge. Modern methods combine multiple factors for stronger security.

## One-Time Passwords (OTP)

OTPs are valid for a limited time (typically minutes) and cannot be reused. They prove **ownership** -- that the user controls a specific device or email account.

### Flow

1. User enters username, phone number, or email
2. Server generates an OTP with an expiration time
3. Server sends OTP to user's device via SMS or email
4. User enters the received OTP
5. Server compares generated OTP with user's input -- if they match, login is granted
![[auth-otp-flow.webp]]
### SMS Delivery Risks

SMS-based OTP is vulnerable to:

- **SIM swapping** -- attacker convinces mobile carrier to transfer number to a new SIM
- **SS7 interception** -- exploiting telecom protocol vulnerabilities to redirect messages
- **Phone number porting** -- social engineering carrier support

Recommended alternatives: authenticator apps (Google Authenticator, Authy) or email delivery.

## SSO (Single Sign-On)

SSO allows access to multiple applications with a single set of credentials. It relies on a **Central Authentication Service (CAS)** server.

### Flow

**First login (e.g., Gmail):**

1. User attempts to access Gmail → redirected to CAS server
2. CAS verifies credentials, creates a **Ticket Granting Ticket (TGT)** stored as a **Ticket Granting Cookie (TGC)** in the browser (global session)
3. CAS generates a **Service Ticket (ST)** for Gmail, redirects user back with ST
4. Gmail validates ST with CAS → user gains access
![[auth-sso-cas-flow.webp]]
**Subsequent app (e.g., YouTube):**

5. Browser already has TGC → CAS recognizes authenticated status
6. CAS generates a new ST for YouTube → user accesses YouTube without re-entering credentials

### SSO Protocols

| Protocol | Format | Best For |
|---|---|---|
| **SAML** (Security Assertion Markup Language) | XML | Enterprise applications |
| **OIDC** (OpenID Connect) | JWT (built on OAuth 2.0) | Consumer apps, new applications |

OIDC is preferred for new applications -- supports web, mobile, and JavaScript clients.

### SSO Benefits

- Users remember one strong password instead of many weak ones
- Reduced phishing risk (fewer login prompts)
- Centralized access control for IT departments
- Seamless cross-application experience

## OAuth 2.0 and OpenID Connect (OIDC)

OAuth 2.0 is an **authorization** framework. OIDC is an **authentication** layer built on top of it. Together they power "Sign in with Google/GitHub/Facebook" flows.

- **OAuth 2.0**: Issues short-lived tokens for delegated access -- the third-party app never sees the user's password
- **OIDC**: Adds an **ID Token** (JWT) containing user identity data (name, email, profile picture)
![[auth-oauth2-roles.webp]]
### OAuth 2.0 Roles

| Role | Description | Example |
|---|---|---|
| **Resource Owner** | The end user who controls their data | You |
| **Resource Server** | Hosts protected resources | Google's user profile API |
| **Client** | The app requesting access | A third-party website |
| **Authorization Server** | Issues tokens after authentication | Google's auth server |

### Grant Types

| Grant Type | Use Case | Status |
|---|---|---|
| **Authorization Code** | Most apps (web, mobile with backend) | Recommended |
| **Authorization Code + PKCE** | SPAs and mobile apps (no backend secret) | Recommended |
| **Implicit** | SPAs (frontend-only) | **Deprecated** -- use Auth Code + PKCE |
| **Resource Owner Password** | Trusted first-party apps | Limited use |
| **Client Credentials** | Server-to-server, no user involved | Active |

### Authorization Code Grant Flow

The most common and secure grant type:

1. Client redirects user's browser to authorization server with requested **scopes** (permissions)
2. User reviews and **grants or denies** the access request
3. Authorization server redirects browser back to client with an **authorization code** (short-lived, one-time use)
4. Client sends the authorization code + its own credentials to the authorization server (back-channel, server-to-server)
5. Authorization server validates and returns an **access token** (and optionally a refresh token)

The key security property: the access token is never exposed to the browser. The authorization code is exchanged server-side.
![[auth-oauth2-code-grant-flow.webp]]
### Why Implicit Grant Is Deprecated

The implicit grant returned the access token directly in the URL fragment -- visible in browser history, referrer headers, and logs. Authorization Code + PKCE achieves the same goal without exposing tokens.

## Biometric Authentication

Uses unique physical characteristics (face, fingerprint, retina) to verify identity. Falls under the **inherence** factor.

### Limitations

- Biometric data can be stolen or replicated (unlike passwords, you can't change your fingerprint)
- Not everyone has distinct or consistent biometric features
- Privacy concerns around biometric data storage
- **Should never be the sole factor** -- always combine with other methods (MFA)

## Multi-Factor Authentication (MFA)

MFA requires **two or more** authentication factors from different categories:

```
Something you KNOW  (password, PIN)
        +
Something you OWN   (phone, hardware key)
        +
Something you ARE   (fingerprint, face)
```

Using two factors from the same category (e.g., password + security question) is **not** true MFA -- both are knowledge factors.

### Time-based OTP (TOTP)

The most common MFA implementation. Uses a shared secret and the current timestamp to generate codes.
![[auth-totp-flow.webp]]
**Setup:**

1. User enables 2FA in application
2. Server generates a **secret key** and encodes it in a QR code
3. User scans QR code with authenticator app (Google Authenticator, Authy)
4. App stores the secret key locally on device

**Usage:**

5. Every **30 seconds**, the app generates a 6-digit code: `HMAC-SHA1(secret, floor(timestamp / 30))`
6. User enters the code into the application
7. Server independently generates the same code using its copy of the secret
8. If codes match → access granted

Standards: TOTP is defined in RFC 6238, HOTP (counter-based) in RFC 4226. Both developed under OATH (Initiative for Open Authentication).

## Passwordless Authentication with FIDO

The FIDO (Fast IDentity Online) Alliance, backed by Google, Apple, and Microsoft (joint pledge May 2022), aims to eliminate server-stored passwords entirely.
![[auth-fido-timeline.webp]]
### Core Principle

**Private keys never leave the user's device.** The server only stores the public key. Authentication uses asymmetric cryptography -- even if the server is breached, no credentials are exposed.

### FIDO2 Framework

| Component | Role |
|---|---|
| **WebAuthn** | Web API standard for built-in authenticators (on-device biometrics, PINs) |
| **CTAP** (Client-to-Authenticator Protocols) | Enables external authenticators (security keys, mobile devices, wearables) |

### Registration Flow

1. Relying Party (RP) server sends a **challenge** to the client via WebAuthn
2. Client forwards challenge to the authenticator
3. User approves via biometric/PIN/security key
4. Authenticator creates a **new key pair** -- private key stored locally, public key + credential ID + attestation sent back
5. Client forwards public key, client data, and attestation to RP server
6. RP server validates and stores the public key for the user

### Authentication Flow

1. RP server sends a challenge
2. User approves via biometric/PIN
3. Authenticator signs the challenge with the private key
4. Server verifies the signature with the stored public key
![[auth-fido2-registration-flow.webp]]
### Adoption

Already supported by: Google, Microsoft (Windows Hello), Apple (Touch ID/Face ID passkeys), Dropbox, GitHub. FIDO works **alongside** existing protocols (SSO, OAuth) -- it replaces the password step, not the entire flow.

## Evolution Summary

```
Passwords (HTTP Basic)
  └─ Problem: no state, credentials every request
      │
Session-Cookie
  └─ Problem: stateful, CSRF, scalability
      │
Token / JWT
  └─ Problem: still needs initial password
      │
SSO (SAML / OIDC)
  └─ One password for many apps
      │
OAuth 2.0 + OIDC
  └─ Delegated access, no password sharing with third parties
      │
OTP / MFA
  └─ Multiple factors, harder to compromise
      │
FIDO2 (WebAuthn + CTAP)
  └─ No server-stored passwords at all
     Private key never leaves device
```

## Comparison of Passwordless Methods

| | OTP | SSO | OAuth 2.0 + OIDC | Biometric | FIDO2 |
|---|---|---|---|---|---|
| **Factor type** | Ownership | Knowledge (single password) | Knowledge (delegated) | Inherence | Ownership + Inherence |
| **Password required** | Yes (+ OTP) | Yes (once) | Yes (at identity provider) | No | No |
| **Server stores secret** | Yes (OTP seed) | Yes (at CAS) | No (token-based) | Yes (biometric template) | **No** (only public key) |
| **Phishing resistant** | Partially | No | Partially | Yes | **Yes** (cryptographic) |
| **Cross-platform** | Yes (SMS/email) | Yes (via CAS) | Yes (standard protocols) | Device-dependent | Yes (passkeys sync) |
| **User experience** | Extra step each login | Seamless after first login | One-click social login | Seamless (face/finger) | Seamless (face/finger/key) |
| **Best for** | MFA second factor | Enterprise apps | Third-party integrations | Device unlock | Future of web auth |

## Key Takeaways

- **Three authentication factors**: knowledge (passwords), ownership (devices), inherence (biometrics) -- true MFA uses factors from different categories
- **OTP** adds an ownership factor but SMS delivery is vulnerable to SIM swapping -- prefer authenticator apps
- **SSO** centralizes authentication via a CAS server; SAML for enterprise, OIDC for consumer apps
- **OAuth 2.0** handles authorization (delegated access), **OIDC** adds authentication (identity verification) on top
- The **Authorization Code + PKCE** grant is now the recommended flow; implicit grant is deprecated
- **Biometrics** should never be the sole factor -- combine with passwords or tokens
- **TOTP** (RFC 6238) generates time-based codes from a shared secret; the standard behind Google Authenticator
- **FIDO2** (WebAuthn + CTAP) eliminates server-stored passwords entirely -- private keys never leave the device
- The industry is moving toward **passkeys** (FIDO2) as the default authentication method
![[auth-evolution-summary.webp]]
## See Also

- [Authentication Methods](authentication-methods.md) -- Part 1: Password, Session-Cookie, Token, and JWT
