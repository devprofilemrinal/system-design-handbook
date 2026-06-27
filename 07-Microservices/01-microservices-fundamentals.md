# 01 — Microservices Fundamentals

> **07-Microservices Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Are Microservices?

Microservices is an architectural style where an application is built as a collection of small, independently deployable services. Each service:

- Owns a single, well-defined business capability
- Runs in its own process
- Has its own database (owns its data)
- Communicates with other services over a network
- Can be deployed independently of other services

The opposite of microservices is a **monolith** — a single deployable unit containing all the application's functionality.

```
MONOLITH:                          MICROSERVICES:
┌─────────────────────┐            ┌──────────┐  ┌──────────┐
│                     │            │  User    │  │  Order   │
│  User logic         │            │ Service  │  │ Service  │
│  Order logic        │            └──────────┘  └──────────┘
│  Payment logic      │                 │              │
│  Notification logic │            ┌──────────┐  ┌──────────┐
│  [one database]     │            │ Payment  │  │ Notify   │
└─────────────────────┘            │ Service  │  │ Service  │
       (one unit)                  └──────────┘  └──────────┘
                                   (each independent)
```

---

## 2. Why Microservices Exist — The Problems With Monoliths at Scale

Monoliths are not inherently bad. They start as the right choice. Over time, as teams and codebases grow, specific problems emerge.

| Monolith Problem | How It Manifests |
|---|---|
| **Scaling is all-or-nothing** | One component needs more capacity → must scale the entire application |
| **Deployment coupling** | Changing one feature requires deploying the entire application → risky, slow |
| **Technology lock-in** | Entire application must use the same language, framework, runtime |
| **Team coupling** | Many teams working in one codebase → merge conflicts, coordination overhead, slow releases |
| **Fault propagation** | A bug in one component can crash the whole application |
| **Long build/test cycles** | The entire codebase must build and test for every change |

> **These problems are real — but they only appear at a specific scale.** A team of 5 engineers on a new product does not have these problems. Microservices solve problems of scale and organisational complexity, not inherent technical inferiority of monoliths.

---

## 3. The Monolith vs Microservices Decision

This is one of the most important architectural decisions and one of the most frequently made incorrectly.

### Start With a Monolith

A well-structured monolith is the right starting point for almost every new product. Here is why:

- **Domain boundaries are unclear early.** Microservices require knowing your domain well enough to split it correctly. Wrong boundaries are far more painful in microservices than in a monolith (the cost of refactoring across service boundaries is enormous).
- **Operational overhead is zero.** A monolith has one deployment, one log stream, one database to manage.
- **Development speed is higher.** No network calls, no distributed transactions, no service discovery.

### When to Move to Microservices

Move when specific, demonstrable pain points appear — not in anticipation of them.

| Signal | What It Indicates |
|---|---|
| **Independent scaling need** | Component X needs 10× resources; scaling the whole app wastes 9× | 
| **Deployment independence need** | Team A's deployments are blocked by Team B's changes |
| **Technology mismatch** | One component genuinely needs a different language/runtime |
| **Team size** | >2 pizza teams working in the same codebase with coordination overhead |
| **Fault isolation need** | One component's failure causes unacceptable system-wide outages |

> **Rule:** If you can't identify which of these signals you're experiencing, you don't need microservices yet.

### The Modular Monolith — The Ignored Middle Ground

Between a tangled monolith and full microservices lies the **modular monolith**: a single deployable unit with clear internal module boundaries — separate modules, separate databases within the application, clean interfaces between them.

```
Modular Monolith:
┌─────────────────────────────────┐
│ [User Module] | [Order Module]  │
│      ↕ clean interfaces ↕      │
│ [Payment Module] | [Notify Mod] │
│ [User DB] [Order DB] [Pay DB]   │
└─────────────────────────────────┘
(one deployment, but internally modular)
```

A modular monolith gives you clean domain boundaries (making future extraction to microservices easy) while avoiding distributed systems complexity. It is underused and underrated.

---

## 4. Core Principles of Microservices

### Single Responsibility

Each service owns exactly one business capability. If a service does two unrelated things, split it.

```
Good: Order Service (creates, tracks, cancels orders)
Bad:  Order-and-Notification Service (two different capabilities)
```

### Own Your Data

Each service has its own database. No two services share a database. No direct database access across service boundaries.

```
✅ Order Service → Order DB
   Payment Service → Payment DB
   (services communicate via API, not by querying each other's DB)

❌ Order Service → Payment DB (direct DB access)
   (creates tight coupling at the data layer; defeats the purpose)
```

### Design for Failure

Every inter-service call can fail. Services must handle downstream failures gracefully — with timeouts, retries, circuit breakers, and fallbacks.

### Decentralised Everything

Decentralised data management (each service owns its DB), decentralised governance (teams choose their own tools within guardrails), decentralised deployment (each service deploys independently).

---

## 5. Microservices Trade-offs — The Honest Picture

Microservices are not a free upgrade. They trade one set of problems for another.

