# Fundamentals and Fallacies

> A distributed system is a set of machines that must cooperate over an unreliable network, and almost every hard problem in the field follows from the fact that parts can fail independently and without announcement.

## Overview

A single-process program has a convenient property: it either works or it crashes. Its function calls return, its memory is shared, and it has one clock. A distributed system gives up all of that. Components run on different machines, talk through a network, and fail on their own schedules. Some of them are slow rather than dead, and from the outside slow and dead look identical.

You are building a distributed system the moment your service calls another service, a database over the network, a queue, or a third-party API. You do not need hundreds of nodes. Two processes and a socket are enough to inherit every problem in this section.

The core theme of this handbook is the difference between a developer who writes code and an engineer who owns outcomes. Distributed systems are where the difference shows up fastest. A developer writes `response = client.charge(card)` and moves on. An engineer asks what happens if the call times out, whether the charge happened, whether retrying will charge twice, and how anyone would find out.

## Core Concepts

### Why distribute at all

Teams distribute systems for a small number of reasons.

- **Scale.** One machine cannot hold the data or serve the load.
- **Availability.** One machine is a single point of failure.
- **Latency.** Users are far from the data center.
- **Organization.** Separate teams need to deploy independently.

Each reason is legitimate. Each also has a price, and the price is paid in complexity that never goes away. If one well-run machine meets your needs, it is usually the better engineering choice.

### Partial failure

The defining property is **partial failure**. Some parts work while others do not, and you often cannot tell which is which. When you send a request and receive no reply, at least four things may have happened.

1. The request never arrived.
2. The request arrived, the work was done, and the reply was lost.
3. The request arrived and the receiver is still working on it.
4. The receiver crashed partway through the work.

A timeout does not tell you which one occurred. It only tells you that you stopped waiting. This is why so much of distributed design is about making operations safe to repeat. See [Messaging and Idempotency](./messaging-and-idempotency.md).

### The fallacies of distributed computing

A group of engineers at Sun Microsystems, with contributions from several people, collected a list of false assumptions that newcomers make. The list is widely known as the eight fallacies of distributed computing.

| Fallacy | What reality looks like |
|---|---|
| The network is reliable | Packets are dropped, reordered, duplicated and delayed. Links flap. Load balancers restart. |
| Latency is zero | Every hop costs time, and the tail is much worse than the median. |
| Bandwidth is infinite | Large payloads, chatty protocols and replication traffic saturate links. |
| The network is secure | Traffic can be observed, altered or spoofed unless you protect it. |
| Topology does not change | Instances move, scale, and are replaced. Addresses go stale. |
| There is one administrator | Many teams and vendors control parts of the path, with different policies. |
| Transport cost is zero | Serialization, copying and network egress consume CPU and money. |
| The network is homogeneous | Different hardware, protocol versions and libraries must interoperate. |

The list is old, and the lesson is not. Teams rediscover each item the hard way, usually during an incident.

### System models

When reasoning about an algorithm, we state what we assume about the environment.

- **Network model.** Synchronous (bounded delay), asynchronous (no bound), or partially synchronous (usually bounded, occasionally not). Real networks are the last kind.
- **Node failure model.** Crash-stop (a node halts and stays halted), crash-recovery (it halts and later returns, possibly with lost memory), or Byzantine (it may behave arbitrarily or maliciously).
- **Timing.** Whether clocks are bounded in how far they drift.

Most production systems assume crash-recovery nodes and a partially synchronous network. Byzantine models matter when participants do not trust each other, as in some public ledgers, and are rarely needed inside one organization's data center.

### Failure detection is guessing

A failure detector decides that a node is dead. In an asynchronous network it cannot be certain. The usual mechanism is a timeout on heartbeats.

- A short timeout detects real failures quickly but raises false alarms during load spikes or garbage collection pauses.
- A long timeout avoids false alarms but leaves users waiting for a dead node.

A false positive is not harmless. If a system declares a healthy node dead and replaces it, two nodes may briefly both believe they are in charge. Preventing that outcome is a central job of [Consensus](./consensus.md).

### Latency, tail latency, and fan-out

A request that touches many backends is as slow as its slowest dependency. If each of 100 calls has a 1 percent chance of being slow, a request that waits for all of them is slow most of the time. Engineers therefore measure percentiles (p95, p99), not averages, and apply techniques such as hedged requests, timeouts, and limiting fan-out.

