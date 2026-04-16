# Password, Session, Cookie, Token, JWT, SSO, OAuth - Authentication Explained - Part 2

> Source: ByteByteGo Newsletter, Alex Xu, 2023-04-12
> URL: https://blog.bytebytego.com/p/password-session-cookie-token-jwt-sso-oauth

## Passwordless Authentication

We have covered three types of authentication so far: HTTP basic authentication, session-cookie authentication, and token-based authentication. They all require a password. However, there are other ways to prove your identity without a password.

When it comes to authentication, there are three factors to consider:

- Knowledge factors: something you know, such as a password
- Ownership factors: something you own, such as a device or phone number
- Inherence factors: something unique to you, such as your biometric features

Passwords fall under "something you know". One-Time Passwords (OTP) prove that the user owns a cell phone or a device, while biometric authentication proves "something unique to you".

## One-Time Passwords (OTP)

One-Time Passwords (OTP) are widely used as a more secure method of authentication. Unlike static passwords, which can be reused, OTPs are valid for a limited time, typically a few minutes. This means that even if someone intercepts an OTP, they can't use it to log in later. Additionally, OTPs require "something you own" as well as "something you know" to log in. This can be a cell phone number or email address that the user has access to, making it harder for hackers to steal.

However, it's important to note that using SMS as the delivery method for OTPs can be less secure than other methods. This is because SMS messages can be intercepted or redirected by hackers, particularly if the user's phone number has been compromised. In some cases, attackers have been able to hijack phone numbers by convincing the mobile carrier to transfer the number to a new SIM card (SIM swapping). Once the attacker has control of the number, they can intercept any OTPs sent via SMS. For this reason, it's recommended to use alternative delivery methods, such as email or mobile apps, whenever possible.

### OTP Flow

1. The user wants to log in and is asked to enter a username, cell phone number, or email.
2. The server generates an OTP with an expiration time.
3. The server sends the OTP to the user's device via SMS or email.
4. The user enters the OTP received in the login box.
5. The server compares the generated OTP with the one the user entered. If they match, login is granted.

Alternatively, a hardware or software key can be used to generate OTPs for multi-factor authentication (MFA). For example, Google 2FA uses a software key that generates a new OTP every 30 seconds.

## SSO (Single Sign-On)

Single Sign-On (SSO) is a user authentication method that allows us to access multiple systems or applications with a single set of credentials. SSO streamlines the login process, providing a seamless user experience across various platforms.

The SSO process mainly relies on a Central Authentication Service (CAS) server.

### SSO Flow

1. When we attempt to log in to an application, such as Gmail, we're redirected to the CAS server.
2. The CAS server verifies our login credentials and creates a Ticket Granting Ticket (TGT). This TGT is then stored in a Ticket Granting Cookie (TGC) on our browser, representing our global session.
3. CAS generates a Service Ticket (ST) for our visit to Gmail and redirects us back to Gmail with the ST.
4. Gmail uses the ST to validate our login with the CAS server. After validation, we can access Gmail.

When we want to access another application, like YouTube, the process is simplified:

5. Since we already have a TGC from our Gmail login, CAS recognizes our authenticated status.
6. CAS generates a new ST for YouTube access, and we can use YouTube without inputting our credentials again.

### SSO Protocols

- **SAML** (Security Assertion Markup Language): Widely used in enterprise applications. Communicates authentication and authorization data in XML format.
- **OIDC** (OpenID Connect): Popular in consumer applications. Handles authentication through JSON Web Tokens (JWT) and builds on OAuth 2.0 framework. Preferred for new applications -- supports web-based, mobile, and JavaScript clients.

## OAuth 2.0 and OpenID Connect (OIDC)

While OAuth 2.0 is primarily an authorization framework, it can be used in conjunction with OpenID Connect (OIDC) for authentication purposes. OIDC is an authentication layer built on top of OAuth 2.0, enabling the verification of a user's identity and granting controlled access to protected resources.

When using "Sign in with Google" or similar features, OAuth 2.0 and OIDC work together to streamline the authentication process. OIDC provides user identity data in the form of a standardized JSON Web Token (JWT).

OAuth 2.0 provides "secure delegated access" by issuing short-lived tokens instead of passwords, allowing third-party services to access protected resources with the resource owner's permission.

### OAuth 2.0 Roles

