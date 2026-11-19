# Open Source Security

> Almost every modern application is mostly open source code written by strangers, so securing your software means managing trust in the supply chain, not only in your own code.

## Overview

A typical application directly depends on a handful of packages and indirectly on hundreds more. Each dependency is code you run with your application's privileges, maintained by people you may never have met. Their mistakes become your vulnerabilities. Their compromised accounts become your incidents.

Open source security has two sides. As a consumer, you decide what to trust and how to detect when that trust is misplaced. As a maintainer, you are a link in other people's supply chains, and your practices affect everyone downstream.

Public incidents have made the risk concrete. The Log4Shell vulnerability in a widely used Java logging library (2021) showed how a flaw in one component can reach a vast number of systems, many of which did not know they contained it. The compromise of the xz Utils project (disclosed in 2024) showed that an attacker can patiently gain a maintainer's trust and insert a backdoor. Incidents in package registries, such as malicious packages published under names resembling popular ones, show that attackers also exploit the ecosystem's mechanics. The details of each are public record. The lesson is general: trust is a security boundary.

Generated code adds a new vector. An AI assistant may suggest a package that does not exist, and an attacker can register that name. The assistant may suggest an outdated or vulnerable usage pattern. Verifying what you install and why is a fundamental skill. See [Security](../10-security/README.md) for the broader discipline.

## Core Concepts

### The threat model for dependencies

| Threat | Description |
|---|---|
| Known vulnerability | A published flaw in a version you use |
| Malicious package | A package built to harm, including typosquats and dependency confusion |
| Compromised maintainer or build | A legitimate package altered by an attacker |
| Abandoned package | No one is fixing flaws, and no one will respond to reports |
| Malicious or careless contribution | A change merged into a project that introduces a flaw |
| Install-time execution | Scripts that run during installation with your privileges |
| Secrets exposure | Credentials accidentally committed to repositories or logs |

### Know what you run

- **Dependency inventory.** Use your package manager's lockfile for reproducible builds. Commit it.
- **Software bill of materials.** An SBOM, in a standard format such as SPDX or CycloneDX, lists the components in a build. It lets you answer "are we affected?" quickly when a new vulnerability is announced.
- **Vulnerability identifiers.** Vulnerabilities are tracked with CVE identifiers and often with ecosystem advisories. Open-source vulnerability databases, such as OSV, aggregate them.
- **Severity scoring.** CVSS scores provide a common scale, but a score does not tell you whether the vulnerable code path is reachable in your application.

### Consumer practices

1. **Pin and lock.** Commit lockfiles and use exact versions in deployments.
2. **Scan continuously.** Run dependency scanning in continuous integration and on a schedule, because new vulnerabilities are disclosed against old code.
3. **Update on a cadence.** Small, frequent updates are easier than rare large ones. Automated update tools can open pull requests for review.
4. **Evaluate before adopting.** Check maintenance activity, release history, number of maintainers, issue response, security policy, and whether the functionality is worth a dependency. A small utility is sometimes better written yourself.
5. **Reduce the surface.** Remove unused dependencies. Fewer packages mean fewer risks.
6. **Verify names.** Check spelling and the publisher before installing. Confirm a package an assistant suggested actually exists and is the one you intend.
7. **Limit install-time and runtime privileges.** Build in isolated environments. Avoid giving build jobs access to production secrets.
8. **Plan for emergencies.** Know how to identify affected systems, patch, and redeploy quickly.

### Maintainer practices

- **Publish a security policy.** A `SECURITY` file says how to report a vulnerability privately and what response to expect. Use private vulnerability reporting features of your hosting platform where available.
- **Protect accounts.** Enable multi-factor authentication on source hosting and package registries. Prefer hardware-backed or phishing-resistant methods. Many registries now require or strongly encourage this for maintainers of popular packages.
- **Protect the build.** Build releases in automated, reproducible pipelines. Limit who can trigger releases. Consider trusted publishing mechanisms offered by registries, which avoid long-lived tokens. Generate provenance attestations where supported.
- **Protect the branch.** Require review and passing checks before merge. Restrict force-pushes.
- **Handle disclosure responsibly.** Acknowledge reports, reproduce, fix in private, coordinate a release and advisory, request a CVE when appropriate, and credit the reporter.
- **Review contributions carefully.** Be especially attentive to changes in build scripts, binary files, test fixtures that contain opaque data, and new maintainers who push for access quickly.
- **Keep secrets out.** Use secret scanning, and rotate any credential that was exposed even briefly.

### Frameworks and tools worth knowing

These are real, widely referenced efforts. Verify the current state of each before relying on it.

- **OpenSSF Scorecard** evaluates a project's security practices with automated checks.
- **SLSA** (Supply-chain Levels for Software Artifacts) is a framework describing increasing assurance in build integrity.
- **Sigstore** provides tooling for signing and verifying software artifacts.
- **Dependabot and similar tools** automate dependency update proposals.
- **Software composition analysis tools** scan for known vulnerabilities and license issues.

### Responding to a disclosed vulnerability

