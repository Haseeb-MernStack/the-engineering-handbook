# Messaging and Idempotency

> Asynchronous messaging decouples services in time and load, but it delivers messages at least once, so correctness depends on consumers that tolerate duplicates.

## Overview

Services can talk synchronously, waiting for a reply, or asynchronously, by handing a message to a broker or log and moving on. Messaging brings real benefits. The sender does not need the receiver to be up. Bursts are absorbed by a queue. Work can be spread across consumers. New consumers can be added without changing the producer.

It also moves complexity. A message can be lost, delivered twice, delivered late, or delivered out of order. The sender cannot always know which happened. You cannot wish this away. You design for it, and the main tool is idempotency: making repeated processing of the same message harmless.

## Core Concepts

### Queues and logs

| Model | Behavior | Typical use |
|-------|----------|-------------|
| Queue | Each message goes to one consumer in a group and is removed after acknowledgment | Task distribution, work offloading |
| Publish/subscribe | Each message goes to every subscriber | Notifications, fan-out |
| Replicated log | Messages are appended and retained; consumers track their own position | Event streaming, replay, rebuilding state |

A log lets a consumer re-read history. That supports recovery, new consumers, and reprocessing after a bug fix. A traditional queue deletes messages once handled.

### Delivery semantics

- **At most once.** Send and do not retry. No duplicates. Messages may be lost.
- **At least once.** Retry until acknowledged. No loss, but duplicates occur.
- **Exactly once.** Each message has exactly one effect.

True exactly-once delivery over an unreliable network is not achievable in the general case. Two Generals style arguments show a sender cannot distinguish "the message was lost" from "the acknowledgment was lost". What systems offer is **effectively once processing**: at-least-once delivery combined with deduplication or idempotent effects. Some platforms provide transactional features that make this work inside their own boundary. Those guarantees usually stop at the boundary. Writing to an external database or calling an external API reopens the problem.

### Idempotency

An operation is idempotent if applying it many times has the same effect as applying it once.

- Naturally idempotent: `SET balance_status = 'closed'`, deleting a row by key, putting an object at a fixed path.
- Not idempotent: `balance = balance + 10`, appending a row, sending an email, charging a card.

Make non-idempotent operations safe with an **idempotency key**: a unique identifier for the logical operation, created by the caller and reused on every retry.

```python
def handle_payment(msg, db):
    key = msg["idempotency_key"]
    with db.transaction() as tx:
        if tx.exists("processed", key):
            return  # duplicate, already applied
        tx.insert("processed", key)
        tx.execute(
            "UPDATE accounts SET balance = balance - %s WHERE id = %s",
            (msg["amount"], msg["account_id"]),
        )
```

The deduplication record and the effect must commit atomically. If the key is recorded but the effect fails, the operation is lost. If the effect happens but the key is not recorded, a retry repeats it.

For external calls, pass the idempotency key to the provider if it supports one. Otherwise record intent first, then act, then record completion, and design a reconciliation process for the ambiguous middle state.

### Ordering

Brokers typically guarantee order only within a partition or a single queue with one consumer. To keep related events in order, route them by a key, such as an account ID, to the same partition. Global ordering is expensive and rarely needed. Consumers must also handle retries, which can reorder processing within a stream.

Where order matters, put a sequence number or version in the message. The consumer can ignore events older than state it already holds.

### Acknowledgment and redelivery

A consumer receives a message, processes it, then acknowledges. If it crashes before acknowledging, the broker redelivers. Acknowledging before processing risks loss. Acknowledging after processing risks duplicates. Choose the second and make processing idempotent.

### Dead-letter queues

A message that repeatedly fails is a poison message. Retrying forever blocks progress. After a bounded number of attempts, move it to a dead-letter queue for inspection, with the error and original metadata attached.

### The outbox pattern

A service often must update its database and publish an event. Doing both without a shared transaction means one can succeed and the other fail. The outbox pattern writes the event to a table in the same database transaction as the state change. A separate relay reads the outbox and publishes. Delivery is at least once, so consumers deduplicate.

```text
BEGIN
  UPDATE orders SET status = 'paid' WHERE id = 42
  INSERT INTO outbox (id, topic, payload) VALUES (..., 'order.paid', {...})
COMMIT
-- relay: read unpublished outbox rows, publish, mark as published
```

### Sagas

Multi-service workflows cannot rely on one ACID transaction. A saga is a sequence of local transactions, each with a compensating action that undoes it if a later step fails. Compensation is a business-level reversal, such as a refund, not a rollback. It can itself fail, so it must be idempotent and retryable.

