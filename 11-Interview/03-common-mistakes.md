# 03 — Common Mistakes in System Design Interviews

> **11-Interview Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## Overview

Every mistake in this document is drawn from patterns that interviewers at top companies consistently report as reasons for failing or downlevelling a candidate. None of them are about lacking knowledge — they're about how you approach the problem.

Read each one, recognise the pattern, and internalise the correction. Practice sessions specifically targeting your personal weak points are far more valuable than studying more content.

---

## Mistake 1 — Jumping to a Solution Without Clarifying

**What it looks like:**
```
Interviewer: "Design a social media platform."
Candidate:   "Okay, so I'll start with a load balancer, then..."
```

**Why it fails:** The interviewer doesn't know if you understand what you're building. A social media platform for 1,000 users is architecturally different from one for 1 billion. A read-heavy platform (Instagram) is different from a write-heavy one (Twitter). You've started solving without knowing the problem.

**The correction:** Spend 5 minutes asking questions before drawing a single box. Define scope, scale, and key trade-offs explicitly. Confirm your understanding before proceeding.

---

## Mistake 2 — Designing Without Estimations

**What it looks like:**

Candidate designs a system with "a database" and "some caches" without establishing whether they're handling 1,000 or 1,000,000 requests per second.

**Why it fails:** Design decisions are meaningless without scale context. Putting a cache in front of a database is unnecessary at 100 RPS and critical at 100,000 RPS. Without estimations, you're guessing — and the interviewer knows it.

**The correction:** Always estimate traffic, storage, and bandwidth before designing. Let the numbers drive the architecture. "At 400,000 writes/sec, a single relational DB saturates — so I need either sharding or a write-optimised NoSQL store."

---

## Mistake 3 — Over-Engineering for Unknown Scale

**What it looks like:**

Candidate immediately proposes 12 microservices, 3 message queues, multi-region active-active deployment, and distributed consensus — for a system serving 10,000 users.

**Why it fails:** Over-engineering is as bad as under-engineering. It signals poor judgment, inability to match solution complexity to problem complexity, and wasted resources. Interviewers want to see you calibrate.

**The correction:** Let the scale estimations guide complexity. 10K DAU → start simple. 500M DAU → justify the complexity you add. Always match the solution to the actual problem.

---

## Mistake 4 — Being Technology-First Instead of Problem-First

**What it looks like:**
```
"I'll use Kafka here because it's really scalable."
"I'll use MongoDB because it's flexible."
"I'll use microservices because that's the modern approach."
```

**Why it fails:** Naming a technology without explaining why it fits this specific problem reveals shallow understanding. An interviewer will immediately ask "why Kafka over SQS?" and a candidate who chose it reflexively has no answer.

**The correction:** Always frame the choice as a solution to a specific problem. "I need to fan-out this event to four independent consumers — Kafka's consumer group model is designed for exactly that. SQS would require four separate queues with message copying."

---

## Mistake 5 — No Trade-off Reasoning

**What it looks like:**

Candidate makes decisions confidently but when asked "why did you choose that?" answers with "because it's better" or "because it's faster" without elaboration.

**Why it fails:** System design is about trade-offs. Every choice sacrifices something. A candidate who can't articulate what they're giving up doesn't understand the choice they made.

**The correction:** For every significant decision, say: "I'm choosing X over Y. X gives me [benefit] at the cost of [trade-off]. Given our requirement for [Z], that trade-off is acceptable because..."

---

## Mistake 6 — Ignoring the Hard Problems

**What it looks like:**

Candidate designs the happy path beautifully — user sends a request, server processes it, database stores it, response returns. Doesn't address what happens when a node fails, when traffic spikes, when two users write simultaneously.

**Why it fails:** The interesting problems in system design are the edge cases: consistency during network partition, duplicate processing after a retry, hot partitions under uneven load. A candidate who only addresses the sunny day scenario hasn't done system design.

**The correction:** After the happy path, proactively ask yourself: "What can go wrong here? How does this fail gracefully?" Address at least two or three failure modes explicitly.

---

## Mistake 7 — Choosing the Wrong Database Without Justification

**What it looks like:**

Candidate uses a relational database for everything, or uses NoSQL for everything, without explaining why the data model and access pattern fit the chosen database type.

**Why it fails:** The choice of database is one of the highest-signal decisions in a system design interview. Getting it wrong — or worse, not being able to explain it — signals a fundamental gap.

**The correction:** Justify every database choice with: the access pattern, the consistency requirement, the scale, and why the chosen database fits better than the alternatives. "I need time-ordered writes per channel at 200K writes/sec with efficient range reads — Cassandra's wide-column model with channel_id as the partition key and timestamp as the clustering key is designed for exactly this."

---

## Mistake 8 — Shallow Depth, Wide Coverage

**What it looks like:**

Candidate rushes through the entire system in 20 minutes, mentioning every component but going deep on none. The design is a collection of boxes with arrows between them.

**Why it fails:** Senior engineers are evaluated on their ability to go deep. Anyone can draw boxes. The signal is in the details: how exactly does the matching algorithm work? What's the partition strategy for this database? How do you handle the retry after a timeout?

**The correction:** It's better to cover 60% of the system deeply than 100% superficially. When you reach a complex component, go deep on it. Let the interviewer decide when to move on.

---

## Mistake 9 — Treating the Interviewer as an Examiner

**What it looks like:**

