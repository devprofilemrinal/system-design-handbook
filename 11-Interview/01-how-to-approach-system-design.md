# 01 — How to Approach System Design Interviews

> **11-Interview Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Interviewers Are Actually Evaluating

System design interviews are not tests of memorisation. Interviewers are not looking for the "correct" answer — most problems have no single correct answer. They are evaluating:

| What They Observe | What They're Actually Testing |
|---|---|
| How you handle an ambiguous problem | Can you structure your thinking under pressure? |
| The questions you ask | Do you understand what matters before designing? |
| How you justify decisions | Can you reason through trade-offs, not just recite patterns? |
| How you respond to pushback | Are you defensive or genuinely thinking? |
| What you choose to go deep on | Do you know what the hard problems actually are? |
| How you communicate | Can you explain complex ideas clearly to different audiences? |

> **The single biggest mistake candidates make:** jumping straight into a solution without clarifying requirements. An interviewer who asks "design Twitter" wants to see how you handle ambiguity — not whether you know Twitter's architecture.

---

## 2. The Framework — Five Phases

Every system design interview should follow this structure. The time allocations assume a 45-minute interview — adjust proportionally.

```
Phase 1: Clarify Requirements        (5 minutes)
Phase 2: Estimate Scale              (5 minutes)
Phase 3: High-Level Design           (15 minutes)
Phase 4: Deep Dive                   (15 minutes)
Phase 5: Wrap-Up                     (5 minutes)
```

Never skip Phase 1 and 2. Candidates who jump to Phase 3 immediately almost always design the wrong system or fail to identify the hard problems.

---

## 3. Phase 1 — Clarify Requirements (5 minutes)

Your goal: transform a vague prompt into a specific, scoped problem.

### The Questions to Ask

**Functional scope:**
```
"What are the core features we need to support?"
"What is explicitly out of scope?"
"Who are the users — consumers, businesses, developers?"
"What is the primary use case vs. secondary ones?"
```

**Scale:**
```
"How many daily active users are we expecting?"
"What is the expected read/write ratio?"
"What is the expected data volume over 5 years?"
"Is this a global system or regional?"
```

**Non-functional priorities:**
```
"What matters more — availability or consistency for this data?"
"What is the acceptable latency for the core flow?"
"Are there compliance requirements (GDPR, PCI-DSS)?"
"What does 'down' mean for this system — total outage or degraded?"
```

**Existing context:**
```
"Are we building from scratch or integrating with existing systems?"
"Is there a mobile app, web app, or API-only?"
```

### What Good Clarification Looks Like

```
Interviewer: "Design a messaging system like WhatsApp."

Poor candidate:
  "Okay, so I'll start with a load balancer..."

Good candidate:
  "Before I start, let me make sure I understand the scope.
   Are we supporting one-to-one messaging, group chats, or both?
   Should messages be end-to-end encrypted or is server-side encryption sufficient?
   What scale are we targeting — tens of millions or hundreds of millions of users?
   Do we need to support offline message delivery?
   Is read receipts and typing indicators in scope?"

After getting answers:
  "So to confirm: one-to-one and group messaging, server-side encryption,
   500 million DAU, offline delivery required, read receipts in scope but
   typing indicators are out of scope for today. Is that right?"
```

---

## 4. Phase 2 — Estimate Scale (5 minutes)

Back-of-envelope calculations drive architectural decisions. They tell you what kind of system you're building before you design it.

### The Standard Estimates to Make

**Traffic:**
```
Daily Active Users (DAU): given or assumed
Requests per second (RPS):
  Read RPS  = DAU × reads_per_user / 86,400
  Write RPS = DAU × writes_per_user / 86,400
Peak RPS  = average × 2-3×
```

**Storage:**
```
Daily data written = write RPS × seconds_per_day × record_size
5-year storage     = daily × 365 × 5
```

**Bandwidth:**
```
Ingress  = write RPS × average_payload_size
Egress   = read RPS × average_payload_size
```

### What Estimations Tell You

```
Example: Chat system, 500M DAU, 40 messages/user/day

Write RPS = 500M × 40 / 86,400 ≈ 231,000 writes/sec
Peak      = 231,000 × 2 ≈ 462,000 writes/sec

→ Cannot use a single relational DB for writes
→ Need a write-optimised store (Cassandra) or horizontal sharding
→ Need async processing (Kafka) to absorb peaks

Message size = 100 bytes × 86,400 sec × 231,000 ≈ 2 TB/day
5-year storage = 2 TB × 365 × 5 ≈ 3.65 PB

→ Storage is significant — need archival strategy
→ Hot/cold tiering required
```

> **Estimations are not about precision. They're about order of magnitude.** Is this 1,000 writes/sec or 1,000,000? That difference determines everything. Being off by 2× is fine. Being off by 1,000× means your architecture is wrong.

---

## 5. Phase 3 — High-Level Design (15 minutes)

Design the system at the architecture level. Draw the major components and explain how they connect.

### The Sequence

```
1. Start with the API
   → What are the primary endpoints?
   → What do requests and responses look like?

2. Draw the high-level components
   → Client → Load Balancer → Services → Databases
   → Show where caching sits
   → Show where async processing is needed

3. Walk through the core user flow
   → "When a user sends a message, here's what happens..."
   → Trace the request through every component

4. Address the critical NFRs
   → "For availability, I'd use X because..."
   → "For the write throughput we estimated, I need Y because..."
```

### What to Draw

