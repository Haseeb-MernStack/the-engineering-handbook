# Requirements and Estimation

> Most project failures begin with building the wrong thing or misjudging the cost of the right thing, so engineers must clarify what is needed and estimate honestly under uncertainty.

## Overview

Code is the last step in a chain that starts with a need. If the need is misunderstood, even flawless code fails. Requirements work is the practice of finding out what is needed, why, and what "done" means. Estimation is the practice of predicting the effort, time, and capacity involved.

Both are about uncertainty. Requirements are incomplete and change. Estimates are guesses about work you have not yet done. Neither can be made exact. Both can be made better, and both can be communicated honestly.

A developer who writes code waits for a complete specification and then reports a single number when asked. An engineer who owns outcomes asks questions that expose hidden assumptions, separates what is known from what is not, and gives estimates as ranges with stated confidence and risks.

This topic covers eliciting and structuring requirements, non-functional requirements, back-of-envelope capacity estimates, and effort estimation.

## Core Concepts

### Requirements are about problems, not solutions

A stakeholder often arrives with a solution: "add a button that exports to CSV." Behind it is a problem: "finance needs monthly figures in a spreadsheet." Understanding the problem may reveal a better solution, or show the button is not enough.

Useful questions:

- Who needs this, and what are they trying to accomplish?
- What do they do today, and what is painful about it?
- What does success look like, and how will we measure it?
- What happens if we do not build it?
- What are the constraints: deadline, budget, law, existing systems?
- What is out of scope?

### Functional and non-functional requirements

**Functional requirements** describe what the system does: "a user can reset a password using an emailed link."

**Non-functional requirements** (also called quality attributes) describe how well it does it. They shape architecture more than most features do.

| Attribute | Example of a testable statement |
|---|---|
| Performance | 95% of search requests complete within a stated time at expected peak load |
| Availability | The service meets an agreed uptime target measured monthly |
| Durability | Committed orders survive the loss of a single machine |
| Scalability | The system handles a stated growth in users without redesign |
| Security | Personal data is encrypted in transit and at rest; access is logged |
| Maintainability | A new engineer can run the system locally from the documentation |
| Compliance | Data for certain users stays in a given region |
| Usability / accessibility | Core flows are usable with a keyboard and screen reader |

A non-functional requirement without a number or test is a wish. "Fast" cannot be verified. "p95 under a stated duration at a stated load" can.

### Writing requirements that can be tested

Aim for statements that are specific, unambiguous, and verifiable. Acceptance criteria in a given-when-then form help:

```
Given a registered user with a valid reset token,
When they submit a new password that meets the policy,
Then the password is changed, all existing sessions are revoked,
  and a confirmation email is sent.

Given an expired reset token,
When the user submits the form,
Then the request is rejected with a clear message and no change is made.
```

Include the unhappy paths. Most defects live there.

### Surfacing hidden requirements

Stakeholders state what they remember. Implicit needs go unsaid until they are violated:

- Behavior on failure and on bad input
- Limits (size, rate, quantity)
- Data retention and deletion
- Auditing and permissions
- Migration of existing data
- Operations: monitoring, deployment, rollback, support tooling
- Edge cases: time zones, currencies, empty states, concurrency

A useful habit is to walk through the whole lifecycle of the data and the user journey and ask at each step "what could go wrong, and what should happen?"

### Back-of-envelope estimation

Before building, estimate the scale. The point is order of magnitude, to decide what design is plausible.

Worked example: a service that stores user-uploaded images.

- Assume 1 million active users, 2 uploads per user per day on average.
- Daily uploads: 2 million. Per second on average: 2,000,000 / 86,400, about 23 per second. Peak may be several times the average, so plan for a few hundred per second at the peak.
- Assume 2 MB per image. Daily storage growth: 2,000,000 x 2 MB = 4 TB per day. Per year: roughly 1.5 PB before replication.

Conclusions follow immediately. Average request rate is modest and a single database handles the metadata. Storage volume is large and calls for object storage with lifecycle policies, not database blobs. Replication multiplies the figure. Cost needs a conversation now, not later.

The numbers are assumptions, flagged as such. Their value is in showing which factors dominate. If the assumption about image size is wrong by a factor of two, you know the storage figure moves by the same factor.

Useful habits:

- Write down assumptions and units.
- Convert to per-second figures, and consider peak versus average.
- Consider read-to-write ratio, data growth, and retention.
- Round aggressively. Precision is false here.
- Sanity-check against something you know.

### Estimating effort

Software effort estimates are notoriously unreliable. Reasons include:

- Work is new each time, unlike repeating a manufacturing process.
- Unknowns surface during the work.
- People estimate the happy path and forget testing, review, deployment, and integration.
- Optimism is common, and estimates are often treated as commitments.

Practical methods:

- **Break the work down.** Smaller pieces are easier to reason about, and gaps become visible. Beware false precision from over-splitting.
- **Use reference classes.** Compare with similar past work and what it actually took, not what was planned.
- **Give ranges.** "Three to six weeks, most likely four" says more than "four weeks". A wide range signals uncertainty.
- **Name the biggest risks.** Unknown integrations, unclear requirements, and unfamiliar technology are where estimates break.
- **Spike the unknown.** A short time-boxed investigation reduces the largest uncertainty cheaply.
- **Re-estimate as you learn.** The cone of uncertainty narrows as work progresses. An old estimate is not a promise.
- **Estimate as a group.** Different people remember different complications. Disagreement is useful information.

Distinguish three things people conflate: an **estimate** (a prediction), a **target** (what the business would like), and a **commitment** (a promise). Mixing them turns honest estimates into pressure.

### Scope, time, and quality

Scope, schedule, and quality are linked, and resources help less than expected. Adding people to a late project often slows it, because of onboarding and coordination costs. When the date is fixed, the lever that works is scope. Cut features, not tests.

