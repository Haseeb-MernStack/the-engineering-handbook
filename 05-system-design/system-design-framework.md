# System Design Framework

> A system design framework is a repeatable sequence of questions that turns a vague problem into a justified design, and is the same skill used in real design reviews.

## Overview

System design is the practice of deciding what components a system needs, how they interact, where data lives, and how the whole thing behaves under load and failure. It appears in two settings: interviews and real work. The real setting matters more. Real design means writing a proposal, defending it in a review, and living with the result.

Beginners often jump to boxes and technology names. "Use Kafka and Redis." Seniors start elsewhere: what problem are we solving, for whom, how big, how reliable, and what can we leave out? The design follows from the answers.

A framework does not replace judgment. It protects you from skipping the steps that matter and gives reviewers a shared structure to check. It also makes your reasoning visible, which is the part others can verify.

## Core Concepts

### The sequence

| Step | Question | Output |
|---|---|---|
| 1. Clarify | What are we building, and for whom? | Scope, users, use cases |
| 2. Requirements | What must it do, and how well? | Functional and non-functional requirements |
| 3. Estimate | How much load and data? | Order-of-magnitude numbers |
| 4. Interfaces and data | What are the APIs and entities? | API sketch, data model |
| 5. High-level design | What are the components? | Diagram of the main path |
| 6. Deep dive | Where is the hard part? | Detailed design of 1-3 components |
| 7. Failure and bottlenecks | What breaks, and what limits growth? | Risks and mitigations |
| 8. Evolve | What changes later? | Phases, open questions |

### Step 1: Clarify

Ask until the problem is bounded. Who are the users? What is the core action? What is explicitly out of scope? Ambiguity left here becomes rework later. Writing a short list of non-goals is as valuable as the goals.

### Step 2: Requirements

**Functional**: what the system does. "Users can shorten a URL and be redirected."

**Non-functional**: how well. Latency, availability, durability, consistency, security, cost, and operability. Make them measurable. "Fast" is not a requirement. "95% of redirects complete within 100 ms" is. Pick numbers from your product needs, not from habit.

Identify which requirement is dominant. A payment system prioritizes correctness and durability over latency. A feed prioritizes latency and availability over perfect freshness. The dominant requirement drives most choices.

### Step 3: Estimate

Rough arithmetic decides whether you need one machine or a hundred. See [Capacity Estimation](./capacity-estimation.md). Even a crude estimate prevents designs that are 100 times too big or too small.

### Step 4: Interfaces and data

Define the main operations before the internals.

```
POST /v1/links        {long_url}        -> {short_code}
GET  /{short_code}                      -> 301/302 redirect
```

Then sketch the core entities and their relationships, access patterns, and the keys used to look them up. Data access patterns choose the storage technology far more than preference does. Ask: what are the most frequent reads and writes? What must be consistent?

### Step 5: High-level design

Draw the simplest design that meets the requirements. Start with one path, the most important one, from client to storage and back.

```
Client -> Load balancer -> App servers -> Cache -> Database
                                  \-> Queue -> Workers
```

Resist adding components that no requirement demands. Every box is something to deploy, monitor, secure, and pay for.

### Step 6: Deep dive

Choose the parts where the design is risky or non-obvious and work them through: key generation, ranking, partitioning, consistency, hot spots. Explain the options and why you chose one. This is where depth shows.

### Step 7: Failure and bottlenecks

Walk the diagram and ask of each component: what if it is slow, down, or wrong? What if a dependency is? Where does load concentrate first? What happens at ten times the expected traffic? See [Reliability and Availability](./reliability-and-availability.md).

### Step 8: Evolve

State what you would build first and what you would defer. A design that starts simple and has a stated growth path beats one that is complex from day one.

### Running a design review

A design review is the same framework applied to a written proposal. A useful proposal contains:

