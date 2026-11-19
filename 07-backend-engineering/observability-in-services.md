# Observability in Services

> Observability is the ability to understand what a running service is doing, and why, from the telemetry it emits: logs, metrics and traces.

## Overview

Production behaves differently from your laptop. You cannot attach a debugger to it, and the failure you need to explain has often already passed. What you have is the evidence the system left behind.

Observability is a design property, like testability. A service is observable if you can ask new questions about its behavior ("why are requests from this tenant slow since 14:00?") without shipping new code to answer them. Monitoring tells you that something is wrong. Observability helps you find out why.

A developer adds a `console.log` when something breaks. An engineer decides in advance which signals will be needed at 3 a.m. and builds them in, because after the incident starts it is too late.

This topic covers instrumentation inside a service. For alerting, SLOs and platform-level monitoring, see [Observability and Monitoring](../09-devops-cloud-infrastructure/observability-and-monitoring.md).

## Core Concepts

### The three signals

| Signal | Answers | Strength | Weakness |
|---|---|---|---|
| Logs | What happened in this specific event? | Rich detail, context | Expensive at volume, hard to aggregate |
| Metrics | How much, how often, how fast over time? | Cheap, aggregable, good for alerts | No per-request detail, limited dimensions |
| Traces | Where did this request spend its time across services? | Shows causality and latency breakdown | Needs propagation everywhere, usually sampled |

They complement each other. A metric shows error rate rising. A trace shows which downstream call is slow. A log shows the exact error message.

### Structured logging

Write logs as structured records (typically JSON), not free text. Machines can then filter and aggregate them.

```json
{"ts":"2026-03-02T10:15:04.211Z","level":"error","service":"orders",
 "msg":"payment failed","order_id":"o_123","trace_id":"4bf92f35",
 "error":"timeout","duration_ms":3001}
```

Guidelines:

- Use levels consistently: error for things needing attention, warn for recoverable anomalies, info for significant events, debug off by default in production.
- Include identifiers: request id, trace id, tenant or user id where appropriate.
- Log events and decisions, not every line of code.
- Never log secrets, tokens, full card numbers or unnecessary personal data. See [Authentication and Authorization](./authentication-and-authorization.md) and the [security section](../10-security/README.md).
- Log an error once, where it is handled, not at every layer it passes through.

### Metrics

A metric is a numeric time series. Common types: counters (only increase), gauges (go up and down), histograms (distribution of values).

Useful frameworks:

- **RED** for request-driven services: Rate, Errors, Duration.
- **USE** for resources: Utilization, Saturation, Errors.
- Google's "four golden signals": latency, traffic, errors, saturation.

Prefer histograms for latency and look at percentiles. Averages hide the slow tail that users feel. Measure successful and failed request latency separately, since fast errors can mask slow successes.

### Cardinality

Each unique combination of label values creates a separate time series. A label such as `user_id` or a raw URL with ids can produce millions of series, raising cost and destabilizing the metrics store. Use low-cardinality labels (route template, status class, region) in metrics. Put high-cardinality identifiers in logs and traces.

### Distributed tracing

A trace follows one request across services. It consists of spans, each with a name, start time, duration and attributes, linked by parent relationships. A trace context (for example the W3C Trace Context headers) is passed on every outbound call so spans join into one tree.

```
GET /checkout            [=====================================] 480ms
  auth check             [==]
  inventory.reserve         [=========]
  payment.charge                       [==========================] 310ms
    db.query                              [===]
```

Tracing needs consistent propagation, including through queues. Include the trace context in message headers so asynchronous work stays linked. See [Queues and Background Jobs](./queues-and-background-jobs.md).

Because tracing every request is costly, services sample. Head sampling decides at the start and is cheap. Tail sampling decides after the trace completes and can keep all errors and slow requests, at higher complexity.

### Correlation

The power comes from joining signals. Put the trace id in every log line. Attach exemplars or links from metrics to traces where supported. With this, you can go from an alert to a graph to a slow trace to the exact log in minutes.

### Health checks

Distinguish liveness (the process should be restarted if this fails) from readiness (the instance should receive traffic). A liveness check that depends on the database can restart every instance during a database outage and make things worse. Keep liveness shallow.

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Verbose logging | More context when debugging | Storage and ingestion cost, noise, privacy risk |
| High metric cardinality | Fine-grained slicing | Cost, slow queries, backend instability |
| Full tracing | Complete picture | Overhead, cost |
| Aggressive sampling | Low cost | May miss rare events |
| Vendor agent | Fast setup | Lock-in, opaque overhead |
| Open standards for instrumentation | Portability | Some integration work |

Telemetry is not free. It adds latency, bandwidth and bills. It also becomes a data-protection liability if it captures personal data. Decide what you keep, how long, and who can read it.

## Failure Modes

- **Logs without context.** "Error occurred" with no id or inputs. Impossible to correlate.
- **Log flooding.** An error loop writes millions of lines, filling disk or exhausting the logging budget just when you need it. Rate-limit repeated messages.
- **Sensitive data leakage.** Request bodies or headers logged wholesale expose credentials. Use allowlists and redaction.
- **Cardinality explosion.** A new label doubles the metric bill overnight or crashes the store. Review labels in code review.
- **Broken trace propagation.** One service drops the header and traces fragment. Detect with a metric of traces missing expected spans.
- **Averages only.** Mean latency looks fine while p99 is terrible.
- **Alert fatigue.** Noisy alerts get ignored. Alert on symptoms users feel, not every cause.
- **Observability that fails with the system.** Logging blocks on a downed collector and takes the service with it. Make telemetry export asynchronous and lossy under pressure.
- **Clock skew.** Timestamps from different hosts disagree, making timelines misleading. Use synchronized time.

## In Practice

### Minimum bar for a new service

1. Structured logs with request and trace ids.
2. RED metrics per route, with latency histograms.
3. Traces on inbound and outbound calls, with context propagated through queues.
4. Dependency metrics: call rate, errors and latency per downstream.
5. Saturation metrics: pool usage, queue depth, memory, CPU.
6. Distinct liveness and readiness endpoints.
7. A dashboard answering "is it healthy?" at a glance, and a runbook link per alert.

### Worked example

Users report slow checkouts. The latency histogram shows the p99 rose at 14:00 while the p50 did not. Filtering traces by duration shows the slow ones all wait on `payment.charge`. The span attributes show one payment region. The logs for those traces show repeated connection timeouts. Without any new deploy you have isolated the cause to one dependency and region. This is observability working: new questions answered from existing data.

### Developer vs. engineer

The developer adds logs after the bug. The engineer asks, while designing the feature, "how will I know it is broken, and how will I find out why?"

### Verifying AI-generated code

Generated code often logs entire request objects, uses string-concatenated unstructured messages, puts user ids in metric labels, or logs and rethrows at every layer. It rarely propagates trace context. Review telemetry like any other behavior.

## Further Reading

- Charity Majors, Liz Fong-Jones and George Miranker, *Observability Engineering*.
- Betsy Beyer et al. (eds.), *Site Reliability Engineering*, chapter on monitoring distributed systems (the four golden signals).
- Cindy Sridharan, *Distributed Systems Observability*.
- W3C, "Trace Context" Recommendation.
- Official OpenTelemetry documentation on instrumentation and context propagation.
