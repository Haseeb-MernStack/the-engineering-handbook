# Design Docs and RFCs

> A design doc or RFC is a short written proposal that exposes the reasoning behind a technical decision so that others can improve it before the expensive work begins.

## Overview

Changing a design on paper costs minutes. Changing it after implementation costs weeks. A design doc moves the thinking, and the disagreement, to the cheap end of the timeline.

Two related artifacts appear in most teams. A **design doc** usually describes how one team will build something. An **RFC** (request for comments) invites a wider audience, often several teams or a whole community, to review a proposal that affects them. Some organizations use the terms interchangeably. Others add **ADRs** (architecture decision records), which are short records of a single decision after it has been made.

The purpose is not paperwork. The purpose is to make decisions better and to leave a record of why they were made. When someone asks in two years "why is it built this way?", the document should answer.

This matters in an age of generated code. A model can propose an implementation in seconds, but it cannot know your constraints, your users or the history behind your system. The design doc is where a human states the problem, weighs options and takes responsibility for the choice.

## Core Concepts

### When to write one

Write a design doc when:

- The change is hard to reverse (data models, public APIs, service boundaries).
- It affects several teams or systems.
- It takes more than a few weeks of effort.
- There are several credible approaches and you need to choose.
- You are uncertain and want others to find your blind spots.

Do not write one for a small, easily reversed change. A short ticket or pull request description is enough. A rough rule: the effort of the document should be proportionate to the cost of being wrong.

### Anatomy of a good doc

A good doc has a clear problem, explicit goals and non-goals, a proposed design, alternatives and an honest list of risks. The most commonly skipped and most valuable parts are non-goals and alternatives.

- **Goals** say what success is. Prefer measurable ones.
- **Non-goals** state what this design deliberately does not address. They prevent scope creep and misreading.
- **Alternatives considered** show that you explored the space and why you rejected other options. Without them, reviewers will suggest them again.
- **Trade-offs** state what the chosen design costs.

### Design-doc outline template

Copy and adapt this outline. Delete sections that do not apply.

```markdown
# Title: <short, specific name of the change>

Status: Draft | In review | Approved | Superseded
Author(s): <names>
Reviewers: <names or teams>
Last updated: <date>

## Summary
Two or three sentences: what you propose and why. A reader should be able to stop here and know the gist.

## Context and problem
What is the situation today? What is broken, missing or costly? Include evidence: incidents, metrics, user reports.

## Goals
- Measurable outcomes this design must achieve.

## Non-goals
- Things this design intentionally does not do.

## Proposed design
### Overview
A diagram and a plain description of the main components and how they interact.
### Data model and interfaces
Schemas, APIs, events. Show examples.
### Key flows
Walk through the main scenarios step by step, including failure paths.
### Dependencies
What this relies on and what relies on it.

## Alternatives considered
For each option: what it is, why it is attractive, why it was not chosen.

## Trade-offs and risks
What does this design cost? What could go wrong? What is the plan if it does?

## Security, privacy and compliance
Data handled, access control, threats considered.

## Testing and rollout
How will you verify it? How will you release it safely (flags, staged rollout, rollback)?

## Observability and operations
Metrics, logs, alerts, runbooks, on-call impact.

## Cost and capacity
Expected load, storage and money. State assumptions.

## Open questions
Unresolved issues and who will resolve them.

## Milestones
Rough phases and ordering, not a detailed schedule.

## Decision log
Date, decision, rationale. Updated as the design evolves.
```

### RFC process

An RFC process adds structure to wider review:

1. **Draft.** The author writes the proposal.
2. **Socialize.** Talk to the most affected people before circulating. No one should be surprised by a document.
3. **Review period.** A fixed window, such as a week or two, with named reviewers and open comments.
4. **Decision.** A named decision maker or group accepts, rejects or requests changes, and records why.
5. **Record.** The final doc is archived and linked from the code or the repository.

Python's PEPs and the Rust RFC repository are public examples of this process at community scale.

### ADRs

