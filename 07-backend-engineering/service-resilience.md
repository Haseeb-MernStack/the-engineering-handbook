# Service Resilience

> Resilience is the set of techniques, including timeouts, retries, backoff, circuit breakers and rate limiting, that keep a service useful when its dependencies or its callers misbehave.

## Overview

In a distributed system, partial failure is normal. A dependency is slow, a network drops packets, a deploy restarts half the fleet, a client sends ten times the usual traffic. The question is not whether this happens but whether one small failure stays small.

Resilience patterns are mostly about containing blast radius. A slow dependency should not exhaust your threads. A failing dependency should not receive a flood of retries that keeps it down. A greedy client should not starve everyone else. Each pattern defends against a specific way that failures spread.

A developer calls a remote service and handles the success case. An engineer asks what the call does when the other side is slow, wrong, down or recovering, and what happens to everything else in the meantime.

## Core Concepts

### Timeouts

Every network call needs a timeout. Without one, the default is often unlimited or very long. A hung dependency then holds a connection, a thread and memory for as long as it likes, and capacity drains until the caller fails too.

Guidelines:

- Set separate connect and read (or total) timeouts.
- Base values on the dependency's observed latency distribution, for example a bit above a high percentile, not on a guess.
- Propagate a deadline: if the user-facing request has two seconds left, a downstream call should not wait five.
- Timeouts must be shorter as you go deeper in the call chain, or callers time out while callees keep working.

```ts
const controller = new AbortController();
const timer = setTimeout(() => controller.abort(), 800);
try {
  const res = await fetch(url, { signal: controller.signal });
  return await res.json();
} finally {
  clearTimeout(timer);
}
```

A timeout does not tell you whether the operation happened. The request may have succeeded after you gave up. This is why retries need idempotency.

### Retries

Retries turn transient failures into successes. They are also the most common cause of self-inflicted outages.

Retry only when all hold:

- The error is plausibly transient (connection reset, timeout, 503, a 429 with Retry-After). Never retry validation errors or most 4xx responses.
- The operation is idempotent, or carries an idempotency key.
- There is retry budget left.

### Backoff and jitter

Waiting between retries gives the dependency time to recover. Exponential backoff doubles the wait each attempt. Jitter randomizes it so that thousands of clients do not retry in lockstep.

```ts
function delayMs(attempt: number, base = 100, cap = 10_000) {
  const exp = Math.min(cap, base * 2 ** attempt);
  return Math.random() * exp; // full jitter
}
```

Cap both the delay and the number of attempts. Respect `Retry-After` when the server provides it.

### Retry amplification

If each of three layers retries three times, one user action can produce 27 calls against the bottom service just when it is weakest. Retry at one layer, usually the one closest to the user, or use a retry budget: allow retries only up to a small fraction of total traffic.

### Circuit breakers

A circuit breaker watches calls to a dependency and stops making them when it appears unhealthy.

```
 CLOSED --(failure rate over threshold)--> OPEN
   ^                                         |
   |                                   (wait period)
   +----(probe succeeds)---- HALF-OPEN <-----+
```

- Closed: calls flow, failures are counted.
- Open: calls fail immediately without touching the dependency.
- Half-open: a few probe calls test recovery.

Breakers fail fast, which frees your resources, and give the dependency room to recover. They need a sensible fallback: a cached value, a default, a degraded feature or a clear error.

### Bulkheads

Isolate resources so one failing dependency cannot consume all of them. Use separate connection pools, thread pools or concurrency limits per dependency. If the recommendations service hangs, checkout should still work.

### Rate limiting and load shedding

Rate limiting protects you from callers. Common algorithms:

| Algorithm | Behavior | Note |
|---|---|---|
| Fixed window | N requests per interval | Simple, allows bursts at window edges |
| Sliding window | Smoother count over a moving interval | More state |
| Token bucket | Tokens refill at a rate, bursts allowed up to bucket size | Common default |
| Leaky bucket | Constant outflow rate | Smooths bursts, adds queueing |
| Concurrency limit | Cap in-flight requests | Directly protects capacity |