### Backpressure

If producers outpace consumers, queues grow without bound. Bound queue size or age, monitor consumer lag, and apply backpressure: slow producers, shed load, or scale consumers.

## Trade-offs

**Sync against async.** Async improves resilience and smooths load. It makes flow harder to trace, delays errors, and complicates user feedback, since the caller no longer gets an immediate result.

**Queue against log.** Logs enable replay and multiple consumers and require retention and position management. Queues are simpler for work distribution and give up history.

**Strict ordering against throughput.** A single ordered stream limits parallelism. Partitioning by key preserves order per entity and allows scale, while giving up global order.

**Deduplication window.** Keeping every key forever is costly. Keeping them for a limited time bounds storage, but a duplicate arriving after expiry is processed twice. Pick a window longer than the maximum realistic redelivery delay.

**Choreography against orchestration.** Services reacting to each other's events are loosely coupled and hard to follow end to end. A central orchestrator is easier to reason about and becomes a coupling point.

**Retries.** Retries recover from transient faults and amplify load during outages. See [Failure Modes and Resilience Patterns](./failure-modes-and-resilience-patterns.md).

## Failure Modes

### Duplicate side effects

A consumer sends an email or charges a card, crashes before acknowledging, and does it again on redelivery. Prevent with idempotency keys. Detect by monitoring duplicate rates and by reconciling against downstream records.

### Lost messages

A producer publishes before the database commit, then the transaction rolls back, or the reverse. Events and state disagree. The outbox pattern closes this gap. Reconcile periodically by comparing source of truth with derived state.

### Poison messages blocking a partition

One malformed message fails forever and halts a partition. Cap retries, use dead-letter queues, and alert on their depth. Validate messages against a schema at the producer and the consumer.

### Unbounded consumer lag

A slow downstream makes lag grow. Data staleness grows, then storage limits are hit. Alert on lag in time, not only message count. Provide a plan to scale consumers or shed work.

### Out-of-order effects

An "update" is processed before its "create", or an old update overwrites a new one. Carry versions, partition by entity, and make handlers tolerate arrival of events whose prerequisites are missing, by parking or retrying them.

### Retry storms

All consumers retry immediately after an outage, hitting a recovering service with a spike. Use exponential backoff with jitter and cap total attempts.

### Schema drift

Producers change message formats and consumers break. Treat messages as public contracts. Evolve them compatibly: add optional fields, avoid changing meaning, and version when you must.

### Replay surprises

Replaying a log to rebuild state re-triggers side effects such as emails. Separate pure state building from effectful actions, or mark replays so effectful handlers skip them.

## In Practice

### Checklist for a new consumer

- Is the handler idempotent? How is that enforced?
- Is the dedup record committed atomically with the effect?
- What is the retry limit, backoff policy, and dead-letter destination?
- What ordering does it need, and what is the partition key?
- Which schema does it accept, and how are incompatible messages handled?
- Which metrics exist: throughput, lag, error rate, dead-letter depth, processing time?
- What happens on replay?

### Worked example: order confirmation email

An order service emits `order.paid`. A notification consumer sends a confirmation email.

Naive handler: receive, send email, acknowledge. A crash between send and acknowledge causes a second email.

Improved handler: derive a key from `order_id` and `template`. Insert the key into a `sent_notifications` table with a unique constraint. If the insert succeeds, send. If the email provider accepts the key as an idempotency token, pass it. If the process dies after insert but before sending, the next attempt sees the key exists and skips, so the email is never sent. To avoid that, store a status: `pending` then `sent`. A retry that finds `pending` re-sends, accepting a possible duplicate in a rare crash window. This is a deliberate choice: for an email, a rare duplicate is better than a lost confirmation. For a payment it would be the opposite. The right behavior is a business decision, not a library setting.

### Developer vs. engineer

A developer enables the broker's "exactly once" option and considers the problem solved. An engineer reads what the guarantee covers, notes it ends at the broker boundary, and makes the consumer idempotent anyway.

## Further Reading

- Gregor Hohpe and Bobby Woolf, *Enterprise Integration Patterns*.
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapters on stream processing and the end-to-end argument.
- Chris Richardson, *Microservices Patterns* (outbox and saga patterns).
- Hector Garcia-Molina and Kenneth Salem, "Sagas".
- Pat Helland, "Life beyond Distributed Transactions: an Apostate's Opinion".
- Jay Kreps, "The Log: What every software engineer should know about real-time data's unifying abstraction".
