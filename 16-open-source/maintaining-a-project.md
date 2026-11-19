# Maintaining a Project

> Maintaining a project means owning its long-term cost: every feature accepted, every issue opened, and every release shipped becomes work someone must carry.

## Overview

Creating a project is easy. Maintaining one is a different job. The code is the smallest part of it. Maintainers triage, review, release, answer questions, manage expectations, and decide what the project will not do.

The central fact of maintainership is asymmetry. Users file issues at no cost. Each issue costs the maintainer attention. A project that is popular and under-resourced can feel like a service with unpaid customers. Understanding that asymmetry explains most maintainer behavior, including the "no" that surprises newcomers.

This is the engineer-versus-coder distinction in its purest form. A coder ships a library. An engineer owns what happens to that library after release: its bugs, its users, its security posture, and its eventual retirement. The skills are transferable to any production system you will ever own.

## Core Concepts

### Scope and a clear promise

Every project should state what it is, who it is for, and what it deliberately does not do. A short "non-goals" list in the README prevents a large share of unwanted requests. A project with a narrow, well-defended scope is easier to maintain than one that tries to satisfy everyone.

### Essential project files

| File | Purpose |
|---|---|
| `README` | What the project is, how to install and use it, where to get help |
| `LICENSE` | The terms under which others may use it |
| `CONTRIBUTING` | How to propose changes, run tests, and what review looks like |
| `CODE_OF_CONDUCT` | Expected behavior and how it is enforced |
| `SECURITY` | How to report vulnerabilities privately |
| `CHANGELOG` | What changed in each release, written for users |
| Issue and PR templates | Prompts that elicit the information you need |

### Issue triage

Triage is the practice of giving each incoming item a quick, honest disposition. A workable routine:

1. Is it a duplicate? Link and close.
2. Is it reproducible? If not, ask for a minimal reproduction and set a time limit.
3. Is it in scope? If not, say so kindly and explain why.
4. Is it a bug, feature request, or question? Label it.
5. Does it need a decision? Assign it, or mark it as awaiting maintainer input.

Labels should be few and meaningful. Examples: `bug`, `enhancement`, `question`, `needs-reproduction`, `good first issue`, `help wanted`, `wontfix`.

### Review and merge standards

Write down what "mergeable" means:

- Tests pass in continuous integration.
- New behavior has tests.
- Public behavior changes are documented.
- The change fits the project's design and scope.
- At least one maintainer other than the author approves, where the team size allows it.

Automate everything that can be automated: formatting, linting, test runs, dependency updates. Reserve human attention for design and correctness. See [Developer Tools](../15-developer-tools/README.md) and [Testing and Quality](../11-testing-quality/README.md).

### Releases and versioning

A release is a promise. Many projects follow Semantic Versioning, in which a major version signals breaking changes, a minor version adds backward-compatible features, and a patch version fixes bugs. Semantic versioning only works if you define what counts as your public API and honor it.

A repeatable release process typically includes:

- A release branch or tag from a green main branch.
- A changelog entry describing user-visible changes.
- Automated build and publish steps, so releases do not depend on one person's laptop.
- Signed or provenance-attested artifacts where your ecosystem supports it.
- A deprecation policy: how long old behavior stays, and how users are warned.

### Saying no

Declining is a core maintainer skill. A good refusal is brief, specific, and respectful. It names the reason (scope, maintenance cost, conflict with design) and, where possible, points to an alternative such as a plugin, a fork, or a different tool. Silence is worse than no. It leaves the contributor guessing and leaves the issue open indefinitely.

### Sharing the load

A project with one maintainer has a bus factor of one. Reduce it deliberately:

- Document release and administrative procedures.
- Grant triage rights early. Triagers help without touching code.
- Promote consistent contributors to maintainers, with clear criteria.
- Keep accounts, signing keys, and package registry access recoverable by more than one person.

## Trade-offs

