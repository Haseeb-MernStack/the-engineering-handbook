# Failure Modes and Resilience Patterns

> Distributed systems fail partially, slowly, and in correlated ways; resilience comes from bounding every dependency and designing the degraded behavior in advance.

## Overview

A single program mostly either works or crashes. A distributed system is different. Some components work while others do not. A dependency may be up but slow, or return wrong answers, or succeed for half of requests. These partial failures are harder than clean crashes because they are ambiguous and because they spread.

Resilience is the ability to keep delivering acceptable service while parts fail. It is not the absence of failure. It is designed: you decide what happens when each dependency is slow, down, or wrong, and you limit how far a failure can travel.

This topic collects the failure modes and the patterns that contain them. It ties together the earlier topics in this section, especially [Fundamentals and Fallacies](./fundamentals-and-fallacies.md), [Replication and Partitioning](./replication-and-partitioning.md), and [Messaging and Idempotency](./messaging-and-idempotency.md).

## Core Concepts

### How failures differ

- **Crash:** the process stops. Easy to detect, in principle.
- **Omission:** messages are lost.
- **Timing:** responses arrive late. Indistinguishable from a crash until a timeout fires.
- **Byzantine or corrupt:** wrong data is returned, from bugs, bit rot, or malicious actors.
- **Gray failure:** a component is degraded but passes simple health checks. It answers pings and fails real work.

Slow is often worse than down. A dead dependency fails fast. A slow one holds threads, connections, and memory while callers wait.

### Cascading failure

One overloaded component slows, callers pile up waiting, their resources exhaust, and they fail, pushing load onto others. Retries add traffic. The failure spreads upstream and sideways. Many large outages follow this shape: a local fault, amplified by feedback loops.

### Timeouts

Every remote call needs a timeout. Without one, the worst case is infinite. Choose timeouts from measured latency, for example somewhat above a high percentile of normal behavior, and keep them shorter than the caller's own deadline.

Propagate a deadline across calls. If a request has 800 ms left, a downstream call should not be given a fresh 5 seconds.

### Retries

Retries mask transient faults. Use them carefully:

- Retry only idempotent operations, or those with idempotency keys.
- Use exponential backoff with jitter so clients spread out.
- Cap attempts and total time.
- Use a retry budget, for example limiting retries to a fraction of normal traffic, so that retries cannot multiply load.
- Retry at one layer only. If three layers each retry three times, one request becomes up to twenty-seven.

```python
import random, time

def call_with_retry(fn, attempts=4, base=0.1, cap=2.0):
    for i in range(attempts):
        try:
            return fn()
        except TransientError:
            if i == attempts - 1:
                raise
            delay = min(cap, base * (2 ** i))
            time.sleep(random.uniform(0, delay))  # full jitter
```

### Circuit breakers

A circuit breaker watches calls to a dependency. When failures exceed a threshold, it opens and fails calls immediately, sparing the dependency and freeing the caller. After a cool-down it allows a few trial calls. If they succeed, it closes.

```text
CLOSED --failures exceed threshold--> OPEN
OPEN --cool-down elapsed--> HALF-OPEN
HALF-OPEN --trial succeeds--> CLOSED
HALF-OPEN --trial fails--> OPEN
```

### Bulkheads

Isolate resources so one failing dependency cannot consume everything. Use separate thread pools, connection pools, or queues per dependency or per tenant. A slow recommendations service then exhausts its own pool and not the one serving checkout.

### Load shedding and backpressure

When demand exceeds capacity, serving everyone badly is worse than serving some well. Reject excess work early and cheaply, ideally lowest priority first. Return a clear overload signal so clients back off. Bounded queues turn overload into rejection rather than unbounded latency.

### Graceful degradation

Decide the reduced mode in advance. If recommendations are down, show popular items. If the profile service is down, show a generic page. If search is slow, serve cached results. The degraded path must be tested, because untested fallback code usually fails when first used.

### Redundancy and failure domains

Redundancy helps only if failures are independent. Replicas in the same rack, zone, or deploy pipeline share fate. Spread across failure domains: machines, racks, zones, regions. Remember software is a failure domain too. The same bad release on every replica fails them all together.

### Health checks

A shallow check, "process is running", misses gray failures. A deep check that calls every dependency can cause cascading restarts when one dependency blips. Separate liveness (restart me) from readiness (send me traffic), and keep liveness conservative.

### Observability

You cannot fix what you cannot see. Instrument the signals that show user pain and saturation: request rate, error rate, latency distributions, saturation of queues and pools. Use distributed tracing to follow a request across services, and propagate correlation IDs. Alert on symptoms users feel, and keep cause-level metrics for diagnosis.

### Testing failure

