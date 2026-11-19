# Queues and Background Jobs

> A queue decouples the work a request triggers from the request itself, trading simplicity for resilience, throughput and the ability to absorb bursts.

## Overview

Not everything belongs inside a request. Sending email, resizing images, calling a slow partner API, generating a report and fanning out notifications all take time, can fail independently and do not need to finish before the user gets a response. A queue lets the request record the intent to do the work and return. A separate worker performs it later.

This changes the shape of the system. A synchronous call has one outcome the caller sees immediately. A queued job has an outcome nobody is watching. It can fail silently, run twice, run late, run out of order or never run. The engineering work is making those outcomes visible and safe.

A developer wires up a queue library and sees the job run once on a laptop. An engineer asks what happens when the worker crashes mid-job, when the same message is delivered twice, when the queue grows faster than it drains, and who finds out.

## Core Concepts

### Producers, brokers, consumers

A producer enqueues a message. A broker stores it durably. A consumer pulls or receives it, does the work and acknowledges. If the consumer never acknowledges, the broker assumes failure and makes the message available again.

```
 request --> [producer] --> [ queue / broker ] --> [worker] --> side effect
                                  |                    |
                                  +-- redelivery <-----+  (no ack)
```

### Delivery guarantees

| Guarantee | Meaning | Cost |
|---|---|---|
| At most once | Message may be lost, never duplicated | Silent data loss |
| At least once | Message is never lost, may be duplicated | Consumers must tolerate duplicates |
| Exactly once | Each message has one effect | Only achievable end to end by combining at-least-once delivery with idempotent handling |

Most brokers offer at-least-once delivery. Treat "exactly once" claims with care. They usually hold inside one system's boundary, not across your database and a third-party API.

### Idempotency

A handler is idempotent if running it twice has the same effect as running it once. This is the central design property of job processing. Techniques:

- Use a natural or generated idempotency key per job and record completed keys.
- Prefer set-style operations ("set status to shipped") over increments.
- Pass the key to downstream APIs that support it.
- Make the "check, act, record" sequence atomic, for example inside one database transaction.

```ts
async function handleChargeJob(job: { id: string; orderId: string }) {
  await db.tx(async (tx) => {
    const done = await tx.oneOrNone(
      "SELECT 1 FROM processed_jobs WHERE job_id = $1", [job.id]);
    if (done) return; // duplicate delivery, already handled
    await chargeCustomer(job.orderId, { idempotencyKey: job.id });
    await tx.none("INSERT INTO processed_jobs(job_id) VALUES ($1)", [job.id]);
  });
}
```

Note the limit of this example: the external charge is not inside the transaction. The idempotency key passed to the payment provider is what makes a crash between the charge and the insert safe.

### Visibility timeouts, acknowledgements and retries

While a worker holds a message it is hidden from others for a visibility timeout (or lease). If the work takes longer than the timeout, the message reappears and a second worker starts it. Set the timeout above realistic job duration, or extend the lease while working.

Retries should use backoff and a limit. See [Service Resilience](./service-resilience.md).

### Dead-letter queues

A dead-letter queue (DLQ) receives messages that exceeded their retry limit or are unprocessable. It turns an endless retry loop into a visible, inspectable pile. A DLQ nobody watches is only a slower way to lose data.

### The transactional outbox

A common bug: update the database, then publish a message. If the process dies between the two, the state changed but nothing was announced. The reverse order publishes events about changes that never committed.

The outbox pattern writes the message to an `outbox` table in the same transaction as the state change. A separate relay reads the table and publishes. Delivery is then at least once, and consumers must be idempotent.

### Ordering and partitioning

Global ordering does not scale. Most systems guarantee order only within a partition, shard or message group. Pick a key (for example the account id) so that messages that must stay ordered share it. Understand that retries and concurrency can still reorder work unless the broker says otherwise.

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Async instead of sync | Faster responses, burst absorption, isolated failures | Eventual consistency, harder debugging, no immediate result |
| Queue vs. log-style broker | Queues are simple work distribution. Logs allow replay and multiple independent readers | Logs need partition and offset management. Simple queues cannot easily replay |
| Many small jobs | Fine-grained retry, parallelism | Overhead per job, more load on the broker |
| Few large jobs | Less overhead | Long runtimes, expensive retries, lease problems |
| Strict ordering | Simpler consumer logic | Lower parallelism, head-of-line blocking |
| Database-backed job table | One fewer system, transactional enqueue | Polling load, contention at high volume |
| Dedicated broker | Throughput, purpose-built features | Another system to run, secure and monitor |

A queue also moves failure rather than removing it. The user no longer sees an error, so you must build the channel through which they eventually learn the outcome: a status endpoint, a notification, or a retry button.

## Failure Modes

- **Poison message.** One malformed message crashes the consumer on every attempt, blocking a partition or burning retries. Detect with repeated redelivery counts. Prevent with validation on enqueue, bounded retries and a DLQ.
- **Duplicate processing.** A lease expires mid-job or an ack is lost. The customer is charged twice. Prevent with idempotency, not with hope.
- **Unbounded backlog.** Producers outpace consumers. Latency grows silently, then memory or disk fills. Detect with queue depth and age of the oldest message. Prevent with autoscaling on lag, rate limits on producers, or backpressure.
- **Retry storm.** A downstream outage causes every job to retry at once, and recovery is crushed by the herd. Use exponential backoff with jitter and a concurrency cap.
- **Lost enqueue.** The dual-write problem above. Use an outbox.
- **Stuck or zombie workers.** A worker holds leases but makes no progress. Detect with per-job duration metrics and heartbeats.
- **Schema drift.** A producer deploys a new message shape before consumers understand it. Version messages and deploy consumers first.
- **Silent DLQ growth.** Alert on any non-zero DLQ depth for important queues.

## In Practice

### Design checklist for a new job

1. What is the idempotency key, and where is it recorded?
2. What is the maximum safe duration, and is the lease longer?
3. Which errors are retryable and which are permanent?
4. What is the retry schedule and limit?
5. What happens when it lands in the DLQ, and who is alerted?
6. Does ordering matter, and what is the partition key?
7. How does the user or calling system learn the result?
8. What is the maximum acceptable delay, and what alerts when it is exceeded?

### Metrics worth having

Queue depth, age of oldest message, enqueue rate, completion rate, failure rate, retry count, job duration percentiles, DLQ depth. Age of oldest message is usually the best single health signal because it measures what users experience.

### Developer vs. engineer

The developer's job runs once in a demo. The engineer kills the worker mid-job, delivers the message twice, and fills the queue to see what breaks.

### Verifying AI-generated code

Generated consumers often assume exactly-once delivery, retry forever with no backoff, swallow exceptions and acknowledge anyway, or enqueue after commit without an outbox. Check each of these explicitly. Ask the generated code the questions in the checklist above.

## Further Reading

- Martin Kleppmann, *Designing Data-Intensive Applications*, chapters on messaging, stream processing and the end-to-end argument for exactly-once.
- Gregor Hohpe and Bobby Woolf, *Enterprise Integration Patterns*.
- Chris Richardson, *Microservices Patterns*, sections on the transactional outbox and sagas.
- Your broker's official documentation on delivery semantics, visibility timeouts and dead-letter handling.
- J. H. Saltzer, D. P. Reed and D. D. Clark, "End-to-End Arguments in System Design" (1984).
