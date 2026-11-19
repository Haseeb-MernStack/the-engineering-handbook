# Time, Clocks, and Ordering

> Machines in a distributed system cannot agree on what time it is, so correct systems order events by causality or by explicit agreement instead of trusting wall-clock timestamps.

## Overview

Programmers treat time as a shared fact. Logs are sorted by timestamp. "Last write wins" compares two timestamps. A lock expires after thirty seconds. Each of those habits assumes that every machine reads the same clock. They do not.

Every machine has its own oscillator, and each one runs slightly fast or slow. Clocks are corrected over the network, and that correction is itself subject to delay and error. Two events that happened in a given order in real time may carry timestamps in the opposite order. A system that trusts those timestamps will silently lose data.

This topic explains which clocks exist, what each is good for, and how to establish order when physical time cannot be trusted.

## Core Concepts

### Two kinds of physical clock

- **Time-of-day (wall) clock.** Returns a calendar time. It can jump forward or backward when synchronized, when a leap second is applied, or when an administrator changes it. It is suitable for showing a time to a human.
- **Monotonic clock.** Counts elapsed time from an arbitrary starting point and never goes backward. It is suitable for measuring durations and timeouts on a single machine. Its values are meaningless across machines.

A common bug is measuring an elapsed duration with the wall clock. If the clock steps backward during the measurement, the "duration" is negative or wildly wrong.

### Clock synchronization and its limits

Protocols such as NTP estimate the offset between a local clock and a reference. The accuracy depends on network conditions and can degrade without any local sign of trouble. Some environments use dedicated hardware such as GPS receivers or atomic references to get tighter bounds. Even then, the correct description is not "the time is T" but "the time is within an interval of T plus or minus an error bound".

### Happens-before

Lamport observed that what matters is often not the absolute time but the order of cause and effect. The relation "A happens before B" is defined by three rules.

1. If A and B occur on the same process and A occurs first, A happens before B.
2. If A is the sending of a message and B is its receipt, A happens before B.
3. The relation is transitive.

Two events with no such chain between them are **concurrent**. Concurrent does not mean simultaneous. It means neither could have influenced the other.

### Logical clocks

A **Lamport clock** is a counter on each process.

- Increment before each local event.
- Attach the counter to every outgoing message.
- On receipt, set the local counter to the larger of its own value and the message's value, then add one.

This guarantees that if A happens before B, then A's timestamp is smaller than B's. The reverse does not hold: a smaller timestamp does not prove causality. Lamport clocks give a consistent ordering, not a way to detect concurrency.

```python
class LamportClock:
    def __init__(self):
        self.t = 0

    def tick(self):            # local event or send
        self.t += 1
        return self.t

    def receive(self, remote_t):
        self.t = max(self.t, remote_t) + 1
        return self.t
```

### Vector clocks

A **vector clock** keeps one counter per process. Comparing two vectors tells you whether one event happened before the other or whether they are concurrent.

```text
A = [2, 1, 0]   B = [2, 2, 0]   -> A before B (every entry <= , one <)
A = [2, 1, 0]   C = [1, 3, 0]   -> concurrent (neither dominates)
```

Version vectors apply the same idea to replicas of data. When two versions are concurrent, the system has detected a real conflict and must resolve it. Cost: the vector grows with the number of participants, so practical systems prune it or use variants.

### Hybrid logical clocks

A hybrid logical clock combines a physical timestamp with a logical counter. It stays close to wall-clock time, which is useful for humans and for range queries, while preserving causal ordering. Several distributed databases use this approach.

### Bounded-uncertainty clocks

Some systems expose time as an interval with a known error bound and wait out the uncertainty before committing. The idea is to pay latency, equal to the width of the uncertainty window, in exchange for a real-time ordering guarantee. It requires careful infrastructure and is not something to improvise.

### Total order versus partial order

Causality gives a **partial order**: some pairs of events are ordered, others are concurrent. Many applications need a **total order**, where every pair is ordered, for example a replicated log. A total order can be produced by agreement among nodes, which is the job of [Consensus](./consensus.md), or by a single sequencer that assigns numbers.

### Leases and fencing

A **lease** is a lock that expires. It relies on clocks, which is dangerous. A client may acquire a lease, then pause (a long garbage collection, a stalled virtual machine), then resume and believe it still holds the lease after it has expired and been granted to someone else. The standard defense is a **fencing token**: a number that increases with every grant. The protected resource rejects any request carrying a token older than one it has already seen.

