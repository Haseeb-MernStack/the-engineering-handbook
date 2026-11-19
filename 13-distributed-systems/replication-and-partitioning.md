# Replication and Partitioning

> Replication copies data to several nodes for durability and availability; partitioning splits data across nodes for scale; most real systems need both and pay for each.

## Overview

A single machine has limits. It can fail. It can hold only so much data. It can serve only so many requests. Two techniques address this.

**Replication** keeps copies of the same data on multiple nodes. It improves durability, availability, and read capacity, and it can place data near users.

**Partitioning**, also called sharding, splits the dataset so that each node holds only a part. It improves capacity and write throughput.

They are orthogonal. A typical large system partitions data into shards and replicates each shard. The difficulty is not the idea. It is keeping copies consistent, deciding where data lives, and surviving the moments when either mapping changes.

Note that "partition" here means splitting data. It is unrelated to a network partition, though the two interact.

## Core Concepts

### Replication topologies

| Approach | Writes go to | Strengths | Weaknesses |
|----------|--------------|-----------|------------|
| Single leader | One leader, copied to followers | Simple, no write conflicts | Leader is a write bottleneck and failover point |
| Multi-leader | Several leaders, often one per region | Local writes in each region, tolerates region isolation | Write conflicts must be resolved |
| Leaderless | Any replica, using quorums | No failover step, high availability | Weaker guarantees, repair needed |

### Synchronous and asynchronous replication

With synchronous replication, the leader waits for followers to confirm before acknowledging a write. No acknowledged write is lost if the leader dies. A slow or dead follower stalls writes.

With asynchronous replication, the leader acknowledges immediately. Writes are fast and the leader is independent of followers. If the leader fails before followers catch up, acknowledged writes are lost.

Many systems choose a middle path: wait for one or a few followers, replicate to the rest asynchronously.

### Replication lag and its anomalies

Asynchronous followers trail the leader. Reading from a follower exposes users to anomalies:

- **Read-your-writes violation.** A user saves a profile, reloads, and sees the old version.
- **Non-monotonic reads.** A user sees newer data, then older data, as requests hit different replicas.
- **Violated causality.** A reply is visible before the question it answers.

Mitigations include routing a user's reads to the leader shortly after they write, pinning a session to one replica, and tracking a version or log position the client has already seen so replicas can wait until they reach it. See [Consistency Models and CAP](./consistency-models-and-cap.md).

### Failover

When a leader fails, the system must detect it, choose a new leader, and redirect clients. Each step has hazards:

- Detection uses timeouts. Too short causes false failovers. Too long extends downtime.
- A new leader that lacked the old leader's last writes may drop them or conflict with them when the old leader returns.
- The old leader may still think it leads. Without fencing, two leaders accept writes.

### Quorum reads and writes

In leaderless systems with `n` replicas, a write goes to `w` nodes and a read queries `r` nodes. If `w + r > n`, read and write sets overlap, so a read sees at least one up-to-date copy. This is helpful but not a guarantee of linearizability. Sloppy quorums, concurrent writes, and failed partial writes can still produce stale reads.

### Conflict resolution

When multiple nodes accept writes, conflicting updates occur. Options:

- **Last write wins** by timestamp. Simple, but silently discards data and depends on clocks. See [Time, Clocks, and Ordering](./time-clocks-and-ordering.md).
- **Version vectors** to detect concurrency, then merge by application logic or ask the user.
- **CRDTs**, data types whose merge operation is commutative, associative, and idempotent, so replicas converge without coordination.

### Partitioning strategies

**Range partitioning** assigns contiguous key ranges to shards. Range scans are efficient. Sequential keys, such as timestamps, concentrate load on the newest shard.

**Hash partitioning** hashes the key to choose a shard. Load spreads evenly. Range queries must touch every shard.

**Consistent hashing** places nodes and keys on a ring so that adding or removing a node moves only a fraction of keys. Naive `hash(key) % N` reshuffles almost everything when `N` changes.

**Fixed number of partitions.** Create many more partitions than nodes. Move whole partitions between nodes when scaling. This is a common and operationally simple design.

```python
import hashlib

def shard_for(key: str, num_partitions: int) -> int:
    digest = hashlib.sha256(key.encode()).digest()
    return int.from_bytes(digest[:8], "big") % num_partitions
```

The partition-to-node mapping is then a separate, small, replicated table.

### Secondary indexes

Queries by a non-partition key are hard. A **local index** stays with each shard. Writes are cheap, but reads must query all shards. A **global index** is itself partitioned by the indexed value. Reads are targeted, but one write may update several shards, usually asynchronously.