- **Resource owner**: The end user, who controls access to their personal data.
- **Resource server**: The Google server hosting user profiles as protected resources. Uses access tokens to respond to protected resource requests.
- **Client**: The device (PC or smartphone) making requests on behalf of the resource owner.
- **Authorization server**: The Google authorization server that issues tokens to clients.

### OAuth 2.0 Grant Types

- **Authorization code grant**: The most complete and versatile mode, suitable for most application types.
- **Implicit grant**: Designed for applications with only a frontend. No longer recommended.
- **Resource owner password credentials grant**: Used when users trust a third-party application with their credentials.
- **Client credentials grant**: Suitable for cases without a frontend, like command-line tools or server-to-server communication.

### Authorization Code Grant Flow

1. The client directs the resource owner's user agent (typically a web browser) to the authorization server, presenting the requested permissions (scopes).
2. The resource owner either grants or denies the client's access request.
3. If the resource owner grants access, the authorization server redirects the user agent back to the client using a redirection URI containing an authentication code.
4. The client uses the authentication code to request an access token from the authorization server. The code and redirection URI must match those issued in step 3.
5. After successful validation, the authorization server provides an access token, enabling the client to access the user's protected resources within the defined scope.

Note: The implicit grant is no longer recommended due to security concerns (exposure of access tokens). The Authorization Code Grant with PKCE (Proof Key for Code Exchange) is now the preferred approach for frontend applications.

While OAuth 2.0 with OIDC still requires password entry, it is considered "passwordless" in the sense that users don't need to register a new account or create passwords for third-party websites.

## Biometric Authentication

Biometric authentication uses unique physical characteristics of a person, such as facial features, fingerprints, or retinal patterns, to verify their identity. During registration, biometric features are extracted and stored securely in a backend system.

When the user logs in, the backend system compares the live biometric features with the stored ones. If they match, the user is granted access.

Limitations:
- Biometric features can be stolen or replicated
- Some people may not have distinct or consistent biometric features
- Should not be used as the sole authentication factor -- combine with other factors (MFA)

## Multi-Factor Authentication (MFA)

Multi-Factor Authentication (MFA) requires users to provide more than one form of identification to access protected resources.

MFA typically involves using two out of three authentication factors:
- Something you **know** (password or PIN)
- Something you **own** (phone or token)
- Something you **are** (fingerprint or face recognition)

### Time-based OTP (TOTP)

Two-Factor Authentication (2FA) using time-based codes relies on TOTP (Time-based One-Time Passwords). The algorithms use a seed and a changing factor (timestamp for TOTP, counter for HOTP) to generate the result.

**Setup (Steps 1-4):**
1. User enables 2FA in an application
2. Server generates a secret key and creates a QR code containing issuer, user, and secret key
3. User scans QR code with a 2FA app (e.g., Google Authenticator)
4. App stores the secret key locally on user's device

**Usage (Steps 5-10):**
5. Every 30 seconds, the 2FA app generates a 6-digit OTP using the stored secret key and current timestamp
6. User enters OTP into the application
7. Server generates its own OTP using its copy of the secret key
8. Server compares both OTPs -- if they match, access is granted

## Passwordless Authentication with FIDO (WebAuthn, CTAP)

On May 5, 2022, Google, Apple, and Microsoft pledged to support passwordless sign-in across their platforms. These standards are developed by the FIDO Alliance and the W3C. FIDO enables users to sign in using passkeys across their devices with biometric or security key authentication.

### FIDO2 Framework

- **WebAuthn**: Defines a standard web API with built-in authenticators like on-device biometrics or PINs
- **CTAP** (Client-to-Authenticator Protocols): Enables the use of external authenticators (FIDO Security Keys, mobile devices, wearables)

The basic idea: credentials (private keys) are owned by the user and managed by an authenticator. The application server only stores the public key -- private keys never leave the users' devices.

### FIDO Registration Flow

1. When the user registers, the Relying Party (RP) server sends a challenge to the RP client application using WebAuthn.
2. The client application forwards the challenge to an authenticator.
3. The user approves the request using the selected authentication method (biometric, PIN, security key).
4. A new key pair is created -- private key stored with authenticator, public key sent back to RP client along with credential ID and attestation.
5. The RP client sends the public key, client data, and attestation to the RP server.
6. The RP server validates the response and registers the public key for the user.

WebAuthn is adopted by major platforms: Google, Microsoft (Windows Hello), Dropbox. FIDO is not intended to replace existing federation protocols (SSO, OAuth) but works alongside them.