Resilience claims are hypotheses. Test them: inject latency, errors, dropped packets, and paused or killed instances. Start in test environments, then carefully in production with limited blast radius. Run game days to practice the human response.

## Trade-offs

**Timeouts.** Short timeouts protect callers and cause false failures under normal variance. Long timeouts tolerate slow responses and tie up resources.

**Retries.** They improve success rates and can turn a brownout into a blackout. A strict budget lowers recovery from brief faults but prevents amplification.

**Circuit breakers.** They protect dependencies, but a poorly tuned breaker flaps, or stays open after recovery, or trips on a single noisy endpoint and blocks healthy ones.

**Redundancy.** Extra capacity and regions cost money and add operational complexity, including data consistency across regions. Active-active gives faster failover and is harder than active-passive.

**Graceful degradation.** Each fallback is more code, more tests, and more states. Too many degraded modes become a source of bugs.

**Load shedding.** It protects the system and deliberately fails some users. Someone must decide who, and that decision is a product choice.

**Chaos testing.** It finds weaknesses early and carries risk and a need for solid safeguards and team maturity.

## Failure Modes

### Retry storm

A dependency slows, clients time out and retry, load rises, the dependency slows further. Detect through the ratio of retries to first attempts. Prevent with budgets, jitter, and backoff.

### Thundering herd

A cache entry expires or a service restarts and many clients simultaneously hit the origin. Use request coalescing, staggered expirations, and warm-up.

### Metastable failure

The system enters a bad state that persists even after the original trigger is gone. For instance, a cold cache after a restart overloads the database, which keeps cache fills slow, which keeps the cache cold. Recovery requires reducing load, for example shedding traffic, until the loop breaks. Design for it by keeping capacity headroom and having a load-shedding lever.

### Resource exhaustion

Leaks and unbounded growth: file descriptors, threads, connections, memory, disk. Set limits and alert at saturation thresholds before the cliff.

### Dependency-as-single-point-of-failure

A "highly available" service depends on a shared component, such as DNS, an auth service, a config store, or a certificate, with no fallback. When it fails, everything fails. Map dependencies, including hidden and shared ones. Decide for each whether to fail open or closed.

### Correlated failure from deploys

A bad release or configuration pushed everywhere at once. Use staged rollouts, canaries, automatic rollback on error-rate regression, and kill switches via feature flags. Treat configuration changes like code changes.

### Clock and certificate expirations

Expired certificates and bad time handling cause sudden, system-wide failures. Monitor expiry dates, automate renewal, and keep clocks synchronized. See [Time, Clocks, and Ordering](./time-clocks-and-ordering.md).

### Untested failover

Failover paths that never run tend not to work. Exercise them regularly.

## In Practice

### Per-dependency checklist

For every remote dependency, write down:

- Timeout and deadline propagation.
- Retry policy: idempotent? budget? jitter?
- Circuit breaker or other fast-fail mechanism.
- Isolation: separate pool or bulkhead.
- Fallback behavior and whether it has been tested.
- Fail open or fail closed, and why.
- Metrics and alerts: latency, errors, saturation.

### Worked example: checkout with a slow recommendations service

Recommendations add context on the checkout page. Latency rises from 40 ms to 6 seconds. The checkout service has a 10 second timeout and shares a thread pool across all calls. Threads fill up waiting. Checkout stops responding, and revenue stops over a non-essential feature.

Fixes, in order of value:

1. Set a tight timeout, around the latency the page can afford, on the recommendations call.
2. Put it in its own bounded pool so it cannot starve checkout.
3. Add a circuit breaker that returns an empty list when open.
4. Render checkout without recommendations as a designed fallback, and test it.
5. Alert on fallback rate so the team learns about the degradation.

Nothing here needs new technology. It needs someone to ask which dependencies are essential.

### Developer vs. engineer

A developer wraps a call in try/catch and logs the error. An engineer asks what the caller experiences when the call takes thirty seconds instead of failing, what the load is when every client retries, and how they would know it happened.

### Heuristics

- Assume every call can fail, hang, or return garbage.
- Prefer failing fast over waiting long.
- Make the degraded mode the designed behavior, not an afterthought.
- After every incident, ask which limit or isolation was missing, and add it.
- Write blameless postmortems focused on system conditions and fixes.

## Further Reading

- Michael Nygard, *Release It!*.
- Betsy Beyer et al., eds., *Site Reliability Engineering* and *The Site Reliability Workbook*.
- Jeffrey Dean and Luiz Barroso, "The Tail at Scale".
- Peter Alvaro et al., "Lineage-driven Fault Injection".
- Nathan Bronson et al., "Metastable Failures in Distributed Systems".
- Marc Brooker, writings on timeouts, retries, and backoff in the AWS Builders' Library.