| What You Gain | What You Pay |
|---|---|
| Independent scaling per service | Network latency on every inter-service call |
| Independent deployment | Distributed tracing needed to debug a request |
| Technology flexibility | Data consistency across services is hard |
| Fault isolation | Operational complexity: N services to deploy, monitor, scale |
| Team autonomy | Distributed transactions require Saga pattern |
| Clear ownership | Service discovery and API versioning overhead |

> **The microservices tax is real and constant.** Every system built as microservices pays the operational and complexity overhead on every feature, every bug fix, every deployment. Only pay this tax if the benefits genuinely justify it.

---

## 6. How to Know if Your Service Is the Right Size

"Micro" in microservices does not mean "small." It means "focused." A service is the right size when:

- One team can own and understand it completely
- It can be rewritten in 2–4 weeks if needed
- It has one clear reason to change
- Its API surface is stable

A service is too large if:
- Multiple teams need to modify it frequently
- Deployments are risky because many things can break
- It does more than one business capability

A service is too small if:
- It cannot function without always calling another service
- It shares a database with another service
- Changes always require coordinating with another service simultaneously

```
Too fine-grained (nanoservices):
  GetUserName Service, GetUserEmail Service, GetUserPhone Service
  → Every user profile operation requires 3 network calls
  → Chatty, fragile, unnecessary

Right-sized:
  User Service (owns all user profile data and operations)
```

---

## 7. Microservices and Conway's Law

Conway's Law: *"Any organisation that designs a system will produce a design whose structure is a copy of the organisation's communication structure."*

This is not just an observation — it's a design principle. Microservice boundaries should mirror team boundaries. If Team A owns the User Service and User Database, and Team B owns the Order Service and Order Database, they can deploy independently and move at their own pace.

```
Aligned (Conway's Law working for you):
  Team A → owns User Service → deploys User Service independently
  Team B → owns Order Service → deploys Order Service independently

Misaligned (Conway's Law working against you):
  Team A and Team B both modify User Service
  → Coordination overhead → slow releases → defeats microservices purpose
```

> **If your team structure doesn't match your service boundaries, your microservices will have the problems of a distributed monolith — all the complexity of microservices, none of the benefits.**

---

## 8. How Large Companies Apply This

| Company | Approach | Source |
|---|---|---|
| **Netflix** | ~700 microservices; each owned by a small team; pioneered chaos engineering to handle distributed failures | Netflix Tech Blog (public) |
| **Amazon** | "Two-pizza teams" — service ownership aligns with team size; each team owns its service end-to-end | Jeff Bezos public mandate (widely cited) |
| **Uber** | Started monolith → moved to microservices as scale demanded it; documented domain-driven decomposition | Uber Eng Blog (public) |
| **StackOverflow** | Deliberately stayed a modular monolith for many years; serves millions with small team and simple architecture | Stack Overflow Blog (public) |

> **Inferred:** Internal details vary; the architectural choices are publicly documented.

---

## 9. Best Practices

- **Start with a modular monolith.** Extract services only when a specific scaling or team-ownership pain point is proven.
- **One service, one database.** Never share a database between services.
- **Align service boundaries with team boundaries** — Conway's Law is real.
- **Design every service to handle downstream failure** — timeouts, retries, circuit breakers from day one.
- **Define service contracts (APIs) carefully** — changing a service's API breaks its consumers.
- **Extract services one at a time** — big-bang rewrites fail; the Strangler Fig pattern is safer (covered in file 04).

---

## 10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Microservices from day one | All the complexity, none of the scale benefits | Start monolith; extract when pain is proven |
| Services sharing a database | Tight data coupling defeats service independence | One database per service, strictly |
| Wrong service boundaries | Services always deployed together; constant coordination | Align with domain and team boundaries |
| Distributed monolith | Microservices architecture with monolith coupling | Enforce independent deployability and data ownership |
| No fault tolerance | One service failure cascades to full outage | Circuit breakers + timeouts on all inter-service calls |

---

## 11. Interview Questions

1. What is the difference between a monolith and microservices?
2. When should you choose a monolith over microservices?
3. What is Conway's Law and how does it affect microservice design?
4. What is a modular monolith? Why is it underrated?
5. What does "own your data" mean in microservices?
6. What is a distributed monolith and why is it the worst of both worlds?
7. How do you know if a service is the right size?

---

## 12. Summary

| Concept | Key Takeaway |
|---|---|
| **Microservices** | Small, independently deployable services each owning a business capability |
| **vs Monolith** | Monolith is the right start. Microservices solve specific scale/team problems. |
| **Modular monolith** | The underrated middle ground. Clean boundaries without distributed complexity. |
| **Own your data** | Each service has its own DB. No sharing. No cross-service DB queries. |
| **Conway's Law** | Service boundaries must match team boundaries or you get a distributed monolith. |
| **The tax** | Operational complexity, network latency, distributed transactions. Always paid. |

---

## 13. Cross References

**Prerequisites:** System Design Fundamentals · Distributed Systems Fundamentals (DS #1)

**Related Topics:** Service Discovery (BB #7) · API Gateway (BB #2) · Fault Tolerance (NFR #6)

**What to Learn Next:** 02-service-decomposition.md

---

*System Design Engineering Handbook — 07-Microservices Series*