<div align="center">

# The Engineering Handbook

**The field guide for people who build systems that last.**

*Writing code is a skill. Engineering is a discipline. This handbook is about the discipline.*

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)
[![Sections](https://img.shields.io/badge/sections-18-informational.svg)](#curriculum)
[![Maintained](https://img.shields.io/badge/maintained-yes-success.svg)](#roadmap)

[Why this exists](#why-this-exists) ·
[Developer vs. Engineer](#developer-vs-engineer) ·
[Curriculum](#curriculum) ·
[Learning paths](#learning-paths) ·
[Contributing](#contributing)

</div>

---

## Why this exists

Tutorials teach you how to use a tool. Bootcamps teach you how to ship a feature. Very little teaches you how to **reason about a system**: why it was designed this way, where it will break, what it will cost, and what you give up with every decision.

That gap is what separates a person who writes code from an engineer who owns outcomes. It is also the gap that matters most now that generating code has become cheap.

This handbook is an attempt to close it. It is organized as a structured, opinionated, practitioner-oriented reference covering the full breadth of modern software engineering, from first principles to production operations to career growth.

## Who this is for

| If you are... | You will get... |
|---|---|
| **A new engineer** | A map of the territory, so you know what you don't know yet. |
| **A developer moving toward engineering** | The mental models, trade-off thinking, and vocabulary of the profession. |
| **A mid-level engineer** | Depth in the areas you haven't owned: distributed systems, security, reliability, performance. |
| **A senior engineer or lead** | A shared reference for mentoring, design reviews, and hiring conversations. |
| **A self-taught or career-switching builder** | The foundations that are easy to skip and expensive to lack. |

## Developer vs. Engineer

The titles are often used interchangeably. The disciplines are not the same.

A developer's central question is *"Does it work?"*
An engineer's central question is *"Will it keep working, at scale, under failure, as requirements change, and at an acceptable cost?"*

| Dimension | Developer mindset | Engineering mindset |
|---|---|---|
| **Goal** | Make the feature work | Make the system dependable over time |
| **Scope** | The function, the file, the ticket | The system, the team, the lifecycle |
| **Failure** | A bug to fix | A condition to design for in advance |
| **Decisions** | "What is the fastest way to build this?" | "What are the trade-offs, and what will this cost in a year?" |
| **Quality** | Passes when I run it | Verified by tests, observability, and review |
| **Complexity** | Handled when it appears | Actively managed and minimized |
| **Requirements** | Implemented as written | Questioned, clarified, and challenged |
| **Ownership** | Ends at merge | Continues through deploy, operate, and retire |
| **Documentation** | Optional | Part of the deliverable |
| **Security & cost** | Someone else's concern | Design constraints from day one |

Neither is lesser. Every engineer starts as a developer, and the transition is a matter of habit and perspective, not a title. **This handbook is built to accelerate that transition.**

## Engineering in the age of AI

AI coding assistants are now part of the toolchain, and this handbook treats them as exactly that: a tool.

What AI does well: generating boilerplate, translating between languages, explaining unfamiliar code, drafting tests, accelerating exploration.

What remains the engineer's responsibility, and cannot be delegated:

- **Problem framing.** Deciding what should be built, and what should not.
- **Architecture and trade-offs.** Choosing between options under real constraints: latency, cost, team size, risk, deadlines.
- **Verification.** Knowing whether generated code is correct, secure, and maintainable. You cannot review what you do not understand.
- **Accountability.** When the system fails at 3 a.m., the pager does not go to the model.
- **Judgment under ambiguity.** Most real engineering problems have incomplete requirements and conflicting priorities.

The more code is generated, the more valuable it becomes to be the person who can **read it critically, design around it, and take responsibility for it.** Strong fundamentals are what make AI a multiplier instead of a liability. Section [14: AI Engineering](./14-ai-engineering) covers this in depth.

## Curriculum

Eighteen sections, ordered from foundations to practice. Each is a self-contained module, and together they form a complete picture of the profession.

### Foundations

| # | Section | Focus |
|---|---|---|
| 01 | [Engineering Fundamentals](./01-engineering-fundamentals) | How computers work, the engineering mindset, problem solving, trade-off thinking |
| 02 | [Programming Concepts](./02-programming-concepts) | Paradigms, type systems, memory, concurrency, error handling, clean code |
| 03 | [Data Structures & Algorithms](./03-data-structures-algorithms) | Core structures, complexity analysis, algorithmic patterns, when each applies |

### Design & Architecture

| # | Section | Focus |
|---|---|---|
| 04 | [Software Architecture](./04-software-architecture) | Patterns, modularity, coupling and cohesion, domain-driven design, evolution |
| 05 | [System Design](./05-system-design) | Designing for scale, reliability, and cost; capacity planning; design reviews |
| 06 | [Database Engineering](./06-database-engineering) | Relational and NoSQL, modeling, indexing, transactions, replication, migrations |

### Building

| # | Section | Focus |
|---|---|---|
| 07 | [Backend Engineering](./07-backend-engineering) | APIs, services, authentication, queues, caching, background processing |
| 08 | [Frontend Engineering](./08-frontend-engineering) | Browser fundamentals, rendering, state, accessibility, web performance |

### Running in Production

| # | Section | Focus |
|---|---|---|
| 09 | [DevOps, Cloud & Infrastructure](./09-devops-cloud-infrastructure) | CI/CD, containers, orchestration, infrastructure as code, observability |
| 10 | [Security](./10-security) | Threat modeling, secure coding, authN/authZ, secrets, supply chain |
| 11 | [Testing & Quality](./11-testing-quality) | Test strategy, automation, quality gates, reliability practices |
| 12 | [Performance & Scalability](./12-performance-scalability) | Profiling, bottlenecks, load testing, caching, optimization discipline |
| 13 | [Distributed Systems](./13-distributed-systems) | Consistency, consensus, failure modes, messaging, partitioning |

### Modern Practice

| # | Section | Focus |
|---|---|---|
| 14 | [AI Engineering](./14-ai-engineering) | LLM applications, evaluation, AI-assisted development, responsible use |
| 15 | [Developer Tools](./15-developer-tools) | Git, editors, debuggers, shell, automation, productivity workflows |

### Career & Craft

| # | Section | Focus |
|---|---|---|
| 16 | [Open Source](./16-open-source) | Contributing, maintaining, licensing, working in public |
| 17 | [Engineering Career](./17-engineering-career) | Levels, communication, leadership, interviews, long-term growth |
| 18 | [Real-World Projects](./18-real-world-projects) | End-to-end builds that apply every section above |

## Learning paths

You do not have to read this front to back. Pick the path that matches your goal.

| Path | Sections | Outcome |
|---|---|---|
| **Foundations first** | 01 → 02 → 03 → 04 | Solid footing before specializing |
| **Backend engineer** | 02 → 06 → 07 → 05 → 10 → 12 | Build and scale reliable services |
| **Frontend engineer** | 02 → 08 → 11 → 12 → 10 | Fast, accessible, maintainable interfaces |
| **Full-stack engineer** | 01 → 06 → 07 → 08 → 09 → 10 | End-to-end ownership |
| **Platform / DevOps** | 09 → 10 → 13 → 12 → 05 | Operate systems at scale |
| **Interview preparation** | 03 → 05 → 04 → 17 | Algorithms, design, and communication |
| **Staff-track growth** | 05 → 13 → 04 → 17 → 18 | System-level judgment and influence |

## How each section is structured

Every section follows the same shape so the handbook stays predictable and scannable:

1. **Overview**: what it is and why it matters in practice.
2. **Core concepts**: the ideas you must understand, explained precisely.
3. **Trade-offs**: where the real decisions are, and what each choice costs.
4. **Failure modes**: how things break in production and how to anticipate it.
5. **In practice**: examples, checklists, and decision frameworks.
6. **Further reading**: primary sources, books, and papers worth your time.

## Principles behind this handbook

- **Depth over breadth of buzzwords.** Concepts are explained, not name-dropped.
- **Trade-offs over absolutes.** There are no best practices without context.
- **Tool-agnostic where possible.** Tools change; principles endure.
- **Production reality.** Failure, cost, and operability are first-class topics.
- **Primary sources.** Claims link to papers, specifications, and authoritative references.
- **Honest scope.** Where something is contested or evolving, we say so.

## Repository structure

```text
the-engineering-handbook/
├── 01-engineering-fundamentals/
├── 02-programming-concepts/
├── 03-data-structures-algorithms/
├── 04-software-architecture/
├── 05-system-design/
├── 06-database-engineering/
├── 07-backend-engineering/
├── 08-frontend-engineering/
├── 09-devops-cloud-infrastructure/
├── 10-security/
├── 11-testing-quality/
├── 12-performance-scalability/
├── 13-distributed-systems/
├── 14-ai-engineering/
├── 15-developer-tools/
├── 16-open-source/
├── 17-engineering-career/
├── 18-real-world-projects/
├── CONTRIBUTING.md
├── LICENSE
└── README.md
```

## Roadmap

- [x] Define the 18-section structure
- [ ] Complete foundations (01–03)
- [ ] Complete design and architecture (04–06)
- [ ] Complete building and production sections (07–13)
- [ ] Complete modern practice and career sections (14–17)
- [ ] Ship end-to-end reference projects (18)
- [ ] Add diagrams, checklists, and review templates throughout

Progress is tracked through [issues](../../issues) and [pull requests](../../pulls).

## Contributing

Contributions that raise the quality bar are welcome: corrections, clearer explanations, better examples, missing failure modes, and new references. Please read [CONTRIBUTING.md](CONTRIBUTING.md) before opening a pull request. It describes the quality standard, the chapter template, and the review process.

## License

Released under the [MIT License](LICENSE). You are free to use, share, and adapt this material with attribution.

---

<div align="center">

**If this handbook helps you, consider giving it a star and sharing it with another engineer.**

*Build things that work. Engineer things that last.*

</div>