- Problem and goals; non-goals.
- Requirements with numbers.
- Proposed design with a diagram.
- Alternatives considered and why they were rejected.
- Risks, failure modes, and rollout plan.
- Open questions.

Reviewers look for unstated assumptions, missing failure handling, unjustified complexity, and mismatched numbers. Record the outcome as a decision (see [Architecture Decision Records](../04-software-architecture/architecture-decision-records.md)).

## Trade-offs

Every design is a set of trades. Make them explicit.

| Choice | Gain | Cost |
|---|---|---|
| Simple design first | Fast to build, easy to operate | May need rework at higher scale |
| Design for large scale now | Fewer migrations later | Complexity and cost paid before any need exists |
| Strong consistency | Simple reasoning for users and developers | Higher latency, lower availability in some failures |
| Eventual consistency | Availability and speed | Stale reads; harder application logic |
| Build | Fit and control | Engineering time and ongoing maintenance |
| Buy or use managed | Speed, less operations | Cost, lock-in, less control |
| More detail upfront | Fewer surprises | Time spent on guesses that may be wrong |

The time budget is also a trade-off. In a review or interview, spending most of the time on the diagram leaves none for failure analysis, which is usually where the value is.

## Failure Modes

**Solution first.** The design begins with a technology and then searches for a problem. Prevent by writing requirements before drawing.

**Unmeasurable requirements.** "Highly available and fast" cannot be checked. Convert to numbers and tie them to user impact.

**Skipping estimation.** The design needs 50 database nodes or one. Nobody checked. A five-minute calculation would have shown it.

**Over-engineering.** Sharding, multi-region, and a service mesh for a system with a few hundred users. The cost shows up as slow delivery and fragile operations.

**Happy path only.** The diagram shows everything working. In production, a slow dependency exhausts the thread pool and the whole system stalls. Always include a failure walk.

**Buzzword substitution.** "We will use a cache" without saying what is cached, how it is invalidated, and what happens on a miss storm.

**Ignoring operations.** No plan for deployment, monitoring, migrations, or on-call. The design works until the first incident.

**Single-person design.** Nobody else reviewed it, so blind spots survive. Review early, when changes are cheap.

## In Practice

### Pocket checklist

- [ ] Problem, users, and non-goals written down.
- [ ] Requirements are measurable and ranked.
- [ ] Back-of-envelope numbers computed and assumptions listed.
- [ ] APIs and data access patterns defined.
- [ ] Simplest viable architecture drawn.
- [ ] Two or three hard problems explored with alternatives.
- [ ] Failure of each component considered.
- [ ] Observability, deployment, and rollback covered.
- [ ] Growth path and deferred items stated.

### Questions worth asking of any design

- What is the most likely thing to break first?
- What data can we never lose?
- What happens if this dependency is down for an hour?
- How will we know it is unhealthy before users tell us?
- What is the cheapest version that proves the idea?
- How do we undo this change?

### Developer vs. engineer

A developer is given a feature and designs the code for it. An engineer asks about volume, failure, ownership, and cost, and can say what the design will cost to run and where it will strain.

### Verifying AI-generated designs

AI tools produce architectures that look credible: load balancer, cache, queue, database replicas. Review them critically:

- Does each component trace to a stated requirement? Remove those that do not.
- Are the numbers real, or invented? Insist on your own estimate.
- Is failure handling specific, or just "add retries"?
- Are consistency and data-loss behaviors stated?
- Did it quietly assume managed services you do not have?

Use the tool to enumerate alternatives and to challenge assumptions. Do not let it decide scope.

## Further Reading

- Martin Kleppmann, *Designing Data-Intensive Applications*.
- Alex Xu, *System Design Interview* (volumes 1 and 2).
- Mark Richards and Neal Ford, *Fundamentals of Software Architecture*.
- Google, *Site Reliability Engineering* and *The Site Reliability Workbook*, on design for operations.
- Gregor Hohpe, *The Software Architect Elevator*.
