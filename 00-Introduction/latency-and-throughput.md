# Latency & Throughput

> **NFR Deep Dive #1** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Overview

Latency and throughput are the two fundamental performance metrics of any system. They are independent — a system can have low latency and low throughput, or high latency and high throughput. Optimizing one often trades off against the other.

| Metric | Definition | Unit | Question It Answers |
|---|---|---|---|
| **Latency** | Time to complete a single request | ms, µs, s | How *fast* is one operation? |
| **Throughput** | Number of requests handled per unit time | RPS, QPS, TPS | How *many* operations per second? |

**Analogy — a highway:**
- **Latency** = how long one car takes to travel from entry to exit.
- **Throughput** = how many cars pass the exit per minute.
- A wider highway (more lanes) increases throughput but does not make any single car faster. A higher speed limit reduces latency.

---

## 2. Why These Metrics Matter

A system that is functionally correct can still fail in production if it is slow or cannot handle load. These two metrics are the primary lens through which performance is specified, measured, and optimized.

- **Latency** directly shapes user experience. Amazon found every 100ms of added latency measurably reduced sales (widely cited; based on public statements).
- **Throughput** determines capacity. If demand exceeds throughput, requests queue, latency spikes, and the system eventually collapses.

These two are linked through queuing: as throughput approaches the system's maximum capacity, latency rises sharply (see Section 7).

---

## 3. Latency — Deep Dive

### 3.1 Latency Is Always a Sum

The latency a user experiences is the sum of every hop in the request path. You cannot optimize what you do not decompose.

```
Total Latency = Network + Queuing + Processing + I/O + Serialization

Client → [Network] → Load Balancer → [Queue] → Service
       → [DB Query / I/O] → [Compute] → [Serialize] → Response
```

| Component | Typical Contribution | Cause |
|---|---|---|
| Network (same region) | 0.5–2 ms | Physical distance, routing |
| Network (cross-continent) | 100–300 ms | Speed of light is a hard limit |
| Queuing | 0 ms → seconds | Load — grows as utilization rises |
| Database query | 1–100 ms | Index quality, query complexity, lock contention |
| Compute | µs–ms | Algorithm efficiency |
| Serialization | µs–ms | Payload size, format (JSON vs binary) |

### 3.2 Never Use Averages — Use Percentiles

An average hides the worst experiences. This is the single most important concept in latency.

| Metric | Meaning | Example |
|---|---|---|
| **P50 (median)** | Half of requests are faster than this | 20 ms |
| **P95** | 95% of requests are faster than this | 80 ms |
| **P99** | 99% of requests are faster than this | 200 ms |
| **P99.9** | 99.9% of requests are faster than this | 800 ms |

**Why averages lie:** A system with P50 = 10ms but P99 = 5,000ms has a serious problem that an average of ~60ms completely masks.

**The tail latency amplification problem:** If one page makes 100 backend calls and waits for all of them, the *slowest* call determines page load time. With a P99 of 200ms per call, nearly *every* page load hits at least one slow call.

```
P(no slow call) = 0.99^100 ≈ 0.366

→ 63% of page loads experience at least one P99-tail call.
```

> **Rule:** Specify and monitor latency at P95, P99, and P99.9 — never as an average alone.

### 3.3 Reference Latency Numbers (Orders of Magnitude)

Every engineer should know these to reason about design without measuring.

| Operation | Approximate Latency |
|---|---|
| L1 cache reference | ~1 ns |
| Main memory (RAM) reference | ~100 ns |
| SSD random read | ~16 µs |
| Network round trip (same datacenter) | ~500 µs |
| Disk (HDD) seek | ~2–10 ms |
| Network round trip (cross-continent) | ~100–150 ms |

> **Key insight:** Memory is ~100,000× faster than disk. This is *why* caching exists — moving hot data from disk to memory collapses latency by orders of magnitude.

### 3.4 How to Reduce Latency

