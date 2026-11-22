# Cheat Sheet: Choosing a Database

> Pick the storage model from your access patterns, consistency needs and operational capacity, not from popularity.

## Use when

You are starting a service, adding a new kind of data, or reconsidering a store that no longer fits. Do this before writing the schema, because migrating storage engines later is expensive.

## Questions first

Answer these in writing. If you cannot, find out before choosing.

| Question | Why it matters |
| --- | --- |
| What are the top five read and write operations, and how often does each run? | Access patterns decide the model |
| Do you query by key only, or by arbitrary fields, ranges, joins or text? | Flexible queries favor relational and search stores |
| What consistency do you need? Can two users briefly see different values? | Strong consistency narrows options and raises cost |
| Do multiple records need to change atomically? | Multi-record transactions favor relational stores |
| How large will the data and traffic get in one and three years? | Single-node limits and sharding needs |
| What is the read-to-write ratio and the latency target? | Drives indexing, caching and replication |
| How stable is the schema? | Fixed structure favors relational; evolving shapes may suit documents |
| Who will operate it, and what do they already know? | Operational skill is a real constraint |
| What are the retention, backup, recovery and compliance requirements? | Rules out options early |

## Comparison

| Model | Strong at | Costs and limits | Typical fit |
| --- | --- | --- | --- |
| Relational (SQL) | Joins, constraints, transactions, ad hoc queries, mature tooling | Horizontal scaling of writes needs effort; schema changes need care | Business data with relationships and integrity rules |
| Document | Flexible shape, reading a whole aggregate in one fetch | Cross-document joins and transactions are weaker or costlier; duplication | Content, catalogs, profiles with varied attributes |
| Key-value | Very fast lookup by key, simple scaling | No rich queries; you design every access path up front | Sessions, caches, feature flags, counters |
| Wide-column | High write throughput, partition-oriented scale | Query patterns fixed by the partition key; limited joins | Time series, event logs, large partitioned datasets |
| Search engine | Full-text search, relevance, faceting | Usually not the system of record; eventual consistency; reindexing | Search and analytics over text |
| Graph | Traversals over relationships | Smaller ecosystem; less suited to bulk aggregates | Recommendations, network and dependency analysis |
| Object or file storage | Cheap, durable large blobs | No query on contents; latency | Media, backups, exports |

Product capabilities within each category vary, and they change over time. Check the documentation of the specific product for transactions, consistency and limits.

## Decision guide

1. **Start with a relational database** unless a requirement clearly rules it out. It handles most workloads, supports transactions and flexible queries, and its operational practices are well understood.
2. **Need atomic changes across several records, or strict integrity?** Keep relational.
3. **Access is almost entirely by key, with huge scale or very low latency?** Consider key-value, with the design of keys as the main work.
4. **Data is read and written as self-contained aggregates with varying shape, and you rarely join?** Consider a document store.
5. **Constant heavy writes, partitioned by an obvious key such as device or tenant plus time?** Consider wide-column.
6. **Users search text or filter by many facets?** Add a search engine **beside** your main store, fed from it.
7. **Queries are mostly multi-hop relationship traversals?** Consider a graph store, after checking whether recursive queries in a relational database suffice.
8. **Large files?** Use object storage and keep metadata in your main database.

Most systems use more than one store. Name a **system of record** for each kind of data, and treat other copies as derived and rebuildable.

## When the default is wrong

- **You expect scale beyond a single node for writes** soon, with clear partitioning. Plan sharding or choose a store built for it, and accept the loss of some joins.
- **Latency requirements are tight** and data fits in memory. A cache or in-memory store may serve reads, with a durable store behind it.
- **The data is naturally append-only events.** An append-oriented log or time-series store may fit better than updating rows.
- **Your team cannot operate it.** Prefer a managed service or a technology the team knows. A good fit run badly is worse than an average fit run well.
- **Compliance restricts location or deletion.** Check these before choosing.

## Costs people forget

- **Migrations.** Changing models later means rewriting data access and moving data under load.
- **Dual writes.** Keeping two stores in sync can fail halfway. Prefer one writer and a derived copy fed from it.
- **Operations.** Backups, restores you have actually tested, upgrades, monitoring, capacity and failover.
- **Lock-in.** Proprietary query features and APIs are hard to leave.
- **Consistency gaps.** Replicas and caches can serve old data. Decide where that is acceptable.

## Review checklist

- [ ] Access patterns are written down with expected volume.
- [ ] Consistency and transaction needs are explicit.
- [ ] A system of record is named for each data type.
- [ ] Growth was estimated, with a plan for when a single node is not enough.
- [ ] Backup and restore were tested, not assumed.
- [ ] The team can operate and debug the store.
- [ ] Schema evolution and migration approach is defined.
- [ ] Data retention and compliance needs are covered.
- [ ] A smaller, boring option was considered and the reason for rejecting it is recorded.

## Go deeper

- [Database Engineering](../06-database-engineering/README.md)
- [Choosing a Data Structure](../03-data-structures-algorithms/choosing-a-data-structure.md)
- [Distributed Systems](../13-distributed-systems/README.md) for consistency models and replication
- [System Design](../05-system-design/README.md) for scaling and capacity planning

Back to [Cheat Sheets](./README.md).
