# Event-Driven Architecture

> Event-driven architecture structures a system around facts that have happened, so components can react independently instead of calling each other directly; it buys decoupling and resilience at the price of harder reasoning and debugging.

## Overview

In a request-driven design, component A calls component B and waits. A must know B exists, where it is, and what it expects. If B is slow or down, A is affected.

In an event-driven design, A announces that something happened: `OrderPlaced`, `PaymentCaptured`, `ProfileUpdated`. Zero or more other components react. A does not know who they are. New reactions can be added without changing A.

This is a real structural benefit. It is also a real source of trouble. Control flow is no longer visible in one place. Ordering, duplicates, and partial failure become your problem. Many teams adopt events for the decoupling and discover the costs in production.

An engineer treats the choice as a trade-off, not a style. Use events where independence and asynchrony are required. Use plain calls where a caller needs an answer now.

## Core Concepts

### Events, commands, and queries

These are different things and should be named differently.

| Kind | Meaning | Tense | Expected receivers |
|---|---|---|---|
| Event | A fact that already happened | Past: `OrderPlaced` | Any number, including none |
| Command | A request to do something | Imperative: `CapturePayment` | Exactly one handler |
| Query | A request for information | Question: `GetOrder` | One responder |

Events cannot be rejected. The receiver may ignore them or fail to process them, but the fact stays true. Commands can be refused. Mixing the two, for example publishing an "event" named `SendEmail`, produces hidden coupling: the publisher expects a specific reaction.

### Brokers and delivery

A broker (queue, log, or topic system) sits between publishers and consumers. Two broad models exist:

- **Queue**: each message goes to one consumer in a group. Good for distributing work.
- **Log / topic**: messages are appended to an ordered, retained log. Many independent consumer groups read at their own pace and can replay. Good for broadcasting facts.

Most brokers offer at-least-once delivery. Exactly-once delivery between independent systems is not something to assume. Some systems offer exactly-once processing within a bounded scope, with conditions. Read the guarantees precisely.

### Event payload styles

- **Notification**: carries only an identifier. Consumers call back for details. Small, but creates coupling and load on the source.
- **Event-carried state transfer**: carries the data consumers need. Consumers need no callback, but the payload is a published contract that is hard to change.

### Event sourcing and CQRS

These are separate ideas that are often bundled with events.

- **Event sourcing** stores state as the sequence of events, not as current rows. Current state is derived by replaying. You gain a full history and the ability to rebuild views. You pay in complexity: schema evolution of old events, replay time, and unfamiliar tooling.
- **CQRS** separates the model used for writes from the model used for reads. It can exist without event sourcing.

Neither is required to use events for integration. Adopt them only for a specific need, such as an audit-critical domain.

### Idempotency and ordering

Because delivery can repeat, every consumer must be idempotent: processing the same event twice must have the same effect as once. Common techniques:

- Record processed event IDs and skip repeats.
- Use natural idempotent operations (set a value, not increment it).
- Use conditional writes with a version number.

Ordering is usually guaranteed only within a partition or key. Design so that events that must be ordered share a key, for example all events for one order ID. Do not assume global ordering.

### The dual-write problem and the outbox

A service often must update its database and publish an event. Doing both separately can fail halfway: the row is saved but the event is lost, or the reverse.

The transactional outbox pattern fixes this. Write the event to an `outbox` table in the same database transaction as the state change. A separate relay reads the outbox and publishes to the broker, marking rows as sent.

```sql
BEGIN;
UPDATE orders SET status = 'PLACED' WHERE id = 42;
INSERT INTO outbox (id, aggregate_id, type, payload, created_at)
VALUES (gen_id(), 42, 'OrderPlaced', '{"orderId":42}', now());
COMMIT;
```

The relay may publish a row twice if it crashes after publishing but before marking. That is acceptable because consumers are idempotent.

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Async events over sync calls | Producer survives consumer outages; independent scaling | Eventual consistency; no immediate result; harder debugging |
| Many consumers per event | Add features without touching the producer | Nobody sees the whole flow; impact analysis is hard |
| Log with replay | Rebuild state, add new consumers later | Retention cost; old event formats must stay readable |
| Rich event payloads | Consumers are self-sufficient | Payload becomes a contract; changes break consumers |
| Event sourcing | Complete history, auditability | Steep learning curve; costly schema changes; querying needs projections |
| Choreography (reactions) | No central coordinator | Process logic scattered; cycles and gaps are easy to create |
| Orchestration (coordinator) | Process visible in one place | Coordinator is a central component and can become a bottleneck for change |

