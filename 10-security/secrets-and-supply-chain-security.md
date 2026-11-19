# Secrets and Supply Chain Security

> Modern software is mostly code you did not write and credentials you must not leak, so protecting secrets and the path from source to production is as important as protecting your own logic.

## Overview

Two problems dominate real-world breaches that do not involve novel exploits. First, secrets leak: an API key in a repository, a database password in a log, a token in a screenshot. Second, trusted components turn hostile: a compromised dependency, a poisoned build step, a tampered artifact.

Both come from the same fact. Your system's security depends on things outside your direct control. A typical service imports hundreds of transitive packages, runs in a container built from a base image someone else maintains, is built by a pipeline with deployment credentials, and talks to cloud APIs using long-lived keys. Each of those is a link in a chain, and attackers choose the weakest one.

This topic covers how to keep credentials out of places they should not be, how to know what you depend on, and how to make the path from commit to production verifiable. See also [Authentication, Authorization and Cryptography Basics](./authentication-authorization-and-cryptography-basics.md) for key handling fundamentals and [Secure Development Lifecycle](./secure-development-lifecycle.md) for where these controls sit in a process.

## Core Concepts

### What counts as a secret

A secret is any value whose disclosure grants access or breaks a guarantee: passwords, API keys, private keys, signing keys, session tokens, database connection strings, webhook signing secrets, and encryption keys. Treat anything that authenticates a machine to another machine as a secret.

### Secret lifecycle

```mermaid
flowchart LR
    A[Create] --> B[Store]
    B --> C[Distribute to workload]
    C --> D[Use]
    D --> E[Rotate]
    E --> B
    D --> F[Revoke on compromise]
```

Each stage has its own risk. Creation needs strong randomness. Storage needs encryption and access control. Distribution should be short-lived and authenticated. Use should avoid logs and error messages. Rotation should be routine, not heroic. Revocation must be fast and rehearsed.

### Where secrets belong

| Location | Verdict | Reason |
|----------|---------|--------|
| Source code or committed config | Never | Repository history is permanent and widely copied |
| Container image layers | Never | Layers persist and are readable by anyone with the image |
| Plain environment variables | Acceptable with care | Easy to inject, but visible to child processes, crash dumps, and some debug tooling |
| Secrets manager, fetched at runtime | Preferred | Central access control, audit trail, rotation |
| Workload identity (short-lived credentials issued to a running service) | Best where available | No long-lived secret to steal |
| Developer laptop `.env` files | Acceptable for local development only, git-ignored | Must contain development-only credentials |

The strongest pattern is to eliminate stored secrets. If a service can prove its identity through the platform (for example a cloud role or federated workload identity) and receive short-lived credentials, there is nothing durable to leak.

### Vulnerable versus fixed: configuration

```python
# VULNERABLE: a credential committed to the repository
DB_URL = "postgres://app:hunter2@db.internal:5432/orders"
STRIPE_KEY = "sk_live_..."   # real key pasted into source

# FIXED: read from the runtime environment, fail loudly if absent
import os

def require_env(name: str) -> str:
    value = os.environ.get(name)
    if not value:
        raise RuntimeError(f"Missing required configuration: {name}")
    return value

DB_URL = require_env("DATABASE_URL")
```

Environment injection is a transport mechanism, not storage. The value should come from a secrets manager or platform secret store at deploy time.

### Preventing leaks

Layered controls work better than any single one:

1. **Pre-commit scanning** on developer machines to catch secrets before they are committed.
2. **Server-side scanning** in the repository host and CI, because local hooks can be skipped.
3. **Log redaction** for known sensitive fields and token patterns.
4. **Short-lived credentials** so leaks expire quickly.
5. **Scoped credentials** so a leaked key can do little.
6. **Separate secrets per environment** so a development leak does not expose production.

### If a secret leaks

Assume it is compromised the moment it is public, even briefly. Automated scrapers find exposed keys quickly.

1. Revoke or rotate the secret first.
2. Investigate usage logs for the exposure window.
3. Remove it from code, but understand that deleting it from the latest commit does not remove it from history or from forks and clones.
4. Find the root cause and add a control so the same path cannot recur.

Rewriting history is secondary. The rotated secret is the fix.

### Supply chain basics

The software supply chain is everything between a developer's intent and running code: source, dependencies, build systems, artifact storage, and deployment.

**Know what you ship.** Maintain a **software bill of materials** (SBOM): a machine-readable inventory of components in each build. Formats such as SPDX and CycloneDX exist. An SBOM does not make software safe, but you cannot respond to a newly disclosed vulnerability without knowing whether you are affected.

**Control what you pull.**

- Pin dependency versions and commit lockfiles so builds are reproducible.
- Verify integrity hashes recorded in lockfiles.
- Use an internal proxy or registry for approved packages where scale justifies it.
- Pin container base images by digest, not a floating tag.
- Prefer well-maintained packages with active review, and be wary of tiny packages that add little.

**Resist dependency confusion and typosquatting.** Attackers publish packages with names close to popular ones, or with the name of an internal package on a public registry. Configure registries so internal names resolve only to internal sources. Review new dependency names carefully.

**Harden the build.**

- Run builds in ephemeral, isolated environments.
- Give pipelines the minimum permissions needed, and prefer short-lived federated credentials over stored deploy keys.
- Require review for changes to pipeline definitions, since those change what runs with powerful credentials.
- Separate build from deploy, and sign artifacts.

