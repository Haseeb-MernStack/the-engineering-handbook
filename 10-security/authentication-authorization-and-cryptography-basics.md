# Authentication, Authorization and Cryptography Basics

> Authentication establishes who is calling, authorization decides what they may do, and cryptography provides the primitives that make both trustworthy; each must be used through proven building blocks rather than invented.

## Overview

Three questions sit under almost every security decision. Who are you? What are you allowed to do? Can I trust that this data is intact and private? The first is authentication, the second is authorization, and the third is where cryptography applies.

Confusing them causes real defects. A system can authenticate perfectly and still let any logged-in user read any record. It can use strong encryption and still leak data because keys are stored beside the ciphertext. Identity, permission, and cryptography are separate concerns that must be designed together.

The most important rule in this area is a negative one: do not invent your own. Use established protocols, maintained libraries, and managed services. Cryptography in particular fails silently. A flawed implementation works in every functional test and breaks only under attack.

## Core Concepts

### Authentication

Authentication verifies a claimed identity using one or more factors:

- Something you know (password, PIN)
- Something you have (hardware key, authenticator app, phone)
- Something you are (biometric, usually checked locally on a device)

**Multi-factor authentication** combines at least two different categories. Factors that resist phishing, such as hardware-backed credentials using the WebAuthn standard, are stronger than codes that a user can be tricked into typing into a fake site.

#### Passwords

If you store passwords, store them using a slow, salted, purpose-built password hashing function. Current widely recommended choices include Argon2id, scrypt, and bcrypt; follow the OWASP Password Storage Cheat Sheet for current parameters. Do not use fast general hashes (such as plain SHA-256) for passwords, and never use reversible encryption.

```python
# Illustrative: using a maintained library with sensible defaults
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher()  # library defaults; review them against current guidance

def register(password: str) -> str:
    return ph.hash(password)          # salt is generated and embedded

def verify(stored_hash: str, password: str) -> bool:
    try:
        return ph.verify(stored_hash, password)
    except VerifyMismatchError:
        return False
```

Prefer not to store passwords at all when you can delegate to an identity provider.

#### Sessions and tokens

After login, the system issues a credential the client presents on each request.

| Mechanism | How it works | Strength | Weakness |
|-----------|--------------|----------|----------|
| Server-side session ID in a cookie | Server stores session state; cookie holds an opaque random ID | Easy to revoke instantly | Needs shared session storage when scaled out |
| Signed token (such as JWT) | Claims are signed; server verifies signature | Stateless verification | Hard to revoke before expiry; easy to misuse |
| Opaque access token with introspection | Token is a random string checked against an authority | Revocable, no claims leakage | Extra network call |

Cookies for browser sessions should be marked `HttpOnly`, `Secure`, and with a suitable `SameSite` attribute. Rotate the session identifier on login and on privilege changes. Set sensible idle and absolute expiry.

If you use JWTs, pin the accepted algorithms in the verifier, check issuer, audience, and expiry, and keep lifetimes short. Never accept a token that claims to be unsigned. Many JWT incidents come from verifier configuration, not from the format itself.

#### OAuth 2.0 and OpenID Connect

OAuth 2.0 is an authorization delegation framework: it lets an application obtain limited access to a resource on a user's behalf. OpenID Connect adds an identity layer on top for authentication. For most applications, use an established provider or library and the authorization code flow with PKCE for public clients. Do not use OAuth alone as proof of identity, and do not implement the protocol by hand. The relevant specifications are RFC 6749 (OAuth 2.0) and RFC 7636 (PKCE).

### Authorization

Authorization answers: may this actor perform this action on this resource?

| Model | Decision based on | Fits when | Drawback |
|-------|-------------------|-----------|----------|
| Role-based (RBAC) | The roles assigned to the user | Few, stable job functions | Role explosion; cannot express "own records only" alone |
| Attribute-based (ABAC) | Attributes of user, resource, and context | Fine-grained or contextual rules | Harder to reason about and test |
| Relationship-based | Graph of relationships (owner, member, viewer) | Sharing, hierarchies, collaboration | Needs a dedicated model and storage |

Most real systems combine them: roles for coarse access, ownership checks for object-level access.

Principles that apply to any model:

- **Deny by default.** Missing rule means no access.
- **Enforce on the server.** The UI hides, the server decides.
- **Check at every layer that matters.** Object-level checks belong close to data access.
- **Centralize policy.** Scattered `if user.role == "admin"` checks drift and contradict each other.
- **Least privilege.** Grant the narrowest access that lets the work happen, for the shortest time.
- **Separate identity from permission.** A valid token proves who; it does not prove what.

```python
def can_edit_document(user, doc) -> bool:
    if doc.owner_id == user.id:
        return True
    return doc.id in user.editable_document_ids  # explicit grant, otherwise False
```

Service-to-service calls need authorization too. Inside a network, every service should authenticate its callers (for example with mutual TLS or short-lived signed credentials) and authorize specific operations. Network location alone is not identity.

### Cryptography basics

| Primitive | Purpose | Notes |
|-----------|---------|-------|
| Cryptographic hash | Integrity fingerprint, one-way | Not encryption. Not for passwords on its own. |
| Message authentication code (MAC) | Integrity and authenticity with a shared secret | HMAC is the common construction. |
| Symmetric encryption | Confidentiality with one shared key | Use authenticated encryption (AEAD) modes such as AES-GCM or ChaCha20-Poly1305. |
| Asymmetric encryption and signatures | Key exchange, identity, signing | Slower; used to exchange keys or sign, not bulk data. |
| Key derivation function | Turn a secret into keys | Use purpose-built functions, not ad hoc hashing. |
| Cryptographically secure random generator | Tokens, keys, nonces | Use the operating system source, never a general-purpose random function. |

