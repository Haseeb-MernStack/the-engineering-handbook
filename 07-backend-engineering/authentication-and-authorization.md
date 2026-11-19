# Authentication and Authorization

> Authentication establishes who is calling; authorization decides what that caller may do, and confusing the two is the root of most access-control bugs.

## Overview

Authentication (authn) answers "who are you?" Authorization (authz) answers "are you allowed to do this, to this resource, right now?" They are separate concerns with separate failure modes. A system can authenticate perfectly and still let any logged-in user read any other user's data.

Backend services sit at the point where identity becomes a decision. Every request must carry proof of identity, the service must validate it, and then the service must apply policy to the specific action. Doing this once at the edge is not enough. In a system with several services, each service must enforce its own authorization.

This topic covers the mechanics. For threat modeling, secrets handling, and broader hardening, see the [Security section](../10-security/README.md).

## Core Concepts

### Credentials and sessions

| Mechanism | How it works | Typical use |
|-----------|--------------|-------------|
| Server-side session | Server stores session state; client holds an opaque ID in a cookie | Browser apps served by one backend |
| Signed token (for example, JWT) | Client holds a signed, self-describing token; server verifies the signature | Stateless APIs, service-to-service calls |
| Opaque access token | Random string; server looks it up or asks the issuer to introspect | APIs that need instant revocation |
| API key | Long-lived secret identifying an application | Simple server-to-server integrations |
| Mutual TLS | Both sides present certificates | Service-to-service inside a trust boundary |

Passwords, if you store them at all, must be hashed with a slow, salted, purpose-built algorithm (such as argon2, scrypt, or bcrypt), never a fast general-purpose hash. Prefer delegating login to an identity provider and avoid owning password storage.

### OAuth 2.0 and OpenID Connect

OAuth 2.0 is a framework for delegated authorization. A user lets an application act on their behalf without sharing a password. The application receives an access token with limited scope. OpenID Connect (OIDC) adds an identity layer on top: an ID token that tells the application who the user is.

Key points:

- The authorization code flow with PKCE is the recommended flow for browser and mobile apps. The implicit flow is discouraged.
- Access tokens are for the resource server (your API). ID tokens are for the client. Do not accept an ID token as an API credential.
- Scopes limit what a token can do. They are coarse. They do not replace per-resource checks.
- Short-lived access tokens plus refresh tokens limit the damage of a leak.

### Validating a token

A backend that accepts signed tokens must check, at minimum:

1. The signature, using a key from a trusted source, with an expected algorithm fixed in configuration (never taken from the token header).
2. The issuer (`iss`) and audience (`aud`) match your expectations.
3. The expiry (`exp`) and not-before (`nbf`) times, allowing a small clock skew.
4. Any required scopes or claims.

```ts
import { jwtVerify, createRemoteJWKSet } from "jose";

const jwks = createRemoteJWKSet(new URL(process.env.JWKS_URL!));

export async function authenticate(authHeader?: string) {
  const token = authHeader?.replace(/^Bearer /, "");
  if (!token) throw new HttpError(401, "missing credentials");
  const { payload } = await jwtVerify(token, jwks, {
    issuer: process.env.TOKEN_ISSUER!,
    audience: "orders-api",
    algorithms: ["RS256"],
  });
  return { userId: payload.sub as string, scopes: String(payload.scope ?? "").split(" ") };
}
```

### Authorization models

| Model | Decision based on | Strength | Weakness |
|-------|-------------------|----------|----------|
| RBAC (role-based) | The caller's roles | Simple, easy to audit | Role explosion; cannot express "own resources only" alone |
| ABAC (attribute-based) | Attributes of caller, resource, action, context | Expressive | Harder to reason about and test |
| ReBAC (relationship-based) | Relationships in a graph (owner, member, editor) | Natural for sharing models | Needs a relationship store and consistent updates |
| ACL (per-resource lists) | Explicit entries on each resource | Precise | Management overhead at scale |

Most real systems combine them: roles for coarse capabilities, plus an ownership or relationship check on each resource.

### Where to enforce

- **Gateway.** Good for authentication, coarse scope checks, and rate limits.
- **Service.** Required for resource-level decisions, because only the service knows who owns what.
- **Data layer.** Row-level filters (for example, always including the tenant ID in queries) provide a second line of defense.

A good default is to centralize the policy logic in one module or policy engine, and call it from every handler. Scattered inline `if` statements drift apart over time.

```ts
async function getOrder(ctx: Ctx, orderId: string) {
  const order = await orders.findById(orderId);
  if (!order) throw new HttpError(404);
  if (!policy.can(ctx.user, "order:read", order)) {
    // Return 404 to avoid confirming the resource exists, if that matters for your threat model.
    throw new HttpError(404);
  }
  return order;
}
```

### Service-to-service identity

