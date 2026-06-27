# 02 — CAP Theorem

> **06-Distributed-Systems Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is the CAP Theorem?

The CAP theorem, stated by Eric Brewer in 2000 and formally proved by Gilbert and Lynch in 2002, says:

> **A distributed system can guarantee at most two of three properties simultaneously: Consistency, Availability, and Partition Tolerance.**

These three properties are defined precisely:

| Property | Definition |
|---|---|
| **Consistency (C)** | Every read receives the most recent write or an error. All nodes see the same data at the same time. |
| **Availability (A)** | Every request receives a response — not an error. The response may not be the most recent write. |
| **Partition Tolerance (P)** | The system continues operating even when network partitions occur — when some nodes cannot communicate with others. |

---

## 2. Why Partition Tolerance Is Not Optional

Before discussing the trade-off, understand why you can't simply drop P.

A **network partition** is when the network between nodes breaks — some nodes can communicate with each other but not with others. This is not a rare edge case. Network partitions happen constantly in production: a switch reboots, a cable is cut, a cloud availability zone has a networking issue, a firewall drops packets.

```
Normal:               Partition:
  A ←→ B ←→ C          A ←→ B     C (isolated)
  (all connected)       (B and C can't talk)
```

In any distributed system that runs across multiple machines, you **cannot prevent** network partitions. You can reduce their frequency and duration, but you cannot eliminate them.

Therefore: **P is mandatory.** It's not a choice. The real choice is between C and A — what does your system do *when* a partition occurs?

> **The CAP theorem is really asking: during a network partition, do you prioritise Consistency or Availability?**

---

## 3. The Real Trade-off: CP vs AP

### CP — Consistency + Partition Tolerance

When a partition occurs, the system **refuses to serve requests** rather than risk returning inconsistent data. It sacrifices availability to preserve correctness.

```
Partition occurs: Node A can't reach Node B

CP system behaviour:
  Request arrives at Node A: "What is the balance of account 123?"
  Node A: "I can't confirm I have the latest data — refusing to answer."
  → Returns error or times out
  → User sees an error

Guarantee: Any data returned is correct and up-to-date.
Cost: Some requests fail during a partition.
```

**When to choose CP:**
- Financial systems: a stale bank balance could allow overdrafts
- Inventory: selling an item that's already sold is a real-world problem
- Booking systems: double-booking a seat or hotel room
- Distributed locks and coordination: correctness is critical

**Examples:** HBase, ZooKeeper, etcd, traditional RDBMS clusters

### AP — Availability + Partition Tolerance

When a partition occurs, the system **keeps responding** — but may return stale or inconsistent data. It sacrifices consistency to preserve availability.

```
Partition occurs: Node A can't reach Node B

AP system behaviour:
  Request arrives at Node A: "What is the balance of account 123?"
  Node A: "Last I knew it was £350." (may be stale)
  → Returns £350
  → User gets a response

Guarantee: Always responds.
Cost: Response may not reflect the most recent writes.
```

**When to choose AP:**
- Social media feeds: seeing a post 2 seconds later is fine
- Product catalogues: showing a slightly stale price is acceptable
- DNS: must always respond; slight staleness is tolerable
- Shopping carts: availability matters more than perfect consistency

**Examples:** Cassandra, DynamoDB (default), CouchDB, Riak

---

## 4. Visualising the Trade-off

```
          Consistency
               │
               │
        CP     │     CA (not useful in
      systems  │   distributed systems)
               │
───────────────┼──────────────────── Availability
               │
        (P is  │   AP
      mandatory│  systems
       always) │
               │
          Partition
          Tolerance
```

The CA corner (consistency + availability without partition tolerance) is only achievable in a single-node system or a system connected by a perfectly reliable network. Neither exists in practice at scale.

---

## 5. PACELC — The Fuller Model

CAP only describes behaviour **during** a partition. Daniel Abadi proposed PACELC (2012) to capture the trade-off that exists **even without a partition** — the latency vs consistency trade-off.

```
P → A or C       (during a Partition, choose Availability or Consistency)
E → L or C       (Else — during normal operation — choose Latency or Consistency)
```

**Why PACELC matters:** Partitions are rare. The L vs C trade-off happens on **every single request**. Strong consistency requires waiting for multiple nodes to agree — this adds latency. Eventual consistency returns immediately — faster but potentially stale.

| System | During Partition | Normal Operation | Classification |
|---|---|---|---|
| **Cassandra** | Availability | Latency (low) | PA/EL |
| **DynamoDB (default)** | Availability | Latency | PA/EL |
| **PostgreSQL (single node)** | Consistency | Consistency | — |
| **Google Spanner** | Consistency | Consistency | PC/EC |
| **ZooKeeper** | Consistency | Consistency | PC/EC |
| **MongoDB (default)** | Availability | Latency | PA/EL |

> **PACELC is more useful than CAP for everyday design decisions** because most design decisions happen in normal operation, not during partitions.

---

## 6. Consistency Models — A Spectrum

The CAP theorem's "C" is binary — either you have strong consistency or you don't. But in practice, consistency is a spectrum. Understanding where a system sits on this spectrum is crucial.

```
STRONG ◄────────────────────────────────────────► WEAK
(correct, expensive)                     (fast, may be stale)

Linearisability → Sequential → Causal → Read-your-writes → Eventual
```

