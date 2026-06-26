# Fault Tolerance

> **NFR Deep Dive #6** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Overview

Fault tolerance is a system's ability to **keep operating correctly even when some of its components fail**. The goal is not to prevent failures — that's impossible at scale — but to ensure that individual failures don't bring down the whole system.

> **Core philosophy:** *Assume everything fails.* Disks die, networks drop, servers crash, dependencies time out. A fault-tolerant system is designed so that any single failure is contained, absorbed, and recovered from — ideally without users ever noticing.

---

## 2. Fault vs Error vs Failure

These three terms are distinct, and the distinction matters.

```
FAULT    →    ERROR    →    FAILURE
(cause)      (state)       (visible effect)

A disk goes bad (fault) → a read returns garbage (error)
→ the user sees wrong data (failure)
```

> **Fault tolerance breaks this chain.** It stops a *fault* from becoming an *error*, or an *error* from becoming a *failure* — by detecting and handling it before users are affected.

---

## 3. Types of Faults

| Fault Type | Example | Difficulty |
|---|---|---|
| **Crash (fail-stop)** | Server dies cleanly and stops responding | Easiest — silence is detectable |
| **Omission** | Messages or responses get dropped | Medium — looks like slowness |
| **Timing** | Component responds too slowly | Hard — is it dead or just slow? |
| **Byzantine** | Component behaves arbitrarily/maliciously, sends wrong data | Hardest — can't trust the node |

> Most systems design for crash and omission faults. Byzantine fault tolerance (used in blockchains, aerospace) is far more expensive and rarely needed in typical applications.

---

## 4. Redundancy — The Foundation

You cannot tolerate the loss of something you have only one of. Redundancy means having spare capacity so failures are survivable.

| Redundancy Type | How It Works |
|---|---|
| **Active-active** | All replicas serve traffic; load redistributes if one dies | 
| **Active-passive** | Standby waits idle; takes over when primary fails |
| **N+1 / N+2** | Provision one (or two) more than the minimum needed |
| **Geographic** | Replicas in different datacenters/regions survive site loss |

```
Active-Active:                   Active-Passive:
  ┌───┐  ┌───┐  ┌───┐             ┌─────────┐   ┌─────────┐
  │ ✓ │  │ ✓ │  │ ✓ │             │ PRIMARY │   │ STANDBY │
  └───┘  └───┘  └───┘             │  (live) │   │ (idle)  │
  all serving; lose one,          └─────────┘   └─────────┘
  others absorb load               failover on primary death
```

> **No single point of failure (SPOF):** the goal of redundancy is that no one component's failure can take down the system. Audit every layer — LB, app, DB, network — for SPOFs.

---

## 5. Failure Detection

You can't tolerate a fault you haven't noticed. Fast, accurate detection is half the battle.

| Mechanism | Purpose | Pitfall |
|---|---|---|
| **Heartbeats** | Nodes periodically signal "I'm alive" | Network blip → false "dead" |
| **Health checks** | LB probes endpoints; routes around unhealthy nodes | Too aggressive → flapping |
| **Timeouts** | Treat a non-response as failure after a deadline | Too short → false positives |
| **Monitoring** | Watch error rates, latency; alert on anomalies | Alert fatigue if poorly tuned |

> **The detection dilemma:** Detect too fast → false positives (healthy nodes wrongly killed, "flapping"). Detect too slow → users suffer during the gap. Tuning this balance is core to fault-tolerant design.

---

## 6. Resilience Patterns — Containing Failure

Redundancy keeps spares ready; these patterns stop a failure from *spreading*.

| Pattern | Problem Solved | Mechanism |
|---|---|---|
| **Circuit Breaker** | A failing dependency drags callers down | After N failures, stop calling; fail fast; periodically retry |
| **Retry + Backoff + Jitter** | Transient glitches cause needless failures | Retry with growing, randomized delays |
| **Timeout** | A slow call ties up resources indefinitely | Cap wait time on every external call |
| **Bulkhead** | One overloaded dependency starves everything | Isolate resources per dependency (separate pools) |
| **Fallback** | A failed call breaks the whole feature | Return a default, cached, or degraded response |
| **Load shedding** | Overload collapses the entire system | Drop low-priority requests to protect the rest |

### Circuit Breaker — the key pattern

```
CLOSED  ──(failures exceed threshold)──→  OPEN
  ↑                                         │
  │                                  (wait timeout)
  │                                         ▼
  └──(test succeeds)──  HALF-OPEN  ←─(try one request)

CLOSED    = normal, calls pass through
OPEN      = dependency presumed dead; fail instantly, don't even try
HALF-OPEN = cautiously test if it has recovered
```

> **Analogy:** Like an electrical breaker — it trips to isolate one faulty appliance, protecting the whole house from burning down. Without it, one slow dependency exhausts all your threads and your *entire* service goes down — a **cascading failure**.

