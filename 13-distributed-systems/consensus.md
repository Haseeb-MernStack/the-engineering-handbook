# Consensus

> Consensus is how a group of unreliable machines agrees on a single value or sequence of values, and it is the foundation under leader election, locks, and strongly consistent storage.

## Overview

Several machines must often act as one. They must agree on who the leader is, which transaction committed, or what order operations happened in. Any node may crash. Messages may be delayed, dropped, or reordered. Yet the group must not disagree, and it must keep making progress.

This is the consensus problem. A correct consensus protocol gives three guarantees:

- **Agreement:** no two nodes decide different values.
- **Validity:** the decided value was proposed by some node.
- **Termination:** every non-faulty node eventually decides.

A foundational result, usually called FLP, shows that in a fully asynchronous system no deterministic protocol can guarantee all three if even one node may crash. Real protocols sidestep this. They guarantee agreement and validity always, and they guarantee termination only when the network is behaving well enough for a while. Safety never depends on timing. Liveness does.

Most engineers never implement consensus. You will use it through a coordination service, a replicated database, or a log. You still need to understand it, because its costs and failure behavior leak into your system: write latency, availability during partitions, and what "the cluster has no leader" means at 3 a.m.

This topic builds on [Consistency Models and CAP](./consistency-models-and-cap.md) and [Time, Clocks, and Ordering](./time-clocks-and-ordering.md).

## Core Concepts

### Quorums and majorities

Consensus protocols rely on quorums. A quorum is a subset of nodes large enough that any two quorums overlap. With `N = 2f + 1` nodes, a majority is `f + 1`. Any two majorities share at least one node. That shared node carries knowledge from one decision to the next, so the group cannot split into two independent decisions.

The cluster tolerates `f` crash failures. Five nodes tolerate two failures. Three tolerate one. Four nodes tolerate only one, which is why even-sized clusters add cost without adding tolerance.

| Nodes | Majority | Crash failures tolerated |
|-------|----------|--------------------------|
| 3     | 2        | 1                        |
| 4     | 3        | 1                        |
| 5     | 3        | 2                        |
| 7     | 4        | 3                        |

### Leaders, terms, and epochs

Most practical protocols, including Paxos variants and Raft, elect a leader. The leader orders client requests and replicates them. Leadership is scoped by a monotonically increasing number, called a term, epoch, or ballot. A node that sees a higher term steps down. Messages with stale terms are rejected.

The term is what makes a deposed leader harmless. A leader that was partitioned away may still believe it leads. Once it contacts a majority, or any node with a higher term, it learns it is stale.

### The replicated log

The usual way to apply consensus is to agree on an ordered log of commands. Every node applies the same commands in the same order to a deterministic state machine. All replicas then end in the same state. This is state machine replication.

```text
client -> leader: append(cmd)
leader -> followers: AppendEntries(term, index, cmd)
followers -> leader: ack
leader: once a majority has stored the entry, mark it committed
leader -> client: result (after commit and apply)
```

An entry is committed once a majority has durably stored it. A committed entry is never lost, even if the leader crashes, because any future leader must be elected by a majority that overlaps the one that stored it.

### Raft in brief

Raft was designed to be understandable. It splits the problem into three parts:

1. **Leader election.** A follower that hears no heartbeat before a randomized timeout becomes a candidate, increments the term, and requests votes. A node grants at most one vote per term, and only to a candidate whose log is at least as up to date as its own. A majority of votes wins.
2. **Log replication.** The leader sends entries to followers and tracks which are stored on a majority.
3. **Safety.** The election restriction guarantees that a new leader already holds every committed entry.

Paxos solves the same problem with a different structure. Single-decree Paxos agrees on one value through a prepare phase and an accept phase. Multi-Paxos chains instances to build a log. Both rely on the same quorum-overlap idea.

### Fault models

Most production consensus assumes crash-stop or crash-recovery faults. Nodes fail by stopping, perhaps restarting with stable storage intact. They do not lie. Tolerating arbitrary or malicious behavior needs Byzantine fault tolerant protocols, which require more nodes (`3f + 1`) and more message rounds. They are used where participants do not trust each other, such as some blockchain systems. Inside one organization's datacenter, crash-fault protocols are the norm.

### Where consensus shows up

- Coordination services that store small, critical metadata: configuration, leader leases, service membership.
- Replicated databases that need linearizable writes within a shard.
- Distributed transaction commit across shards.
- Cluster managers that track which node owns which work.

## Trade-offs