### Rebalancing and routing

Rebalancing moves data when nodes join or leave. It consumes network and disk, and competes with live traffic. Clients must find the right node: through a routing tier, through any-node forwarding, or through a client holding a cached partition map that must tolerate staleness.

## Trade-offs

**Replication factor.** More copies improve durability and read capacity but multiply storage and write cost. Three is a common baseline. Fewer is cheaper and riskier.

**Sync against async.** Synchronous gives durability and costs latency and availability. Asynchronous gives speed and costs possible data loss on failover.

**Read from followers.** It scales reads and lowers load on the leader. It exposes stale and non-monotonic reads that the application must tolerate or handle.

**Single against multi-leader.** Multi-leader improves regional write latency and isolation. It adds conflict resolution, which is hard to get right and hard to debug.

**Hash against range.** Hashing avoids hot ranges and loses efficient scans. Range keeps scans and invites hotspots.

**Number of shards.** More shards give flexibility and parallelism. They add metadata, connection overhead, and cross-shard complexity. Cross-shard transactions and joins become expensive or unsupported.

## Failure Modes

### Hot partitions

Skewed access concentrates load on one shard. A celebrity user, a tenant much larger than others, or a monotonically increasing key can saturate one node while the rest idle. Detect with per-shard load metrics, not cluster averages. Mitigate by choosing better keys, adding a salt or suffix to spread hot keys, splitting hot ranges, or caching.

### Data loss on failover

Asynchronous replication plus leader failure loses the unreplicated tail. Prevent by requiring some synchronous acknowledgment for important data. Detect by measuring replication lag and by reconciling after every failover.

### Split brain

Two nodes both act as leader after a partition or a premature failover. Use quorum-based leader election and fencing tokens. See [Consensus](./consensus.md).

### Replication lag spikes

A long transaction, a bulk load, or a slow follower causes lag to grow. Users read stale data, and failover becomes unsafe. Alert on lag in both time and bytes. Throttle bulk operations. Decide in advance which reads may tolerate lag.

### Rebalancing storms

Adding a node triggers heavy data movement that starves live traffic, causing latency spikes that look like an outage. Throttle rebalancing, schedule it off-peak, and add nodes before capacity is critical rather than during an emergency.

### Resharding too late

A system that assumed one database discovers it needs shards when it is already saturated. Retrofitting a partition key into queries, schemas, and unique constraints is slow and risky. Choose a partition key early, and keep tenant or entity identifiers in every table so later splitting is possible.

### Cross-shard invariants

A uniqueness constraint or foreign key spanning shards cannot be enforced locally. Duplicates appear. Either route those operations through one authoritative shard, use a coordination step, or accept and reconcile.

## In Practice

### Choosing a partition key

Check each candidate against these questions:

- Does most traffic include this key, so queries touch one shard?
- Is the distribution even, now and as data grows?
- Will it create a hot spot on recent data?
- Is it stable? Changing a record's key means moving it.
- Do the operations you most need to keep atomic share the key?

### Worked example: multi-tenant SaaS

A product stores per-tenant data. Tenant ID is a natural key: queries are tenant-scoped, and tenants rarely interact. The team hashes tenant ID into many fixed partitions and maps partitions to nodes.

Problem: one enterprise tenant is far larger than the rest. Its partition is overloaded. Options include moving that tenant to a dedicated shard, which a directory table makes possible, or sub-partitioning that tenant by a secondary key. The directory table, a lookup from tenant to shard, was cheap to add early and is the reason this fix is easy.

### Developer vs. engineer

A developer adds a read replica and points reporting queries at it. An engineer asks how far behind it can be, what happens when users read their own writes from it, what the failover plan is, and which queries must never use it.

### Operational checklist

- Replication lag is graphed and alerted.
- Failover has been rehearsed, including the return of the old leader.
- Backups exist separately from replicas. Replication copies deletes and corruption too.
- Per-shard load and size are visible.
- The partition map is replicated, versioned, and cached with a clear staleness policy.
- Rebalancing can be throttled.

## Further Reading

- Martin Kleppmann, *Designing Data-Intensive Applications*, chapters on replication and partitioning.
- Giuseppe DeCandia et al., "Dynamo: Amazon's Highly Available Key-value Store".
- David Karger et al., "Consistent Hashing and Random Trees".
- Marc Shapiro et al., "Conflict-free Replicated Data Types".
- Fay Chang et al., "Bigtable: A Distributed Storage System for Structured Data".
