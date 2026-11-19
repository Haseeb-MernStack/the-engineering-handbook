# Trade-offs and Decision Making

> Engineering is choosing among imperfect options under uncertainty, so the quality of your decisions depends on how clearly you see costs, constraints, and reversibility.

## Overview

There is rarely a best solution in the abstract. There is a best solution for a particular set of goals, constraints, and risks at a particular time. Faster means less consistent. Simpler means less flexible. Safer means slower. Every choice buys something and pays with something else.

A developer who writes code often asks "what is the right way?" and looks for a rule. An engineer who owns outcomes asks "what are we optimizing for, what are we willing to give up, and how will we know if we chose badly?" The second question has an answer you can defend and revisit.

This topic provides a framework for making and recording decisions. It covers how to name the options, what to compare, how to account for uncertainty and reversibility, and how decisions fail.

## Core Concepts

### Every decision has a context

The same question can have different answers in different contexts. A relational database may be right for a payments system and wrong for a high-volume event firehose, or the reverse, depending on consistency needs, volume, team skills, and budget. Write down the context before the answer:

- What problem are we solving, and for whom?
- What are the hard constraints (law, budget, deadline, existing systems)?
- What qualities matter most (correctness, latency, cost, simplicity, time to market)?
- What is the scale now, and what is plausible in a year?
- Who will operate this?

### Quality attributes pull against each other

Common pairs:

| Gain | Usually costs |
|---|---|
| Lower latency | Higher cost, more complexity (caching, replication), or weaker consistency |
| Higher availability | Cost, operational complexity, sometimes consistency |
| Stronger consistency | Latency and availability under failure |
| Higher throughput | Per-item latency (batching), memory |
| Flexibility | Simplicity and sometimes performance |
| Speed of delivery | Polish, test depth, or future cost |
| Security | Convenience and sometimes speed |
| Portability | Use of platform-specific strengths |

Recognizing these pairs lets you notice when a requirement quietly asks for both sides. "Always available, always consistent, and fast" is a request for a trade-off, not a design.

### Reversibility decides how careful to be

Some decisions are easy to undo: a function name, an internal library, a feature flag. Others are hard to undo: a public API shape, a data format written to disk, a primary key choice, a vendor contract, a security model.

- **Two-way doors**: decide quickly, with the information you have. Learn from the result.
- **One-way doors**: slow down, gather evidence, get review, and prototype if possible.

Treating every decision as a one-way door causes paralysis. Treating them all as two-way doors causes expensive surprises.

### Make costs explicit

Compare options on dimensions that matter, including costs people forget:

- Build effort and time
- Operating effort and on-call burden
- Runtime cost
- Learning curve for the team
- Risk and blast radius if it fails
- Lock-in and exit cost
- Effect on future changes
- Security and compliance impact

The cost of ownership usually exceeds the cost of building. Software is maintained far longer than it is first written.

### Reasoning under uncertainty

You rarely have full information. Techniques that help:

- **Separate facts from assumptions.** List assumptions and mark which are cheap to check.
- **Prototype or spike** to turn the riskiest assumption into evidence.
- **Pre-mortem**: imagine the decision failed a year from now and list why. This finds risks that optimism hides.
- **Set a decision deadline.** More information has diminishing value, while delay has a real cost.
- **Define what would change your mind.** If you cannot, you are not reasoning, you are defending.
- **Prefer options that keep choices open** when the cost of keeping them open is small.

### Recording the decision

A lightweight decision record captures what was chosen and why, so later readers can tell whether the reasons still hold.

```markdown
# ADR-012: Use a message queue for order notifications

Status: Accepted
Context: Notification sending is slow and fails independently of checkout.
Options considered: (1) call provider inline, (2) background job table, (3) managed queue
Decision: Option 3.
Consequences: Checkout latency no longer depends on the provider.
  We now operate a queue, handle duplicate delivery, and monitor lag.
Revisit if: volume or provider behavior changes materially.
```

The most valuable parts are the options you rejected and the consequences you accepted.

### Common heuristics, and their limits

| Heuristic | Useful because | Misleading when |
|---|---|---|
| Keep it simple | Complexity is a lasting cost | "Simple" ignores a real requirement |
| Don't repeat yourself | Duplication spreads bugs | Two things look alike but change for different reasons |
| You aren't gonna need it | Speculative generality is usually wrong | A cheap hook now avoids an expensive migration later |
| Make it work, make it right, make it fast | Orders effort sensibly | "Later" never arrives for correctness or security |
| Use boring technology | Mature tools have known failure modes | The boring choice truly cannot meet the need |
| Measure before optimizing | Avoids wasted effort | You ignore known design limits that no measurement will fix |

Heuristics are compressed experience. They are starting points, not verdicts.

## Trade-offs

Decision making has its own internal trade-offs.

- **Speed vs. rigor.** Deliberate analysis reduces error and delays action. Match rigor to stakes and reversibility.
- **Consensus vs. clarity.** Broad agreement builds buy-in and can dilute the decision or stall it. A single accountable decision-maker who consults widely is often faster. It requires trust and a record of reasoning.
- **Standardization vs. fit.** Using the same stack everywhere cuts operating cost and training. It forces some problems into a poor fit. Allow exceptions with a clear bar for them.
- **Local vs. global optimum.** A choice that is best for your team may raise costs for others, such as a custom format nobody else can read. Weigh the system-wide effect.
- **Short-term vs. long-term.** Taking on technical debt can be rational, such as a deadline with real consequences. It is a loan with interest. Track it and plan repayment, or it compounds.
- **Cost of analysis.** Writing a long design document for a small, reversible choice is itself a bad trade-off.

