# 02 — Functional & Non-Functional Requirements

> **00-Introduction** — Engineering Handbook
> Language-agnostic · 8 min read

---

## 1. Why Requirements Come First

Every architectural mistake starts the same way: someone designed a system without fully understanding what it needed to do, or how well it needed to do it.

Requirements are not bureaucracy. They are the constraints that make every subsequent decision meaningful. Without them, you are guessing. With them, every architectural choice has a reason.

> **A design without requirements is just a diagram. Requirements are what give the diagram meaning.**

There are two types: **Functional Requirements** define what the system does. **Non-Functional Requirements** define how well it does it. You need both. Getting either wrong produces a system that either does the wrong thing, or does the right thing badly.

---

## 2. Functional Requirements — What the System Does

A functional requirement describes a specific capability the system must provide. It is always expressed from the perspective of what a user or external system can do with it.

### Characteristics of a Good FR

```
✓ Describes a behaviour, not an implementation
✓ Testable — you can verify it passes or fails
✓ Scoped — clear what is included and what is not
✓ Unambiguous — no room for two interpretations

Bad:  "The system should handle messages"
Good: "A user can send a text message (up to 500 characters)
       to any other registered user"

Bad:  "The system should be fast"
Good: "The search results page loads in under 200ms"  ← this is actually NFR
```

### How to Identify FRs

Ask: **"What can a user do with this system?"**

Every answer is a functional requirement candidate. Then scope it — which of those are in scope for this design? Which are explicitly out of scope?

```
For a ride-sharing app:

In scope:
  ✓ Rider can request a ride with a pickup and destination
  ✓ System matches rider to nearby driver
  ✓ Both parties see real-time location during the trip
  ✓ Payment is processed automatically at trip end
  ✓ Rider and driver can rate each other

Out of scope (explicitly):
  ✗ Ride scheduling (book in advance)
  ✗ Multiple stops in one trip
  ✗ Carpooling / shared rides
  ✗ Driver background check workflow
```

Explicitly stating out-of-scope is not laziness — it is discipline. It prevents scope creep and keeps the design focused on what was actually agreed.

---

## 3. Turning a Vague Prompt Into FRs

In interviews and in real projects, requirements arrive as vague briefs. "Design a messaging system like WhatsApp." Your job is to convert that into a precise set of FRs.

### The Clarifying Questions

For any system, ask these categories of questions:

**Who are the users?**
```
"Is this consumer-facing or B2B?"
"Are there different user types — e.g. sender and receiver, or rider and driver?"
"Authenticated users only, or anonymous too?"
```

**What are the core actions?**
```
"What is the primary thing a user does in this system?"
"What does 'success' look like for a user completing that action?"
"Are there secondary actions that support the primary one?"
```

**What is the scope?**
```
"Is feature X in scope for today's design?"
"If we had to cut one feature to ship on time, which would it be?"
"Are there any hard constraints — regulatory, technical, or business?"
```

**What are the data flows?**
```
"Does data come in from external systems?"
"Is the output stored, displayed, forwarded, or all three?"
"Who reads the data and when?"
```

### Example: Converting "Design Twitter" to FRs

```
Clarifying questions asked:
  Q: One-to-one messaging or public posts?
  A: Public posts — tweets visible to all followers

  Q: Does the feed need to be chronological or ranked?
  A: Chronological for this design

  Q: Do we need search and trending topics?
  A: No — out of scope

  Q: Video and images or text only?
  A: Text only for this design

  Q: What's the follow model — mutual or one-directional?
  A: One-directional (you can follow someone without them following back)

Resulting FRs:
  FR-1: Users can post tweets (text, up to 280 characters)
  FR-2: Users can follow other users (one-directional)
  FR-3: A user's home feed shows tweets from accounts they follow,
         in reverse chronological order
  FR-4: New tweets appear in followers' feeds within 5 seconds
  FR-5: Users can like and retweet

  Out of scope: Search, trending topics, direct messages,
                media uploads, notifications, ads
```

---

## 4. Non-Functional Requirements — How Well the System Does It

An NFR describes a quality attribute of the system — a constraint on how it must behave, not what it must do. These are often the harder requirements to get right, because they're less visible until they're violated.

### The Nine Core NFRs

| NFR | The Question It Answers |
|---|---|
| **Latency** | How fast does a single request complete? |
| **Throughput** | How many requests can the system handle per second? |
| **Availability** | What fraction of the time is the system accessible? |
| **Scalability** | How does the system grow as load increases? |
| **Reliability** | Does it do what it promises, consistently? |
| **Consistency** | Do all users see the same data at the same time? |
| **Durability** | Is committed data guaranteed to survive failures? |
| **Fault Tolerance** | Does it continue working when components fail? |
| **Security** | Is data and access protected against threats? |

Each of these has a dedicated deep-dive in the `01-NFR/` section. Here we focus on how to extract the right NFRs for a given system.

---

## 5. How to Identify NFRs

NFRs come from understanding two things: **what the system is used for** (determines the failure cost) and **who is using it** (determines the usage pattern).

### The Failure Cost Question

Ask: **"What happens to the business if this NFR is violated?"**

```
System: Online payment processing

If latency exceeds 3 seconds:
  → Users abandon checkout → direct revenue loss
  → NFR: P99 latency < 1 second on the payment flow

If availability drops below 99.9%:
  → Hours of downtime per year → significant revenue loss
  → NFR: 99.99% availability (five nines)

If a payment is processed twice:
  → Legal liability, fraud, customer trust destroyed
  → NFR: Exactly-once payment processing (idempotency required)

If payment data is breached:
  → Regulatory fines (PCI-DSS), brand destruction
  → NFR: PCI-DSS Level 1 compliance, data encrypted at rest and in transit
```

