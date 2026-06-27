# Consistency

> **NFR Deep Dive #5** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Overview

Consistency means that all parts of a distributed system agree on the **same view of the data**. When data is replicated across many nodes, consistency defines *what value a read returns* after a write — especially whether it can return stale data.

> **The core tension:** In a single database, consistency is free. The moment you replicate data across nodes (for availability and scale), keeping every copy identical becomes a fundamental, unavoidable trade-off against availability and latency.

> ⚠️ **Note:** "Consistency" means two different things depending on context — the **C in ACID** (transactional integrity) and the **C in CAP** (replica agreement). They are not the same. Both are covered below.

---

## 2. The Consistency Spectrum

Consistency is not binary — it's a spectrum from strict to loose. Stronger consistency costs more latency and availability.

```
STRONG ←──────────────────────────────────────────→ WEAK
(correct, slow)                              (fast, may be stale)

Linearizable → Sequential → Causal → Eventual
```

| Model | Guarantee | Cost | Use Case |
|---|---|---|---|
| **Strong / Linearizable** | Every read sees the most recent write, instantly, everywhere | Highest latency; lower availability | Bank balances, inventory, locks |
| **Sequential** | All nodes see operations in the same order (not necessarily real-time) | High | Distributed logs |
| **Causal** | Causally-related operations seen in order; unrelated ones may differ | Medium | Comment threads, chat |
| **Eventual** | If writes stop, all replicas *eventually* converge | Lowest; highest availability | Social feeds, DNS, view counts |

> **Eventual consistency** is the most common model at scale. "Eventually" usually means milliseconds — but the system makes no guarantee about *when* during that window a read might see old data.

---

## 3. Strong vs Eventual — The Practical Trade-off

```
STRONG CONSISTENCY
  Write → replicate to all → confirm → only then readable
  ✅ Always correct          ❌ Slower; unavailable if a replica is unreachable

EVENTUAL CONSISTENCY
  Write → confirm immediately → replicate in background
  ✅ Fast; always available  ❌ Reads may briefly return stale data
```

**Concrete example — a "like" counter:**
- *Eventual:* You like a post; the count updates on your screen but a friend sees the old count for 2 seconds. Harmless. Choose availability.
- *Strong:* You withdraw $100; every node must agree the balance dropped *before* allowing another withdrawal. Stale data = overdraft. Choose consistency.

> **The decision rule:** Ask "what is the cost of a user seeing slightly stale data?" If it's harmless (a like count) → eventual. If it's dangerous (a bank balance) → strong.

---

## 4. CAP Theorem — Why You Must Choose