---

## 7. Cascading Failures — The Thing to Fear Most

A cascading failure is when one component's failure overloads others, which then fail, spreading until the whole system collapses.

```
Service A slows down
   → callers pile up waiting (threads exhausted)
      → callers become unresponsive
         → THEIR callers pile up
            → entire system down
```

**Defenses:** circuit breakers (stop calling the slow service), timeouts (don't wait forever), bulkheads (isolate the damage), load shedding (drop excess load), and **backpressure** (signal upstream to slow down).

---

## 8. Graceful Degradation

When something fails, serve *reduced* functionality instead of total failure.

```
Recommendation engine down?  → show generic popular items, not an error
Search service down?          → show cached/browse results
Image CDN down?               → show text + placeholder, page still works
```

> **Principle:** Degrade in layers. The core function (e.g. checkout) must survive even if peripheral features (recommendations, reviews) fail. Identify which features are essential vs. optional *before* the outage.

---

## 9. Recovery — Self-Healing

Fault tolerance includes *automatically restoring* full capacity after a failure.

| Mechanism | Behaviour |
|---|---|
| **Automatic failover** | Promote a replica/standby when the primary dies |
| **Auto-restart** | Supervisor restarts crashed processes |
| **Auto-replacement** | Orchestrator spins up a new node to replace a dead one |
| **Checkpointing** | Save state periodically; resume from last good point |

---

## 10. How Large Companies Apply This

| Company | Application | Source |
|---|---|---|
| **Netflix** | Chaos Monkey kills random instances in prod, forcing fault-tolerant design; Hystrix pioneered circuit breakers | Netflix Tech Blog (public) |
| **Amazon** | Cells/bulkheads isolate failures; multi-AZ redundancy by default | AWS public docs |
| **Google** | Health-checked load balancing; automatic node replacement; SRE failure drills | *Google SRE Book* (public) |
| **Erlang/telecom** | "Let it crash" philosophy — supervisors restart failed processes automatically | Erlang/OTP public design |

> **Inferred:** Internal implementation details vary; the patterns (chaos engineering, circuit breakers, bulkheads, supervision) are publicly documented.

---

## 11. Best Practices

- **Assume every component will fail** — design from that premise.
- **Eliminate SPOFs** — redundancy at every layer.
- **Wrap every external call** in timeout + circuit breaker.
- **Make retries safe** with idempotency + backoff + jitter.
- **Isolate with bulkheads** so one failure can't starve everything.
- **Design graceful degradation** — know your essential vs optional features.
- **Automate recovery** — failover, restart, replacement; minimize human-in-the-loop.
- **Test with chaos engineering** — inject real failures regularly; an untested mechanism will fail.

---

## 12. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No circuit breakers | One slow dependency cascades to total outage | Circuit breakers + timeouts everywhere |
| Retries without backoff | Retry storm overwhelms the recovering service | Exponential backoff + jitter |
| Retries without idempotency | Duplicate side effects (double charges) | Idempotency keys |
| Hidden SPOFs | "Redundant" system still has one fatal component | Audit every layer for SPOFs |
| Redundancy on the same host/rack | Correlated failure kills all "copies" at once | Spread across AZs/regions (anti-affinity) |
| Never testing failure paths | Fault-tolerance code is broken when needed | Chaos testing + game days |

---

## 13. Interview Questions

1. Distinguish fault, error, and failure.
2. What is a cascading failure and how do you prevent one?
3. Explain the circuit breaker pattern and its three states.
4. Active-active vs active-passive redundancy — trade-offs?
5. What is a bulkhead and what does it protect against?
6. Why must retries use backoff *and* idempotency?
7. Give an example of graceful degradation in a real system.
8. What is the detection dilemma (fast vs slow failure detection)?

---

## 14. Summary

| Concept | Key Takeaway |
|---|---|
| **Fault tolerance** | Keep working correctly despite component failures. |
| **Philosophy** | Assume everything fails; contain and recover. |
| **Redundancy** | Foundation — no SPOF, spread across zones. |
| **Detection** | Heartbeats, health checks, timeouts — tune to avoid false positives. |
| **Resilience patterns** | Circuit breaker, bulkhead, retry, fallback — stop cascades. |
| **Cascading failure** | The main threat; isolate to prevent system-wide collapse. |
| **Degradation** | Reduced service beats total outage. |
| **Recovery** | Automate failover, restart, replacement. |

---

## 15. Cross References

**Prerequisites:** System Design Fundamentals · Availability (NFR #2) · Reliability (NFR #4)

**Related Topics:** Resilience Patterns · Redundancy & Replication · Load Balancing · Chaos Engineering

**What to Learn Next:** Disaster Recovery (RTO/RPO) · Distributed Consensus · Observability & Monitoring

---

*System Design Engineering Handbook — NFR Series*