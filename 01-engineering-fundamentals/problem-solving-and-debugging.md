# Problem Solving and Debugging

> Debugging is applied scientific method: form a hypothesis, test it, and let evidence, not intuition, decide what is wrong.

## Overview

Most of an engineer's time goes to understanding existing behavior, not writing new code. Something is slow, wrong, or flaky, and you must find out why. The skill is not knowing every answer. It is having a reliable process for reaching answers you can trust.

Weak debugging looks like random changes: edit something, run again, hope. It sometimes works and teaches nothing. Strong debugging looks like investigation. You reproduce the problem, narrow where it lives, form a specific explanation, test the explanation, and fix the cause. You then confirm the fix and add protection against recurrence.

The same process applies to broader problem solving. Before you can solve a problem you must state it precisely. Many "hard" problems turn out to be unclear ones.

This topic gives a method for both, along with the traps that make smart people waste days.

## Core Concepts

### Define the problem

State three things:

- **Expected behavior**: what should happen.
- **Actual behavior**: what does happen, with exact output, errors, and timing.
- **The gap**: the smallest description of the difference.

"The app is broken" is not a problem statement. "Uploading a file over 5 MB returns a 500 error, and the log shows a timeout from the storage client" is.

### The debugging loop

```
Observe -> Reproduce -> Narrow -> Hypothesize -> Test -> Fix -> Verify -> Prevent
   ^                                                 |
   +--------------- hypothesis refuted --------------+
```

1. **Observe.** Collect facts: error messages, stack traces, logs, metrics, recent changes, who is affected.
2. **Reproduce.** Find the steps that trigger the problem reliably. A reproducible bug is half solved. If you cannot reproduce it, gather more data from production instead of guessing.
3. **Narrow.** Shrink the search space. Which component? Which input? Which version? Which time?
4. **Hypothesize.** Write a specific, falsifiable claim: "The timeout happens because the client's default timeout is 3 seconds and large uploads take longer."
5. **Test.** Design the cheapest experiment that could prove the claim false. Run it.
6. **Fix.** Change the cause, not the symptom.
7. **Verify.** Confirm the original reproduction now passes and nothing else broke.
8. **Prevent.** Add a test, a check, an alert, or a note that makes this class of bug easier to catch next time.

### Narrowing techniques

- **Binary search the history.** If it worked in version A and fails in version C, test B. Version control tools can automate this by bisecting commits.
- **Binary search the code.** Disable half the pipeline, or log values at the midpoint, and see which half is faulty.
- **Minimize the input.** Remove parts of the failing input until the smallest case remains. A tiny failing case is easier to reason about than a large one.
- **Compare working and failing.** What differs between a request that works and one that fails: data, environment, timing, user, configuration?
- **Change one variable at a time.** Otherwise you cannot tell which change mattered.

### Reading evidence

- **Stack traces** show where the failure surfaced, not always where it started. Read from the failure point upward, and ask how the bad value got there.
- **Logs** record what the code chose to say. Absence of a log line is also information. Check timestamps and correlation identifiers.
- **Metrics** show shape over time: when did it start, does it correlate with a deploy, load, or a dependency?
- **Traces** follow a request across services and show where time goes.
- **Core dumps and heap profiles** show state at a moment, valuable for crashes and memory growth.

### Tools and methods

| Tool | Best for | Limitation |
|---|---|---|
| Print/log statements | Quick insight, production-safe if structured | Requires code change; can change timing |
| Interactive debugger | Inspecting state step by step | Hard in production; poor for timing and concurrency bugs |
| Profiler | CPU and allocation hotspots | Sampling overhead; shows where, not why |
| Tracing | Latency across services | Needs instrumentation in advance |
| Test case | Pinning a bug so it cannot return | Only as good as the reproduction |
| Rubber duck explanation | Exposing wrong assumptions | Needs you to be honest about what you do not know |

### Classes of bugs

- **Deterministic bugs** fail the same way every time. Reproduce and step through them.
- **Intermittent bugs** fail sometimes. Suspect concurrency, timing, caches, retries, uninitialized state, or environmental differences. Increase the logging, then look for what differs when it fails.
- **Heisenbugs** change when you observe them, usually because the observation shifts timing. Prefer low-impact recording over breakpoints.
- **Data-dependent bugs** appear only for certain inputs. Look at the data that fails.
- **Environment bugs** occur only in one place. Compare versions, configuration, permissions, and resource limits.
- **Scale bugs** appear only under volume. Reproduce with load or realistic data.

### Problem solving beyond bugs

When the problem is "how do we build this?" the same discipline applies:

- **Restate it** in your own words. Check it with the person who asked.
- **Break it down** into smaller problems you can solve independently.
- **Solve a simpler version** first. Remove constraints, then add them back.
- **Look for a known pattern.** Many problems are variants of ones already solved. See [Choosing a Data Structure](../03-data-structures-algorithms/choosing-a-data-structure.md) for one example.
- **Work an example by hand** before writing code.
- **Check your solution** against edge cases: empty, one element, maximum size, duplicates, invalid input.

## Trade-offs

