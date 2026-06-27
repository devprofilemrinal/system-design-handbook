# 01 — Distributed Systems Fundamentals

> **06-Distributed-Systems Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is a Distributed System?

A distributed system is a collection of independent computers that appear to users as a single coherent system. The machines communicate over a network to coordinate work, share data, and collectively provide a service.

You interact with distributed systems constantly — Google Search, WhatsApp, Netflix, your bank. Each of these runs across hundreds or thousands of machines simultaneously.

The appeal is obvious: more machines means more capacity. But distribution introduces an entirely new class of problems that simply don't exist on a single machine.

> **The fundamental challenge:** A single machine either works or it doesn't. In a distributed system, components can partially fail, messages can be delayed or lost, and different parts of the system can disagree about what is true — all simultaneously, all silently.

---

## 2. The 8 Fallacies of Distributed Computing

Peter Deutsch at Sun Microsystems identified eight assumptions engineers incorrectly make when building distributed systems. Every one of these assumptions leads to production failures when violated.

| Fallacy | Reality |
|---|---|
| **1. The network is reliable** | Networks drop packets, connections time out, links fail. Always. |
| **2. Latency is zero** | Network calls take time. Cross-datacenter calls take 50–150ms. |
| **3. Bandwidth is infinite** | Bandwidth is finite and contended. Large payloads cause bottlenecks. |
| **4. The network is secure** | Networks are hostile. Encryption and auth are mandatory, not optional. |
| **5. Topology doesn't change** | Servers are added, removed, fail, and move. IPs change. |
| **6. There is one administrator** | Multiple teams own different parts. Coordination is hard. |
| **7. Transport cost is zero** | Serialisation, network I/O, and infrastructure all have real costs. |
| **8. The network is homogeneous** | Different machines, OS versions, library versions coexist. |

> **Internalise these.** Every distributed system design problem is ultimately caused by violating one of these assumptions. When a system fails in production, trace it back to which fallacy was ignored.

---

## 3. Why Distribution Is Hard — The Core Problems

### 3.1 Partial Failure

In a single machine, a failure is total — it either works or it crashes. In a distributed system, components can fail independently. A network call might:

- Succeed
- Fail (connection refused)
- Time out (you don't know if it succeeded or not)
- Succeed but the response is lost on the way back

```
Service A calls Service B:

A → request → B → processes → response →
                                         X (network drops response)
A: "Did B succeed? I don't know."
B: "I completed the work."

→ A retries. B processes twice. Duplicate side effect.
```

This is why **idempotency** is so critical in distributed systems — you must be able to safely retry any operation because you can never be certain the first attempt completed.

### 3.2 Unreliable Networks

Networks are asynchronous — there is no upper bound on how long a message might take to arrive, or whether it arrives at all. This makes it impossible to distinguish between:

- A slow response (the other node is busy)
- A dead node (the other node has crashed)
- A network partition (the nodes can't communicate but both are alive)

```
Timeout after 5 seconds — what happened?

Possibility A: Node crashed
Possibility B: Node is slow; response is still coming
Possibility C: Network partition; node is alive but unreachable
Possibility D: Response sent, lost in transit

You cannot tell these apart from the outside.
```

> **There is no way to know if a remote node has failed or is just slow.** This is why distributed systems design always involves timeouts, retries, and planning for the worst case.

### 3.3 No Global Clock

On a single machine, you have one clock. All events can be ordered by timestamp. In a distributed system, each machine has its own clock, and those clocks drift apart constantly — by milliseconds or more.

```
Machine A: event at timestamp 10:00:00.100
Machine B: event at timestamp 10:00:00.050

Did A's event happen before B's? You cannot tell — B's clock might be behind.
```

This makes it fundamentally impossible to determine the true ordering of events across machines using wall-clock time alone. This is why logical clocks exist — covered in file 03.

### 3.4 Consistency vs Availability Under Partitions

When a network partition occurs (two parts of the system can't communicate), every distributed system must make a choice: keep responding (availability) or refuse to respond until consistency is restored. There is no third option.

This is the CAP theorem — covered in depth in file 02.

---

## 4. Key Properties We Want From Distributed Systems

| Property | Meaning |
|---|---|
| **Scalability** | Handle increasing load by adding nodes |
| **Availability** | Respond to requests even when some components fail |
| **Consistency** | All nodes agree on the same data at the same time |
| **Fault tolerance** | Continue operating correctly despite partial failures |
| **Performance** | Low latency and high throughput at scale |

The hard truth: **these properties conflict**. You cannot have all of them simultaneously. Every distributed system is a set of deliberate trade-offs between them. Knowing which trade-offs are appropriate for a given system is the core skill of distributed systems design.

---

## 5. The Two Failure Models

Understanding how components fail shapes how you design for resilience.

### Crash-Stop (Fail-Stop)

A node either works correctly or stops completely. It does not send wrong data — it just stops responding. This is the easiest failure mode to handle.

```
Detection: timeout → assume dead → route around it
Recovery: restart the process or replace the node
```

Most server failures in practice are crash-stop: a process crashes, runs out of memory, or is killed.

### Byzantine Failure

A node behaves arbitrarily — it can send incorrect, contradictory, or malicious messages while appearing to be alive. Named after the Byzantine Generals Problem.

```
Node sends "value = 5" to some nodes and "value = 7" to others.
How do other nodes agree on the correct value when one node is lying?
```

Byzantine fault tolerance (BFT) is required in adversarial environments — blockchains, distributed voting, aerospace. It requires `3f + 1` nodes to tolerate `f` Byzantine failures, making it expensive.

> **Most distributed systems in software engineering assume crash-stop failures only.** Byzantine fault tolerance is for systems where nodes may be compromised by external attackers. Unless you're building a blockchain or aerospace system, you do not need BFT.

---

## 6. Timeouts — The Universal Tool

Since you can never know if a remote node has failed or is just slow, timeouts are the primary mechanism for detecting and handling failures in distributed systems.

```
Rule: Every network call must have a timeout.
      A call with no timeout can block a thread forever.

If timeout fires:
  → Treat as failure
  → Retry (if idempotent)
  → Circuit break (if repeatedly failing)
  → Return error or fallback
```

**Setting timeouts correctly:**

| Too short | False positives — treat slow-but-working nodes as dead |
|---|---|
| Too long | Real failures block resources for a long time |
| Right | Based on the P99 latency of the called operation + buffer |

```
If P99 of the DB query is 50ms:
  Timeout = 150–200ms (3–4× P99)
  → Catches real failures
  → Doesn't falsely trigger on slow-but-successful calls
```

---

## 7. Retries — Handle Them Carefully

When a call fails or times out, retrying seems obvious. But retries in distributed systems require careful design.

### Exponential Backoff

Don't retry immediately — each retry waits longer than the last. This prevents a recovering service from being instantly re-overwhelmed.

```
Attempt 1: fails → wait 1 second
Attempt 2: fails → wait 2 seconds
Attempt 3: fails → wait 4 seconds
Attempt 4: fails → wait 8 seconds
(cap at some maximum, e.g. 30 seconds)
```

### Jitter

Add randomness to retry delays. Without jitter, all clients that experienced the same failure retry at the same moment — creating a thundering herd that overwhelms the recovering service.

```
Without jitter: all 1,000 clients retry at exactly T+2s → 1,000 simultaneous hits
With jitter:    clients retry between T+1.5s and T+2.5s → spread out → manageable
```

### The Retry Budget

Don't retry indefinitely. Set a maximum retry count or time budget. After that, fail fast and return an error or fallback.

> **Only retry idempotent operations.** If an operation is not idempotent, retrying after a timeout (where you don't know if the first attempt succeeded) causes duplicate side effects. Fix the idempotency first; then retry safely.

---

## 8. The Two Generals Problem — Why Perfect Reliability Is Impossible

The Two Generals Problem is a thought experiment that proves a fundamental impossibility in distributed systems.

```
Two armies (A1 and A2) want to attack a city simultaneously.
They can only communicate by sending messengers through enemy territory.
Messengers can be captured (messages can be lost).

General A1 sends: "Attack at dawn"
General A2 receives it, sends confirmation.
A1 receives confirmation, sends confirmation of confirmation.
...

No matter how many confirmations are exchanged,
neither general can ever be CERTAIN the other received the last message.

→ Perfect agreement over an unreliable channel is impossible.
```

**The distributed systems implication:** You can never achieve 100% guarantee that two nodes have agreed on something over an unreliable network. Every protocol is an approximation — it reduces uncertainty to an acceptable level, but never to zero.

This is why:
- Exactly-once delivery is theoretically impossible without careful design
- Two-phase commit can block indefinitely
- All consensus protocols have failure cases

---

## 9. How These Concepts Appear in Interviews

When an interviewer asks you to design a distributed system, they are testing whether you understand:

| They ask | You should address |
|---|---|
| "How does service A call service B?" | Timeouts, retries, circuit breakers, idempotency |
| "What happens if a node fails?" | Partial failure, detection delay, failover |
| "How do you ensure data consistency?" | CAP trade-off, consistency model choice |
| "How do services agree on a leader?" | Consensus protocol (Raft/Paxos) |
| "How do you distribute load?" | Consistent hashing, partitioning |
| "How do you order events?" | Logical clocks, Kafka partitions |

---

## 10. Best Practices

- **Assume the network will fail** — design every cross-service call with timeouts and retries.
- **Make operations idempotent** — so retries are always safe.
- **Use exponential backoff with jitter** — prevents retry storms on recovering services.
- **Design for partial failure** — what does your system do when only one of five dependencies is down?
- **Embrace eventual consistency** where strong consistency isn't required — it's cheaper and more available.
- **Make failure visible** — timeouts, errors, and circuit-breaker trips must all be logged and alerted on.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No timeouts on network calls | Thread pool exhaustion; cascading failure | Every call has an explicit timeout |
| Retrying non-idempotent operations | Duplicate side effects (double charges) | Make operations idempotent first |
| Assuming clocks are synchronised | Wrong event ordering; stale-data bugs | Use logical clocks or version numbers |
| Ignoring partial failure | System behaves incorrectly when a dependency is degraded | Design and test for every failure mode |
| Treating distributed systems like a single machine | Missing entire classes of failure | Explicitly design for network unreliability |

---

## 12. Interview Questions

1. What is a distributed system and what makes it hard?
2. Name three of the eight fallacies of distributed computing and explain why each matters.
3. What is partial failure? Why is it harder to handle than a total failure?
4. Why can't you use wall-clock timestamps to order events across distributed nodes?
5. What is the Two Generals Problem and what does it tell us?
6. Why must retries use exponential backoff with jitter?
7. What is the difference between crash-stop and Byzantine failure?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Distributed system** | Independent computers appearing as one. Distribution introduces a new class of failures. |
| **8 Fallacies** | The network is not reliable, zero-latency, secure, or infinite. Violating these causes production failures. |
| **Partial failure** | Components fail independently. You can't always tell if a call succeeded. |
| **No global clock** | Clocks drift. Wall-clock time cannot order distributed events. |
| **Timeouts** | The universal tool. Every network call must have one. |
| **Retries** | Only for idempotent ops. Exponential backoff + jitter always. |
| **Two Generals** | Perfect agreement over unreliable network is impossible. |

---

## 14. Cross References

**Prerequisites:** System Design Fundamentals · Availability (NFR #2) · Fault Tolerance (NFR #6)

**Related Topics:** Consistency (NFR #5) · Replication (DB #5) · Messaging Fundamentals

**What to Learn Next:** 02-cap-theorem.md · 03-consistency-and-clocks.md

---

*System Design Engineering Handbook — 06-Distributed-Systems Series*