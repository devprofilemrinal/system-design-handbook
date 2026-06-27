# 04 — Microservices Patterns

> **07-Microservices Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Overview

Microservices patterns are reusable solutions to recurring design problems that appear specifically in distributed service architectures. Each pattern addresses a specific pain point — migration, data management, interface design, or cross-cutting concerns.

This document covers the patterns most frequently encountered in system design interviews and real-world microservices systems.

| Pattern | Problem It Solves |
|---|---|
| **Strangler Fig** | How to safely migrate from a monolith |
| **Anti-Corruption Layer** | How to integrate with legacy or external systems without polluting your domain |
| **Backend for Frontend (BFF)** | How to serve different clients with different data needs |
| **Sidecar** | How to add cross-cutting concerns without modifying service code |
| **Saga** | How to manage distributed transactions *(deep dive in Messaging section)* |
| **CQRS** | How to separate read and write models for performance and clarity |
| **Bulkhead** | How to isolate failures to prevent cascade |

---

## 2. Strangler Fig Pattern

Named after the strangler fig tree, which grows around a host tree and gradually replaces it.

### The Problem

You have a working monolith. You want to move to microservices. Rewriting the entire system at once (a "big bang" rewrite) is extremely risky — the new system must perfectly replicate the old one before anything can go live. Most big-bang rewrites fail or take far longer than planned.

### The Solution

Extract one capability at a time while keeping the monolith running. Route requests to the new service for the extracted capability; everything else still goes to the monolith. Over time, the monolith shrinks and the new services grow until the monolith is fully replaced.

```
Phase 1: All traffic → Monolith
  [Client] → [Monolith (User + Order + Payment + Notification)]

Phase 2: Extract User Service
  [Client] → [Router/Proxy]
               ├─ /users/* → [User Service]  (new)
               └─ /*       → [Monolith]       (everything else)

Phase 3: Extract Order Service
  [Client] → [Router/Proxy]
               ├─ /users/*  → [User Service]
               ├─ /orders/* → [Order Service]  (new)
               └─ /*        → [Monolith]

Phase N: Monolith fully replaced
  [Client] → [API Gateway]
               ├─ /users/*    → [User Service]
               ├─ /orders/*   → [Order Service]
               └─ /payments/* → [Payment Service]
```

**Key principles:**
- Extract one bounded context at a time
- The monolith is not modified (or minimally so) — you route around it
- Each extraction can be rolled back independently if it fails
- The system is always in a deployable, working state

> **The Strangler Fig is the only safe way to migrate a production monolith.** Any approach that requires a complete rewrite before going live is a project extinction risk.

---

## 3. Anti-Corruption Layer (ACL)

### The Problem

Your service needs to integrate with a legacy system, a third-party API, or an external partner. Their data model and terminology differ from yours. If you let their concepts bleed into your code, your service becomes polluted with their abstractions — making it hard to change their contract without touching your core logic.

### The Solution

Build a translation layer between your domain model and the external system. The ACL translates their data shapes and concepts into your ubiquitous language.

```
External Payment Provider (Stripe):
  {
    "charge": {
      "amount": 5000,           (in cents)
      "currency": "usd",
      "source": "tok_abc123",   (Stripe token)
      "status": "succeeded"
    }
  }

Your domain model:
  Payment {
    amount: Money(£50.00),      (your Money type)
    method: PaymentMethod.CARD,
    status: PaymentStatus.COMPLETED
  }

Anti-Corruption Layer (translates between them):
  StripePaymentAdapter.toPayment(stripeCharge) → Payment
  StripePaymentAdapter.toStripeCharge(payment) → StripeCharge
```

**Benefits:**
- Your domain model stays clean and independent of external systems
- Changing the external provider means changing only the ACL, not the whole codebase
- Tests can mock the ACL rather than calling the real external system

---

## 4. Backend for Frontend (BFF)

### The Problem

Different clients (mobile app, web app, third-party API) need different data shapes from the same backend services. The mobile app needs a lightweight response (small payload, battery-sensitive). The web app needs rich data. The partner API needs a different authentication model.

Without BFF, you either:
- Over-fetch: return all data for every client (wastes mobile bandwidth)
- Under-fetch: return minimal data and make clients do multiple calls
- Pollute the backend API with client-specific logic (messy, hard to maintain)

### The Solution

Create a dedicated API layer for each client type. Each BFF aggregates calls to backend services and returns exactly what its client needs.