Internal calls need identity too. Options include mutual TLS with workload identities, or short-lived signed tokens issued to each service. Passing the end user's identity downstream (as a signed claim) lets the downstream service make user-level decisions. Do not rely on "it came from inside the network" as proof of anything.

## Trade-offs

- **Stateless tokens versus server sessions.** Signed tokens avoid a lookup per request and scale simply. The cost is revocation: a valid token stays valid until it expires. Short lifetimes, a deny-list, or introspection reduce this, at the cost of reintroducing state. Server sessions revoke instantly but require shared storage.
- **Token lifetime.** Short lifetimes limit exposure but increase refresh traffic and the chance of clients failing during refresh. Long lifetimes are convenient and dangerous.
- **Rich claims in tokens.** Embedding roles and permissions in the token avoids lookups but makes changes slow to propagate and tokens large.
- **RBAC versus finer models.** RBAC is easy to explain to auditors. Finer models fit real sharing rules but cost more to build, test, and operate.
- **Central policy engine versus inline checks.** A central engine gives consistency and auditability, and adds a dependency and a learning curve. Inline checks are quick and drift.
- **Build versus buy identity.** Using a managed identity provider reduces the risk of subtle errors in login flows. It adds vendor dependence and cost.

## Failure Modes

- **Broken object-level authorization (IDOR).** The endpoint `/orders/123` checks that the user is logged in, not that order 123 is theirs. Changing the ID reads another customer's data. This is among the most common API vulnerabilities. Detect with tests that call every endpoint as a different user. Prevent with a mandatory ownership check in a shared access layer.
- **Trusting client-supplied identity.** The service reads `user_id` from the request body or a header that the client can set. Prevent by deriving identity only from the validated credential.
- **Accepting unsigned or wrongly signed tokens.** Historic bugs include honoring a token whose header says the algorithm is "none", or confusing symmetric and asymmetric keys. Prevent by fixing allowed algorithms in configuration and using a maintained library.
- **Skipping audience checks.** A token issued for service A is accepted by service B. Prevent by validating `aud`.
- **Authorization only at the edge.** An internal service trusts every caller, so one compromised service reaches everything. Prevent with per-service enforcement and least-privilege service identities.
- **Stale permissions.** A user is removed from a team but their token still carries the old role for an hour. Decide what staleness is acceptable and set lifetimes accordingly.
- **Leaked long-lived secrets.** API keys in repositories or logs. Detect with secret scanning in CI. Prevent with rotation and short-lived credentials.
- **Tenant leakage.** A query forgets the tenant filter and returns rows across customers. Prevent with row-level security or a repository layer that always injects the tenant.
- **Verbose auth errors.** "Wrong password" versus "no such user" enables account enumeration. Return the same response for both.
- **Logging tokens.** Authorization headers captured in access logs. Redact them. See [Observability in Services](./observability-in-services.md).

## In Practice

### Checklist for a new endpoint

- Is the caller authenticated with a validated credential?
- What action is being performed, and on which resource?
- Which policy function decides, and is it called on every code path (including bulk and export endpoints)?
- Does the failure response avoid leaking whether the resource exists, where that matters?
- Is there a test that a different user, a different tenant, and an unauthenticated caller are each refused?
- Are denials logged with who, what, and why (without secrets)?

### Worked example: ownership bug review

A reviewer sees:

```ts
router.get("/invoices/:id", requireLogin, async (req, res) => {
  res.json(await invoices.findById(req.params.id));
});
```

The route is authenticated but not authorized. The fix is to load the invoice, check that `invoice.accountId` is among the caller's accounts via the policy module, and return 404 otherwise. Then add a test: user B requests user A's invoice and gets 404.

### Developer vs. engineer

A developer makes the login work and moves on. An engineer lists every action in the system, decides who may perform each, writes those rules in one place, and proves with tests that the forbidden cases are refused. The engineer also plans for revocation and key rotation before an incident forces it.

### When AI generates auth code

AI-generated code often looks correct and omits the hard parts. Verify:

- Token validation checks signature, issuer, audience, and expiry, with algorithms pinned.
- Handlers check resource ownership, not just login.
- Secrets come from configuration or a secret manager, not literals.
- Password handling uses a vetted slow hash, or better, delegates to an identity provider.
- Cookies set `HttpOnly`, `Secure`, and an appropriate `SameSite` value.
- Nothing invents a custom cryptographic scheme.

## Further Reading

- RFC 6749, The OAuth 2.0 Authorization Framework (IETF).
- RFC 7636, Proof Key for Code Exchange (PKCE) (IETF).
- RFC 7519, JSON Web Token (JWT) (IETF), and RFC 8725, JSON Web Token Best Current Practices.
- OpenID Connect Core specification (OpenID Foundation).
- OWASP API Security Top 10 and the OWASP Authentication and Session Management cheat sheets (owasp.org).
- Ross Anderson, Security Engineering (access control chapters).
