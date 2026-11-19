# OWASP Top 10 and Secure Coding

> The OWASP Top 10 is a consensus list of the most common classes of web application risk, and secure coding is the discipline of making those classes structurally difficult to introduce.

## Overview

The Open Worldwide Application Security Project (OWASP) publishes an awareness document called the Top 10. It groups the most widely observed web application risks into categories and is revised periodically using data from the industry and community input. The category names and their order change between editions, so treat any specific ranking as a snapshot. Check the current edition on the OWASP site rather than memorizing numbers.

The Top 10 is a starting vocabulary, not a complete standard. Passing it does not make an application secure. Its value is that it gives teams a shared language for recurring failure patterns, and those patterns are remarkably stable. The same underlying mistakes appear year after year: untrusted data interpreted as code, missing access checks, weak handling of secrets, and unpatched components.

Secure coding is the practice of removing those mistakes by construction. The best defenses do not depend on a developer remembering to be careful. They make the safe path the default path: a query API that cannot concatenate, a template engine that escapes by default, a framework that denies access unless a rule allows it.

## Core Concepts

### The recurring risk families

Rather than recite a list that will date, group the categories by cause.

| Family | Root cause | Typical defense |
|--------|-----------|-----------------|
| Injection (SQL, OS command, template, LDAP, and others) | Untrusted data mixed into an interpreter's command language | Separate code from data: parameterized APIs, no string building |
| Cross-site scripting | Untrusted data rendered into a page as markup or script | Context-aware output encoding, a strict content security policy |
| Broken access control | Missing or inconsistent authorization checks | Deny by default, central enforcement, object-level checks |
| Cryptographic failures | Weak or misused cryptography, data exposed in transit or at rest | Vetted libraries, modern algorithms, TLS everywhere |
| Security misconfiguration | Insecure defaults, verbose errors, open storage, unnecessary features | Hardened baselines, config as code, automated scanning |
| Vulnerable and outdated components | Known flaws in dependencies | Inventory, update cadence, automated alerts |
| Authentication failures | Weak credential handling, broken session management | Proven identity libraries, MFA, session rotation |
| Software and data integrity failures | Trusting unsigned updates, unsafe deserialization, tampered pipelines | Signature verification, avoid native deserialization of untrusted data |
| Logging and monitoring failures | Attacks are not recorded or noticed | Security-relevant events logged and alerted on |
| Server-side request forgery | The server fetches a URL chosen by the attacker | Allowlists, network egress controls, no raw URL fetching |
| Insecure design | Missing controls at the design level | Threat modeling, secure design patterns |

Insecure design deserves emphasis. No amount of careful coding fixes a design that never included an authorization model. See [Threat Modeling](./threat-modeling.md).

### Input handling: validate, encode, and separate

Three different operations are often confused:

- **Validation** checks that input meets expectations (type, length, range, format). Prefer allowlists to denylists.
- **Encoding** transforms data so it is inert in a specific output context (HTML body, HTML attribute, URL, JavaScript, SQL). The correct encoding depends on where the data lands.
- **Separation** keeps data and commands in different channels so data is never parsed as a command.

Validation reduces noise. It is not a substitute for separation or encoding. A name field that legitimately contains an apostrophe must still be safe in a query.

### Vulnerable versus fixed: SQL injection

Vulnerable pattern: the query text is built by concatenating user input.

```python
# VULNERABLE: user input becomes part of the SQL text
def find_user(conn, email):
    query = "SELECT id, email FROM users WHERE email = '" + email + "'"
    return conn.execute(query).fetchone()
```

Fixed: the query text is constant, and values travel separately as parameters.

```python
# FIXED: parameterized query; the driver keeps data out of the SQL grammar
def find_user(conn, email):
    query = "SELECT id, email FROM users WHERE email = ?"
    return conn.execute(query, (email,)).fetchone()
```

Parameters cover values, not identifiers. If a column name or sort direction comes from the user, map it through an allowlist:

```python
ALLOWED_SORT = {"created": "created_at", "name": "display_name"}

def list_users(conn, sort_key):
    column = ALLOWED_SORT.get(sort_key, "created_at")  # default, never raw input
    return conn.execute(f"SELECT id, display_name FROM users ORDER BY {column}").fetchall()
```

The f-string is acceptable here only because `column` can take one of a fixed set of values chosen by the code.

### Vulnerable versus fixed: cross-site scripting

```javascript
// VULNERABLE: untrusted text is interpreted as HTML
element.innerHTML = comment.body;

// FIXED: treat it as text
element.textContent = comment.body;
```

Modern frameworks escape by default. Danger appears when developers opt out (`dangerouslySetInnerHTML`, `v-html`, `|safe`). If rich text must be rendered, sanitize it with a maintained sanitizer library and restrict the allowed tags, rather than writing your own filter.

### Vulnerable versus fixed: broken access control

```javascript
// VULNERABLE: authenticated, but not authorized for this specific object
app.get("/invoices/:id", requireLogin, async (req, res) => {
  const invoice = await db.invoices.findById(req.params.id);
  res.json(invoice);
});

// FIXED: scope the lookup to the caller
app.get("/invoices/:id", requireLogin, async (req, res) => {
  const invoice = await db.invoices.findOne({
    id: req.params.id,
    ownerId: req.user.id,
  });
  if (!invoice) return res.status(404).end();
  res.json(invoice);
});
```