```
At minimum, your diagram should show:
  ✅ Client entry point (browser/mobile/API)
  ✅ Load balancer / API Gateway
  ✅ Core services (name them by function)
  ✅ Databases (with the type — relational, NoSQL, cache)
  ✅ Async layer (queue/stream) if needed
  ✅ CDN if content-heavy

You do NOT need to show:
  ❌ Every microservice
  ❌ Internal network topology
  ❌ DevOps/CI-CD
  ❌ Monitoring (unless asked)
```

### Verbalize Your Reasoning

Don't just draw — explain every decision as you make it.

```
"I'm putting a cache between the service and the database here
 because the read/write ratio we estimated is 100:1, and the hot
 queries are always the same recent messages — high cache hit rate.
 I'd use Redis with a 24-hour TTL and cache-aside pattern."

Not:
"I'll add a cache here."
```

---

## 6. Phase 4 — Deep Dive (15 minutes)

The interviewer will direct you to the most interesting part of the problem. Common deep dive areas:

```
Data model / schema design
Database selection and justification
Specific algorithm (matching, ranking, routing)
How you handle a specific scale challenge
How a particular failure mode is handled
How you ensure consistency for a critical operation
```

### How to Handle the Deep Dive

**Lead with the problem, then the solution:**

```
"The hard problem here is [X]. Let me explain why it's hard,
 then walk through how I'd solve it."

Example:
"The hard problem in a social feed is the celebrity problem —
 when a user with 50 million followers posts, we'd need to write
 to 50 million timeline caches simultaneously, which would take
 minutes. Let me walk through how I'd handle this..."
```

**Show you know the trade-offs:**

```
"I could solve this with approach A, which gives us [benefit]
 but costs [trade-off]. Approach B gives us [benefit] but costs
 [trade-off]. Given our requirement for [X], I'd choose B because..."
```

**If you don't know something, say so honestly:**

```
"I'm not certain about the exact implementation of X, but
 conceptually I'd approach it by [reasoning]. The key properties
 I'd need are [A, B, C]. Does that direction make sense?"
```

---

## 7. Phase 5 — Wrap-Up (5 minutes)

Close the interview actively. Don't wait to be dismissed.

```
"Let me quickly summarise what we've designed..."
  → One sentence per major component
  → One sentence on the key trade-off you made

"If I had more time, the next things I'd address are..."
  → Shows you know what's incomplete
  → Shows you can prioritise

"Are there any specific areas you'd like to go deeper on?"
  → Gives the interviewer a chance to probe where they want
  → Shows confidence rather than relief it's over
```

---

## 8. Time Management

The most common failure mode is spending too long on Phase 1-2 and running out of time before the deep dive. The deep dive is where senior candidates differentiate themselves.

```
If you're behind:
  Phase 3: sketch the HLD quickly; don't perfect it
  Say: "I'll note these details and come back if we have time"
  Get to the deep dive — that's where the signal is

If you're ahead:
  Go deeper on trade-offs
  Proactively address follow-up questions before they're asked
  Discuss failure modes and how the system degrades gracefully
```

---

## 9. Communication Tips

**Think aloud:**
```
"I'm thinking about whether to use SQL or NoSQL here.
 The access pattern is... the scale is... so I'm leaning toward..."

Not: [silent for 30 seconds] "I'll use Cassandra."
```

**Check in with the interviewer:**
```
"Does this level of detail make sense before I go deeper?"
"Is there a specific area you'd like me to focus on?"
"I'm making the assumption that X — is that reasonable?"
```

**Use precise language:**
```
Say: "I'll use Cassandra for the message store because it's write-optimised
      and the partition key on channel_id gives us efficient range reads"

Not: "I'll use a NoSQL database because it's more scalable"
```

---

## 10. Common Signals That Differentiate Candidates

| Junior Signal | Senior Signal |
|---|---|
| Jumps to a solution immediately | Clarifies requirements before designing |
| Picks technology without justification | Explains why one technology fits better than another |
| Treats every requirement equally | Identifies which requirements drive the hardest problems |
| Designs the happy path only | Proactively addresses failure modes |
| Gets defensive when challenged | Engages with pushback: "that's a good point, let me reconsider" |
| Covers everything shallowly | Goes deep on 2-3 hard problems |
| Says "it's more scalable" | Quantifies: "at 500K writes/sec, a single Postgres primary saturates" |
| Ignores trade-offs | Makes trade-offs explicit and justifies them |

---

## 11. The Mental Model — What You're Actually Doing

Think of the system design interview as a **collaborative engineering conversation**, not an oral exam. The interviewer is a senior colleague. You're designing something together.

```
Your role:
  Lead the conversation
  Make decisions and justify them
  Ask for input when genuinely uncertain
  Treat pushback as collaboration, not attack

Their role:
  Probe your reasoning
  Guide you toward interesting problems
  Evaluate how you think, not just what you know
```

---

## 12. Summary

| Phase | Duration | Goal |
|---|---|---|
| **Clarify** | 5 min | Turn vague prompt into scoped problem |
| **Estimate** | 5 min | Back-of-envelope math that drives design |
| **HLD** | 15 min | Architecture with justified decisions |
| **Deep Dive** | 15 min | Go deep on 2-3 hard problems |
| **Wrap-Up** | 5 min | Summarise, acknowledge gaps, invite questions |

**The three things that matter most:**
1. Clarify before designing — never assume
2. Justify every decision — "because" is the most important word
3. Make trade-offs explicit — there is no perfect design, only deliberate choices

---

## 13. Cross References

**Read Next:** 02-estimation-cheatsheet.md · 03-common-mistakes.md · 04-trade-off-vocabulary.md

**Then:** Start with 10-Case-Studies/01-url-shortener.md and work through all 30

---

*System Design Engineering Handbook — 11-Interview Series*