| Technique | How It Helps | Trade-off |
|---|---|---|
| **Caching** | Serves hot data from memory, skipping slow I/O | Stale data; cache invalidation complexity |
| **CDN / Edge** | Moves data physically closer to users | Cost; only helps cacheable content |
| **Database indexing** | Turns full scans (O(n)) into lookups (O(log n)) | Slower writes; more storage |
| **Connection pooling** | Reuses connections, skipping setup cost | Pool tuning required |
| **Async / parallelism** | Runs independent work concurrently | Complexity; harder to reason about |
| **Better algorithms** | Reduces compute time fundamentally | Engineering effort |
| **Compression / binary formats** | Smaller payloads serialize and transmit faster | CPU cost to compress/decompress |

---

## 4. Throughput — Deep Dive

### 4.1 Common Units

| Unit | Stands For | Used For |
|---|---|---|
| **RPS** | Requests per second | Web servers, API gateways |
| **QPS** | Queries per second | Databases, search |
| **TPS** | Transactions per second | Payment systems, ledgers |
| **MPS** | Messages per second | Queues, event streams |

### 4.2 Estimating Required Throughput

Throughput targets are *derived* from functional requirements via back-of-envelope math. This is a core system-design-interview skill.

```
Example: messaging system

Assumptions:
  500 million daily active users
  Each user sends 40 messages/day
  Peak traffic = 2× the daily average

Daily messages   = 500M × 40           = 20 billion/day
Average MPS       = 20B ÷ 86,400 sec    ≈ 231,000 MPS
Peak MPS          = 231,000 × 2          ≈ 462,000 MPS
```

The result (~500K MPS at peak) immediately rules out single-node designs and mandates horizontal scaling, partitioning, and async processing.

### 4.3 How to Increase Throughput

| Technique | How It Helps | Trade-off |
|---|---|---|
| **Horizontal scaling** | Add machines; distribute load | Requires statelessness, load balancing |
| **Load balancing** | Spreads requests across instances | Adds a hop; needs health checks |
| **Database read replicas** | Offloads reads from the primary | Replication lag → stale reads |
| **Sharding / partitioning** | Splits data and load across nodes | Cross-shard queries become hard |
| **Batching** | Amortizes fixed per-request cost | Increases latency per item |
| **Async processing** | Removes slow work from the request path | Eventual consistency |
| **Connection pooling** | More concurrent work with fewer resources | Tuning required |

---

## 5. The Core Trade-off: Latency vs Throughput

These two metrics frequently pull in opposite directions. The clearest example is **batching**.

```
WITHOUT batching:
  Each request processed immediately.
  → Low latency per request.
  → Lower throughput (fixed overhead paid per request).

WITH batching:
  Requests grouped and processed together.
  → Higher throughput (overhead amortized across the batch).
  → Higher latency (early requests wait for the batch to fill).
```

| Optimize For | Approach | Best For |
|---|---|---|
| **Low latency** | Process immediately, no waiting | Interactive systems, user-facing APIs |
| **High throughput** | Batch, buffer, parallelize | Data pipelines, analytics, bulk jobs |

> **Design principle:** Decide *up front* which one matters for each path. A checkout API optimizes for latency; a nightly billing job optimizes for throughput.

---

## 6. The Relationship: Little's Law

Little's Law connects the two metrics with concurrency. It is the most useful formula in performance engineering.

```
L = λ × W

L = average number of requests in the system (concurrency)
λ = throughput (requests per second)
W = average latency (seconds per request)
```

**Worked example:**

```
A service handles λ = 1,000 RPS.
Each request takes W = 0.2 sec (200ms latency).

Concurrent requests in flight:
L = 1,000 × 0.2 = 200

→ The system must support 200 simultaneous in-flight requests
  (e.g. 200 threads, connections, or async slots) to sustain this load.
```

**Why this matters:** It tells you how to size thread pools and connection pools. If latency rises to 400ms at the same 1,000 RPS, you now need 400 concurrent slots. **Rising latency silently increases your concurrency requirement** — a common cause of cascading failures.

---

## 7. Why Latency Explodes Near Capacity

As throughput approaches a system's maximum, latency does not rise linearly — it rises *exponentially*. This is queuing theory.