During a **network partition** (nodes can't talk to each other), a distributed system can guarantee only **two of three**: Consistency, Availability, Partition tolerance.

```
        Consistency (C)
            /\
           /  \
          / CP \   ← reject requests to stay correct
         /      \
        /────────\
       / AP       \  ← serve stale data to stay up
      /____________\
 Availability(A)  Partition tolerance(P)
```

**Since network partitions are unavoidable in distributed systems, P is mandatory.** The real choice is C vs A *during a partition*:

| Choice | During a partition... | Examples |
|---|---|---|
| **CP** (consistency) | Refuse to respond rather than return wrong data | Banking, bookings, ZooKeeper, etcd |
| **AP** (availability) | Keep responding, accept temporary staleness | Cassandra, DynamoDB, DNS, shopping carts |

> When the network is healthy (no partition), you get both C and A. CAP only forces the choice *during* a partition.

---

## 5. PACELC — The Fuller Picture

CAP only describes behavior *during* partitions. PACELC adds what happens normally:

```
if Partition (P):  choose Availability (A) or Consistency (C)
Else (E):          choose Latency (L)      or Consistency (C)
```

> **The everyday trade-off is L vs C.** Even with no partition, strong consistency costs latency (you must wait for replicas to agree). This is the trade you actually pay every day — partitions are rare; latency is constant.

---

## 6. ACID vs BASE — Two Philosophies

| | ACID (strong) | BASE (eventual) |
|---|---|---|
| **Stands for** | Atomicity, Consistency, Isolation, Durability | Basically Available, Soft state, Eventual consistency |
| **Philosophy** | Correctness first | Availability first |
| **Typical DB** | Relational (PostgreSQL, MySQL) | NoSQL (Cassandra, DynamoDB) |
| **Best for** | Financial, transactional data | High-scale, high-availability data |

**ACID's "C"** = a transaction moves the DB from one *valid state* to another (constraints, foreign keys hold). This is about **transactional integrity**, different from CAP's "replicas agree."

---

## 7. Techniques to Manage Consistency

| Technique | What It Does | Trade-off |
|---|---|---|
| **Quorum (R + W > N)** | Read/write enough replicas to guarantee overlap | Tunable consistency vs latency |
| **Two-phase commit (2PC)** | Coordinator ensures all nodes commit or none do | Strong, but slow and blocks on coordinator failure |
| **Consensus (Raft, Paxos)** | Nodes agree on a single value despite failures | Strong, fault-tolerant, but complex |
| **Versioning / vector clocks** | Detect and resolve conflicting concurrent writes | App must handle conflict resolution |
| **Read-your-own-writes** | A user always sees their *own* updates immediately | Partial guarantee; simpler than full strong |

### Quorum intuition

```
N = total replicas, W = nodes to confirm a write, R = nodes to read

If  R + W > N  →  read and write sets always overlap
                 →  a read is guaranteed to see the latest write.

Example: N=3, W=2, R=2 → 2+2 > 3 → strong consistency, still tolerates 1 node down.
```

---

## 8. How Large Companies Apply This

| Company | Application | Source |
|---|---|---|
| **Amazon (DynamoDB)** | Tunable consistency — choose eventual (fast) or strong (correct) per read | AWS public docs |
| **Google (Spanner)** | Globally *strong* consistency using synchronized atomic clocks (TrueTime) | Public research paper |
| **Cassandra users** | Tunable per-query consistency levels (ONE, QUORUM, ALL) | Apache Cassandra docs |
| **Banking systems** | Strong consistency mandatory — stale balance = real money lost | Industry standard |

> **Inferred:** Most large systems mix models — strong consistency for money/inventory, eventual for feeds/counts/recommendations.

---

## 9. Best Practices

- **Match the model to the data.** Strong for money/inventory; eventual for feeds/counts.
- **Default to eventual at scale** unless correctness genuinely demands strong.
- **Use read-your-own-writes** to hide eventual consistency from the acting user.
- **Use quorums** to tune the consistency/latency balance precisely.
- **Make conflict resolution explicit** in eventually-consistent stores (last-write-wins, merge, etc.).
- **Don't pay for strong consistency you don't need** — it costs latency on every request.

---

## 10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Strong consistency everywhere | Needless latency, lower availability | Reserve strong for data that truly needs it |
| Eventual consistency for money | Overdrafts, oversold inventory, double-spends | Strong consistency for financial/inventory data |
| Ignoring replication lag | Users don't see their own just-made change | Read-your-own-writes; read from primary after write |
| No conflict resolution strategy | Concurrent writes silently lose data | Define merge/versioning rules upfront |
| Confusing ACID-C with CAP-C | Wrong reasoning about guarantees | Keep transactional integrity vs replica agreement separate |

---

## 11. Interview Questions

1. Explain the consistency spectrum from strong to eventual.
2. State the CAP theorem. Why is partition tolerance non-negotiable?
3. When would you choose AP over CP? Give a concrete example of each.
4. What does PACELC add over CAP?
5. Explain how quorum (R + W > N) gives strong consistency.
6. Difference between ACID consistency and CAP consistency?
7. How would you hide eventual consistency from a user who just posted a comment?

---

## 12. Summary

| Concept | Key Takeaway |
|---|---|
| **Consistency** | All replicas agree on the same data view. |
| **Spectrum** | Strong (correct, slow) → Eventual (fast, may be stale). |
| **CAP** | During a partition, choose Consistency *or* Availability. |
| **PACELC** | Even without partitions, trade Latency vs Consistency daily. |
| **Quorum** | `R + W > N` guarantees a read sees the latest write. |
| **ACID vs BASE** | Correctness-first vs availability-first philosophies. |
| **Rule** | Match the model to the cost of stale data. |

---

## 13. Cross References

**Prerequisites:** System Design Fundamentals · Availability (NFR #2) · Reliability (NFR #4)

**Related Topics:** CAP Theorem (deep dive) · Replication · Distributed Transactions · Consensus (Raft/Paxos)

**What to Learn Next:** Fault Tolerance (NFR #6) · Distributed Consensus

---

*System Design Engineering Handbook — NFR Series*