```
Mobile App   → [Mobile BFF]  ─┐
Web App      → [Web BFF]     ─┼→ [User Service]
                               ├→ [Order Service]
Partner API  → [Partner BFF] ─┘  [Payment Service]

Mobile BFF: returns minimal user profile + last 5 orders (small payload)
Web BFF: returns full profile + all orders + recommendations (rich)
Partner BFF: returns only data the partner is authorised to see
```

**Benefits:**
- Each client gets exactly the data it needs in one call (no over/under fetching)
- Client-specific logic stays in the BFF, not in backend services
- Backend services remain clean and generic

**Trade-off:** More BFF services to maintain. Accept this cost when clients genuinely have very different requirements.

> **GraphQL is an alternative to BFF for flexible client-driven querying.** Instead of a BFF per client type, a single GraphQL layer lets clients specify exactly what fields they want. Use GraphQL when client requirements are highly variable; use BFF when client types are well-defined and different in fundamental ways.

---

## 5. Sidecar Pattern

### The Problem

Every service needs cross-cutting functionality: logging, metrics collection, tracing, health checks, mTLS, service discovery registration. If each service implements these itself:
- Code is duplicated across all services
- Implementation quality is inconsistent
- Different teams implement the same thing differently
- Upgrading the logging library means touching every service

### The Solution

Deploy a **sidecar process** alongside every service instance. The sidecar runs in the same network namespace as the service and handles cross-cutting concerns on its behalf.

```
┌─────────────────────────────────────┐
│            Pod / VM                 │
│  ┌───────────────┐  ┌────────────┐ │
│  │  Your Service │  │  Sidecar   │ │
│  │               │  │            │ │
│  │  (business    │  │  - logging │ │
│  │   logic only) │  │  - metrics │ │
│  │               │  │  - tracing │ │
│  └───────────────┘  │  - mTLS    │ │
│         │           │  - health  │ │
│         └───────────┘            │ │
└─────────────────────────────────────┘
         All traffic through sidecar
```

**Sidecar advantages:**
- Service code focuses purely on business logic
- Consistent cross-cutting behaviour across all services
- Update the sidecar (new logging format, new tracing standard) → all services updated without code changes
- Language-agnostic: Python service and Go service can have the same sidecar

**The service mesh** (Istio, Linkerd) is the productionised version of the sidecar pattern — it deploys a sidecar to every service instance automatically.

---

## 6. CQRS — Command Query Responsibility Segregation

### The Problem

A single data model often makes poor trade-offs between writes (which need normalised, transactional data) and reads (which need denormalised, pre-joined, fast-query data).

An order management system might write normalised orders and order lines (good for integrity) but need to query "all orders with product details, customer name, and shipping status" (requires expensive joins across many tables).

### The Solution

Separate the write model (commands) from the read model (queries). They use different data stores optimised for their respective operations.

```
WRITE SIDE (Commands):
  Client → [Command Handler] → [Write Store (PostgreSQL)]
  Normalised, ACID transactions, optimised for writes
  "CreateOrder", "UpdateStatus", "AddItem"

READ SIDE (Queries):
  Client → [Query Handler] → [Read Store (Elasticsearch/Redis/Cassandra)]
  Denormalised, optimised for specific queries, may be stale
  "GetOrderDashboard", "SearchOrders", "GetUserOrderHistory"

  Write Store → [Events] → [Projector] → Read Store
  (write events trigger updates to the read store)
```

**Benefits:**
- Reads and writes scaled independently
- Read store is optimised for specific query patterns (can use different DB technology)
- Write store stays clean and normalised

**Costs:**
- Eventual consistency between write and read stores — reads may be slightly stale
- More infrastructure to manage (two stores, projection pipeline)
- Increased complexity for relatively simple CRUD scenarios

> **CQRS is not always the right choice.** For simple applications, one model is simpler and correct. Apply CQRS when read and write performance requirements genuinely diverge, or when you have multiple very different query patterns.

---

## 7. Bulkhead Pattern

### The Problem

Service A depends on Service B and Service C. Service B starts responding slowly, holding threads open. Service A's thread pool is exhausted waiting for B. Now Service A can't process requests to Service C either — even though C is perfectly healthy. One slow dependency brings down all others.

### The Solution

Isolate resource pools per dependency. Give each downstream dependency its own thread pool, connection pool, or queue. One pool exhausting doesn't affect the others.