**Consistency against availability.** A consensus group stays available only while a majority can talk to each other. In a partition, the minority side cannot make progress. This is the CAP trade-off made concrete. You gain linearizable behavior. You lose writes on the minority side.

**Latency.** Every committed write needs at least one round trip from the leader to a majority, plus durable writes on each. Across regions, that round trip is tens or hundreds of milliseconds. Placing replicas across distant regions buys survivability and costs latency on every write.

**Throughput.** A single leader orders all writes, so it is a bottleneck. Systems scale by running many independent consensus groups, one per shard. That adds complexity: cross-shard operations need an extra coordination layer.

**Cluster size.** More nodes tolerate more failures but make each commit wait on more acknowledgments and more network traffic. Three or five nodes are typical. Larger groups are rare for the consensus core.

**Build against buy.** Writing your own consensus is expensive and error prone. Using an established library or service costs you a dependency and some operational burden, and limits how you can tune behavior. For nearly all teams, buying is correct.

## Failure Modes

### Split brain from bad leases

A leader that assumes it still leads after its lease expired can act alongside a new leader. Both write. Data diverges. This happens when lease checks use local clocks and a process pause, such as a long garbage collection or a stalled VM, makes the leader wake up believing it still has time.

Prevention: use fencing tokens. Each lock or leadership grant carries a monotonically increasing number. The storage layer rejects any write carrying a lower number than one it has seen.

```python
def write(resource, data, token):
    if token < resource.highest_token_seen:
        raise StaleLeaderError()
    resource.highest_token_seen = token
    resource.apply(data)
```

### Leader flapping

When timeouts are too tight for real network jitter, followers repeatedly suspect a healthy leader and start elections. Each election pauses writes. Throughput collapses while the cluster is technically healthy. Detect it by alerting on term changes per hour. Fix it by tuning election timeouts above realistic tail latency, and by fixing the slow disk or overloaded node that is causing missed heartbeats.

### Slow disks

Consensus requires durable writes before acknowledging. A slow fsync on a leader or on a majority of followers stalls the whole group. Monitor commit latency and disk sync duration. Give consensus nodes fast, dedicated storage.

### Loss of quorum

Losing a majority halts writes. Causes include correlated failure (all replicas in one rack or zone), a bad rolling upgrade that takes down two of three nodes, or a membership change done carelessly. Spread nodes across failure domains. Change membership one node at a time using the protocol's supported procedure. Rehearse recovery.

### Treating the coordination service as a database

Coordination services are built for small data and modest write rates. Storing large values or high-volume state in them overloads the leader and endangers every system that depends on them. Keep payloads small and keep application data elsewhere.

## In Practice

### Decision checklist

- Do you need agreement on a single order or a single owner? If not, consider weaker models that avoid consensus entirely.
- Can a managed or existing coordination service do the job? Prefer it to writing protocol code.
- What is the behavior when a majority is unreachable? Is halting acceptable?
- Are replicas spread across independent failure domains?
- Do all lock holders use fencing tokens against the protected resource?
- Do you alert on leader changes, commit latency, and quorum health?

### Worked example: a job scheduler with one active runner

Requirement: exactly one worker runs the nightly billing job.

A naive approach has each worker check a timestamp row and claim it if stale. Two workers can claim at once, and a paused worker can resume and run again.

A safer design acquires a lease in a coordination service. The lease returns a fencing token. The worker passes the token to the billing database with each write. The database rejects stale tokens. Even if two workers briefly believe they hold the lease, only the one with the newest token can write. You have moved the correctness guarantee from "the lease is exclusive" to "stale writers are rejected", which holds regardless of timing.

### Developer vs. engineer

A developer sees a distributed lock library and uses it to make a job run once. An engineer asks what happens when the lock holder pauses for thirty seconds, what the protected resource does with a stale writer, and what the job does when the coordination service is unreachable. They then write down the decision: fail closed, fail open, or degrade.

### Heuristics

- Prefer idempotent operations so that duplicated execution is harmless. Then you need consensus less often.
- Use consensus for small control-plane data, not bulk data-plane traffic.
- Test with injected partitions and paused processes, not only with killed processes.

## Further Reading

- Diego Ongaro and John Ousterhout, "In Search of an Understandable Consensus Algorithm" (the Raft paper).
- Leslie Lamport, "Paxos Made Simple".
- Fischer, Lynch, and Paterson, "Impossibility of Distributed Consensus with One Faulty Process".
- Tushar Chandra, Robert Griesemer, and Joshua Redstein, "Paxos Made Live".
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter on consistency and consensus.
- Martin Kleppmann, "How to do distributed locking" (essay on fencing tokens).
