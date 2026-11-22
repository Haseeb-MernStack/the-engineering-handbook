# Self-Assessment: Programming Concepts

> Check what you can do today, not what you have read. Each item describes observable behavior. Be strict: if you could not do it in front of a colleague without looking it up, leave it unchecked.

## How to use this page

1. Work through one topic at a time. Check an item only if you can do it **unprompted** and **explain why**.
2. Count the checked items for each topic and use the score guide at the end.
3. Revisit in three to six months. The goal is a shrinking list of unchecked items, not a perfect score.

## Programming Paradigms

- [ ] I can describe imperative, object-oriented, functional and event-driven styles, and name a problem each suits.
- [ ] I can explain the difference between composition and inheritance, and why composition is often preferred.
- [ ] I can write a pure function and explain what makes it easier to test.
- [ ] I choose a style to fit the problem and the team, not out of habit.

## Type Systems

- [ ] I can explain static versus dynamic typing, and strong versus weak typing, without mixing them up.
- [ ] I use types to make invalid states hard to represent.
- [ ] I validate external input at the boundary, since types do not check runtime data by themselves.
- [ ] I can explain what generics provide and where they add complexity.
- [ ] I can explain how a null or missing value causes failures, and how my language helps prevent it.

## Memory Management

- [ ] I can say where a value lives, stack or heap, and when it is released.
- [ ] I can explain how garbage collection works at a high level and what causes pauses.
- [ ] I can describe how a memory leak arises in a garbage-collected language.
- [ ] I can use a profiler or heap snapshot to find what holds memory.
- [ ] I can explain ownership or reference counting in at least one language.

## Concurrency and Parallelism

- [ ] I can explain the difference between concurrency and parallelism.
- [ ] I can describe a race condition and show how I would fix one.
- [ ] I can describe how a deadlock forms and how to avoid it.
- [ ] I can name the shared mutable state in a concurrent design and how it is protected.
- [ ] I bound concurrency, using limits on threads, pools and queues, instead of letting it grow with load.
- [ ] I know what blocking a thread or an event loop costs.

## Error Handling

- [ ] I can separate expected failures from bugs and handle each differently.
- [ ] I set explicit timeouts on every call that leaves the process.
- [ ] I retry only operations that are safe to retry, with backoff and a limit.
- [ ] I do not swallow errors, and I add enough context for someone to diagnose them.
- [ ] I can describe graceful degradation for a failing dependency.

## Clean Code and Refactoring

- [ ] I name things for what they mean, not how they are implemented.
- [ ] I can recognize common smells, such as long functions, duplicated logic and unclear dependencies.
- [ ] I refactor in small steps with tests passing between steps.
- [ ] I can tell duplication that should be removed from duplication that is coincidence.
- [ ] I can explain why simpler code is not always shorter code.

## Reviewing generated code

- [ ] I check generated code for swallowed errors, missing timeouts and unbounded growth.
- [ ] I check that types and validation match the real data, not the example in the prompt.
- [ ] I look for hidden shared state and blocking calls in async code.

## Score guide

Count the checked items **per topic**.

| Checked in the topic | Where you are | What to do |
| --- | --- | --- |
| All or all but one | Solid | Move on. Test yourself with the [question bank](./question-bank.md). |
| About half | Partial | Reread the topic, then write a small program that exercises each unchecked item. |
| Fewer than half | Gap | Start with the topic's **Overview** and **Core Concepts**, then work through the **Failure Modes** section. |

## Related

- Practice questions: [Question Bank](./question-bank.md)
- Back to the [section index](./README.md)