**Verify what you deploy.** Sign build artifacts and verify signatures before deployment. Frameworks such as SLSA (Supply-chain Levels for Software Artifacts) describe progressively stronger guarantees about provenance: who built an artifact, from what source, using what process.

### Dependency vulnerability management

Scanners compare your inventory against public vulnerability databases. They produce a lot of output. Triage by:

- Is the vulnerable code actually reachable in your application?
- Is it exposed to untrusted input?
- Is there a fixed version, and what does upgrading break?
- Is there a mitigation (configuration, input filtering, network control) while you upgrade?

Reachability matters, but do not use it as a permanent excuse. Code paths change.

## Trade-offs

| Choice | Benefit | Cost |
|--------|---------|------|
| Central secrets manager | Audit, rotation, access control | Availability dependency, operational setup, bootstrap problem (how does the workload authenticate to it?) |
| Short-lived credentials | Leaks expire fast | More moving parts; clock and refresh failures cause outages |
| Pinning dependencies | Reproducible, reviewed builds | Stale dependencies unless updates are automated |
| Automatic dependency updates | Fast patching | Noise, test load, risk of accepting a malicious release without review |
| Vendoring dependencies | Control and offline builds | Maintenance burden, repository size, update lag |
| Private package mirror | Control, availability, scanning | Infrastructure to run and keep current |
| Signing and provenance | Tamper evidence | Key management, pipeline complexity, workflow friction |
| Few dependencies | Smaller attack surface | More code to write and maintain yourself, with its own bugs |

There is no zero-dependency answer for most products. The honest position is to treat each dependency as a liability with a benefit and to be able to say why it is worth it.

Automation versus review is a contested trade-off. Automatically merging dependency updates reduces exposure to known vulnerabilities but increases exposure to a malicious new release. Many teams auto-update patches with strong test gates and review larger changes.

## Failure Modes

**Secret in version control history.** Found by scanners or by attackers months later. Detect with historical scans. Prevent with pre-commit and server-side checks, and fix by rotation.

**Secret in logs or error reports.** A request dump or debug log includes an Authorization header. Redact by default, and review what crash reporters capture.

**Over-privileged credentials.** A CI token with admin rights to the whole cloud account. Scope each credential to one purpose and review permissions periodically.

**Long-lived, shared keys.** The same key used by ten services for three years. Nobody can rotate it without risking an outage, so nobody does. Design for rotation from day one: support two valid keys during a rollover.

**Rotation untested.** The first rotation happens during an incident and breaks production. Rehearse routine rotation.

**Compromised dependency.** A maintainer account is taken over, or a new release contains malicious code that runs at install or import time. Mitigate with pinning, review of dependency diffs for critical packages, restricted install-time scripts where tooling allows, and egress controls that limit what compromised code can reach.

**Dependency confusion.** A build fetches a public package where an internal one was intended. Configure scoped registries and reserve internal names.

**Poisoned CI.** An untrusted contribution triggers a pipeline that has access to secrets. Do not expose secrets to workflows triggered by untrusted code, and require approval for those runs.

**Mutable tags.** A base image tag moves to different content between builds. Pin by digest.

**Vulnerability fatigue.** Hundreds of scanner findings, none triaged, so critical ones hide. Set a policy for severity, reachability, and time to fix, and measure it.

## In Practice

### Baseline checklist

- [ ] No secrets in source, images, or build logs; scanning enforced server-side.
- [ ] A secrets manager or workload identity is the source of production credentials.
- [ ] Credentials are scoped to one purpose and one environment.
- [ ] Every secret has an owner and a rotation procedure that has been exercised.
- [ ] Lockfiles are committed; builds use them.
- [ ] Base images are pinned and updated on a schedule.
- [ ] An SBOM is produced per build and stored with the artifact.
- [ ] Dependency alerts are triaged against a written policy.
- [ ] Pipeline definitions require review; pipelines run with least privilege.
- [ ] Artifacts are signed and verified before deploy where your platform supports it.

### Worked example: adding a dependency

Before adding a package, answer in the pull request:

1. What problem does it solve, and what is the cost of writing it ourselves?
2. Who maintains it, how active is it, and how many maintainers does it have?
3. What does it pull in transitively?
4. Does it run code at install time?
5. What license does it use, and is that compatible with ours?
6. How would we replace it if it were abandoned or compromised?

Five minutes of review is cheap against years of exposure.

### Developer vs. engineer

A developer adds a package that makes the feature work. An engineer treats the package as an ongoing obligation: pinned, tracked in the SBOM, owned by someone, and replaceable.

### When AI generates code

- Check that every package it imports exists, is the real well-known one, and is spelled correctly. Generated code can reference packages that do not exist, which an attacker can register.
- Remove any placeholder or example credentials, and confirm that nothing real was pasted into prompts or context. Treat text sent to external tools as a disclosure channel.
- Review generated pipeline and infrastructure files for overly broad permissions and unpinned actions or images.
- Do not install suggested packages by running generated shell commands without reading them.

## Further Reading

- Michael Howard and David LeBlanc, *Writing Secure Code*. A classic that still frames trust and input thinking.
- NIST, *Secure Software Development Framework (SSDF)*, SP 800-218, published on the NIST website.
- SLSA specification (slsa.dev), describing levels of build integrity and provenance.
- OWASP Cheat Sheet Series: Secrets Management and Software Supply Chain Security (owasp.org).
- Evan Gilman and Doug Barth, *Zero Trust Networks*. Useful for workload identity and service authentication.

Back to the section: [Security](./README.md)
