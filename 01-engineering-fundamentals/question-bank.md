# Question Bank: Engineering Fundamentals

> Questions for interviews, design reviews and self-testing. Each has a model answer behind a toggle, so you can commit to your own answer first.

## How to use this page

- **Self-testing.** Answer aloud or in writing before opening the toggle. Compare for gaps, not for matching wording.
- **Interviewing.** Pick questions by level and use *Red flags* as a rubric. Follow up with "what would change your answer?"
- **Design review.** The *Applied* questions work as prompts for reviews and planning.

Difficulty: **E** entry, **M** mid-level, **S** senior.

## Mindset

### 1. (E) What is the difference between a developer and an engineer?

<details>
<summary>Model answer</summary>

Both write code. The difference is in what they take responsibility for. A developer's focus is that the feature works. An engineer's focus is that the system keeps working over time: under load, under failure, as requirements change, and at an acceptable cost. The engineer designs for failure, verifies with tests and monitoring, and owns the result through deployment and operation.

**Strong answers:**
- Say it is a matter of habits and scope, not title or seniority.
- Give a concrete example, such as adding a timeout and an alert along with the feature.

**Red flags:** Defines it by years of experience or by knowing more languages.
</details>

### 2. (M) A stakeholder asks for a "quick export button". What do you do before building it?

<details>
<summary>Model answer</summary>

Find the underlying need. Ask who uses it, what they do with the data, how often, how much data, and what happens today. Ask about format, permissions, sensitive fields, and what failure looks like. Then state the options and cost of each, for example a synchronous download for small data versus a background job for large exports.

**Strong answers:**
- Raise data volume, access control and auditing early.
- Propose the smallest option that meets the need, and say what it leaves out.

**Red flags:** Starts coding at once, or refuses without exploring the need.
</details>

## How computers work

### 3. (E) Why is reading from memory much faster than reading from disk or the network?

<details>
<summary>Model answer</summary>

Each level is farther from the CPU and built for capacity rather than speed. Registers and CPU caches are fastest, then main memory, then local storage, then network. Each step down is typically orders of magnitude slower in latency. Programs that keep their working data close, and access it sequentially, run faster.

**Strong answers:**
- Mention that exact numbers vary by hardware, and that ratios matter more than any single figure.
- Connect to caching and batching.

**Red flags:** Quotes precise latency numbers as universal.
</details>

### 4. (M) Your service is slow but CPU use is low. What could be limiting it?

<details>
<summary>Model answer</summary>

Low CPU with slow responses usually means the program is waiting. Candidates include disk or network I/O, a slow downstream dependency or database, lock contention, a connection pool or thread pool that is exhausted, garbage collection pauses, or memory pressure causing swapping. Check queue depths, pool usage, dependency latency and wait time, not just CPU.

**Strong answers:**
- Propose how to tell them apart: traces, per-dependency timing, pool metrics, thread dumps.

**Red flags:** Suggests adding CPUs.
</details>

### 5. (M) What is the difference between a process and a thread?

<details>
<summary>Model answer</summary>

A process has its own address space and resources, isolated from other processes. Threads run within a process and share its memory. Sharing makes communication cheap but requires synchronization to avoid races. Processes give stronger isolation at the cost of heavier creation and communication.

**Strong answers:**
- Link to the failure modes: a crash in a thread can take down the process, while processes fail independently.
- See [Concurrency and Parallelism](../02-programming-concepts/concurrency-and-parallelism.md).
</details>

## Problem solving and debugging

### 6. (E) A bug appears only in production. How do you start?

<details>
<summary>Model answer</summary>

Gather evidence before changing anything. Establish when it started and what changed (deploys, config, traffic, data). Look at logs, metrics and traces around the failures. Find what is different about failing requests versus passing ones. Form a hypothesis, find a way to reproduce it, such as a test, a staging replay or a captured input, and test the hypothesis.

**Strong answers:**
- Check recent changes first.
- Mitigate if users are affected, for example by rolling back, then investigate.
- Avoid changing several things at once.

**Red flags:** Edits code in production, or "fixes" by guessing.
</details>

### 7. (M) A test fails once in a hundred runs. How do you approach it?

<details>
<summary>Model answer</summary>