Contrast with a different system:

```
System: Internal analytics dashboard

If latency is 5 seconds:
  → Slightly annoying, but acceptable for hourly reports
  → NFR: P99 latency < 10 seconds

If it's down for 1 hour:
  → Team waits for data → minor inconvenience
  → NFR: 99.5% availability is fine

If data is 1 hour old:
  → For analytics, slightly stale data is fine
  → NFR: Eventual consistency, data up to 1 hour stale acceptable
```

**The system's purpose determines which NFRs matter and how strictly.** Don't copy NFR numbers from one design to another — derive them.

### The Usage Pattern Question

Ask: **"What does the load look like in numbers?"**

This is where estimation comes in. You cannot set meaningful NFR targets without knowing the scale.

```
For a messaging system:
  "500 million daily active users, 40 messages per user per day"
  → 231,000 message writes per second at average
  → 462,000 at peak

This immediately drives NFRs:
  → Throughput: system must handle 500,000 writes/sec (with headroom)
  → Storage: 2 TB/day, 3.6 PB over 5 years
  → These numbers rule out single-node designs
  → They mandate write-optimised distributed storage (Cassandra)
```

---

## 6. The NFR Specification Format

NFRs must be measurable. Vague NFRs are useless — you can't build to them or verify them.

| ❌ Vague NFR | ✅ Measurable NFR |
|---|---|
| "The system should be fast" | "P99 latency < 200ms for search queries" |
| "The system should be reliable" | "99.99% availability measured over a rolling 30-day window" |
| "The system should handle high traffic" | "Must sustain 50,000 reads/sec and 5,000 writes/sec under normal load" |
| "Data should be safe" | "No data loss after a confirmed write; data encrypted at rest with AES-256" |
| "The system should scale" | "Must scale to 10× current load within 24 hours by adding nodes" |

The measurable version gives engineers a target to design for and a test to verify against.

---

## 7. NFRs Drive Architecture

This is the most important point about NFRs: **they directly determine which architectural choices are possible and which are not.**

```
NFR: P99 read latency < 10ms
→ Disk access (~10ms) cannot be on the critical read path
→ Requires in-memory caching (Redis)
→ Cache invalidation strategy becomes a core design problem

NFR: 99.99% availability
→ No single point of failure
→ Every component needs redundancy
→ Requires leader-follower replication with automatic failover
→ Circuit breakers on all inter-service calls

NFR: No duplicate payments (idempotency)
→ Idempotency keys required on all payment endpoints
→ Unique constraint on idempotency key in DB
→ Retry logic must always include the same key

NFR: Data fresh within 5 seconds
→ Push model required (pull/polling introduces too much lag)
→ WebSockets or Server-Sent Events for delivery
→ Kafka for event-driven propagation
```

**An NFR is not a nice-to-have annotation at the bottom of a document. It is an architectural constraint that shapes every decision above it.**

---

## 8. The FR/NFR Template

When starting any design — in an interview or in real life — fill this out first.

```
SYSTEM: ___________________________

FUNCTIONAL REQUIREMENTS
  Core (must have):
    FR-1:
    FR-2:
    FR-3:

  Secondary (nice to have, out of scope today):
    -
    -

  Explicitly out of scope:
    -
    -

NON-FUNCTIONAL REQUIREMENTS
  Latency:
    Read P99:
    Write P99:

  Throughput:
    Reads/sec (avg / peak):
    Writes/sec (avg / peak):

  Availability:
    Target (%):
    Acceptable downtime per month:

  Consistency:
    Model: [Strong / Eventual / Tunable]
    Justification:

  Durability:
    Can we lose data? [Yes / No / N seconds acceptable]

  Scale:
    Current users:
    Storage (5 year):

  Security:
    Key concerns (auth, encryption, compliance):
```

In an interview, completing this template (verbally, while drawing) in the first 5-8 minutes is the difference between a well-structured design and one that meanders.

---

## 9. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Skipping requirements and jumping to design | Designing the wrong system | Always spend 5 minutes on requirements first |
| Vague NFRs ("system should be fast") | No basis for architectural decisions | Quantify every NFR with a number and a percentile |
| Treating all NFRs as equally important | Over-engineering non-critical paths | Rank NFRs by the cost of violating them |
| Not stating what's out of scope | Scope creep; interviewer thinks you're ignoring features | Explicitly say what you're not designing and why |
| Copying NFR targets from another system | Wrong targets for your actual use case | Derive NFRs from the specific system's failure cost and usage pattern |
| Forgetting that NFRs conflict | Inconsistent design choices | Explicitly resolve conflicts ("we chose availability over consistency because...") |

---

## 10. Summary

| Concept | Key Point |
|---|---|
| **Functional Requirements** | What the system does. User-facing capabilities. Testable and scoped. |
| **Non-Functional Requirements** | How well it does it. Quality attributes. Must be measurable. |
| **Clarifying questions** | The tool for converting vague prompts into precise requirements |
| **Out of scope** | Explicit — not an afterthought |
| **NFRs drive architecture** | Every major architectural decision traces back to an NFR |
| **Measurable NFRs** | P99 latency, requests/sec, availability %, durability guarantee — always with numbers |

---

## 11. Cross References

**Read Next:** `03-hld-and-lld.md`

**Deep dives on each NFR:** `01-NFR/` — all 9 files

**In interviews:** `11-Interview/01-how-to-approach-system-design.md` (Phase 1 and 2 of the framework)

---

*System Design Engineering Handbook — 00-Introduction*