```
Latency
  │                                    ╱
  │                                  ╱
  │                               ╱
  │                          ╱
  │                  ╱╱
  │        ___──────
  │───────
  └────────────────────────────────────── Utilization
  0%        50%        80%   90%  95% 100%
```

| Utilization | Latency Behaviour |
|---|---|
| < 50% | Stable, low latency |
| 70–80% | Latency begins to climb |
| > 90% | Latency rises sharply — small load increases cause large spikes |
| ~100% | Queues grow unbounded; system becomes unresponsive |

> **Operational rule:** Never run a system at sustained high utilization. Target **60–70%** to leave headroom for traffic spikes. The remaining capacity is not "waste" — it is your latency insurance.

---

## 8. How Large Companies Apply This

| Company | Application | Source |
|---|---|---|
| **Netflix** | Targets low video start-time (P99); pre-positions content at edge via CDN to cut network latency | Netflix Tech Blog (public) |
| **Amazon** | Publicly stated that added latency reduces conversion; latency budgets enforced per service | Public statements |
| **Discord** | Optimizes message-delivery latency; chose Cassandra for write throughput at scale | Discord Eng Blog (public) |
| **Google** | Coined "tail at scale"; uses request hedging (send duplicate request, take first response) to cut P99 | *The Tail at Scale*, Dean & Barroso (public paper) |

> **Inferred:** Exact internal latency budgets are not public. The techniques above are described in public engineering material.

---

## 9. Best Practices

- **Specify latency as percentiles** (P95/P99/P99.9), never as an average.
- **Set explicit latency budgets** per service so the end-to-end sum stays within target.
- **Derive throughput targets** from FRs using back-of-envelope math, including peak (not just average).
- **Run at 60–70% utilization** to keep latency stable and absorb spikes.
- **Use Little's Law** to size thread and connection pools correctly.
- **Cache the hot path** — memory is ~100,000× faster than disk.
- **Load-test before launch** — you cannot know real P99 without production-like load.

---

## 10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Measuring only average latency | Tail latency hidden; real UX is poor | Track P99 / P99.9 |
| Sizing for average load, not peak | System collapses during spikes | Size for peak with headroom |
| Running at 90%+ utilization | Latency explodes on small load increases | Keep utilization ≤ 70% |
| Optimizing latency and throughput identically | Wrong trade-off for the workload | Pick the metric that matters per path |
| Ignoring tail amplification | Fan-out requests make P99 the common case | Reduce per-call P99; use hedging |
| Fixed-size pools ignoring Little's Law | Pool exhaustion under rising latency | Size pools from L = λ × W |

---

## 11. Interview Questions

1. Why should latency be reported as P99 rather than as an average?
2. Explain tail latency amplification in a fan-out request pattern.
3. A service handles 2,000 RPS with 50ms average latency. How many concurrent requests are in flight? *(Answer: L = 2000 × 0.05 = 100)*
4. Why does latency rise exponentially as a system nears full utilization?
5. Give a concrete example where increasing throughput increases latency.
6. How would you estimate the required throughput for a new photo-upload feature?
7. What utilization level should a latency-sensitive service target, and why?

---

## 12. Summary

| Concept | Key Takeaway |
|---|---|
| **Latency** | Time for one request. Always a *sum* of hops. Measure as percentiles. |
| **Throughput** | Requests per unit time. Derive targets from FRs, including peak. |
| **Trade-off** | Batching raises throughput but adds latency. Choose per path. |
| **Little's Law** | `L = λ × W` — connects throughput, latency, and concurrency. |
| **Queuing** | Latency explodes past ~70–80% utilization. Keep headroom. |
| **Reference** | Memory ≈ 100,000× faster than disk → caching is the highest-leverage latency optimization. |

---

## 13. Cross References

**Prerequisites:** System Design Fundamentals (FR, HLD, LLD)

**Related Topics:** Caching Strategies · CDN & Edge Computing · Load Balancing · Database Indexing

**What to Learn Next:** Availability & Reliability (NFR Deep Dive #2) · Scalability (NFR Deep Dive #3)

---

*System Design Engineering Handbook — NFR Series*