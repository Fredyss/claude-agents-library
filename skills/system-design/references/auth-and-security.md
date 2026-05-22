# Authentication, Authorization & Security

Three distinct concerns that are easy to conflate:
- **Authentication (AuthN)** — *who are you?* Proving identity.
- **Authorization (AuthZ)** — *what are you allowed to do?* Deciding access, after identity is established.
- **Security** — defending the whole system against misuse, regardless of who the actor is.

You authenticate first, then authorize, and you secure throughout.

## Contents
- Authentication: methods and credentials
- Sessions vs tokens (JWT)
- OAuth 2.0 and OpenID Connect
- Authorization models (RBAC, ABAC)
- Security: the practical checklist
- Common vulnerabilities

## Authentication

Establishing identity. Built from one or more **factors**: something you know (password), something you have (phone, hardware key), something you are (biometrics). Combining factors = **multi-factor authentication (MFA)**, which dramatically raises the bar against credential theft.

Handling passwords correctly is non-negotiable:
- **Never store plaintext passwords.** Store a salted **hash** using a slow, purpose-built algorithm (bcrypt, scrypt, or Argon2 — never plain MD5/SHA-256, which are too fast and easy to brute-force).
- A **salt** (unique random value per password) defeats precomputed rainbow tables and ensures identical passwords hash differently.
- Slowness is a feature here: it makes brute-forcing expensive.

## Sessions vs tokens

After login, the system needs to remember the user across stateless requests. Two dominant approaches:

### Session-based (stateful)
The server creates a session, stores it server-side (in memory, a DB, or a shared cache), and hands the client an opaque **session ID** (usually in a cookie). Each request sends the ID; the server looks it up.
- **Pros**: easy to revoke instantly (delete the session); the ID reveals nothing.
- **Cons**: the server must store and look up session state, which complicates horizontal scaling — you need a **shared** session store (e.g. Redis), not per-server memory (see `scaling.md` on statelessness).

### Token-based (stateless) — JWT
The server issues a signed **JSON Web Token (JWT)** containing claims (user id, roles, expiry). The client sends it (typically `Authorization: Bearer <token>`); the server **verifies the signature** and trusts the contents — no server-side lookup.
- A JWT has three parts: header, payload (claims), and signature. The signature proves the token wasn't tampered with; it does **not** encrypt the payload — never put secrets in it (it's base64, readable by anyone).
- **Pros**: stateless and scales naturally (any server can verify without shared storage); works well across services.
- **Cons**: **revocation is hard** — a valid token stays valid until it expires. Mitigate with short-lived access tokens plus longer-lived **refresh tokens**, and a denylist for emergency revocation. Keep access-token lifetimes short.

Rule of thumb: sessions for traditional single-app web with easy revocation needs; JWTs for APIs, mobile clients, and multi-service architectures where statelessness matters. Either way, transmit over HTTPS only, and for cookies use `HttpOnly`, `Secure`, and `SameSite`.

## OAuth 2.0 and OpenID Connect

- **OAuth 2.0** is an **authorization** framework for **delegated access** — letting an app act on a user's behalf against another service *without* the user handing over their password ("Connect your Google account"). The app receives a scoped **access token**, not credentials. The **Authorization Code flow (with PKCE)** is the standard for web and mobile apps.
- **OpenID Connect (OIDC)** is a thin **authentication** layer on top of OAuth 2.0. It adds an **ID token** (a JWT identifying the user), which is what powers "Sign in with Google/Apple/GitHub."

The common confusion: OAuth alone is about *access delegation* (authorization); OIDC adds *who the user is* (authentication). "Log in with X" needs OIDC.

## Authorization models

Deciding what an authenticated user may do:

- **RBAC (Role-Based Access Control)**: assign users to roles (admin, editor, viewer); permissions attach to roles. Simple, the most common model, easy to audit. Can get unwieldy with many fine-grained, overlapping roles ("role explosion").
- **ABAC (Attribute-Based Access Control)**: decisions evaluate attributes of the user, resource, action, and context (e.g. "editors may edit documents *in their own department* during business hours"). Far more expressive and fine-grained; more complex to implement and reason about.
- **Other patterns**: ACLs (per-object permission lists) and relationship-based access control (ReBAC, e.g. Google Zanzibar — "can edit because they own the parent folder") for systems where access derives from relationships.

Start with RBAC; reach for ABAC/ReBAC when role-only rules can't express the policy. Enforce authorization **server-side on every request** — never trust the client to hide a button. And follow **least privilege**: grant the minimum access needed.

## Security: the practical checklist

Security is layered ("defense in depth"), applied throughout the design rather than bolted on:

- **Encrypt in transit**: TLS/HTTPS everywhere, no exceptions. Use HSTS.
- **Encrypt at rest**: sensitive data and backups encrypted on disk.
- **Validate and sanitize all input** server-side. Treat every client input as hostile.
- **Use parameterized queries / prepared statements** — the primary defense against SQL injection. Never build queries by string concatenation.
- **Manage secrets properly**: keys and credentials in a secrets manager / environment config, never in source code or logs. Rotate them.
- **Principle of least privilege** for users, services, and database accounts alike.
- **Rate limit and throttle** auth endpoints especially (see `api-design.md`) to blunt brute-force and abuse.
- **Log and monitor** security-relevant events; you can't respond to what you can't see. Avoid logging secrets/PII.
- **Keep dependencies patched** — known-vulnerable libraries are a leading breach vector.

## Common vulnerabilities

Know these by name; they recur constantly (the OWASP Top 10 is the canonical list):

- **Injection (SQL, command, etc.)**: untrusted input interpreted as code. Defense: parameterized queries, input validation, least-privilege DB accounts.
- **Broken authentication**: weak passwords, poor session/token handling, missing MFA. Defense: strong hashing, secure session/token management, MFA.
- **Broken access control**: users acting outside their permissions, e.g. **IDOR** — changing `/orders/123` to `/orders/124` and seeing someone else's data. Defense: enforce authorization on every request against the *actual* owner, not just authentication.
- **XSS (Cross-Site Scripting)**: injecting malicious scripts into pages other users view. Defense: output encoding, Content-Security-Policy, framework auto-escaping.
- **CSRF (Cross-Site Request Forgery)**: tricking a logged-in user's browser into making unwanted requests. Defense: anti-CSRF tokens, `SameSite` cookies.
- **Sensitive data exposure**: weak/absent encryption, secrets in code or logs. Defense: encrypt in transit and at rest, proper secrets management.
- **Security misconfiguration**: default credentials, verbose errors leaking internals, open cloud buckets. Defense: hardened defaults, least privilege, regular review.

When designing, name the threats relevant to the system and the specific control that mitigates each — a security story is "here's the attack, here's the defense," not a list of buzzwords.