```text
Client A gets lease, token 33.   A pauses.
Lease expires. Client B gets lease, token 34.  B writes with 34.
A wakes up, writes with 33.  Storage rejects it: 33 < 34.
```

## Trade-offs

| Approach | What it gives you | What it costs |
|---|---|---|
| Wall-clock timestamps | Human-readable, cheap, no coordination | Skew can reorder events and lose writes |
| Lamport clocks | Order consistent with causality | Cannot detect concurrency |
| Vector clocks | Detect concurrency and conflicts | Metadata grows with participants; app must resolve conflicts |
| Hybrid logical clocks | Causality plus near-physical time | More complex; still needs sync for tight bounds |
| Single sequencer | Simple total order | Throughput limit and a single point of failure unless replicated |
| Consensus-based log | Fault-tolerant total order | Extra round trips and operational complexity |
| Bounded-uncertainty clocks | Strong real-time ordering | Specialized infrastructure; added commit latency |

"Last write wins" deserves its own note. It is simple and converges, but it decides the winner by timestamp and therefore discards one of two concurrent writes without telling anyone. That is acceptable for a cache or a "last seen" field. It is dangerous for anything where each write carries meaning.

## Failure Modes

**Lost updates from clock skew.** Node A's clock runs ahead by a few hundred milliseconds. Its older write carries a newer timestamp and overwrites a genuinely newer write from node B. No error is raised. Detect by comparing replica contents against an authoritative log; prevent by not using wall-clock comparison for conflict resolution of important data.

**Unreliable log ordering.** Logs from many hosts merged by timestamp show an effect before its cause. Engineers waste hours chasing a bug that is only skew. Mitigate with trace and request identifiers and causal span relationships rather than relying on sorted timestamps alone.

**Expiry bugs.** Tokens, certificates, caches and leases that depend on a time comparison misbehave when clocks differ. A certificate "not yet valid" on a host with a slow clock is a classic outage.

**Backward time steps.** A synchronization step moves the clock back. Timers fire twice, durations go negative, and "ensure monotonically increasing ids" invariants break. Use monotonic clocks for durations.

**Process pauses.** A paused process resumes with stale beliefs about leadership or leases. Use fencing tokens and re-check state after any long operation.

**Leap seconds and time-zone handling.** Store instants in UTC; convert only at the edges. Be explicit about how your platform treats leap seconds.

## In Practice

### Developer vs. engineer

The developer sorts by `created_at`. The engineer asks which clock produced the value, how far apart those clocks can be, and whether a wrong order would be harmless or harmful.

### Decision checklist

- Am I measuring a duration on one machine? Use a monotonic clock.
- Am I showing a time to a person? Use the wall clock, stored as UTC.
- Do I need to know if one event could have caused another? Use causal tracking (vector or version vectors).
- Do I need one agreed order of operations? Use a sequencer or a consensus-based log.
- Am I granting exclusive access based on a timeout? Add a fencing token.
- Am I resolving conflicting writes by timestamp? Decide, in writing, that losing one write is acceptable.

### Worked example: ordering comments

A comment service runs on three replicas. Two users reply to the same comment at nearly the same moment, hitting different replicas.

- Ordering by wall-clock time can place the replies in a different order on different replicas if clocks differ.
- Ordering by a per-thread sequence number assigned by one leader gives one order everywhere, at the cost of routing writes for a thread to its leader.
- Ordering by causality (each reply records the ID of its parent) preserves the logical structure and tolerates concurrent siblings, which are displayed in any stable tie-breaking order.

For a comment thread the third option is usually enough. For a bank ledger it is not, and a single ordered log is required.

### Operational habits

- Monitor clock offset on every host and alert on drift.
- Include a request ID in every log line and propagate it across calls.
- Avoid putting a wall-clock comparison in any code path whose failure would corrupt data.

See also [Consistency Models and CAP](./consistency-models-and-cap.md) for how ordering guarantees appear to applications.

## Further Reading

- Leslie Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System".
- Colin Fidge and Friedemann Mattern, the original papers on vector clocks.
- Sandeep Kulkarni and colleagues, "Logical Physical Clocks" (hybrid logical clocks).
- Corbett et al., "Spanner: Google's Globally-Distributed Database" (bounded-uncertainty time).
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter on the trouble with distributed systems.
