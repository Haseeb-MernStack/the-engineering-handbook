# Capacity Estimation

> Capacity estimation is rough arithmetic that turns product assumptions into orders of magnitude for traffic, storage, and bandwidth, so you can tell whether a design needs one machine or one hundred.

## Overview

Before building, you need to know how big the problem is. Not exactly. Within a factor of two or ten is usually enough. A system that handles 50 requests per second and one that handles 50,000 are different designs. Estimation tells you which you are building.

Estimation is also a way to find mistakes. If your numbers say a single table will hold 40 terabytes, or that a single server must handle a million connections, the design needs to change before any code is written.

All numbers here are assumptions. Real systems are measured. Estimation is the first draft of that measurement, and it should be replaced by production data as soon as you have it. The goal is to be roughly right and transparent about assumptions, not precisely wrong.

## Core Concepts

### Principles

1. **State assumptions.** Write each input and its source: "10 million daily active users (assumed)."
2. **Use round numbers.** Powers of ten and simple factors. False precision misleads.
3. **Work in orders of magnitude.** Is it 10, 1,000, or 100,000? Decide that first.
5. **Separate average from peak.** Systems are sized for peaks, not averages.
6. **Check against reality.** Compare with similar systems you have measured.

### Useful conversions

Round, approximate values for mental math:

| Quantity | Approximation |
|---|---|
| Seconds in a day | about 100,000 (86,400 exactly) |
| Seconds in a month | about 2.5 million |
| 1 KB / 1 MB / 1 GB / 1 TB | 10^3 / 10^6 / 10^9 / 10^12 bytes |
| Requests per second from N per day | N / 100,000 |

The rule "N per day divided by 100,000 gives per second" is a quick way to convert. 10 million requests per day is about 100 requests per second on average.

### Peak factor

Traffic is not flat. Daily cycles, events, and launches create peaks. A common assumption is that peak is two to ten times the average. Pick a factor from your own traffic patterns or product knowledge, and state it. If you have no data, use a conservative factor and mark it as a guess.

### Latency orders of magnitude

Approximate relative costs help judge designs. Exact values vary by hardware and change over time, so treat these as rough ratios:

- Memory access is far faster than SSD access.
- SSD access is far faster than a network round trip within a data center.
- A round trip within a data center is far faster than one across continents.
- A disk seek on rotating media is slower than all of the above except cross-continent trips.

The practical lessons: avoid unnecessary network hops, keep hot data in memory, and avoid serial calls across regions. Look up current figures for your hardware if the decision depends on them.

### What to estimate

**Traffic.** Requests per second, split into reads and writes.

**Storage.** Records per day x size per record x retention, plus replication and indexes.

**Bandwidth.** Requests per second x response size, in and out.

**Memory.** For caches: hot data size, usually a fraction of the total.

**Compute.** Requests per second divided by per-instance capacity, with headroom. Per-instance capacity must come from measurement of your code.

### A repeatable template

```
Users:               U  (monthly active)
Active fraction:     a  (daily active / monthly active)
Actions per user/day: n
Daily requests:      U * a * n
Average RPS:         daily requests / 100,000
Peak RPS:            average * peak_factor
Record size:         s bytes
Daily storage:       daily writes * s
Total storage:       daily storage * retention_days * replication * overhead
Bandwidth:           peak RPS * average response size
```

## Trade-offs

| Choice | Gain | Cost |
|---|---|---|
| Quick, rough estimate | Fast; exposes order of magnitude | May miss subtle factors |
| Detailed model | More defensible | Time spent on uncertain inputs; false confidence |
| Size for peak | Survives peaks | Idle capacity most of the time |
| Size for average | Lower cost | Fails during peaks |
| Large headroom | Safety, room for growth | Higher cost |
| Autoscaling | Matches load | Scaling lag; limits; cost surprises; stateful parts do not scale easily |
| Measuring first | Real numbers | Needs a prototype or load test |

The main trade-off is effort against certainty. Estimate quickly to choose a design. Measure to commit to a capacity.

