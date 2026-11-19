# Security

## Overview

Security is a property of how a system is designed, built, and operated. It is not a feature you add. Every engineer makes security decisions daily, often without noticing: how input is handled, who is allowed to see a record, where a credential is stored, which package is added. Those decisions accumulate into the security posture of the product.

AI-assisted coding raises the stakes. Generated code is fast and plausible, and it often reproduces common insecure idioms. An engineer with strong fundamentals can spot the missing authorization check or the interpolated query. One without them cannot tell. This section builds the judgment to review, design, and verify, not just to write code that runs.

## What you will learn

- How to reason about what can go wrong before building, using threat models.
- The recurring families of web application risk and the structural defenses that remove them.
- The difference between authentication, authorization, and cryptography, and how to use each safely.
- How to keep secrets out of the wrong places and how to treat dependencies and builds as part of your attack surface.
- How to embed security into the delivery process so it scales beyond a single reviewer.

## Topics

Suggested reading order:

| Topic | What it covers |
|-------|----------------|
| [Threat Modeling](./threat-modeling.md) | Assets, trust boundaries, data flow diagrams, STRIDE, attack trees, and turning threats into decisions |
| [OWASP Top 10 and Secure Coding](./owasp-top-10-and-secure-coding.md) | Common risk families, input handling, and vulnerable-versus-fixed patterns for injection, XSS, and access control |
| [Authentication, Authorization and Cryptography Basics](./authentication-authorization-and-cryptography-basics.md) | Identity, sessions, tokens, permission models, and safe use of cryptographic primitives |
| [Secrets and Supply Chain Security](./secrets-and-supply-chain-security.md) | Secret lifecycle, leak prevention, dependency risk, SBOMs, and build integrity |
| [Secure Development Lifecycle](./secure-development-lifecycle.md) | Security activities across delivery stages, automated testing types, and response practices |

## Prerequisites

- Comfort reading and writing code in at least one language, and basic web concepts (HTTP, cookies, request and response).
- Familiarity with how a service talks to a database and to other services. See [Backend Engineering](../07-backend-engineering/README.md).
- Basic understanding of version control and CI pipelines.

## What good looks like

- [ ] Before building a feature that crosses a trust boundary, you sketch the data flow and list what could go wrong.
- [ ] You never build queries by concatenating untrusted input, and you can explain why parameters work.
- [ ] Every endpoint you write enforces authorization on the server, including per-object checks, and has tests for it.
- [ ] You use vetted libraries and managed services for authentication and cryptography instead of writing your own.
- [ ] You can state where each secret in your system lives, who can read it, and how it is rotated.
- [ ] You can list your direct dependencies, say why each is there, and respond when one is reported vulnerable.
- [ ] You review AI-generated code for insecure defaults, invented packages, and missing authorization before accepting it.
- [ ] After an incident or finding, you fix the class of flaw with a test, a lint rule, or a shared helper, not only the single instance.
- [ ] You record accepted risks with an owner and a review date.

Back to [The Engineering Handbook](../README.md)