### Choreography versus orchestration

In choreography, each service reacts to events and emits new ones. In orchestration, one component tells others what to do and tracks progress. Choreography suits short, simple flows. For long multi-step processes with compensation (a saga), orchestration is usually easier to understand, test, and operate. This is a judgment call, and teams differ.

## Failure Modes

**Duplicate processing.** A consumer crashes after applying an effect but before acknowledging. The broker redelivers. Without idempotency, a customer is charged twice. Prevent with idempotency keys. Detect with reconciliation jobs that compare effects against source events.

**Lost events.** Dual writes without an outbox drop events when the process dies between steps. Detect by comparing counts between source tables and published streams.

**Poison messages.** One malformed message fails repeatedly and blocks a partition or burns retries. Use a bounded retry count, then move the message to a dead-letter queue with alerting. Dead-letter queues that nobody reads are just slow data loss; assign an owner.

**Out-of-order arrival.** `OrderCancelled` is processed before `OrderPlaced`. The consumer creates a cancelled order or errors. Use per-key ordering, version numbers, and logic that tolerates gaps.

**Consumer lag.** A slow consumer falls behind. The system looks healthy but data is stale for minutes or hours. Monitor lag (age of the oldest unprocessed message) and alert on it, not just on error rates.

**Retry storms.** Failing consumers retry aggressively and overload a dependency that is already struggling. Use exponential backoff with jitter and a circuit breaker.

**Schema breakage.** A producer renames a field and consumers silently misread it. Treat event schemas as versioned contracts. Prefer additive changes. Run compatibility checks in CI.

**Invisible flows.** After a year, nobody knows which services react to `UserDeleted`. A compliance deletion is missed in one. Keep a catalog of events, owners, and consumers. Propagate a correlation ID through every hop so one business action can be traced.

## In Practice

### When events fit

- Several independent parties need to react to the same fact.
- The producer must not be slowed or broken by consumers.
- Work is naturally asynchronous (email, indexing, analytics, fan-out).
- You need to rebuild derived data or add new consumers later.

### When to prefer a direct call

- The caller needs the result to continue.
- The flow is simple, with one producer and one consumer.
- Strong consistency is required across the step.
- The team lacks the tooling to trace and operate asynchronous flows.

### Design checklist

- [ ] Events are named in the past tense and describe business facts.
- [ ] Every event has a stable ID, timestamp, type, schema version, and correlation ID.
- [ ] Producers use an outbox or equivalent to avoid dual writes.
- [ ] Every consumer is idempotent and tested with duplicate delivery.
- [ ] The ordering key is chosen and documented.
- [ ] Retries are bounded; a dead-letter path exists with an owner and an alert.
- [ ] Lag is measured and has an alert threshold.
- [ ] Schema changes are checked for compatibility before release.
- [ ] There is a runbook for replaying events safely.

### Worked example: order confirmation

Placing an order must reserve stock, capture payment, and email the customer.

1. The order service writes the order and an `OrderPlaced` outbox row in one transaction.
2. Inventory and payment react. The email service waits for `PaymentCaptured`.
3. If payment fails, `PaymentFailed` is published and inventory releases the reservation.

The email step is a good fit for events: it can be delayed without harm. The payment step is more delicate, because the customer waits for an answer. Many teams keep payment authorization as a synchronous call and use events only for what follows.

### Developer vs. engineer

A developer adds a message queue because it feels scalable and tests the happy path. An engineer asks what happens on duplicates, reordering, a stuck consumer, and a schema change, and builds the outbox, idempotency, lag alert, and replay procedure before launch.

### Verifying AI-generated designs

Generated event-driven code often shows a publisher and a subscriber and nothing more. Check for:

- Whether the publish and the database write are atomic.
- Idempotent handlers; look for plain inserts and increments.
- Acknowledgement placement: ack after processing, not before.
- Retry limits and a dead-letter path.
- Ordering assumptions that the broker does not actually provide.
- Whether a synchronous call was replaced by events where the caller needs the answer.

## Further Reading

- Gregor Hohpe and Bobby Woolf, *Enterprise Integration Patterns*.
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapters on stream processing and messaging.
- Chris Richardson, *Microservices Patterns*, sections on sagas and the transactional outbox.
- Martin Fowler, "What do you mean by 'Event-Driven'?" (martinfowler.com).
- Vaughn Vernon, *Implementing Domain-Driven Design*, chapter on domain events.
