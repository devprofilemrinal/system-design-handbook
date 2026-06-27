# 03 — Consistency & Clocks

> **06-Distributed-Systems Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. The Problem With Time in Distributed Systems

On a single machine, time is simple: one clock, one sequence of events, clear before/after relationships. In a distributed system, every machine has its own clock, and those clocks are never perfectly synchronised.

```
Machine A: event at 10:00:00.100
Machine B: event at 10:00:00.050

Did A's event happen before B's? Not necessarily.
B's clock might be running 200ms behind A's.
The real order might be: A happened first, B happened second.
```

This is not a solvable hardware problem. Physical clocks drift constantly due to temperature, manufacturing variance, and network synchronisation delays. Even with NTP (Network Time Protocol), clocks across machines can differ by milliseconds to hundreds of milliseconds.

> **You cannot use wall-clock time to establish the causal order of events across distributed nodes.** This is a fundamental constraint — not a limitation to be fixed with better hardware.

---

## 2. Logical Clocks — Lamport Timestamps

Leslie Lamport introduced logical clocks in 1978 to solve the ordering problem without relying on physical time. The core insight: we don't need to know *when* something happened — we only need to know *whether* event A happened before event B.

### The Happens-Before Relationship (→)

Event A **happens-before** event B (written A → B) if:
- A and B are in the same process, and A occurs before B in program order, OR
- A is the sending of a message, and B is the receipt of that message, OR
- There exists some C such that A → C and C → B (transitivity)

If neither A → B nor B → A, then A and B are **concurrent** — they have no causal relationship.

### How Lamport Clocks Work

Each process maintains a counter. Rules:

1. Increment your counter before each event
2. When sending a message, include your current counter value
3. When receiving a message, set your counter to `max(local, received) + 1`

```
Process A:     1    2    3         6    7
                    │              ↑
                 send(2)       receive(5)
                    │              │
Process B:          ↓    3    4    5    6
                 receive(3)
                    │
                 send(3) → process B counter = max(2,3)+1 = 4... wait

Simpler example:
A: counter=1 → sends message with timestamp=1
B: receives, has counter=0 → sets to max(0,1)+1=2
B: counter=2 → sends with timestamp=2
A: receives, has counter=1 → sets to max(1,2)+1=3
```

**What Lamport clocks give you:** If A → B, then timestamp(A) < timestamp(B).

**What they don't give you:** The converse is not guaranteed. timestamp(A) < timestamp(B) does NOT mean A → B. Two concurrent events might have the same or different timestamps with no causal relationship.

---

## 3. Vector Clocks — Capturing Causality Precisely

Vector clocks extend Lamport clocks to capture causality exactly. Each process maintains a **vector** (array) of counters — one per process in the system.

### How Vector Clocks Work

Each process `i` maintains vector `V` where `V[j]` = the number of events from process `j` that process `i` knows about.

Rules:
1. Before each event, increment your own position: `V[i]++`
2. When sending a message, include your full vector
3. When receiving a message, merge: `V[j] = max(V_local[j], V_received[j])` for all j, then increment your own: `V[i]++`

```
3 processes: A, B, C
Initial vectors: A=[0,0,0], B=[0,0,0], C=[0,0,0]

A performs event: A=[1,0,0]
A sends message to B (with vector [1,0,0]):
  B receives: B=[max(0,1), max(1,0)+1, max(0,0)] = [1,1,0]
B sends message to C (with vector [1,1,0]):
  C receives: C=[1,1,1]

Now:
  A=[1,0,0]
  B=[1,1,0]
  C=[1,1,1]
```

**What vector clocks give you:**
- A → B (A happened before B) if and only if A's vector ≤ B's vector component-wise (and at least one component is strictly less)
- A and B are concurrent if neither vector dominates the other

```
A=[2,1,0], B=[1,2,0]
  A[0]=2 > B[0]=1  (A ahead in A's events)
  A[1]=1 < B[1]=2  (B ahead in B's events)
→ Neither dominates → A and B are concurrent (happened independently)
```

### Where Vector Clocks Are Used

