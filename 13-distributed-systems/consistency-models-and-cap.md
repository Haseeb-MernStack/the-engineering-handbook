# Consistency Models and CAP

> A consistency model is a contract about what reads may return when data is replicated, and choosing one means deciding which anomalies your application can tolerate in exchange for latency and availability.

## Overview

When data lives on more than one machine, a read can reach a copy that has not yet seen the latest write. The system must decide what to do. It can make the reader wait, fail, or return something stale. The set of rules it follows is its **consistency model**.

The word "consistency" is overloaded. The C in ACID means that a transaction preserves application invariants. The C in CAP means a specific property about a single replicated object. Consistency in "eventual consistency" is about replica convergence. Engineers who mix these meanings make bad decisions. This topic uses the replication meaning throughout.

## Core Concepts

### The spectrum of models

Models range from strong to weak. Stronger models are easier to program against and cost more in latency and availability.

| Model | Informal guarantee |
|---|---|
| Linearizability | Every operation appears to take effect atomically at one instant between its start and end. All clients see the same single order, consistent with real time. |
| Sequential consistency | All clients see operations in one order consistent with each client's own program order, but not necessarily real time. |
| Causal consistency | If one operation could have influenced another, everyone sees them in that order. Concurrent operations may appear in different orders. |
| Read-your-writes | A client always sees its own earlier writes. |
| Monotonic reads | A client never sees data go backward in time. |
| Eventual consistency | If writes stop, all replicas eventually converge to the same value. Says nothing about what you see before that. |

Session guarantees such as read-your-writes and monotonic reads are weaker than linearizability yet remove the most user-visible surprises. They are often the right target.

### Linearizability in one example

```text
Client A:  write(x=1) ............ done
Client B:                 read(x) -> must return 1 (starts after A finished)
```

If B starts after A's write completed and still reads the old value, the system is not linearizable. This matters for locks, leader election, unique constraints and any case where a stale read causes an incorrect decision.

### Transactions and isolation

Isolation levels describe how concurrent transactions on one or more nodes interact. They are a separate axis from replica consistency. A **serializable** database behaves as if transactions ran one at a time. **Strict serializability** adds the real-time ordering of linearizability. Weaker levels such as snapshot isolation permit anomalies like write skew, where two transactions each read overlapping data and write disjoint data, together violating a rule neither violated alone. See [Database Engineering](../06-database-engineering/README.md).

### The CAP theorem

CAP was stated as a conjecture by Eric Brewer and proved by Gilbert and Lynch. In its precise form, it says that if a network partition occurs, a system must choose between **consistency** (here meaning linearizability) and **availability** (every non-failing node answers every request).

Common misreadings:

- "Pick two of three." Partitions are not optional in a network that can fail. The real choice arises only during a partition.
- "A system is either CP or AP." Systems make different choices for different operations, and some behave differently in different configurations.
- "CAP describes everyday behavior." It says nothing about the latency cost of consistency when there is no partition.

### PACELC

Daniel Abadi's PACELC extends the idea: if there is a **P**artition, choose between **A**vailability and **C**onsistency; **E**lse, when running normally, choose between **L**atency and **C**onsistency. This captures the fact that even healthy systems pay for strong guarantees with round trips.

### Quorums

Many replicated stores use quorums. With N replicas, a write waits for W acknowledgments and a read consults R replicas. If R + W > N, every read set overlaps every write set, so a read sees at least one up-to-date copy.

```text
N=3, W=2, R=2  -> overlap guaranteed (2 + 2 > 3)
N=3, W=1, R=1  -> no guarantee; fast, may read stale data
```

Overlap alone does not give linearizability. Sloppy quorums, concurrent writes, and clock-based conflict resolution can still produce anomalies. Treat quorum settings as a tuning tool, not a proof.

### Convergence without coordination

**CRDTs** (conflict-free replicated data types) are data structures designed so that replicas can accept writes independently and merge deterministically. Counters, sets and registers have CRDT forms. They provide availability and convergence, at the cost of limited operations and sometimes unintuitive semantics, such as how a removed item behaves when concurrently re-added.

## Trade-offs