Candidate works in silence, only speaking when they have something to present. When the interviewer suggests an alternative, the candidate gets defensive or says "yes, that's also a valid approach" without engaging.

**Why it fails:** System design is collaborative at real companies. Interviewers simulate this. They want to see how you think through problems with another engineer, not how you perform a presentation.

**The correction:** Think aloud. Engage with pushback genuinely. "That's interesting — if we use X instead, how would that affect the consistency guarantee we established earlier? Let me think about that..."

---

## Mistake 10 — Ignoring Non-Functional Requirements

**What it looks like:**

Candidate designs a functionally correct system but never mentions availability, latency, consistency, or durability. No SLO mentioned. No discussion of what happens during a network partition.

**Why it fails:** A system that works correctly under ideal conditions isn't production-ready. NFRs separate prototypes from real systems. Not addressing them signals inexperience with production engineering.

**The correction:** After clarifying functional requirements, always clarify NFRs. "What availability do we need? What's the acceptable latency for the core flow? Can we tolerate eventual consistency here?" Then address them explicitly in the design.

---

## Mistake 11 — Making Everything Synchronous

**What it looks like:**

Every operation in the design is a synchronous call. User sends a message → synchronously write to DB → synchronously send notification → synchronously update search index → return response.

**Why it fails:** Synchronous chains are slow, fragile, and don't scale. The response time is the sum of every step. If any step is slow, the user waits. If any step fails, the whole operation fails.

**The correction:** Ask for every operation: "Does the user need to wait for this before they get a response?" If no → move it off the request path into an async queue. Only keep truly blocking operations synchronous.

---

## Mistake 12 — No Single Point of Failure Analysis

**What it looks like:**

The entire design routes through a single instance of a critical component — one database, one message broker, one load balancer — with no discussion of what happens if it fails.

**Why it fails:** High availability is a core NFR for almost every production system. A design with undiscussed SPOFs is incomplete.

**The correction:** After drawing your HLD, explicitly walk through each component and ask "what happens if this fails?" At minimum, discuss replication, failover, and redundancy for the critical path components.

---

## Mistake 13 — Giving Up When Stuck

**What it looks like:**

Candidate hits a hard problem (like the fan-out problem in social feeds, or consistent hashing for cache distribution) and goes silent, or says "I'm not sure how to solve this."

**Why it fails:** Experienced engineers don't always know the answer either. What matters is the reasoning process under uncertainty.

**The correction:** When stuck, verbalise the constraint. "The problem I'm trying to solve is [X]. The challenge is [Y]. One approach would be [A] but that fails because [reason]. Another approach might be [B]..." Show your reasoning. The interviewer may guide you, and even if they don't, they see your process.

---

## Mistake 14 — Treating "Scalable" as a Justification

**What it looks like:**
```
"I'll use Kafka because it's scalable."
"I'll use microservices because they're more scalable."
"I'll use NoSQL because relational databases don't scale."
```

**Why it fails:** "Scalable" is not a justification. Every component is scalable to some point. An interviewer will ask "scalable how? to what order of magnitude? what does that buy you specifically for this problem?" and a candidate using it as a magic word has no answer.

**The correction:** Quantify. "At 400K writes/sec, a single Postgres primary saturates at around 10K TPS. I need a write-optimised solution that can distribute this load. Cassandra with 10 nodes gives me approximately 1M writes/sec — enough headroom for our peak estimate plus growth."

---

## Mistake 15 — Not Knowing When to Stop Going Deeper

**What it looks like:**

Candidate spends 30 minutes on the database schema, covering every column and constraint in exhaustive detail, and never gets to the overall architecture or the interesting distributed systems problems.

**Why it fails:** Time management is a signal. A candidate who can't calibrate depth to available time and problem importance signals poor judgment about what matters.

**The correction:** After 5 minutes on any single component, check in: "I can go deeper here — is this a good use of our time or should I move to [the next important part]?" Let the interviewer guide prioritisation while you demonstrate awareness of what's incomplete.

---

## Summary — The 15 Mistakes

| # | Mistake | Correction |
|---|---|---|
| 1 | Jump to solution | Clarify first — always |
| 2 | No estimations | Math before architecture |
| 3 | Over-engineer | Match complexity to scale |
| 4 | Technology-first | Problem-first, then technology |
| 5 | No trade-offs | Every choice has a cost — name it |
| 6 | Ignore hard problems | Proactively address failure modes |
| 7 | Wrong DB without justification | Justify with access pattern + scale |
| 8 | Wide but shallow | Deep on 2-3 things beats shallow on everything |
| 9 | Treat interviewer as examiner | Collaborate, engage with pushback |
| 10 | Ignore NFRs | Clarify and address them explicitly |
| 11 | Everything synchronous | Async everything that doesn't need to block |
| 12 | Unaddressed SPOFs | Walk through failure for each component |
| 13 | Give up when stuck | Verbalise the constraint, reason through it |
| 14 | "Scalable" as justification | Quantify: "at X writes/sec, Y saturates because..." |
| 15 | Wrong depth calibration | Check in; let interviewer guide prioritisation |

---

## 14. Cross References

**Read First:** 01-how-to-approach-system-design.md

**Read Next:** 04-trade-off-vocabulary.md

**Then practice:** 10-Case-Studies — apply these corrections on every case study

---

*System Design Engineering Handbook — 11-Interview Series*