## Trade-offs

- **Upfront analysis vs. learning by building.** Detailed requirements reduce rework when the problem is well understood. When it is not, building a small version teaches faster than discussion. Match the approach to the uncertainty.
- **Precision vs. cost.** A precise estimate takes effort and may be wrong anyway. For early decisions, a rough range is enough. Refine when the decision is costly.
- **Completeness vs. speed.** Waiting for full requirements delays the start. Starting with gaps risks rework. Resolve the questions that would change the design first and defer the rest.
- **Flexibility vs. commitment.** Fixed scope gives predictability and resists change. Flexible scope adapts and may never finish. Agree on what is fixed and what can move.
- **Building for the stated need vs. anticipating growth.** Designing for ten times the load costs now. Designing only for today may force a rewrite. Choose by how likely and how soon growth is, and by how costly the rewrite would be.
- **Optimistic vs. padded estimates.** Optimistic estimates miss and erode trust. Heavy padding wastes capacity and gets cut. Ranges with explicit risks are more honest than either.

## Failure Modes

**Building to the letter, missing the point.** The team delivers exactly what was written and users cannot do their job. Detect it by demonstrating early and watching real users.

**Ambiguous requirements.** Words like "fast", "secure", "user-friendly", and "real time" mean different things to different people. Prevention: require numbers and examples.

**Missing non-functional requirements.** The feature works and falls over at ten times the expected load, or has no audit trail where the law requires one. Prevention: ask about scale, availability, security, and compliance in every project.

**Scope creep.** Small additions accumulate without any reassessment of the schedule. Prevention: record changes, state their cost, and decide on purpose.

**Requirements frozen too early.** Change is treated as failure, so the team ships an outdated understanding. Prevention: short feedback loops.

**Single-point estimates treated as promises.** A guess becomes a deadline. Prevention: ranges and confidence levels, plus explicit updates when assumptions change.

**Planning fallacy.** People underestimate even when they know similar work ran long. Prevention: use past actuals and an outside view.

**Forgotten work.** Estimates omit testing, review, migration, documentation, deployment, monitoring, and rework. Prevention: use a checklist of activities.

**Hidden dependencies.** Work waits on another team, a vendor, or an approval. Prevention: list dependencies and their owners at the start.

**Wrong-unit capacity math.** Mixing bits and bytes, per-second and per-day, or average and peak, so the design is off by an order of magnitude. Prevention: write units on every line and re-check.

**Estimation as negotiation.** Estimates are pushed down until they match a desired date. The work does not shrink. Prevention: separate the estimate from the target, and negotiate scope instead.

## In Practice

### A requirements checklist

- Can I state the problem, the user, and the success measure?
- What is out of scope, written down?
- Are acceptance criteria testable, including error cases?
- Have I asked about scale, latency, availability, durability, security, and compliance?
- What happens to existing data and users?
- How will this be deployed, monitored, supported, and rolled back?
- Who is the decision-maker for open questions, and by when?
- What are the riskiest assumptions, and how will I test them early?

### An estimation checklist

- Did I break the work into pieces small enough to understand?
- What similar work have we done, and how long did it really take?
- Are testing, review, deployment, monitoring, and documentation included?
- What are the top three risks, and what would each add?
- Is the figure a range with a confidence level?
- Have I distinguished the estimate from the target?
- When will I re-estimate?

### A small capacity calculation in code

Writing the arithmetic down keeps units and assumptions visible and lets others adjust them.

```python
SECONDS_PER_DAY = 86_400

users = 1_000_000
uploads_per_user_per_day = 2
image_mb = 2
peak_factor = 5            # assumption: peak is ~5x average

daily_uploads = users * uploads_per_user_per_day
avg_per_sec = daily_uploads / SECONDS_PER_DAY
peak_per_sec = avg_per_sec * peak_factor
daily_storage_tb = daily_uploads * image_mb / 1_000_000   # MB -> TB (decimal)

print(f"avg {avg_per_sec:.0f}/s, peak {peak_per_sec:.0f}/s, "
      f"{daily_storage_tb:.1f} TB/day")
```

### Developer vs. engineer

| Situation | Developer who writes code | Engineer who owns outcomes |
|---|---|---|
| Vague ticket | Guesses and builds | Asks about the problem, users, and success criteria first |
| Asked "how long?" | Gives one number | Gives a range, states assumptions and risks |
| New feature | Implements the happy path | Lists failure cases, limits, and operational needs |
| Deadline too tight | Works longer hours silently | Presents options: reduce scope, move the date, or accept stated risk |

### When AI generates code or specs

AI can draft requirements, acceptance criteria, and estimates quickly. It does not know your users or system. Review them for:

- Plausible but unsupported numbers. Do not accept figures without a source or an assumption you recognize.
- Missing non-functional requirements and edge cases.
- Generated code that implements the prompt literally. Check it against the real need, not just the prompt.
- Estimates that reflect typing time, not understanding, review, and integration. The time to produce code has dropped, but the time to clarify, verify, and operate it has not.

## Further Reading

- *Software Requirements*, Karl Wiegers and Joy Beatty.
- *Software Estimation: Demystifying the Black Art*, Steve McConnell.
- *The Mythical Man-Month*, Frederick P. Brooks Jr. Classic essays on schedule, staffing, and conceptual integrity.
- *Designing Data-Intensive Applications*, Martin Kleppmann. Chapter material on reliability, scalability, and maintainability.
- *Thinking, Fast and Slow*, Daniel Kahneman. Includes the planning fallacy and the outside view.
- *Specification by Example*, Gojko Adzic. Using concrete examples as requirements.

Back to the section index: [Engineering Fundamentals](./README.md)