## Trade-offs

| Choice | Benefit | Cost |
|---|---|---|
| Stay on one node | Simple, fast, easy to reason about | Capacity ceiling, single point of failure |
| Split into services | Independent deployment, scaling and ownership | Network failures, versioning, operational overhead |
| Synchronous calls | Simple control flow, immediate result | Failures and latency propagate to callers |
| Asynchronous messaging | Decoupling, buffering, retries | Eventual outcomes, harder debugging, duplicates |
| Short failure timeouts | Fast recovery | False positives, flapping, unnecessary failover |
| Long failure timeouts | Stability | Slow recovery, long user-visible outages |

There is no setting of these knobs that is correct in general. The right values depend on the cost of being wrong in each direction for your system.

## Failure Modes

**Retry storms.** A dependency slows down. Callers time out and retry. The retries add load to the struggling dependency, which slows it further. Detect by watching request rates to a dependency rise while its success rate falls. Prevent with bounded retries, exponential backoff with jitter, and retry budgets.

**Gray failure.** A node is not down but is degraded: slow disks, packet loss, a saturated thread pool. Health checks that only test "is the port open" report it healthy. Detect with latency and error-rate signals measured from the callers' point of view, not only with local health endpoints.

**Split brain.** A network partition or long pause leads two nodes to both act as leader. Both accept writes. Data diverges. Prevent with quorum-based leadership and fencing tokens.

**Cascading failure.** One component's overload spreads to others through shared resources or synchronous dependencies. Prevent with load shedding, bulkheads, timeouts that are shorter than the caller's own deadline, and circuit breakers. See [Failure Modes and Resilience Patterns](./failure-modes-and-resilience-patterns.md).

**Hidden coupling.** Two services that seem independent share a database, a DNS provider, a certificate, or a clock source. When the shared piece fails, both fail. Detect by drawing a real dependency map and asking what each box needs in order to start.

**Unbounded waiting.** A call with no timeout can hold a thread forever. Enough of them exhaust the pool. Set an explicit timeout on every network call.

## In Practice

### Developer vs. engineer

| The developer asks | The engineer also asks |
|---|---|
| Does the call work in my test? | What happens if it times out, fails halfway, or succeeds twice? |
| Is the code correct? | Is the system correct when parts are slow, absent or stale? |
| How do I make it faster? | What is the p99, and what is the dependency behind it? |
| Can I add a service? | Who operates it, and what is the failure budget? |

### Checklist for any network call

- Is there an explicit timeout, and is it shorter than the caller's own deadline?
- Is the operation safe to retry? If not, what makes it safe (an idempotency key, a version check)?
- What is the retry policy, and does it include backoff and jitter?
- What does the caller do when the dependency is down: fail, degrade, serve stale data?
- Which metrics and traces will show a problem before users report it?

### A small example

Retrying with backoff and jitter, assuming the operation is idempotent:

```python
import random, time

def call_with_retry(fn, attempts=4, base=0.1, cap=2.0):
    for attempt in range(attempts):
        try:
            return fn(timeout=1.0)
        except TransientError:
            if attempt == attempts - 1:
                raise
            delay = min(cap, base * (2 ** attempt))
            time.sleep(random.uniform(0, delay))  # full jitter
```

Without the jitter, many clients that fail together retry together, and the load arrives in synchronized waves.

### Heuristics

- Reduce the number of network hops on the critical path before optimizing any single hop.
- Prefer designs that fail closed to a known state over designs that depend on timing luck.
- Test failure on purpose: kill processes, add latency, drop packets, in an environment where it is safe to do so.
- Write down which failure assumptions each design depends on. If the assumption is "this never happens", treat that as a bug.

Related reading elsewhere in this handbook: [System Design](../05-system-design/README.md) and [Performance and Scalability](../12-performance-scalability/README.md).

## Further Reading

- Martin Kleppmann, *Designing Data-Intensive Applications*, especially the chapters on the trouble with distributed systems.
- Leslie Lamport, "Time, Clocks, and the Ordering of Events in a Distributed System".
- Fischer, Lynch and Paterson, "Impossibility of Distributed Consensus with One Faulty Process".
- Jeff Dean and Luiz Andre Barroso, "The Tail at Scale".
- Peter Deutsch and colleagues, the "Fallacies of Distributed Computing" (widely reproduced; search by title).
