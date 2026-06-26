# Reliability

> **NFR Deep Dive #4** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Overview

Reliability is the probability that a system performs its function **correctly and consistently** over a period of time. A reliable system produces the right result, every time, even when individual components fail.

> **Reliability vs Availability — the key distinction:**
> - **Availability** = is the system *up*? (responding to requests)
> - **Reliability** = is the system *correct*? (producing right results, not losing data)
>
> A system can be available but unreliable: it responds quickly, but with corrupted or stale data. Reliability is the deeper guarantee.

---

## 2. The Core Metrics: MTBF & MTTR

Reliability is quantified by how often things break and how fast they recover.

| Metric | Full Name | Meaning |
|---|---|---|
| **MTBF** | Mean Time Between Failures | Average time the system runs before failing |
| **MTTR** | Mean Time To Recovery | Average time to restore service after a failure |

```
Availability = MTBF / (MTBF + MTTR)

→ You improve availability by either:
   (a) failing less often      (raise MTBF), or
   (b) recovering faster        (lower MTTR)
```

> **Insight:** Lowering MTTR is often cheaper and faster than raising MTBF. You cannot prevent every failure, but you *can* make recovery fast and automatic. This is why "design for fast recovery" beats "design for no failure."

---

## 3. Reliability vs Related Terms

| Term | Question It Answers |
|---|---|
| **Reliability** | Does it work *correctly*, consistently, over time? |
| **Availability** | Is it *up* right now? |
| **Durability** | Is *stored data* safe and never lost? |
| **Fault tolerance** | Does it *keep working* when a component fails? |

These overlap but are distinct. A bank transfer system must be reliable (correct balances), available (you can transfer now), durable (your money doesn't vanish), and fault-tolerant (a server crash mid-transfer doesn't corrupt anything).

---

## 4. Durability — Don't Lose Data

Durability is the reliability of *stored data*: once the system confirms a write, that data survives crashes, power loss, and hardware failure.

| Technique | How It Protects Data |
|---|---|
| **Replication** | Multiple copies on different nodes — one disk dies, data survives |
| **Write-ahead log (WAL)** | Record the change *before* applying it; replay on crash recovery |
| **Backups + snapshots** | Periodic point-in-time copies for catastrophic recovery |
| **Quorum writes** | Confirm a write only after N replicas acknowledge it |
| **Checksums** | Detect silent data corruption on disk |

> **Durability is measured in nines too.** Cloud object stores often advertise "eleven nines" (99.999999999%) durability — meaning effectively zero chance of losing an object.

---

## 5. Building Reliable Systems

Reliability comes from *assuming components will fail* and designing so failures don't produce wrong results.

### 5.1 Redundancy

No single component should be able to cause incorrect behavior. Replicate critical components and data.

### 5.2 Idempotency

An idempotent operation produces the same result whether run once or many times. This makes **retries safe** — essential in distributed systems where messages get duplicated.

```
❌ Not idempotent:  "add $100 to balance"   → run twice = +$200  (WRONG)
✅ Idempotent:      "set balance to $500"    → run twice = $500   (SAFE)

For operations that aren't naturally idempotent (e.g. payments),
use an idempotency key so duplicates are detected and ignored.
```

> **Idempotency is the single most important property for reliability in distributed systems.** It turns "retry on failure" from dangerous into safe.

### 5.3 Failure Detection

You cannot recover from a failure you don't notice.

| Mechanism | Purpose |
|---|---|
| **Health checks** | Periodic "are you alive?" probes; unhealthy nodes removed |
| **Heartbeats** | Nodes signal liveness; silence = assumed dead |
| **Monitoring + alerting** | Detect anomalies (error rates, latency) and page humans |
| **Timeouts** | A non-responding call is treated as a failure, not waited on forever |

### 5.4 Graceful Recovery

| Pattern | Behaviour |
|---|---|
| **Automatic failover** | Switch to a standby/replica when the primary fails |
| **Self-healing** | Auto-restart crashed processes; auto-replace dead nodes |
| **Rollback** | Revert a bad deployment instantly |
| **Checkpointing** | Save progress so recovery resumes from the last good state |

---

## 6. Data Integrity & Correctness

Reliability includes *not corrupting data*, even under concurrency and partial failure.