Treat it as a real defect. Common causes are timing and ordering assumptions, shared state between tests, reliance on real time or randomness, test data collisions and resource limits. Reproduce by running it many times, in different orders and under load. Capture state at failure. Fix the cause, for example by controlling time and randomness and isolating state, rather than adding retries or sleeps.

**Strong answers:**
- Note that ignoring flaky tests erodes trust in the whole suite.
- See [Testing and Quality](../11-testing-quality/README.md).
</details>

### 8. (S) You have spent two hours on a bug with no progress. What do you change?

<details>
<summary>Model answer</summary>

Step back and question assumptions. List what you believe to be true, and verify each belief with evidence. Reduce the problem by bisecting over commits, inputs or components. Explain it to someone else, since stating the problem often reveals the gap. Check that you are running the code you think you are running. Write down hypotheses already ruled out.

**Strong answers:**
- Talk about methods, not persistence.
- Know when to ask for help and what to bring.
</details>

## Trade-offs and decisions

### 9. (M) How do you decide between building and buying a component?

<details>
<summary>Model answer</summary>

Compare total cost over time, not only the first build. Consider how central it is to your product, the cost of operating and maintaining it, vendor lock-in and exit cost, security and compliance, team skills, delivery time, and fit with your requirements. Buy or adopt for commodity needs. Build where it differentiates you or where no option fits.

**Strong answers:**
- Mention the exit plan and the cost of integration.
- Record the decision and its assumptions.

**Red flags:** Chooses by preference or novelty.
</details>

### 10. (S) How do you decide how much analysis a decision deserves?

<details>
<summary>Model answer</summary>

Match effort to reversibility and impact. Easy-to-reverse choices should be made quickly and revisited with evidence. Hard-to-reverse or high-impact choices, such as a data model, a public API or a vendor commitment, deserve written options and review. Waiting for perfect information is itself a cost.

**Strong answers:**
- Describe a decision record, with context, options, choice and consequences.
- Mention setting a time limit and a trigger for revisiting.
</details>

## Requirements and estimation

### 11. (M) How would you estimate the storage needed for a feature that stores user events?

<details>
<summary>Model answer</summary>

State assumptions explicitly: number of users, events per user per day, average size per event, retention period, and overhead such as indexes and replication. Multiply: for example, users times events per day times event size times retention days, then apply a multiplier for replicas and indexes. Present a range and note which assumption is least certain, and verify it with a small measurement.

**Strong answers:**
- Label numbers as assumptions.
- Sanity check against a known system.

**Red flags:** Gives a single exact figure with no assumptions.
</details>

### 12. (S) A project estimate is "three weeks". How do you test whether that is credible?

<details>
<summary>Model answer</summary>

Ask for the breakdown and the assumptions. Identify unknowns, such as unfamiliar technology, dependencies on other teams, unclear requirements, integration and testing. Compare with similar past work. Ask for a range and a confidence level. Reduce the biggest unknown early, for example with a small spike. Include time for review, deployment, monitoring and documentation.

**Strong answers:**
- Treat an estimate as a statement of uncertainty, not a promise.
</details>

## AI-era verification

### 13. (M) A colleague submits a large change that an assistant largely generated. What do you check beyond whether it passes tests?

<details>
<summary>Model answer</summary>

Check that it solves the actual requirement, not a nearby one. Look for assumptions the tool made, missing error handling, security issues, unbounded resources and unnecessary complexity. Confirm tests test behavior and do not just mirror the implementation. Ask the author to explain the design. If they cannot, the change is not ready.

**Strong answers:**
- Hold generated code to the same standard as any other.
- Remember accountability stays with the author.
</details>

## Scoring and follow-up

Mark each question as *clear*, *partial* or *gap*, then reread the matching topic:

| Questions | Topic |
| --- | --- |
| 1 to 2 | [The Engineering Mindset](./engineering-mindset.md) |
| 3 to 5 | [How Computers Work](./how-computers-work.md) |
| 6 to 8 | [Problem Solving and Debugging](./problem-solving-and-debugging.md) |
| 9 to 10 | [Trade-offs and Decision Making](./trade-offs-and-decision-making.md) |
| 11 to 12 | [Requirements and Estimation](./requirements-and-estimation.md) |
| 13 | [The Engineering Mindset](./engineering-mindset.md) |

Then check yourself with the [Self-Assessment](./self-assessment.md). Back to the [section index](./README.md).
