# 01 — What Is System Design?

> **00-Introduction** — Engineering Handbook
> Language-agnostic · 5 min read

---

## 1. The Simple Definition

System design is the process of defining the **architecture, components, data flows, and trade-offs** of a system to satisfy a set of requirements.

When a product manager says "we need users to be able to send messages to each other", a system designer answers questions like:

- How many users? How many messages per second?
- Where does the data live? What database? Why that one?
- What happens when a server goes down?
- How does the message get from sender to receiver in under a second?
- What does the API look like?

System design is the discipline of answering those questions deliberately — before writing a line of code — so that the system you build actually works at the scale and reliability it needs to.

---

## 2. Why It Matters

A system that is **functionally correct** but **architecturally wrong** fails in production. It might:

- Fall over when traffic doubles
- Lose data when a server crashes
- Take 10 seconds to load when users expect 200ms
- Cost 100× more to run than it should

Good system design prevents these failures. It doesn't mean over-engineering — it means making the right trade-offs for your actual requirements.

> **The most expensive system design decision is the one you make implicitly.** Not choosing a database is still choosing one. Not thinking about scale means you'll discover the problems when it's hardest to fix them.

---

## 3. The Building Blocks of System Design

Every system design discussion involves the same core concerns. These are the lenses through which you evaluate every architectural decision.

```
┌──────────────────────────────────────────────────────────┐
│                  SYSTEM DESIGN CONCERNS                  │
│                                                          │
│   REQUIREMENTS          ARCHITECTURE                     │
│   ├─ Functional         ├─ Components & Services         │
│   └─ Non-Functional     ├─ Data Flow                     │
│                         ├─ APIs                          │
│   STORAGE               └─ Communication patterns        │
│   ├─ What database?                                       │
│   ├─ How structured?    RELIABILITY                      │
│   └─ How much data?     ├─ What happens on failure?      │
│                         ├─ How do we recover?            │
│   SCALE                 └─ How do we detect problems?    │
│   ├─ How many users?                                      │
│   ├─ How many requests? TRADE-OFFS                       │
│   └─ How much data?     └─ What do we sacrifice and why? │
└──────────────────────────────────────────────────────────┘
```

---

## 4. The Vocabulary You Need

These four terms come up in every system design conversation. You need to know exactly what they mean and how they relate.

### Functional Requirements (FR)

**What the system does.** The features and capabilities users interact with.

```
Examples:
  ✓ Users can post tweets of up to 280 characters
  ✓ Users can follow other users
  ✓ A user's feed shows tweets from people they follow
  ✓ Users can like and retweet
```

### Non-Functional Requirements (NFR)

**How well the system does it.** The quality attributes that determine whether the system is production-ready.

```
Examples:
  ✓ The feed must load in under 300ms (P99)
  ✓ The system must be available 99.99% of the time
  ✓ A tweet must appear in followers' feeds within 5 seconds
  ✓ The system must handle 100,000 reads per second
```

### High-Level Design (HLD)

**The architecture.** The major components, how they connect, what databases are used, how data flows, and what the APIs look like. The blueprint before the details.

### Low-Level Design (LLD)

**The detail inside the components.** In system design context: the database schema, the detailed API contracts, the data models, the specific algorithms for the hard problems. Not OOP patterns — the engineering specifics that make each component work.

---

## 5. How They Relate

```
                    START
                      │
                      ▼
           ┌─────────────────────┐
           │  REQUIREMENTS       │
           │  What does it do?   │  ← Functional Requirements
           │  How well?          │  ← Non-Functional Requirements
           └──────────┬──────────┘
                      │
                      ▼
           ┌─────────────────────┐
           │  HIGH-LEVEL DESIGN  │
           │  What are the major │
           │  components and     │
           │  how do they        │
           │  connect?           │
           └──────────┬──────────┘
                      │
                      ▼
           ┌─────────────────────┐
           │  LOW-LEVEL DESIGN   │
           │  What exactly is    │
           │  inside each        │
           │  component?         │
           └─────────────────────┘
```

Requirements drive the architecture. The architecture constrains the details. Each layer justifies the one below it.

---

## 6. System Design in Interviews vs on the Job

The process is the same but the emphasis differs.

| | In an Interview | On the Job |
|---|---|---|
| **Time** | 45 minutes | Days to weeks |
| **Depth** | Breadth + 2-3 deep dives | Full depth on everything |
| **Trade-offs** | Verbalised explicitly | Documented in design docs |
| **Goal** | Show your reasoning | Build the right thing |
| **Unknowns** | Resolved by clarifying questions | Resolved by research and prototyping |

In both contexts, the process is identical: understand requirements → estimate scale → design the architecture → go deep on the hard problems → justify every significant decision.

The interview is a compressed, verbal version of what you'd do in a real design review.

---

## 7. The Mindset

Good system design is not about knowing the answer. It is about asking the right questions and making deliberate trade-offs.

Every system design decision is a trade-off:
- Strong consistency → lower availability
- Low latency → higher cost
- Simple architecture → harder to scale
- High availability → more complexity

The engineer who wins in system design — in interviews and on the job — is the one who can clearly articulate: **"I chose X because of Y, which means I'm accepting Z. Given our requirements, Z is acceptable because..."**

There are no perfect systems. There are only systems whose trade-offs were chosen deliberately for the right reasons.

---

## 8. What to Read Next

This handbook covers all four areas in depth. Suggested order:

```
1. 02-functional-and-non-functional-requirements.md  ← next
   (How to extract and specify requirements properly)

2. 03-hld-and-lld.md
   (What goes into each layer and how to approach them)

3. 01-NFR/  (all 9 files)
   (Deep dives into each non-functional requirement)

4. 02-Building-Blocks/  (all 7 files)
   (The components that appear in every design)

5. 11-Interview/01-how-to-approach-system-design.md
   (The framework for interviews)

6. 10-Case-Studies/  (all 30)
   (Apply everything to real systems)
```

---

*System Design Engineering Handbook — 00-Introduction*