| Choice | Benefit | Cost |
|---|---|---|
| Linearizable reads and writes | Simple reasoning, safe for locks and uniqueness | Higher latency, unavailable on the minority side of a partition |
| Causal or session consistency | Prevents most visible anomalies, scales well | Needs metadata; does not support all invariants |
| Eventual consistency | Highest availability and lowest latency | Application must tolerate stale reads and resolve conflicts |
| Single-leader replication with reads from followers | Easy writes, read scaling | Followers lag; read-your-writes needs routing logic |
| Multi-leader or leaderless | Write availability across regions | Conflicts become the application's problem |

The cost of weaker consistency is not abstract. It is the code you must write to handle conflicts, the bugs from anomalies you did not anticipate, and the support tickets when users see their own changes disappear.

The cost of stronger consistency is real as well. In a geographically distributed deployment, a linearizable write must wait for communication between regions. That adds latency governed by distance, which no software can remove.

## Failure Modes

**Stale read used for a decision.** A service reads a balance or inventory count from a lagging replica and approves an action that the true state would forbid. Detect by comparing decisions with the primary's state in audits. Prevent by sending decision-critical reads to the leader or using a linearizable read.

**Vanishing writes.** A user submits a form, is redirected, and the next page reads from a replica that has not received the write. The data looks lost. Prevent with read-your-writes: route a session's reads to the leader for a short time after a write, or carry a version token.

**Lost update under last-write-wins.** Two clients update the same record concurrently. One update is silently dropped. Prevent with version checks (compare-and-set), merge functions, or CRDTs.

**Write skew.** Two on-call doctors each check that another doctor is on call, then each goes off call. Both transactions were valid in isolation. Prevent with serializable isolation or explicit locking of the checked rows.

**Surprising partition behavior.** A system advertised as consistent returns errors during a partition, or one advertised as available accepts conflicting writes. Test partitions deliberately and read the database's documented behavior, including its known anomalies. Independent testing efforts that probe databases under faults have repeatedly found gaps between marketing claims and real behavior.

**Misconfigured quorum.** Replica settings that allow R + W <= N are chosen for speed and then relied on for correctness.

## In Practice

### Developer vs. engineer

The developer accepts the database's defaults and finds anomalies in production. The engineer states, per data item, what staleness or conflict is acceptable, picks a model to match, and verifies that the configuration delivers it.

### Per-feature decision checklist

For each piece of state, answer:

1. What is the worst thing that happens if a reader sees a value that is a few seconds old?
2. What is the worst thing that happens if two concurrent writers both succeed?
3. Must this operation continue to work during a partition or a region outage?
4. Is there an invariant (uniqueness, non-negative balance) that must never break? If yes, which component enforces it, and is that component linearizable?

### Worked example: an online shop

| Data | Suitable guarantee | Reason |
|---|---|---|
| Product description | Eventual | Stale text for a few seconds is harmless |
| Shopping cart | Session consistency, mergeable | Users must see their own changes; concurrent edits can merge |
| Inventory at checkout | Linearizable or serializable | Overselling has a direct cost |
| Payment status | Linearizable on a single owner | Double charges and missed charges are unacceptable |
| Recently viewed items | Eventual | Losing an entry is harmless |

Different data in the same product use different guarantees. That is normal and desirable.

### Compare-and-set pattern

```sql
UPDATE accounts
SET balance = balance - 50, version = version + 1
WHERE id = 42 AND version = 7;
-- If 0 rows are affected, someone else changed it; reload and retry.
```

### Heuristics

- Start with the strongest model whose cost you can afford, and relax it where measurements justify it.
- Name anomalies explicitly in design reviews: stale read, lost update, write skew.
- Verify behavior with fault-injection tests rather than relying on documentation alone.
- Isolate the small set of data that needs strong guarantees and keep it small.

Compare with [Consensus](./consensus.md), which is the mechanism behind many linearizable systems.

## Further Reading

- Seth Gilbert and Nancy Lynch, "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services".
- Eric Brewer, "CAP Twelve Years Later: How the Rules Have Changed".
- Daniel Abadi, "Consistency Tradeoffs in Modern Distributed Database System Design" (PACELC).
- Maurice Herlihy and Jeannette Wing, "Linearizability: A Correctness Condition for Concurrent Objects".
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter on consistency and consensus.