This class of flaw, an insecure direct object reference, is common because authentication is easy to see and per-object authorization is easy to forget. Returning 404 rather than 403 avoids confirming that the object exists. See [Authentication, Authorization and Cryptography Basics](./authentication-authorization-and-cryptography-basics.md).

### Safer defaults

- Deny by default. Access requires an explicit grant.
- Fail closed. If a check errors, refuse.
- Least privilege for database accounts, service tokens, and cloud roles.
- Errors to users are generic. Details go to logs.
- Prefer high-level, well-reviewed libraries to hand-rolled parsers, encoders, and session handling.

## Trade-offs

| Choice | Benefit | Cost |
|--------|---------|------|
| Strict input validation | Rejects malformed data early | Can reject legitimate edge cases (names, addresses); needs maintenance |
| Output encoding everywhere | Strong, reliable XSS defense | Requires discipline in custom rendering paths |
| Strict Content Security Policy | Limits blast radius of script injection | Breaks inline scripts and third-party tags; needs rollout effort |
| Central authorization layer | Consistent decisions, one place to audit | Adds indirection; poorly designed policies become bottlenecks |
| ORM or query builder | Parameterization by default | Escape hatches (raw queries) reintroduce risk; abstraction can hide inefficiency |
| Rapid dependency updates | Smaller exposure window | More churn, more regression risk |
| Verbose security logging | Better detection and forensics | Cost, noise, and the risk of logging secrets or personal data |

Security controls often trade usability or velocity. State the cost openly and decide with the product owner rather than quietly weakening the control later.

## Failure Modes

**The ORM escape hatch.** A team uses an ORM safely for 95 percent of queries, then one reporting endpoint uses a raw query with string formatting. Detect with static analysis rules that flag string-built queries; prevent with a team rule that raw SQL must use parameters and be reviewed.

**Authorization only at the UI.** The button is hidden, but the endpoint answers anyone. Test every endpoint with a user who lacks the permission. Write automated tests for each role and each object boundary.

**Validation in the client only.** Client checks are a usability feature. The server must enforce everything.

**Sanitizing in the wrong place.** Data is escaped on input for one context and then rendered in another. Encode at output, for the specific context.

**Error messages that disclose.** Stack traces, SQL fragments, and internal hostnames in responses help an attacker map the system. Return a generic message with a correlation ID. Keep details in logs.

**Unsafe deserialization.** Language-native object deserialization of untrusted input can lead to code execution in some runtimes. Prefer plain data formats such as JSON with explicit schema validation.

**Server-side request forgery in "helpful" features.** Link previews, webhooks, and image import fetch URLs for users. Without allowlists and network-level egress rules, those features can reach internal services and cloud metadata endpoints. Restrict destination, resolve and check addresses, and disable redirects or re-validate after them.

**Assuming the framework handles it.** Frameworks cover common cases. Custom code paths, file uploads, redirects, and webhooks fall outside the defaults.

**No detection.** Attacks happen and nobody notices. Log authentication failures, access denials, and input validation failures, and alert on unusual volumes.

## In Practice

### Review checklist for any endpoint

- [ ] Who is allowed to call this, and where is that enforced?
- [ ] Is every object access scoped to the caller or checked against policy?
- [ ] Is every query parameterized?
- [ ] Is every value rendered to HTML, URL, or script context encoded for that context?
- [ ] Are file uploads limited by size and type, stored outside the web root, and served with safe headers?
- [ ] Are redirects restricted to known destinations?
- [ ] Are errors generic to the client and detailed in logs?
- [ ] Are secrets absent from code, logs, and URLs?
- [ ] Is rate limiting applied to expensive or sensitive actions?

### Make the safe thing easy

Encode secure patterns into shared code. Provide one `findOwnedInvoice` helper, one HTTP client wrapper with egress allowlisting, one logging helper that redacts known sensitive fields. Developers will use the easy path. Then add lint rules or CI checks that flag the unsafe alternatives.

### Developer vs. engineer

A developer fixes the specific injection reported by a scanner. An engineer asks why the codebase allowed string-built queries, removes the pattern class with a lint rule and a shared data-access layer, and adds a regression test.

### When AI generates code

Generated code frequently reproduces common but unsafe idioms from its training data, especially in examples and quick scripts. Verify:

- Queries are parameterized, not interpolated.
- Authorization is present per object, not only login.
- Cryptographic choices are current and use library defaults (see [Authentication, Authorization and Cryptography Basics](./authentication-authorization-and-cryptography-basics.md)).
- Dependencies it suggests exist, are maintained, and are the ones you intend. Invented or typo-squatted package names are a real risk (see [Secrets and Supply Chain Security](./secrets-and-supply-chain-security.md)).
- Error handling does not echo internals to the client.

## Further Reading

- OWASP Top 10, current edition, on the OWASP website (owasp.org). Read the category descriptions and prevention notes, not only the titles.
- OWASP Cheat Sheet Series, covering injection prevention, XSS prevention, authorization, and many other topics.
- OWASP Application Security Verification Standard (ASVS), a detailed checklist of verifiable requirements.
- Dafydd Stuttard and Marcus Pinto, *The Web Application Hacker's Handbook*. Deep treatment of how web flaws arise; read it to understand defenses.
- Gary McGraw, *Software Security: Building Security In*. A view of security as an engineering practice.

Back to the section: [Security](./README.md)
