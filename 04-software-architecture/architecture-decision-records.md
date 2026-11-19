# Architecture Decision Records

> An architecture decision record (ADR) is a short, dated document that captures one significant decision, its context, and its consequences, so future engineers understand why the system is the way it is.

## Overview

Code shows what a system does. It rarely shows why. Six months after a decision, the reasons are scattered across chat threads and the memory of people who may have left. A new engineer sees a strange design and either keeps it out of fear or removes it and re-learns the lesson in production.

An ADR records the reasoning at the time of the decision. Michael Nygard popularized the lightweight format. The aim is not paperwork. It is to make decisions explicit, reviewable, and recoverable.

Writing an ADR also improves the decision. Forcing yourself to list the alternatives and the costs exposes weak reasoning before it becomes code. For a team, ADRs turn architecture from private opinion into shared, inspectable engineering.

## Core Concepts

### What deserves an ADR

An architecturally significant decision is one that is costly to reverse, affects several teams or components, or shapes quality attributes such as performance, security, or operability. Examples:

- Choosing a database or messaging technology.
- Splitting a service or merging two.
- Adopting an authentication scheme.
- Defining a public API versioning policy.
- Deciding on a deployment model.

A decision to rename a variable does not need one. A useful test: would a new team member be confused or tempted to undo this in a year?

### Anatomy of an ADR

A compact structure:

```markdown
# ADR-0012: Use a transactional outbox for order events

- Status: Accepted
- Date: 2026-03-14
- Deciders: Orders team, Platform team

## Context
Order changes must be published to the broker. Writing to the database and
the broker separately has lost events during crashes. We need at-least-once
publication without distributed transactions.

## Decision
We will write events to an outbox table in the same transaction as the
order change and publish them with a relay process.

## Alternatives considered
- Dual write with retry: simple, but loses events on crash between steps.
- Change data capture on the orders table: avoids the relay, but couples
  consumers to our table schema.

## Consequences
- Events may be published more than once; consumers must be idempotent.
- We operate one more process (the relay) and must monitor its lag.
- Outbox rows need a cleanup job.
```

### Status and lifecycle

Typical statuses: Proposed, Accepted, Deprecated, Superseded by ADR-NNNN. An ADR is not edited to change history. If the decision changes, write a new ADR that supersedes the old one and link them. The record then shows how thinking evolved.

### Where they live

Keep ADRs in the repository next to the code, for example `docs/adr/0012-use-outbox.md`. Benefits: they are versioned, reviewed through pull requests, and found by the people changing the code. A wiki page disconnected from the repo tends to rot.

Number them sequentially and never reuse numbers.

### Writing good context and consequences

Context states the forces: constraints, requirements, deadlines, what you know and do not know. Write it so a reader without your background can follow it.

Consequences must include the negatives. An ADR that lists only benefits is advocacy. Honest consequences state what becomes harder, what new risks appear, and what must be monitored.

### Lightweight decision process

ADRs fit well with a pull-request flow:

1. Author drafts the ADR with status Proposed and opens a pull request.
2. Affected people comment. Disagreements are resolved in the thread.
3. A named decider accepts or rejects within an agreed time.
4. The status changes to Accepted and the ADR merges, usually before or with the implementation.

Many teams set a rule: a decision is made by whoever is accountable for it, after consulting those affected. Consensus is not required, but objections must be recorded.

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Writing ADRs | Preserved reasoning, better decisions, faster onboarding | Time to write and review |
| Short ADRs | Actually written and read | May omit nuance |
| Long ADRs | Thorough | Often skipped or never finished |
| ADRs in the repo | Versioned, near the code | Harder for non-engineers to find |
| Requiring ADRs for everything | Consistent practice | Bureaucracy; people stop caring |
| Requiring none | No overhead | Decisions are lost and re-litigated |
| Immutable records | Honest history | Readers must follow supersession links to find the current state |

A guideline: an ADR should take thirty to ninety minutes for most decisions. If it takes days, the decision is large and the effort is justified, or the document is overbuilt.

## Failure Modes

**Written after the fact to justify.** The decision is already made and the ADR lists straw-man alternatives. It adds no value. Write it while the decision is still open.

**Never updated.** The system moved on but the ADR still says Accepted. Readers trust stale guidance. Mark superseded ADRs, and review old ones during architecture reviews.

**Too many, too trivial.** The folder fills with minor choices and people stop reading. Keep the bar at significant decisions.

**Hidden negatives.** Consequences list only benefits, so the later pain looks like a surprise. Require a "costs and risks" part.

**No owner or decider.** Debate continues indefinitely. Name the decider and a deadline.

**Unfindable.** ADRs sit in a place nobody searches. Link them from the README, from code comments near the relevant module, and from the onboarding guide.

**Decision without revisit triggers.** A choice that was right for a team of five becomes wrong at fifty. Add a trigger: "Revisit if write volume exceeds X" or "if the team grows beyond Y." Choose the trigger from your own measurements.

## In Practice

### Template to start with

Use the structure above: Title, Status, Date, Deciders, Context, Decision, Alternatives considered, Consequences. Add one line: "Revisit when". Keep a short README in the ADR folder with the index and the template.

### Checklist before accepting an ADR

- [ ] The problem and constraints are stated without assuming the solution.
- [ ] At least two real alternatives were considered, including "do nothing" where relevant.
- [ ] The decision is stated in one clear sentence.
- [ ] Negative consequences and new risks are listed.
- [ ] Operational impact is covered: monitoring, on-call, cost.
- [ ] Reversibility is noted: how hard is it to undo?
- [ ] Affected teams have seen it.
- [ ] A revisit trigger exists.

### Reversible and irreversible decisions

Not all decisions deserve equal care. Choosing a logging library is easy to reverse. Choosing a primary data store or a public API shape is not. Spend deliberation in proportion to the cost of being wrong. Making reversible decisions quickly and irreversible ones carefully is a core skill.

### Developer vs. engineer

A developer picks the tool they know and moves on. A year later nobody remembers why. An engineer writes down the options and the reasons, so the next person can judge whether the reasons still hold. The record also protects the engineer: when the decision is questioned later, the context is on file.

### Verifying AI-generated designs

AI tools can draft ADRs fluently, and the draft will look complete. That is the risk. Verify that:

- The context describes your actual constraints, not generic ones.
- Alternatives are real options you could adopt, not filler.
- Stated consequences are true for your system. Do not accept invented figures or claims about performance.
- A human decider owns the decision. A fluent document is not a decision.

Use AI to draft and to challenge ("what are the weakest assumptions here?"), and keep the judgment human.

## Further Reading

- Michael Nygard, "Documenting Architecture Decisions" (2011 blog post, Cognitect).
- Gregor Hohpe, *The Software Architect Elevator*.
- Mark Richards and Neal Ford, *Fundamentals of Software Architecture*, chapter on architecture decisions.
- The ADR community site, adr.github.io, for templates and tooling.
- Neal Ford, Rebecca Parsons, Patrick Kua, *Building Evolutionary Architectures*, on fitness functions that can enforce decisions.
