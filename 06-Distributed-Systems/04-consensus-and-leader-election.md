# 04 — Consensus & Leader Election

> **06-Distributed-Systems Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Consensus?

Consensus is the problem of getting multiple nodes in a distributed system to **agree on a single value** — even when some nodes may fail or messages may be lost.

This sounds simple. In practice, it is one of the hardest problems in distributed computing. The difficulty comes from the same challenges we identified earlier: partial failures, unreliable networks, no global clock.

```
3 nodes must agree on the next value to write:

Node A proposes: "value = 5"
Node B proposes: "value = 7"
Node C has crashed

What value do A and B agree on?
What if A and B can't communicate with each other?
What if C recovers after they've decided?
```

Consensus protocols answer these questions in a way that guarantees:
- **Agreement:** All non-faulty nodes decide on the same value
- **Validity:** The decided value was proposed by some node (not invented)
- **Termination:** All non-faulty nodes eventually decide (no infinite wait)

---

## 2. Why Consensus Is Needed

Consensus is the foundation of many distributed system components you use every day.

| Use Case | Why Consensus Is Needed |
|---|---|
| **Leader election** | All nodes must agree on exactly one leader |
| **Distributed locking** | All nodes must agree on who holds the lock |
| **Atomic broadcast** | All nodes must receive the same messages in the same order |
| **Replicated state machine** | All replicas must apply the same operations in the same order |
| **Database replication** | All replicas must agree on the order of writes |
| **Configuration management** | All services must see the same configuration at the same time |

> **Without consensus, there is no reliable coordination in distributed systems.** Every form of distributed agreement — from electing a database primary to deciding which service instance handles a job — requires solving consensus.

---

## 3. The FLP Impossibility

Before diving into solutions, understand a fundamental theoretical result:

**Fischer, Lynch, and Paterson (1985) proved:** In an asynchronous distributed system where even one process may fail, no deterministic consensus algorithm can guarantee termination.

```
Translation: if you can't bound message delivery time (asynchronous),
and even one node might crash, you cannot build a consensus algorithm
that always terminates.
```

This sounds devastating. In practice it isn't, because:
- Real systems have partial synchrony — most of the time, messages arrive within a bounded window
- Practical consensus algorithms (Raft, Paxos) guarantee termination under partial synchrony
- They sacrifice termination during extreme conditions (prolonged partitions) but work correctly in normal operation

> **FLP says you can't have all three simultaneously: safety, liveness, and fault tolerance under full asynchrony.** Practical protocols choose to sacrifice liveness (may not terminate) during extreme failures while preserving safety (never produce wrong results).

---

## 4. Paxos — The Classic Protocol

Paxos, introduced by Leslie Lamport in 1989, was the first practical consensus algorithm. It is notoriously difficult to understand but foundational. Most production consensus algorithms are variants of Paxos.

### The Three Roles

- **Proposer:** Initiates the consensus process by proposing a value
- **Acceptor:** Votes on proposals; stores the agreed value
- **Learner:** Learns the outcome (often the same node plays multiple roles)

### The Two Phases

**Phase 1 — Prepare/Promise:**

```
Proposer → Acceptors: "Prepare(n)" (n = unique proposal number)
Acceptors → Proposer: "Promise to not accept proposals < n"
                      + "Here's the highest proposal I've already accepted"
```

**Phase 2 — Accept/Accepted:**

```
Proposer → Acceptors: "Accept(n, value)"
  (value = either the proposer's choice, or the highest previously-accepted value)
Acceptors → Proposer: "Accepted"
  (if they haven't promised a higher-numbered proposal)

If a majority accept → consensus achieved on (n, value)
```

### Why Paxos Is Hard

- Multi-Paxos (for a sequence of values) requires significant extensions
- Leader duels: two proposers competing with increasing proposal numbers, causing livelock
- Gap handling: if a slot in the log is skipped, it must be filled
- The original paper leaves many engineering decisions unspecified

> **No one implements vanilla Paxos.** Production systems use Multi-Paxos, Fast Paxos, or Raft (which reframes Paxos in a more understandable way).

---

## 5. Raft — The Understandable Consensus Algorithm

Raft was designed by Diego Ongaro and John Ousterhout (2014) explicitly to be understandable. It provides the same guarantees as Multi-Paxos with a clearer decomposition.

### Core Concept: Replicated Log

Raft maintains a **replicated log** across all nodes. The log contains commands that all state machines execute in the same order. All nodes applying the same log in the same order produce the same state — this is the foundation of replicated state machines.

```
All nodes have the same log:
[write x=1, write y=2, write x=3, delete z, ...]

Applying this log in order produces the same state on every node.
→ All nodes are identical copies of each other.
```

### The Three Node States

Every Raft node is in one of three states:

```
FOLLOWER → (election timeout) → CANDIDATE → (majority votes) → LEADER
   ↑                                                              │
   └──────────────── (heartbeat from leader) ───────────────────┘
```

**Leader:** Handles all client requests. Replicates log entries to followers. Sends periodic heartbeats to prevent elections.

**Follower:** Passive. Accepts log entries from leader. Votes in elections. If no heartbeat received within timeout → becomes candidate.

**Candidate:** Requests votes from all nodes. If majority vote → becomes leader. If another leader discovered → reverts to follower.

### Leader Election — Step by Step

```
1. System starts. All nodes are followers.
2. Election timeout fires on Node A (randomised — typically 150–300ms).
3. Node A increments term number, votes for itself, sends RequestVote to all.
4. Other nodes receive RequestVote:
   - If they haven't voted in this term → grant vote
   - If they've seen a higher term → update and grant
5. Node A receives majority votes → becomes leader for this term.
6. Node A sends heartbeats immediately to suppress new elections.
```

