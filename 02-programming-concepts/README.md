# Programming Concepts

## Overview

Syntax is the easy part of a language. The lasting part is the set of concepts beneath it: how programs are organized, how values are classified, where data lives, how multiple tasks interact, what happens when things fail and how to keep code changeable. These concepts transfer across languages, and they explain most of the bugs you will meet in production.

An engineer uses them to predict behavior rather than discover it by accident. When generated code arrives, the same concepts are your checklist for whether it is correct, safe and maintainable. Fluency in them is what lets you verify output you did not write.

## What you will learn

- How paradigms shape code, and how to choose among them.
- What type systems guarantee, and where those guarantees end.
- How memory is allocated and reclaimed, and how leaks and pauses arise.
- How to reason about concurrency, shared state and parallel work.
- How to design for failure instead of treating errors as noise.
- How to write code others can read, and change it safely.

## Topics

| Topic | What it covers |
| --- | --- |
| [Programming Paradigms](./programming-paradigms.md) | Imperative, object-oriented, functional, declarative and event-driven styles and when each fits. |
| [Type Systems](./type-systems.md) | Static and dynamic typing, inference, generics, sum types and modeling the domain. |
| [Memory Management](./memory-management.md) | Stack and heap, manual management, garbage collection, ownership and diagnosing leaks. |
| [Concurrency and Parallelism](./concurrency-and-parallelism.md) | Threads, async, shared state, races, deadlocks and bounded concurrency. |
| [Error Handling](./error-handling.md) | Failure kinds, mechanisms, retries, timeouts and graceful degradation. |
| [Clean Code and Refactoring](./clean-code-and-refactoring.md) | Naming, structure, smells and safe, incremental refactoring. |

## Check yourself

| Resource | Use it to |
| --- | --- |
| [Self-Assessment](./self-assessment.md) | Find gaps with a checklist of observable skills. |
| [Question Bank](./question-bank.md) | Practice interview and design-review questions, with model answers and red flags. |

## Prerequisites

- Comfort writing small programs in at least one language.
- The material in [Engineering Fundamentals](../01-engineering-fundamentals/README.md), especially [How Computers Work](../01-engineering-fundamentals/how-computers-work.md).

## What good looks like

- You choose a programming style to fit the problem and the team, not by habit.
- You validate external data at the boundary and use types to make invalid states hard to build.
- You can say where a value lives and when its memory is released.
- You bound every queue, cache, pool and retry, and set explicit timeouts.
- You can name the shared state in a concurrent design and the mechanism that protects it.
- You treat error paths as features, with tests that trigger them.
- You separate refactoring from behavior changes and refactor under test.
- You review generated code for hidden mutation, swallowed errors and unbounded resource use.

Back to [The Engineering Handbook](../README.md).