Practical rules:

1. **Use TLS for data in transit.** Everywhere, including internal traffic. Use current versions, verify certificates, and do not disable verification to make an error go away.
2. **Use authenticated encryption.** Encryption without authentication allows undetected tampering.
3. **Never reuse a nonce with the same key** in modes where that is catastrophic. Prefer libraries that manage nonces for you.
4. **Never roll your own algorithms or protocols.** Choose high-level libraries with opinionated APIs.
5. **Separate keys from data.** Store keys in a key management service or hardware-backed store, not beside the ciphertext. Rotate keys and plan for it from the start.
6. **Use secure randomness.** Predictable tokens are a common authentication flaw.

```python
import secrets

reset_token = secrets.token_urlsafe(32)   # from the OS CSPRNG
# Store only a hash of the token server-side; compare in constant time.
```

Comparing secrets with ordinary equality can leak information through timing; use a constant-time comparison function from the standard library.

## Trade-offs

| Decision | Gains | Costs |
|----------|-------|-------|
| Build login yourself | Full control | You own every flaw: hashing, reset, lockout, enumeration, MFA |
| Delegate to an identity provider | Mature features, less code | Vendor dependence, integration complexity, cost |
| Stateless tokens | Horizontal scale, no session store | Revocation difficulty, larger requests, misuse risk |
| Server-side sessions | Instant revocation | State to manage and replicate |
| Short token lifetimes | Smaller theft window | More refresh traffic and failure handling |
| Fine-grained authorization | Least privilege | Policy complexity, performance, testing burden |
| Encrypting at the application layer | Protects data from database operators | Harder querying, key management burden |
| Encrypting at rest at the disk or database layer | Simple, covers stolen media | Does not protect against a compromised application |
| Strong lockout and rate limits | Resist guessing | Can be abused to lock out real users |

Authentication friction is a real cost. Excessive prompts push users toward weak workarounds. Strong, low-friction options such as passkeys help resolve this tension.

## Failure Modes

**Authentication without authorization.** Every request has a valid session, none check ownership. Test with two accounts and swap identifiers.

**Account enumeration.** Login, signup, or reset responses reveal whether an email exists. Use uniform messages and similar response time.

**Credential stuffing.** Attackers replay passwords leaked elsewhere. Defenses: MFA, rate limiting by account and source, breached-password checks, anomaly detection.

**Token misuse.** Tokens in URLs end up in logs and referrers. Long-lived tokens in local storage are exposed to any script injection. Keep lifetimes short, prefer `HttpOnly` cookies for browser sessions, and avoid putting secrets in URLs.

**Misconfigured verification.** A verifier that accepts any algorithm, skips audience checks, or ignores expiry. Detect with tests that submit expired, wrongly signed, and wrong-audience tokens and expect rejection.

**Privilege escalation through role assignment.** An endpoint that lets users edit their own role field. Treat role and permission fields as protected attributes and never bind them from request bodies automatically.

**Broken key handling.** Keys in source control, shared across environments, never rotated, or stored next to encrypted data. See [Secrets and Supply Chain Security](./secrets-and-supply-chain-security.md).

**Homemade crypto.** Custom ciphers, ad hoc "encryption" with base64 or XOR, unsalted fast hashes for passwords, predictable IVs. Replace with library primitives. Detect in review by searching for low-level crypto calls outside a small, reviewed wrapper module.

**Disabled certificate checking.** A `verify=False` added during debugging reaches production. Make it a lint failure.

**Stale sessions.** Sessions that survive password change, role removal, or employee departure. Invalidate on those events and test it.

## In Practice

### Checklist for a new login system

- [ ] Prefer delegation to an identity provider unless you have a reason not to.
- [ ] Passwords hashed with a modern password hashing function with per-user salt.
- [ ] MFA available, phishing-resistant options where possible.
- [ ] Sessions rotate on login and privilege change, expire on idle and absolute limits.
- [ ] Rate limiting and anomaly detection on authentication endpoints.
- [ ] Uniform responses for unknown accounts.
- [ ] Recovery flow treated as seriously as login, since it is an alternate door.
- [ ] Authorization tests for each role and each object boundary.
- [ ] Authentication events logged without logging credentials.

### Choosing a cryptographic approach

1. Can a managed service do it (key management, secrets, identity)? Use it.
2. If not, is there a high-level library function for the exact task? Use it.
3. If not, stop and consult someone with cryptographic experience before writing code.

### Developer vs. engineer

A developer wires up login and verifies that the happy path works. An engineer writes the negative tests: expired token, tampered token, other user's resource, replayed reset link, and role change mid-session.

### When AI generates code

Authentication and cryptography are areas where generated code often looks correct and is not. Verify:

- Password storage uses a real password hashing function, not a fast hash.
- Randomness for tokens comes from a secure generator.
- JWT verification fixes the algorithm and checks issuer, audience, and expiry.
- Certificate verification is on.
- Hard-coded keys or example secrets are removed.
- Authorization exists per object, not only a login check.
- Library APIs and parameters exist in the version you use, since generated code can mix versions.

## Further Reading

- Ross Anderson, *Security Engineering*. Chapters on access control, protocols, and cryptography in real systems.
- Niels Ferguson, Bruce Schneier, and Tadayoshi Kohno, *Cryptography Engineering*. Practical design principles and common mistakes.
- Jean-Philippe Aumasson, *Serious Cryptography*. A readable introduction to modern primitives.
- RFC 6749, The OAuth 2.0 Authorization Framework, and RFC 7636, Proof Key for Code Exchange (https://www.rfc-editor.org).
- OWASP Cheat Sheet Series: Authentication, Session Management, Password Storage, and Authorization (owasp.org).

Back to the section: [Security](./README.md)