An architecture decision record is a short, dated note: context, decision, consequences. Michael Nygard popularized the lightweight format. ADRs are useful for decisions made in the course of work, where a full design doc would be heavy. See [Software Architecture](../04-software-architecture/README.md).

### Reviewing a design

Good reviewers:

- Read the problem statement first and check that the design addresses it.
- Ask about failure cases, scale assumptions and migration.
- Raise alternatives the author did not list.
- Separate blocking concerns from suggestions.
- Review the design, not the author.

A useful question: "What would make us regret this in a year?"

## Trade-offs

**Thoroughness versus speed.** More review catches more problems but delays work. Scale rigor to risk. Reversible decisions deserve a light process.

**Early review versus polished review.** Sharing a rough draft finds fundamental flaws cheaply but exposes unfinished thinking. Waiting for polish risks sinking time into a bad direction. Prefer early, labeled as draft.

**Consensus versus a decision maker.** Seeking consensus builds buy-in but can stall or dilute the design. A named decider moves faster but may leave dissent unaddressed. Many teams seek input broadly and decide narrowly.

**Documents versus prototypes.** A prototype tests assumptions that prose cannot. It also invites attachment to code. Use prototypes to answer specific questions, and write up what you learned.

**Living docs versus point-in-time records.** Keeping a doc updated is costly and often neglected. Treating it as a dated snapshot with a clear status is more honest. Pair it with the code's own documentation for current behavior.

## Failure Modes

- **The ceremonial doc.** Written after the decision to get approval. Reviewers sense it and disengage. Prevent it by writing before committing, and by genuinely considering alternatives.
- **The novel.** Fifty pages no one reads. Lead with a summary, link out to detail, and cut.
- **No alternatives, no non-goals.** The design appears inevitable, which usually means it was not examined. Require both.
- **Review by silence.** No comments are mistaken for approval. Ask named reviewers directly and set a deadline.
- **Design by committee.** Every comment is accommodated and the result is incoherent. A single author keeps ownership and a decider resolves conflicts.
- **Bike-shedding.** Discussion concentrates on trivial matters, such as naming, because they are easy to opine on. The author should steer attention to the high-risk questions.
- **Stale docs treated as truth.** Someone follows an obsolete design. Mark status clearly and link superseded docs to their successors.
- **Ignoring operations.** A design that says nothing about monitoring, rollout or failure is only half a design.
- **Generated documents unexamined.** A model can produce a plausible-looking document with invented constraints and unjustified choices. The author must verify every claim and own the reasoning.

## In Practice

### A workable lightweight process

1. Write a one-page summary first: problem, proposal, alternatives. Share it with two or three trusted engineers.
2. Revise, then expand into the full template only where risk justifies.
3. Circulate to the affected teams with a clear ask and a review deadline.
4. Hold a short meeting only to resolve open disagreements, not to read the doc aloud.
5. Record the decision and link the doc from the repository.

### Writing tips

- Draw the architecture. One diagram often replaces a page.
- Use concrete numbers for load, latency and cost, and label assumptions.
- Include a "how this fails" section. It improves the design.
- Write the summary last, then put it first.
- Prefer plain language. Define terms.

### After approval

- Break the work into milestones and link tickets to the doc.
- Update the doc when the design changes materially. Add to the decision log.
- When the work ships, note any deviations. They are useful lessons.

### Practice exercise

Pick a feature you built recently. Write a two-page design doc as if you were proposing it today, including two alternatives and the failure modes. Compare it with what you actually did.

## Further Reading

- Michael Nygard, "Documenting Architecture Decisions" (2011), the original ADR proposal.
- Titus Winters, Tom Manshreck and Hyrum Wright, *Software Engineering at Google*. Includes discussion of design documents and review.
- Python Enhancement Proposals, especially PEP 1, and the Rust RFC repository, as examples of public RFC processes.
- Tanya Reilly, *The Staff Engineer's Path*. Covers writing and driving technical decisions.
- Martin Kleppmann, *Designing Data-Intensive Applications*. A model of reasoning explicitly about trade-offs.