## Failure Modes

**Average instead of peak.** The system is sized for the daily mean and falls over at the daily peak. Always compute both.

**Ignoring growth.** Capacity is exact for today. Six months later it is full. Add a growth assumption and a review date.

**Forgetting overheads.** Storage estimates omit indexes, replication, backups, and metadata, which can multiply the raw data size. List them.

**Unit errors.** Bits versus bytes, MB versus MiB, per second versus per day. A factor of eight or 86,400 slips in. Write units on every line.

**False precision.** "We need 37.4 servers." The inputs were guesses. Round, and state a range.

**Untested per-node capacity.** Assuming a server handles a given number of requests per second without measuring. Load test your own code.

**Ignoring fan-out.** One user action triggers many internal calls or writes. A single post delivered to many followers multiplies writes. Estimate the amplification.

**Estimating once.** The numbers are never revisited as real traffic arrives. Compare estimates with production and update.

**Non-linear limits.** Some limits are not proportional: connection counts, lock contention, memory working sets, and disk throughput. Systems degrade sharply near saturation. Leave headroom.

## In Practice

### Worked example: a photo-sharing service (all inputs are assumptions)

Assumptions, stated plainly:

- 10 million monthly active users.
- 20 percent are active daily: 2 million daily active users.
- Each views 50 photos and uploads 0.5 photos per day.
- Average stored photo: 1 MB after processing (assumed).
- Peak factor: 5.

Traffic:

- Views per day: 2 million x 50 = 100 million. Average: 100 million / 100,000 = 1,000 per second. Peak: about 5,000 per second.
- Uploads per day: 2 million x 0.5 = 1 million. Average: about 10 per second. Peak: about 50 per second.
- Read to write ratio: about 100 to 1. This suggests caching and a CDN for reads.

Storage:

- 1 million uploads per day x 1 MB = 1 TB per day.
- One year: about 365 TB of original data. With 3 copies, about 1.1 PB. This is object-storage territory, not a single database. Photos go in an object store, and only metadata goes in the database.

Bandwidth (peak egress):

- 5,000 views per second x 1 MB = 5 GB per second. That is too much for the origin to serve directly. A CDN is not optional; it must serve most of this. Thumbnails would reduce it by an order of magnitude, so image sizes are a design lever.

Metadata:

- 1 million rows per day x 500 bytes = 0.5 GB per day, roughly 180 GB per year. This fits on a single modern database server with replicas, so sharding is not needed on day one.

Conclusions the numbers support: store images in object storage, serve through a CDN, keep metadata in a relational database with read replicas, and defer sharding. They also flag the risky number: the 1 MB average size, which should be validated first.

### Checklist

- [ ] Every input is labeled as assumption or measurement, with units.
- [ ] Average and peak are both computed.
- [ ] Read and write loads are separated.
- [ ] Storage includes replication, indexes, and retention.
- [ ] Fan-out and amplification are considered.
- [ ] Headroom and a growth date are stated.
- [ ] The estimate names the number that would most change the design.
- [ ] A plan exists to replace guesses with measurements.

### Developer vs. engineer

A developer picks infrastructure by habit or fashion. An engineer calculates the expected load, finds which resource will limit first, and explains the design in terms of numbers that others can challenge.

### Verifying AI-generated designs

AI tools may state capacity figures with confidence, including benchmark claims and per-server limits that are not true for your workload. Verify:

- Recompute the arithmetic yourself, including units.
- Treat any quoted throughput or latency figure as unverified until you measure.
- Check that peak, growth, and overhead are included.
- Confirm that assumptions match your product, not a generic example.

## Further Reading

- Jeff Dean, "Numbers Everyone Should Know" (talk slides, widely circulated; values are dated, so treat as ratios).
- Martin Kleppmann, *Designing Data-Intensive Applications*.
- Brendan Gregg, *Systems Performance*.
- Neil Gunther, *Guerrilla Capacity Planning*.
- Alex Xu, *System Design Interview*, chapter on back-of-the-envelope estimation.