1. Determine whether you use the affected component and version, including transitively.
2. Determine whether the vulnerable code path is reachable.
3. Apply the fix or a documented mitigation.
4. Redeploy and verify.
5. Check logs for signs of past exploitation if the severity warrants.
6. Record what happened and improve the process.

## Trade-offs

| Choice | What it gives you | What it costs |
|---|---|---|
| Pin exact versions | Reproducibility; no surprise updates | Fixes do not arrive unless you update; drift creates large upgrades |
| Float to latest | Fast access to fixes | New malicious or broken releases reach you automatically |
| Many small dependencies | Reuse, speed | Wide attack surface; more maintainers to trust |
| Fewer dependencies, more in-house code | Smaller surface; control | Your team owns bugs and security of that code |
| Vendoring dependencies | Isolation from registry problems | Manual updates; vendored code can go stale |
| Automatic update bots | Timely patches | Pull request noise; risk of merging without review |
| Strict scanning gates | Blocks known bad versions | False positives; slowed delivery; alert fatigue |
| Strict maintainer controls (reviews, MFA, signed releases) | Resilience to compromise | Friction for contributors and maintainers; fewer volunteers may stay |
| Public vulnerability discussion | Transparency | Attackers see the details before users patch |
| Private disclosure | Time to fix before exposure | Delays and coordination costs; reporters may feel ignored |

## Failure Modes

**Alert fatigue.** Scanners report hundreds of findings, most irrelevant. Teams stop looking. Prevention: prioritize by reachability and severity, suppress with documented reasons, and tune the tool.

**Stale lockfiles.** Dependencies are pinned and never updated. A vulnerability is announced and the upgrade path spans years of breaking changes. Prevention: scheduled, small updates.

**Blind trust in popularity.** A package with many downloads is assumed safe. Popularity measures reach, not review quality. Prevention: evaluate maintenance and practices directly.

**Typosquatting and name confusion.** A mistyped name installs a malicious package. Prevention: copy names from official documentation, review dependency changes in pull requests, and use registry features that restrict namespaces.

**Dependency confusion.** A build resolves an internal package name to a public package with the same name. Prevention: configure registries and scopes so internal names resolve only internally.

**Social engineering of maintainers.** An unfamiliar contributor builds trust over time, pressures a tired maintainer, and gains commit rights. Prevention: share maintainership among people you know, require review of all changes including from maintainers, and be wary of pressure to bypass process.

**Leaked secrets.** A token is committed, scraped, and used within minutes. Prevention: secret scanning, short-lived credentials, immediate rotation.

**Over-trusting generated suggestions.** An assistant proposes a package name or code pattern. The package does not exist, or it was registered by an attacker. Prevention: verify packages in the registry, check publisher history, and review generated code for insecure patterns.

**Unreachable maintainers.** A vulnerability is reported but no one answers, so the reporter publishes it. Prevention: publish a security contact and monitor it.

**Treating the SBOM as the end.** A list of components that nobody queries provides no protection. Prevention: use it in incident response and automate matching against vulnerability data.

## In Practice

### Dependency review checklist

- [ ] Is the functionality worth a dependency?
- [ ] Is it actively maintained, with recent releases and responsive issues?
- [ ] Does it publish a security policy and advisories?
- [ ] How many maintainers does it have, and who are they?
- [ ] What is its license? See [Licensing Explained](./licensing-explained.md).
- [ ] What does it pull in transitively?
- [ ] Does it run code at install time?
- [ ] Is the name exactly what we intend?

### Maintainer hardening checklist

- [ ] Multi-factor authentication on all accounts with publish rights.
- [ ] Branch protection with required review and checks.
- [ ] Releases built by automation with minimal permissions.
- [ ] A `SECURITY` file with a private reporting path.
- [ ] Secret scanning enabled.
- [ ] At least two people can administer the project and registry entries.
- [ ] Dependencies monitored and updated on a schedule.

### A team routine

Weekly, review automated update pull requests and open advisories. On every new critical disclosure in a component you use, run the affected-systems query from your SBOM and record the outcome. Quarterly, review the dependency list and remove what is unused.

### Related material

- Broader security practice: [Security](../10-security/README.md)
- Automated checks in pipelines: [Testing and Quality](../11-testing-quality/README.md) and [Developer Tools](../15-developer-tools/README.md)
- Sustaining maintainers: [Maintaining a Project](./maintaining-a-project.md)
- Generated code risks: [AI Engineering](../14-ai-engineering/README.md)

## Further Reading

- OpenSSF (Open Source Security Foundation) documentation, including Scorecard and the Concise Guide for Evaluating Open Source Software, openssf.org.
- SLSA specification, slsa.dev.
- Open Source Vulnerabilities (OSV) database and schema, osv.dev.
- Sonatype and other industry annual supply chain reports. Read them critically, as figures vary by vendor and method.
- Ross Anderson, *Security Engineering*. A broad reference on building systems that resist attack.
- Public post-incident write-ups of Log4Shell and the xz Utils backdoor, available from the projects and from security researchers.