```
WITHOUT bulkhead:
  Service A: 100 threads (shared)
  Service B gets slow → 100 threads waiting for B
  Service C calls: no threads available → fails

WITH bulkhead:
  Service A:
    Thread pool for Service B: 30 threads
    Thread pool for Service C: 30 threads
    Thread pool for other work: 40 threads

  Service B gets slow → B's 30 threads fill up
  Service C calls → C's 30 threads still available ✅
```

**Named after ship bulkheads** — the watertight compartments that prevent one flooded section from sinking the whole ship.

**Implementation:** Separate thread pools or semaphores per dependency. Libraries like Resilience4j (conceptually) implement bulkheads alongside circuit breakers.

---

## 8. Combining Patterns — A Real System

Real systems layer multiple patterns. Here's an order management microservice using several at once:

```
Legacy monolith being migrated:
  Strangler Fig: Orders extracted to new service; monolith still handles payments

New Order Service:
  BFF: Mobile BFF returns lightweight order list; Web BFF returns full details
  ACL: Wraps the legacy inventory system's API in clean domain terms
  CQRS: Writes to PostgreSQL; reads from Elasticsearch for search queries
  Sidecar: Handles mTLS, tracing, metrics (Istio sidecar)
  Bulkhead: Separate thread pools for inventory service and payment service calls
  Circuit Breaker: Wraps all synchronous downstream calls
```

---

## 9. How Large Companies Apply These Patterns

| Company | Pattern | Application | Source |
|---|---|---|---|
| **Amazon** | Strangler Fig | Migrated from monolith to services over years; never a big-bang rewrite | Public talks |
| **Netflix** | Bulkhead + Circuit Breaker | Hystrix library pioneered these patterns; each dependency isolated | Netflix Tech Blog (public) |
| **LinkedIn** | CQRS | Separate write and read models for feed; writes to primary, reads from Voldemort | LinkedIn Eng Blog (public) |
| **Uber** | BFF | Separate API layers for rider app, driver app, and partner integrations | Uber Eng Blog (public) |

---

## 10. Best Practices

- **Use Strangler Fig for all monolith migrations** — never big-bang rewrites.
- **Always wrap external systems in an ACL** — protect your domain model from external contamination.
- **Use BFF when clients have genuinely different data needs** — don't force all clients through one API.
- **Apply CQRS only when read/write patterns genuinely diverge** — the complexity must be justified.
- **Implement bulkheads alongside circuit breakers** — isolation and fast-fail work together.
- **Use sidecars for cross-cutting concerns** in a multi-service environment to ensure consistency.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Big-bang monolith rewrite | High risk; long delay before value; often fails | Strangler Fig always |
| No ACL for external APIs | External system's model bleeds into your domain | Wrap all external integrations |
| One BFF for all clients | Over-fetching for mobile; complex client logic | Separate BFF per client type |
| CQRS everywhere | Unnecessary complexity for simple CRUD | Apply only when read/write patterns genuinely differ |
| No bulkhead isolation | One slow dependency starves all others | Separate resource pools per downstream dependency |

---

## 12. Interview Questions

1. What is the Strangler Fig pattern and why is it preferred over a big-bang rewrite?
2. What is an Anti-Corruption Layer? Give an example of when you'd use one.
3. What problem does the Backend for Frontend pattern solve?
4. When would you choose CQRS? What does it cost?
5. What is the Bulkhead pattern? What is it analogous to physically?
6. What is a sidecar and how does it relate to a service mesh?
7. How would you migrate a monolith payment system to microservices safely?

---

## 13. Summary

| Pattern | Problem | Solution |
|---|---|---|
| **Strangler Fig** | Risky monolith migration | Extract one capability at a time; route around monolith |
| **ACL** | External system pollutes domain | Translation layer between external and internal models |
| **BFF** | Different clients, different data needs | Dedicated API per client type |
| **Sidecar** | Cross-cutting concerns in every service | Co-located proxy handles logging, tracing, mTLS |
| **CQRS** | Read/write model mismatch | Separate stores optimised for each |
| **Bulkhead** | One slow dependency starves all | Isolated resource pools per dependency |

---

## 14. Cross References

**Prerequisites:** 01-microservices-fundamentals.md · 02-service-decomposition.md · 03-inter-service-communication.md

**Related Topics:** Messaging Patterns (MS #4) · Fault Tolerance (NFR #6) · Observability (NFR #8)

**What to Learn Next:** 05-microservices-operational.md

---

*System Design Engineering Handbook — 07-Microservices Series*