- **Depth vs. time.** Finding the root cause takes longer than patching the symptom. In an active incident, restoring service comes first (roll back, disable a flag), and root-cause work follows. Do both, in that order.
- **Print statements vs. debugger.** Prints are fast to add and work anywhere but need edits and redeploys. Debuggers give rich state but may not be available in production and can disturb timing.
- **Observability up front vs. later.** Instrumenting in advance costs effort and runtime overhead. It pays back during the first hard incident. Adding it only after the incident means flying blind when it matters.
- **Asking for help early vs. late.** Asking too soon skips learning. Asking too late wastes time and can leave users affected. A useful rule: after a fixed time box of focused effort with no progress, bring in another person, and bring your notes.
- **Fixing the bug vs. fixing the class.** A narrow fix is quick. Removing the class of bug (a type, an invariant, a lint rule) costs more and prevents many future bugs.
- **Reproduction fidelity.** A faithful reproduction is slow and expensive to build. A simplified one is fast and may miss the real cause. Start simple. Return to fidelity if the simple case does not fail.

## Failure Modes

**Confirmation bias.** You decide on a cause and look only for evidence that supports it. Counter it by actively trying to disprove your hypothesis. Ask: what would I see if I were wrong?

**Shotgun debugging.** Changing several things at once. If the bug disappears you do not know why, and it may return. Change one thing at a time and keep notes.

**Fixing the symptom.** Adding a null check where the value should never be null hides a deeper fault. Ask where the bad value came from.

**Assuming the error message is the cause.** The message reports a consequence. A "connection refused" may come from an exhausted pool, a deploy in progress, or a firewall change. Trace upstream.

**Blaming the environment, library, or compiler first.** Occasionally correct, rarely the case. Verify your own code and assumptions before accusing the tools. When you do suspect a library, produce a minimal reproduction to prove it.

**Not reproducing before fixing.** Without a reproduction you cannot know whether the fix worked. You may close the ticket and the bug survives.

**Heroic marathon sessions.** Fatigue degrades judgment. Step away. Explain the problem to someone. Fresh perspective often finds in minutes what hours of staring missed.

**Skipping the write-up.** The team repeats the same mistake. After a significant bug, record what happened, why, how it was found, and what changed.

**Debugging in production carelessly.** Attaching tools or changing state in a live system can make things worse. Prefer read-only investigation, and know how to roll back.

**Race conditions "fixed" with sleeps.** Adding a delay hides a timing bug on your machine and fails elsewhere. Fix the synchronization instead; see [Concurrency and Parallelism](../02-programming-concepts/concurrency-and-parallelism.md).

## In Practice

### A worked example

Symptom: an export job succeeds for most customers but fails for some with `MemoryError`.

1. **Observe.** Failures are from three customers. The job was changed last week to load all rows before writing the file.
2. **Narrow.** Those customers have the largest datasets. Compare row counts: the failing ones are far larger than the rest.
3. **Hypothesize.** "Loading all rows into a list exceeds the memory limit for large customers."
4. **Test.** Run the job for one failing customer with memory metrics on. Memory grows linearly with rows until the process is killed. The hypothesis survives the test.
5. **Fix.** Stream rows in batches and write incrementally.

```python
def export(rows_iter, writer, batch_size=1000):
    batch = []
    for row in rows_iter:
        batch.append(row)
        if len(batch) >= batch_size:
            writer.write_rows(batch)
            batch.clear()
    if batch:
        writer.write_rows(batch)
```

6. **Verify.** The failing customers' exports now complete, with flat memory use.
7. **Prevent.** Add a test with a large synthetic dataset, and a memory alert on the job.

Note what did not happen: nobody just raised the memory limit.

### A debugging checklist

- Can I state expected versus actual behavior precisely?
- Can I reproduce it? If not, what data would let me?
- What changed recently: code, data, config, traffic, dependencies?
- Does it fail for everyone or a subset? What do the failures share?
- What is my current hypothesis, and what result would disprove it?
- Am I changing one thing at a time?
- After the fix, did I verify with the original reproduction?
- What test, alert, or note prevents a repeat?

### Keep a log

Write down what you tried and what you saw. It stops you repeating tests, helps you spot patterns, and gives a teammate the context to help. It becomes the seed of the postmortem.

### Developer vs. engineer

| Situation | Developer who writes code | Engineer who owns outcomes |
|---|---|---|
| Intermittent failure | Reruns until it passes | Looks for what differs when it fails and treats it as a real defect |
| Error in logs | Suppresses the noisy line | Finds what raises it and decides whether it signals a real fault |
| Bug fixed | Closes the ticket | Adds a regression test and checks the fix in production |
| Stuck | Tries more changes | Re-reads evidence, shrinks the case, asks for a second view with notes |

### When AI generates code or helps debug

AI can be a fast second reader, suggesting hypotheses and explaining stack traces. Treat its answers as hypotheses, not findings.

- Give it the exact error, relevant code, and what you have already ruled out.
- Ask for several possible causes and a way to test each.
- Do not apply a suggested fix you cannot explain. A fix that makes the error disappear may hide it.
- Be alert to invented APIs, flags, or function names. Check them against official documentation.
- Generated code is also a source of bugs. When it fails, apply the same loop: reproduce, narrow, test.
- Make sure any generated test would fail without the fix. A test that cannot fail proves nothing.

## Further Reading

- *Debugging: The 9 Indispensable Rules for Finding Even the Most Elusive Software and Hardware Problems*, David J. Agans.
- *Why Programs Fail: A Guide to Systematic Debugging*, Andreas Zeller.
- *How to Solve It*, George Polya. A classic on structured problem solving.
- *The Pragmatic Programmer*, David Thomas and Andrew Hunt. See the material on debugging and tracer bullets.
- *Release It!*, Michael T. Nygard. How systems fail in production and how to find out why.
- *Observability Engineering*, Charity Majors, Liz Fong-Jones, and George Miranda.

Back to the section index: [Engineering Fundamentals](./README.md)