| Choice | What it gives you | What it costs |
|---|---|---|
| Narrow scope | Low maintenance, coherent design | Turns away users and contributors with different needs |
| Broad scope | Larger audience | Larger surface area to test, secure, and document |
| Strict review standards | Higher quality, fewer regressions | Slower merges; contributors may leave |
| Lenient review standards | Welcoming, fast | Debt accumulates; maintainers fix problems later |
| Frequent releases | Fast feedback, small diffs | More release overhead; upgrade fatigue for users |
| Infrequent releases | Stability, fewer upgrades | Large, risky releases; fixes wait for users |
| Strict backward compatibility | User trust | Old design mistakes are kept for years |
| Breaking changes with clear migration | Cleaner design | User friction; some users stay on old versions |
| Adding maintainers | Resilience and capacity | Coordination cost; risk of inconsistent decisions; trust must be extended |
| Accepting funding or sponsorship | Time for the work | Expectations from sponsors; possible pressure on direction |

## Failure Modes

**Maintainer burnout.** Demand grows, volunteers do not, and one person absorbs the abuse and the backlog. The result is a sudden announcement that the project is archived, or silent abandonment. Prevention: set boundaries, publish response-time expectations, delegate early, and treat rest as part of the maintenance plan. See [Sustainable Career and Avoiding Burnout](../17-engineering-career/sustainable-career-and-avoiding-burnout.md).

**The permanent backlog.** Thousands of unlabeled issues make the tracker useless. Contributors cannot find real work and maintainers cannot see real problems. Prevention: triage regularly, close stale issues with an explanation, and accept that an issue tracker is not a promise.

**Feature creep.** Each accepted request is reasonable alone. Together they produce a project nobody can reason about. Prevention: judge every feature against a written scope and the long-term cost of owning it.

**Accidental breaking changes.** A minor release changes behavior that downstream users depended on. Prevention: define the public API, test it, and review diffs for compatibility. Hyrum's Law, the observation that any observable behavior will be depended on by someone, is a useful warning.

**Single point of failure.** One person holds the publishing credentials and becomes unavailable. Prevention: shared access with strong authentication and documented recovery.

**Toxic interactions.** Hostile users or contributors drive away helpful people. Prevention: enforce the code of conduct consistently, early, and without exceptions for popular contributors.

**Abandonment without notice.** The maintainer moves on and leaves users depending on an unmaintained package. Prevention: state the project status in the README, hand over or archive explicitly, and point users to successors. Archiving a project honestly is a legitimate and responsible outcome.

**Supply chain exposure.** Unmaintained dependencies, unreviewed contributions, and compromised maintainer accounts become attack paths. See [Open Source Security](./open-source-security.md).

## In Practice

### A minimal maintainer routine

- Daily or every few days: skim new issues and pull requests; apply labels; respond or defer with a note.
- Weekly: review open pull requests; close stale items; review dependency update proposals.
- Per release: run the release checklist; write the changelog; announce.
- Quarterly: review scope, backlog health, and whether you still want to do this.

### Templates that save time

An issue template for bugs should ask for: version, environment, steps to reproduce, expected result, actual result, and logs. A pull request template should ask: what changed, why, how it was tested, and whether docs were updated. Templates do not remove effort, but they make the missing information obvious.

### Release checklist

- [ ] Main branch is green in continuous integration.
- [ ] Changelog lists user-visible changes and any breaking changes.
- [ ] Version number follows the project's versioning policy.
- [ ] Security fixes have been coordinated with the reporter and disclosed per policy.
- [ ] Artifacts are built by automation, not by hand.
- [ ] Release is tagged and announced.
- [ ] Documentation matches the released version.

### Treating a project like a production service

Apply the same habits you would to any system you operate: tests before changes, monitored dependencies, a documented process for incidents, and a postmortem when something goes wrong. See [Engineering Fundamentals](../01-engineering-fundamentals/README.md).

## Further Reading

- Karl Fogel, *Producing Open Source Software*. Covers running a project, from setup through handling difficult people.
- Nadia Eghbal, *Roads and Bridges: The Unseen Labor Behind Our Digital Infrastructure*. A report on the under-resourced state of shared open source infrastructure.
- Semantic Versioning specification, semver.org.
- Keep a Changelog, keepachangelog.com. A convention for writing changelogs for humans.
- Open Source Guides (opensource.guide), especially the guides on being a good maintainer and finding users.