- **Dynamo-style databases** (Amazon Dynamo, Riak) use vector clocks to detect conflicting writes — when two clients update the same key concurrently, the system detects the conflict via vector clock comparison
- **Distributed version control** (Git's ancestry tracking is conceptually similar)
- **Debugging distributed systems** — reconstructing causal chains of events

---

## 4. The Consistency Models Revisited

Building on CAP (file 02), here are the key consistency models with precise definitions:

### Linearisability (Strong Consistency)

Every operation appears to take effect atomically at a single point in time between its invocation and completion. From any observer's perspective, the system behaves like a single machine.

```
Write: set x=5 (completes at time T)
Any subsequent read from any node returns x=5

Timeline:
──────[write x=5]──────|──────[any read returns 5]──────
                        T
```

**Cost:** Every write must be acknowledged by a quorum before completing. Every read must verify it has the latest value. High latency. Low throughput.

**Used by:** Google Spanner (using TrueTime), ZooKeeper, etcd, single-node databases.

### Sequential Consistency

All nodes see all operations in the same order, but that order doesn't have to match real time. Operations from any single process appear in program order.

```
Process A: write x=1, write x=2
Process B: read x

Sequential consistency: B might read x=1 or x=2, but it will never see x=2 then x=1
(the order within A is preserved globally, but timing relative to B is flexible)
```

Less expensive than linearisability but still strong.

### Causal Consistency

If event A causally precedes event B (A → B), then every node sees A before B. Concurrent events (no causal relationship) can be seen in different orders by different nodes.

```
A posts comment: "I had a great idea"
B replies to A: "Tell me more"

Causal consistency: every node sees A's post before B's reply
(they're causally related; the order must be preserved)

A posts comment: "I had a great idea"
C (independently) posts: "Nice weather today"

These are concurrent → nodes may see them in different orders
```

More available than sequential consistency; sufficient for most social/collaboration use cases.

### Eventual Consistency

If no new writes occur, all replicas will eventually converge to the same value. No guarantee on when. No guarantee about intermediate states.

```
Write x=5 to Node A
Read x from Node B immediately after → might return old value
Read x from Node B 500ms later → probably returns 5
Read x from Node B 5000ms later → definitely returns 5 (partition healed)
```

Maximum availability and performance. Requires application logic to handle stale reads gracefully.

---

## 5. Conflict Resolution in Eventual Consistency

When two clients write to the same key concurrently (before replication catches up), both writes are accepted — creating a conflict. The system must resolve it.

### Last-Write-Wins (LWW)

Keep the write with the latest timestamp; discard the other.

```
Client A writes x=5 at timestamp 100
Client B writes x=7 at timestamp 102
→ Keep x=7 (latest timestamp wins)
```

**Problem:** Clocks drift. Timestamp 100 from A might actually be after timestamp 102 from B if A's clock is fast. LWW silently loses data.

### Vector Clock Conflict Detection

Use vector clocks (Section 3) to detect concurrent writes. Surface the conflict to the application for resolution.

```
Client A writes x=5 → vector clock [1,0]
Client B writes x=7 → vector clock [0,1]
Both are concurrent (neither vector dominates)
→ System stores both versions as a conflict
→ Application (or user) merges: "which is correct?"
```

### CRDT — Conflict-Free Replicated Data Types

Data structures mathematically guaranteed to merge without conflicts. The merge operation is designed so that any order of applying updates produces the same result.

```
Counter CRDT:
  Node A increments by 3
  Node B increments by 5
  Merge: sum of all increments = 8 (regardless of order)
  No conflict possible.

Set CRDT (add-only):
  Node A adds "apple"
  Node B adds "banana"
  Merge: {"apple", "banana"}
  No conflict possible.
```

CRDTs work for specific data types — counters, sets, registers. They don't solve all conflicts — complex business logic still needs application-level resolution.

---

## 6. Google Spanner — Linearisability at Global Scale

Google Spanner is worth understanding because it challenges the assumption that global consistency and high availability cannot coexist.

Spanner achieves **globally linearisable** transactions across datacenters using **TrueTime** — atomic clocks and GPS receivers in every datacenter that bound clock uncertainty to ~7ms.

```
TrueTime API:
  TT.now() returns an interval [earliest, latest]
  The true current time lies somewhere in this interval

Spanner commit rule:
  Before committing a transaction, wait until TT.now().earliest > commit timestamp
  This guarantees that by the time the commit is visible, its timestamp
  is definitively in the past for all nodes
```

By waiting out clock uncertainty at commit time (a few milliseconds), Spanner can assign globally-consistent timestamps and achieve linearisability without sacrificing availability in practice.

> **The lesson from Spanner:** Strong consistency has a real but manageable latency cost. At the scale of Google's infrastructure, global linearisability adds ~10ms of commit latency — often acceptable. The assumption that "distributed = eventual consistency" is wrong. It's a choice.

---

## 7. How Large Companies Apply This

| Company | Mechanism | Application | Source |
|---|---|---|---|
| **Amazon (Dynamo)** | Vector clocks | Detect conflicts in shopping cart writes | Amazon Dynamo paper (public) |
| **Google (Spanner)** | TrueTime + linearisability | Globally consistent financial transactions | Google Spanner paper (public) |
| **Cassandra** | Tunable consistency + LWW | Resolves concurrent writes by timestamp | Apache Cassandra docs |
| **Riak** | Vector clocks + CRDTs | Conflict detection and resolution for distributed data | Basho/Riak public docs |

---

## 8. Best Practices

- **Never use wall-clock time to order events across nodes** — use logical clocks or sequence numbers.
- **Use vector clocks when you need to detect concurrent writes** — LWW silently loses data.
- **Choose the weakest consistency model that your use case tolerates** — each step stronger costs latency and availability.
- **Design application code to handle stale reads** in eventually consistent systems — don't assume freshness.
- **Use CRDTs for collaborative, concurrent-write scenarios** where merge conflicts are inevitable.

---

## 9. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Using timestamps to order distributed events | Wrong ordering when clocks drift | Logical clocks or vector clocks |
| LWW without considering clock drift | Silent data loss on concurrent writes | Vector clocks + conflict detection |
| Assuming eventual = "a few milliseconds" | Stale reads during large partition or replication lag | Design for arbitrary lag; don't assume freshness window |
| Strong consistency everywhere | Latency cost on every request | Reserve strong consistency for operations that genuinely require it |

---

## 10. Interview Questions

1. Why can't you use wall-clock timestamps to order events across distributed nodes?
2. What is a Lamport clock? What does it guarantee?
3. What is a vector clock and what does it add over Lamport clocks?
4. What is linearisability? How does it differ from eventual consistency?
5. How does Amazon Dynamo handle concurrent writes to the same key?
6. What is a CRDT? Give an example of a problem it solves.
7. How does Google Spanner achieve global linearisability despite being a distributed system?

---

## 11. Summary

| Concept | Key Takeaway |
|---|---|
| **Clock drift** | Physical clocks diverge. Wall-clock time cannot order distributed events. |
| **Lamport clocks** | Logical counter. If A→B then timestamp(A) < timestamp(B). Not the converse. |
| **Vector clocks** | Per-process counters. Detect causality and concurrent writes precisely. |
| **Linearisability** | Every read sees the latest write. Single-machine illusion. Highest cost. |
| **Causal consistency** | Causally related events ordered everywhere. Concurrent events may differ. |
| **Eventual** | All replicas converge eventually. Maximum availability. No ordering guarantee. |
| **CRDT** | Data structures that merge without conflict. Works for counters, sets. |
| **Spanner** | Global linearisability via TrueTime. Strong consistency is not impossible. |

---

## 12. Cross References

**Prerequisites:** 01-distributed-systems-fundamentals.md · 02-cap-theorem.md

**Related Topics:** Consistency (NFR #5) · Replication (DB #5) · 04-consensus-and-leader-election.md

**What to Learn Next:** 04-consensus-and-leader-election.md

---

*System Design Engineering Handbook — 06-Distributed-Systems Series*