## Failure Modes

**Decision by default.** Nobody decides, so the first implementation becomes the architecture. Detect it by asking "who chose this, and when?" and finding no answer.

**Optimizing the wrong quality.** Spending effort on peak throughput when the real pain is deploy safety. Prevention: tie each design goal to a stated need and ask for evidence.

**Premature optimization.** Complicating the design for scale you may never reach. The cost is real now and the benefit speculative. Prevention: design for the load you can justify, and make the system measurable so you know when it changes.

**Premature generalization.** Building a framework for a second use case that never comes. Prevention: wait for the second or third real instance before abstracting.

**Under-engineering the irreversible.** Choosing quickly on something costly to change, such as a public API, then living with it. Prevention: identify one-way doors explicitly and apply extra review.

**Sunk cost.** Continuing a failing approach because of effort already spent. Prevention: ask "knowing what I know now, would I start this today?"

**Authority and fashion.** Choosing because a senior person said so, or because a technology is trendy. Prevention: require reasoning tied to this context, whoever proposes it.

**Anchoring on the first option.** The first idea shapes everything after. Prevention: require at least two genuinely different alternatives, including "do nothing".

**Hidden second-order effects.** A change fixes one thing and breaks another elsewhere, such as a retry policy that overloads a dependency. Prevention: for each decision, ask what else it touches and what happens at the boundaries.

**Undocumented decisions.** The reasoning is lost, and later engineers either reverse a good choice or preserve a bad one out of fear. Prevention: short decision records stored with the code.

**Never revisiting.** Context changes and the decision does not. Prevention: record revisit conditions and check them.

## In Practice

### A decision checklist

1. What problem are we solving? Who is affected?
2. What are the constraints and the top two or three qualities that matter?
3. What are at least two real options, plus doing nothing?
4. For each, what does it cost to build, run, and change?
5. How reversible is the choice? What is the exit cost?
6. What are the riskiest assumptions, and can a small experiment test them?
7. What would a pre-mortem say?
8. Who decides, and by when?
9. Where is the decision recorded, and what would make us revisit it?

### A simple weighted comparison

When options are close, a table makes reasoning visible. The scores are judgments, not measurements, and the value is in debating the weights.

| Criterion (weight) | Option A: managed service | Option B: self-hosted |
|---|---|---|
| Time to deliver (3) | 4 | 2 |
| Operating burden (3) | 4 | 2 |
| Control and tuning (1) | 2 | 5 |
| Exit cost (2) | 2 | 4 |
| Running cost at expected scale (2) | 3 | 4 |

If the team disagrees on the outcome, the disagreement usually sits in a weight. That is the real conversation.

### A worked example

A team must choose how to store user sessions.

- **Option 1: signed tokens held by the client.** Servers stay stateless and scale simply. Revoking a session before it expires is hard, and token size grows with content.
- **Option 2: server-side sessions in a shared store.** Revocation is immediate and content is private. The store becomes a dependency that must be highly available, and every request pays a lookup.

The deciding questions: how fast must revocation work, how sensitive is the data, and what is the team able to operate? If instant revocation matters for compliance, Option 2 wins despite the cost. If sessions are low-risk and short-lived, Option 1 may be enough. The right answer follows from the context. Writing it down also records the cost accepted.

### Developer vs. engineer

| Situation | Developer who writes code | Engineer who owns outcomes |
|---|---|---|
| Choosing a database | Picks the one they know | Lists access patterns, consistency needs, and operating cost, then compares |
| Deadline pressure | Cuts tests silently | Names what is being cut, the risk, and when it will be repaid |
| Disagreement | Argues preference | States the criteria and checks which option satisfies them |
| Design review | Defends the design | Presents rejected options and the costs accepted |

### When AI generates code or proposes designs

AI tools tend to propose a single confident answer and rarely state what it costs. Ask directly:

- "Give three approaches and the downsides of each."
- "What breaks at ten times the load?"
- "What would make this choice a mistake?"

Then check the answers against your own constraints. A model does not know your team, budget, traffic, or compliance obligations unless you tell it. The decision remains yours, and so does the accountability. Strong fundamentals are what let you tell a sound trade-off from a plausible-sounding one.

## Further Reading

- *Designing Data-Intensive Applications*, Martin Kleppmann. Trade-offs in storage, consistency, and distribution, stated plainly.
- *Fundamentals of Software Architecture*, Mark Richards and Neal Ford. Architecture characteristics and trade-off analysis.
- *Thinking, Fast and Slow*, Daniel Kahneman. Cognitive biases that distort decisions.
- *A Philosophy of Software Design*, John Ousterhout. Managing complexity as a design goal.
- "Documenting Architecture Decisions", Michael Nygard. The essay that popularized lightweight decision records.
- *Software Architecture: The Hard Parts*, Neal Ford, Mark Richards, Pramod Sadalage, and Zhamak Dehghani.

Back to the section index: [Engineering Fundamentals](./README.md)