Return 429 with `Retry-After` so well-behaved clients can slow down. Limit per client or tenant, not only globally, so one noisy caller cannot use everyone's share.

Load shedding is the server-side complement: when overloaded, reject low-priority work early rather than slowly failing everything. Rejecting quickly is cheaper than timing out.

### Graceful degradation and fallbacks

Decide in advance what the service does without each dependency. A product page can render without reviews. A payment cannot be faked. Classify dependencies as critical or optional and design the optional ones to fail soft.

## Trade-offs

| Technique | Protects against | Costs |
|---|---|---|
| Short timeouts | Resource exhaustion | False failures on slow-but-working calls |
| Retries | Transient faults | Extra load, duplicate effects, longer tail latency |
| Circuit breaker | Cascading failure | Tuning effort, may trip on noise, hides a recovered dependency briefly |
| Bulkheads | Resource contention | Lower utilization, more config |
| Rate limits | Abusive or buggy callers | Rejected legitimate bursts, state to store |
| Fallbacks | Dependency outage | Stale or reduced data, extra code paths that are rarely exercised |

Every protection adds configuration that can be wrong. Defaults copied from a blog post are guesses about someone else's system.

## Failure Modes

- **Missing timeout.** A single slow dependency freezes the service. Detect with in-flight request and thread-pool saturation metrics.
- **Retry storm.** Synchronized retries keep a recovering service down. Use jitter, caps and budgets.
- **Non-idempotent retry.** A retried POST creates two orders. Use idempotency keys.
- **Timeout inversion.** Inner timeouts longer than outer ones, so work continues after the caller left.
- **Breaker flapping.** Thresholds too sensitive, so it opens and closes constantly. Use minimum request volume before evaluating.
- **Untested fallback.** The degraded path was never exercised and breaks when first used. Test it with fault injection.
- **Metastable failure.** Overload causes retries, retries cause more overload, and the system stays broken after the trigger is gone. Shedding load and removing retries is usually how you recover.
- **Shared rate limiter outage.** The limiter's own store fails. Decide whether it fails open or closed, deliberately.

Detect these with dependency latency and error metrics, retry counts, breaker state changes and saturation signals. See [Observability in Services](./observability-in-services.md).

## In Practice

### Checklist for each outbound call

1. Is there a connect and total timeout, derived from a deadline?
2. Is the operation safe to retry? How is idempotency guaranteed?
3. Is backoff exponential with jitter, with a capped attempt count?
4. Does only one layer retry?
5. Is there a bulkhead or concurrency limit?
6. What is the fallback, and is it tested?
7. Which metrics show this dependency's health?

### Worked example

A checkout service calls inventory, pricing and a recommendations service. Recommendations becomes slow. With no timeout, request threads pile up waiting, and checkout stops entirely even though recommendations is optional. The fix: a 300 ms timeout on recommendations, a separate concurrency limit, a breaker that returns an empty list when open, and an alert on its error rate. Checkout stays up and the page is slightly plainer.

### Test it

Resilience is a claim you must verify. Inject latency and errors in staging, kill dependencies, and run load tests past capacity. See [Service Resilience in incident practice](../09-devops-cloud-infrastructure/incident-response-and-sre.md).

### Developer vs. engineer

The developer adds a retry loop because the call sometimes fails. The engineer asks why it fails, whether retrying makes it worse, and what the caller experiences if retries are exhausted.

### Verifying AI-generated code

Generated clients frequently retry immediately in a tight loop, retry on every error type, have no timeout, or wrap a library with a default policy nobody inspected. Read the actual values. Confirm jitter, caps and idempotency.

## Further Reading

- Michael T. Nygard, *Release It!*, on stability patterns such as timeouts, circuit breakers and bulkheads.
- Betsy Beyer et al. (eds.), *Site Reliability Engineering* (Google), chapters on handling overload and addressing cascading failures.
- Marc Brooker, "Exponential Backoff And Jitter," AWS Architecture Blog.
- Amazon Builders' Library, "Timeouts, retries, and backoff with jitter."
- Nathan Bronson et al., "Metastable Failures in Distributed Systems" (HotOS 2021).