**The randomised timeout is critical:** Each node waits a random duration before starting an election. This dramatically reduces the chance of two nodes starting elections simultaneously (split vote).

```
Without randomisation: nodes A, B, C all time out at the same time
→ split vote → no majority → election fails → repeat indefinitely

With randomisation (150ms–300ms):
Node A times out at 150ms → starts election → usually wins before B or C time out
```

### Log Replication — Step by Step

```
1. Client sends command to leader.
2. Leader appends command to its local log (not yet committed).
3. Leader sends AppendEntries RPC to all followers.
4. Followers append to their logs, respond "success."
5. Once a majority have appended → leader commits the entry.
6. Leader applies to state machine, responds to client.
7. Leader notifies followers of commit in next heartbeat.
8. Followers apply committed entries to their state machines.
```

**Commit rule:** An entry is committed only when a majority of nodes have it in their logs. This guarantees that a committed entry is never lost — even if the leader crashes immediately after committing.

```
5 nodes. Leader commits entry E after receiving ACK from 3 nodes (majority).
Leader crashes immediately.

Remaining 4 nodes elect a new leader.
The new leader must have E in its log (it was on 3 of 5 nodes before crash).
→ E is safe. It will be applied on all nodes.
```

### Safety Guarantee

Raft's core safety property: **once a value is committed, no future leader will ever overwrite it.** This is enforced by the election rule: a candidate cannot be elected leader unless its log is at least as up-to-date as any node with a majority.

---

## 6. Split Brain — The Dangerous Scenario

Split brain occurs when a network partition creates two groups, each electing its own leader and accepting writes independently. When the partition heals, the system has two conflicting histories.

```
5 nodes: A(leader), B, C, D, E
Network partition:
  Group 1: A, B (2 nodes)
  Group 2: C, D, E (3 nodes)

Group 2 elects C as new leader (majority).
Both A and C now accept writes.
Both think they are the legitimate leader.

Partition heals:
  A has: [write x=5]
  C has: [write x=7]
  Who wins?
```

**How Raft prevents split brain:**

A leader can only commit entries with acknowledgement from a majority. Group 1 (A, B) has only 2 of 5 nodes — not a majority. A cannot commit any new entries. Group 2 (C, D, E) has 3 of 5 — a majority. C can commit normally.

When the partition heals, A discovers a higher term number from C → A steps down → C's log wins.

> **Raft's quorum requirement is the split-brain prevention mechanism.** A leader who can't reach a majority cannot commit — it is effectively deposed even if it doesn't know it yet.

---

## 7. Real Systems Using Consensus

| System | Protocol | Use |
|---|---|---|
| **etcd** | Raft | Kubernetes cluster state; service discovery; distributed locking |
| **ZooKeeper** | ZAB (Paxos variant) | Distributed coordination; Kafka controller election historically |
| **CockroachDB** | Raft (per shard) | Distributed SQL; each shard uses Raft for replication |
| **Google Chubby** | Paxos | Distributed lock service; Bigtable master election |
| **Consul** | Raft | Service discovery; distributed key-value; leader election |

---

## 8. Best Practices

- **Use an existing consensus system** (etcd, ZooKeeper) rather than implementing your own — consensus is extremely difficult to get right.
- **Understand quorum size** — with N nodes, you need N/2+1 for a majority. Factor this into your deployment decisions (odd number of nodes is conventional: 3 or 5).
- **3 nodes for fault tolerance of 1; 5 nodes for fault tolerance of 2** — more nodes means slower consensus.
- **Don't put consensus on the critical path** — use it for coordination (leader election, config), not for every request.
- **Use randomised election timeouts** to prevent split votes — if you're building on top of Raft.

---

## 9. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Implementing consensus yourself | Subtle bugs that only appear under specific failure conditions | Use etcd, ZooKeeper, or Consul |
| Even number of nodes in a Raft cluster | Ties in voting make leader election unstable | Use odd numbers: 3, 5, 7 |
| Too many Raft nodes (>7) | Consensus latency grows with cluster size | 3 or 5 nodes is the sweet spot for most use cases |
| Confusing leader election with load balancing | Leader does coordination work; load balancing distributes traffic | Separate concerns: consensus for coordination, LB for traffic |
| No monitoring of leader changes | Silent leadership instability goes undetected | Alert on leader election events and frequency |

---

## 10. Interview Questions

1. What is the consensus problem in distributed systems?
2. Why is consensus necessary for leader election?
3. Explain the two phases of Paxos at a high level.
4. What are the three node states in Raft? How do transitions happen?
5. What is split brain? How does Raft's quorum requirement prevent it?
6. Why does Raft use randomised election timeouts?
7. Why would you use etcd or ZooKeeper rather than implementing consensus yourself?

---

## 11. Summary

| Concept | Key Takeaway |
|---|---|
| **Consensus** | All nodes agree on one value despite failures. Hard. Foundational. |
| **FLP** | Under full asynchrony, no deterministic consensus can always terminate. Practical systems assume partial synchrony. |
| **Paxos** | Classic but hard. Two phases. Forms the basis of most practical protocols. |
| **Raft** | Understandable Paxos variant. Leader + log replication + randomised election. |
| **Leader election** | Randomised timeouts prevent split votes. Majority quorum required. |
| **Split brain** | Two leaders simultaneously. Quorum prevents commits without majority. |
| **Use existing systems** | etcd, ZooKeeper, Consul. Don't implement consensus yourself. |

---

## 12. Cross References

**Prerequisites:** 01-distributed-systems-fundamentals.md · 02-cap-theorem.md

**Related Topics:** Replication (DB #5) · Fault Tolerance (NFR #6) · Service Discovery (BB #7)

**What to Learn Next:** 05-consistent-hashing.md

---

*System Design Engineering Handbook — 06-Distributed-Systems Series*