| Model | Guarantee | Cost |
|---|---|---|
| **Linearisability** | Every operation appears to happen atomically at a single point in time. The strongest guarantee. | Very high latency — requires coordination |
| **Sequential consistency** | All nodes see operations in the same order, but not necessarily real-time | High |
| **Causal consistency** | Causally-related operations seen in order; unrelated may differ | Medium |
| **Read-your-writes** | You always see your own writes immediately | Low — partial guarantee only |
| **Eventual consistency** | All replicas converge to the same value — eventually | Lowest — fastest |

**Linearisability** is the strongest. It means every read returns the result of the most recent write — across all nodes — as if the system were a single machine. This is what "strong consistency" means in practice.

**Eventual consistency** is the weakest meaningful guarantee. It says: if writes stop, all replicas will eventually agree. "Eventually" in practice usually means milliseconds to seconds.

---

## 7. Real Systems — Where They Sit

| System | Consistency Model | Why |
|---|---|---|
| **Cassandra** | Tunable (eventual by default) | Designed for AP; quorum reads give stronger consistency |
| **DynamoDB** | Eventually consistent (default) / Strongly consistent (option) | AP by default; strong consistency costs 2× reads |
| **ZooKeeper** | Linearisable | Used for coordination; correctness is non-negotiable |
| **Google Spanner** | Linearisable (globally) | Uses TrueTime (atomic clocks) to achieve global consistency |
| **PostgreSQL (single)** | Linearisable | Single node = no partition problem |
| **MongoDB** | Tunable | Eventual by default; can configure majority reads |

> **Inferred:** Internal consistency mechanisms vary; the classifications above are based on public documentation and research papers.

---

## 8. Applying CAP in System Design Interviews

When designing a system, explicitly state your consistency choice and justify it. Interviewers test whether you can reason through this trade-off rather than blindly choosing one.

**Template:**

```
"For [component], I'm choosing [CP/AP] because:
- The data involved is [financial/inventory/social/...] 
- The cost of [stale data/unavailability] is [unacceptable/acceptable] because [reason]
- Therefore I need [strong/eventual] consistency
- I'll implement this with [specific mechanism]"
```

**Example — ride-sharing system:**

```
Driver location (AP):
  Stale location by 5 seconds = harmless (slightly imprecise ETA)
  Availability = critical (driver app must always update location)
  → Eventual consistency, write to Cassandra

Payment deduction (CP):
  Double charge or missed deduction = real financial error
  Brief unavailability = acceptable (retry in seconds)
  → Strong consistency, write to PostgreSQL with ACID transaction
```

---

## 9. The CAP Theorem Misconception

A common misconception: "CP means no availability, AP means no consistency."

This is wrong. CAP only defines behaviour **during a network partition**, which is a failure mode — not the normal operating condition. When the network is healthy (99.9%+ of the time), both CP and AP systems provide both consistency and availability.

```
No partition (normal operation):
  CP system: consistent AND available ✅
  AP system: consistent AND available ✅

During partition (failure):
  CP system: consistent, NOT available for affected nodes ⚠️
  AP system: available, NOT consistent (may serve stale) ⚠️
```

The choice of CP vs AP only matters when something goes wrong. Choose based on which failure mode your business can better tolerate.

---

## 10. Best Practices

- **Default to AP for most user-facing features** — availability matters more than perfect freshness for social feeds, profiles, search.
- **Use CP for money, inventory, and any resource that can be over-allocated** — the cost of inconsistency is real and immediate.
- **Use tunable consistency** (Cassandra, DynamoDB) to make per-operation decisions rather than committing the entire system to one model.
- **State your consistency choice explicitly in design documents** — it should not be implicit.
- **Remember PACELC** — the latency vs consistency trade-off exists on every request, not just during partitions.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Treating CAP as a choice between all three | Misunderstanding the theorem — P is not optional | P is mandatory; the real choice is C vs A during partitions |
| Choosing strong consistency everywhere | Latency and availability cost on every request | Use strong consistency only where data correctness is critical |
| Choosing eventual consistency for financial data | Stale reads cause overdrafts, over-sells, double-bookings | CP / strong consistency for money and inventory |
| Not knowing your database's default consistency model | Assuming strong consistency when the DB defaults to eventual | Read the documentation; know your defaults |
| Conflating CAP's "C" with ACID's "C" | Different concepts cause design confusion | CAP C = replica agreement; ACID C = constraint validity |

---

## 12. Interview Questions

1. State the CAP theorem. What do each of the three letters mean precisely?
2. Why is partition tolerance not optional in distributed systems?
3. What is the difference between a CP and an AP system? Give an example of each.
4. What does PACELC add to the CAP theorem?
5. What is linearisability? How does it differ from eventual consistency?
6. A payment system must never double-charge. Which side of CAP should it be on?
7. Cassandra is an AP system. How can you get stronger consistency from it?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **CAP** | C + A + P: pick two. P is mandatory. Real choice is C vs A during partitions. |
| **CP** | Refuse to serve stale data. Correctness over availability. |
| **AP** | Always respond. Accept possible staleness. |
| **PACELC** | Latency vs Consistency on every request — more relevant than CAP in normal operation. |
| **Linearisability** | Strongest consistency. Every read sees the latest write. Costs latency. |
| **Eventual** | Weakest guarantee. Always available. Replicas converge over time. |
| **Key insight** | CAP only governs during partition. Both CP and AP are fully available in normal operation. |

---

## 14. Cross References

**Prerequisites:** 01-distributed-systems-fundamentals.md · Consistency (NFR #5)

**Related Topics:** 03-consistency-and-clocks.md · Replication (DB #5) · NoSQL (DB #3)

**What to Learn Next:** 03-consistency-and-clocks.md

---

*System Design Engineering Handbook — 06-Distributed-Systems Series*