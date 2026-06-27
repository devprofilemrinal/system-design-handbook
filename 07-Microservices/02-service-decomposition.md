# 02 — Service Decomposition

> **07-Microservices Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Service Decomposition?

Service decomposition is the process of deciding how to split a system into individual microservices. It is the most consequential decision in microservices architecture — wrong boundaries are extremely expensive to fix once services are in production and other teams depend on them.

The goal is to find boundaries where:
- Each service is cohesive (everything inside belongs together)
- Services are loosely coupled (changes inside one service don't force changes in others)
- Each service can be deployed, scaled, and operated independently

> **The hardest part of microservices is not the technology. It is knowing where to draw the lines.**

---

## 2. Decomposition Strategy 1 — By Business Capability

Split services along the lines of what the business *does*, not how the code is technically structured.

A business capability is something the business does to create value — like "process payments," "manage inventory," or "send notifications." Each capability maps to one service.

```
E-commerce business capabilities:
  ├── User Management     → User Service
  ├── Product Catalogue   → Catalogue Service
  ├── Order Management    → Order Service
  ├── Payment Processing  → Payment Service
  ├── Inventory Tracking  → Inventory Service
  ├── Shipping            → Shipping Service
  └── Notifications       → Notification Service
```

**Why this works:** Business capabilities are stable. The business may change *how* it processes payments (new provider, new rules) but it will always need to process payments. Stable boundaries mean fewer service refactorings over time.

**How to identify capabilities:** Ask "what does this business do?" not "what does the code do?" Interview domain experts (product managers, business analysts). Draw an organisational chart — each department is often a capability.

---

## 3. Decomposition Strategy 2 — Domain-Driven Design (DDD)

Domain-Driven Design gives us the vocabulary and tools to find service boundaries systematically. The most important DDD concept for microservices is the **Bounded Context**.

### Bounded Context

A bounded context is a boundary within which a particular model is defined and applicable. The same word can mean different things in different contexts — and that's okay, as long as the boundary is explicit.

```
The word "Customer" in different contexts:

Sales context:     Customer = a lead being sold to (has: name, company, deal stage)
Support context:   Customer = a user with a ticket (has: name, ticket history, SLA)
Billing context:   Customer = an account to charge (has: name, payment method, invoices)

Same word, three completely different models.
One service should not own all three — it would be doing three different things.
```

Each bounded context maps to one (or sometimes a few closely related) microservices. The boundary of the bounded context is the boundary of the service.

### Ubiquitous Language

Within a bounded context, everyone — engineers, product managers, domain experts — uses the same terms with the same meanings. This shared language is embedded in the code, the APIs, and the data models.

```
In the Order Service:
  Engineers, product managers, and customer service all call it an "Order"
  The API endpoint is /orders
  The database table is orders
  The code class is Order

No synonyms, no ambiguity within the boundary.
```

### Aggregates — The Unit of Consistency

An **aggregate** is a cluster of related domain objects that should be treated as a single unit for data changes. The aggregate root is the entry point — all modifications go through it.

```
Order aggregate:
  Root: Order (id, status, total)
  Contains: OrderLine items, ShippingAddress, PaymentInfo

Rules:
  - You can only modify the Order through the Order root
  - OrderLine items cannot be accessed directly from outside
  - The entire Order is saved/retrieved as one unit
```

Aggregates help define service boundaries: one service typically owns one or a few aggregates. Cross-aggregate operations require coordination across services (Saga pattern).

---

## 4. Decomposition Strategy 3 — By Subdomain

DDD distinguishes between different types of subdomains that deserve different levels of investment:

| Subdomain Type | Description | Strategy |
|---|---|---|
| **Core domain** | What makes the business unique; the competitive advantage | Build it yourself; invest heavily; keep it internal |
| **Supporting domain** | Necessary but not differentiating | Build or buy; less critical |
| **Generic domain** | Commodity functionality used across industries | Buy or use open-source; don't reinvent |

```
Ride-sharing company:

Core domain:    Driver-rider matching algorithm → build, own, protect
Supporting:     Trip management, payment processing → build or buy
Generic:        Email sending, SMS, maps → use third-party APIs

→ Core domain = most important microservice; most engineering investment
→ Generic domain = don't build a microservice; call an external API
```

---

## 5. What Makes a Good vs Bad Boundary

### Signs of a Good Boundary

```
✅ The service can be deployed without coordinating with other services
✅ The service's database changes don't affect other services
✅ A single team fully understands and owns the service
✅ The service's API rarely changes (stable contract)
✅ Features within the service change together (cohesion)
```

### Signs of a Bad Boundary

```
❌ Every change requires simultaneous deployment of two or more services
❌ Services share a database table
❌ A single business feature requires changing three services
❌ The service always calls Service B before it can do anything (tight coupling)
❌ No team feels clear ownership
```

**The change test:** If two services always change together when a feature is added, they should probably be one service. Services that change together belong together.

**The data test:** If Service A needs to read Service B's data constantly to function, the boundary is wrong. Either A is asking for data it should own, or A and B should be merged.

---

## 6. Common Decomposition Mistakes

### Decomposing by Technical Layer (Wrong)

Splitting by technical function (a "database service," a "validation service," an "API service") creates technically-structured services that don't map to business operations. They are always deployed together and have no independent value.

```
❌ Wrong decomposition:
  API Layer Service → Data Processing Service → Database Service
  (these are not business capabilities; they're technical layers)

✅ Right decomposition:
  Order Service (owns its own API layer, business logic, and database)
  Payment Service (owns its own API, logic, and database)
```

### Too Fine-Grained (Nanoservices)

Splitting too small creates chatty services that spend more time calling each other than doing useful work.

```
❌ Too fine-grained:
  GetUserName Service
  GetUserEmail Service
  ValidateUserService
  (three network hops to get a user profile)

✅ Right-sized:
  User Service (returns complete user profile in one call)
```

### Premature Decomposition

Splitting before the domain is understood leads to wrong boundaries. Wrong boundaries are more painful than no boundaries.

```
Timeline:
  Month 1: Build monolith, learn the domain
  Month 6: Boundaries become clear from usage patterns
  Month 12: Extract first service with high confidence in the boundary
  (vs)
  Month 1: Split into 20 microservices based on guesses
  Month 6: Realise boundaries are wrong
  Month 12: Expensive refactoring across service boundaries
```

---

## 7. Practical Decomposition Process

A step-by-step approach for decomposing an existing system or designing a new one:

```
Step 1: Event Storming
  → Workshop with domain experts and engineers
  → List all domain events (things that happen in the business)
  → "OrderPlaced", "PaymentCharged", "ItemShipped", "UserRegistered"
  → Group related events → these groups suggest service boundaries

Step 2: Identify Aggregates
  → What entities own which events?
  → Order aggregate: OrderPlaced, OrderCancelled, OrderShipped
  → Payment aggregate: PaymentCharged, PaymentRefunded

Step 3: Draw Bounded Contexts
  → Each context = one service candidate
  → Check: can this context change independently?
  → Check: does one team naturally own this?

Step 4: Define APIs
  → What does each service expose?
  → What does each service consume?
  → Are there circular dependencies? (if yes, boundary is wrong)

Step 5: Validate With the Change Test
  → For the next 3 planned features, which services change?
  → If two always change together → merge them
  → If one never changes → is it too fine-grained?
```

---

## 8. Event Storming — The Practical Tool

Event Storming is a workshop technique that uses coloured sticky notes to map a business domain:

```
Orange: Domain Events    ("OrderPlaced", "PaymentFailed")
Blue:   Commands         ("PlaceOrder", "CancelOrder")
Yellow: Aggregates       ("Order", "Payment", "User")
Pink:   Policies         ("When PaymentFailed, CancelOrder")
Purple: External Systems ("Stripe", "SendGrid", "Maps API")
```

The output is a domain map that reveals natural service boundaries better than any purely technical analysis.

> **Event Storming works because it forces collaboration between engineers and domain experts.** Engineers discover business rules they didn't know. Domain experts see technical complexity they hadn't considered. The result is boundaries that make sense from both perspectives.

---

## 9. How Large Companies Decompose

| Company | Approach | Source |
|---|---|---|
| **Amazon** | Each service team owns a business capability; two-pizza rule ensures team stays small enough to fully own the service | Public Jeff Bezos mandate |
| **Uber** | Decomposed by domain (trips, drivers, payments, maps) after starting with a monolith; DDD-influenced boundaries | Uber Eng Blog (public) |
| **Netflix** | Business-capability decomposition; chaos testing validates that boundaries are truly independent | Netflix Tech Blog (public) |

---

## 10. Best Practices

- **Decompose by business capability**, not technical function.
- **Use DDD bounded contexts** to find linguistically coherent boundaries.
- **Apply the change test** — services that always change together should be one service.
- **Don't decompose prematurely** — understand the domain first.
- **Run event storming workshops** before decomposing existing systems.
- **Start with 3–5 services**, not 50. Add more as specific needs are proven.
- **Never share a database between services** — it defeats the independence guarantee.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Decomposing by technical layer | Services always deployed together; no independent value | Decompose by business capability |
| Splitting too small | Chatty services; excessive network overhead; hard to trace | Services should be cohesive; fail the change test first |
| Decomposing before domain is known | Wrong boundaries; expensive refactoring | Understand the domain with a modular monolith first |
| Circular service dependencies | A → B → A; impossible to deploy or test independently | Redesign boundaries to remove the cycle |
| Shared database | Coupling at the data layer defeats service independence | One database per service |

---

## 12. Interview Questions

1. What is a bounded context in Domain-Driven Design?
2. How do you decide where to draw the boundary between two services?
3. What is the "change test" for evaluating service boundaries?
4. What is event storming and why is it useful for decomposition?
5. What is the difference between a core domain and a generic domain?
6. What is a distributed monolith and what causes it?
7. You're designing an e-commerce system — how would you decompose it into services?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Decomposition** | The hardest microservices problem. Wrong boundaries are expensive. |
| **By capability** | Split by what the business does. Stable, domain-aligned. |
| **Bounded context** | A boundary where a specific model applies. Maps to a service. |
| **Aggregate** | Unit of consistency. One service owns one or a few aggregates. |
| **Change test** | Always change together → belong together. |
| **Event storming** | Domain mapping workshop. Best tool for finding natural boundaries. |
| **Too small** | Chatty nanoservices. More overhead than value. |
| **Too large** | Multiple capabilities; team coupling; defeats microservices purpose. |

---

## 14. Cross References

**Prerequisites:** 01-microservices-fundamentals.md · Distributed Systems Fundamentals (DS #1)

**Related Topics:** 03-inter-service-communication.md · 04-microservices-patterns.md · Messaging Patterns (MS #4)

**What to Learn Next:** 03-inter-service-communication.md

---

*System Design Engineering Handbook — 07-Microservices Series*