| Threat | Defense |
|---|---|
| **Concurrent writes** | Locks, optimistic concurrency, transactions |
| **Partial failure mid-write** | Atomic transactions (all-or-nothing) |
| **Duplicate processing** | Idempotency keys |
| **Silent corruption** | Checksums, validation |
| **Lost updates** | Versioning / compare-and-swap |

> **Atomicity** is core to reliability: an operation either fully completes or fully rolls back — never half-applied. A transfer that debits one account but crashes before crediting the other must roll back entirely.

---

## 7. Testing for Reliability

You don't *know* a system is reliable until you've tried to break it.

| Practice | What It Validates |
|---|---|
| **Chaos engineering** | Randomly kill components in production; verify the system survives |
| **Load / stress testing** | Behaviour at and beyond expected capacity |
| **Failure injection** | Simulate dependency outages, slow networks, disk failures |
| **Disaster recovery drills** | Confirm backups restore and failover actually works |

> **An untested backup is not a backup. An untested failover is not a failover.** Reliability mechanisms that have never been exercised tend to fail exactly when needed.

---

## 8. How Large Companies Apply This

| Company | Application | Source |
|---|---|---|
| **Netflix** | Chaos Monkey randomly terminates instances to force fault-tolerant design | Netflix Tech Blog (public) |
| **Amazon (S3)** | Advertises 99.999999999% (11 nines) durability via massive replication | AWS public docs |
| **Google (Spanner)** | Uses synchronized clocks + replication for globally consistent, reliable data | Public research paper |
| **Payment systems** | Idempotency keys make retried charges safe (no double-billing) | Common public API patterns |

> **Inferred:** Internal reliability figures are seldom public; the patterns (chaos engineering, replication, idempotency) are widely documented.

---

## 9. Best Practices

- **Lower MTTR aggressively** — fast, automatic recovery often beats preventing failure.
- **Make operations idempotent** so retries are always safe.
- **Replicate data** across nodes/zones; never trust a single copy.
- **Use atomic transactions** to prevent half-applied, corrupting writes.
- **Detect failures fast** with health checks, heartbeats, and timeouts.
- **Automate recovery** — failover, self-healing, instant rollback.
- **Test failure deliberately** — chaos engineering and DR drills, regularly.
- **Back up, and verify restores** — an unverified backup is a liability.

---

## 10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Non-idempotent retries | Duplicate charges, double-processing | Idempotency keys on all critical writes |
| Single copy of data | Permanent loss on disk failure | Replicate + back up |
| No atomic transactions | Corrupted half-written state | Wrap multi-step writes in transactions |
| Never testing failover | Failover fails when it matters | Regular DR drills + chaos testing |
| Confusing available with reliable | "Up" but serving wrong data | Monitor correctness, not just uptime |
| Slow manual recovery | Long outages (high MTTR) | Automate detection and recovery |

---

## 11. Interview Questions

1. What is the difference between reliability and availability?
2. Define MTBF and MTTR. How do they relate to availability?
3. Why is idempotency critical in distributed systems? Give an example.
4. How would you make a payment API safe against duplicate requests?
5. What does "eleven nines of durability" mean, and how is it achieved?
6. Why is lowering MTTR often better than raising MTBF?
7. What is chaos engineering and why does it improve reliability?

---

## 12. Summary

| Concept | Key Takeaway |
|---|---|
| **Reliability** | Correct and consistent results over time — deeper than uptime. |
| **MTBF / MTTR** | Fail less often, OR recover faster. Recovery is usually cheaper. |
| **Durability** | Confirmed data survives any failure. Measured in nines. |
| **Idempotency** | Makes retries safe — the key to distributed reliability. |
| **Atomicity** | All-or-nothing writes prevent corruption. |
| **Testing** | Chaos engineering + DR drills; untested mechanisms fail. |

---

## 13. Cross References

**Prerequisites:** System Design Fundamentals · Availability (NFR #2)

**Related Topics:** Fault Tolerance (NFR #6) · Consistency (NFR #5) · Replication · Disaster Recovery (RTO/RPO)

**What to Learn Next:** Consistency (NFR #5) · Fault Tolerance (NFR #6)

---

*System Design Engineering Handbook — NFR Series*