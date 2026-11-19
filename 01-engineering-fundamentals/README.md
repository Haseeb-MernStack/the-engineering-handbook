# Engineering Fundamentals

## Overview

Fundamentals are the ideas that stay true when languages, frameworks and tools change. This section covers how a computer actually executes your code, how an engineer thinks about ownership, how to find the cause of a failure, how to choose between imperfect options, and how to decide what to build and how long it will take.

These topics matter because they separate writing code from owning outcomes. A developer can make a feature run. An engineer can explain why it runs, predict how it will fail, and defend the choices behind it. That skill matters more as code generation becomes cheap, because the scarce work is judgment: framing the problem, verifying the result and accepting responsibility for it.

## What you will learn

- What the CPU, memory, storage and network do, and what each costs.
- The habits that define an engineer who owns outcomes.
- A repeatable method for debugging, instead of guessing.
- How to make, record and revisit decisions under uncertainty.
- How to turn a vague request into requirements and an honest estimate.

## Topics

| Topic | What it covers |
| --- | --- |
| [The Engineering Mindset](./engineering-mindset.md) | Ownership, outcomes, systems thinking and healthy skepticism. |
| [How Computers Work](./how-computers-work.md) | CPU, memory hierarchy, storage, operating systems and networks as the base of every program. |
| [Problem Solving and Debugging](./problem-solving-and-debugging.md) | Hypothesis-driven debugging, isolating causes and preventing recurrence. |
| [Trade-offs and Decision Making](./trade-offs-and-decision-making.md) | Comparing options, recording decisions and handling reversibility. |
| [Requirements and Estimation](./requirements-and-estimation.md) | Clarifying what to build, expressing uncertainty and sizing work. |

## Prerequisites

None. Basic familiarity with writing a small program in any language helps.

## What good looks like

- You ask what outcome a task serves before you start it.
- You can describe, at a rough level, what the machine does when your code runs.
- You form a hypothesis and test it before changing code while debugging.
- You can reproduce a bug reliably before attempting a fix.
- You write down the options you rejected and why.
- You state estimates as ranges with assumptions, not single dates.
- You verify generated or borrowed code instead of trusting it.
- You add monitoring and tests so a fixed problem stays fixed.

Back to [The Engineering Handbook](../README.md).
