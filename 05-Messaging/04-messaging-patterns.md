# 04 — Messaging Patterns

> **05-Messaging Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Why Patterns Matter

A message broker is infrastructure. Patterns are how you use that infrastructure to solve specific design problems. The same broker (Kafka, RabbitMQ) can implement wildly different architectures depending on which patterns you apply.

This document covers the most important messaging patterns for system design interviews and production systems:

| Pattern | Problem It Solves |
|---|---|
| **Pub/Sub** | One event → many independent consumers |
| **Fan-out** | Broadcast to all subscribers without coupling |
| **Competing Consumers** | Scale processing by distributing load |
| **Event-Driven Architecture** | Decouple services; react to state changes |
| **Saga** | Distributed transaction across services |
| **Event Sourcing** | State as a sequence of events, not current snapshot |
| **Outbox Pattern** | Atomic DB write + message publish |

---

## 2. Publish/Subscribe (Pub/Sub)

In pub/sub, a producer publishes a message to a **topic**. All subscribers to that topic receive a copy of the message independently.

```
Publisher                Topic               Subscribers
                    "order-created"
Order Service  ──→  [order:1001]  ──→  Payment Service (charges card)
                                   ──→  Inventory Service (reserves stock)
                                   ──→  Notification Service (sends email)
                                   ──→  Analytics Service (updates dashboard)
```

**Key characteristic:** The publisher has no knowledge of who the subscribers are. Adding a new subscriber requires no change to the publisher.

**Contrast with direct call:**
```
Direct: Order Service → calls Payment Service
                      → calls Inventory Service
                      → calls Notification Service
        (Order Service knows about and depends on all three)

Pub/Sub: Order Service → publishes "order-created"
         (Order Service knows about nothing; subscribers self-register)
```

> **Pub/Sub is the primary decoupling mechanism in event-driven systems.** New consumers can be added with zero changes to the producer — the system evolves by adding subscribers, not modifying existing services.

---

## 3. Fan-Out

Fan-out is a specific form of pub/sub where one message is delivered to **all** subscribers without any filtering. Every subscriber gets every message.

```
Producer → [Topic] → Subscriber 1 (gets copy)
                   → Subscriber 2 (gets copy)
                   → Subscriber 3 (gets copy)
                   → Subscriber N (gets copy)
```

**Use cases:**
- A new user registration event that must trigger: welcome email, account setup, analytics event, risk check — all simultaneously
- A configuration change that all service instances must receive immediately
- A market data feed that all trading algorithms must process

**Fan-out vs Competing Consumers:**

```
FAN-OUT: all consumers get every message
  → Each consumer does different work with the same event
  → Parallelism of purpose

COMPETING CONSUMERS: each message goes to one consumer
  → All consumers do the same work
  → Parallelism of throughput
```

In Kafka, fan-out is achieved with multiple consumer groups. In RabbitMQ, it's achieved with a fanout exchange.

---

## 4. Event-Driven Architecture (EDA)

In EDA, services communicate entirely through events. A service publishes an event when its state changes. Other services react to that event independently.

```
Traditional (request-driven):
  Order Service calls Payment Service
  Order Service calls Inventory Service
  Order Service calls Notification Service

Event-Driven:
  Order Service: order placed → publishes "order-placed" event
  Payment Service: subscribes to "order-placed" → charges card → publishes "payment-success"
  Inventory Service: subscribes to "order-placed" → reserves stock → publishes "stock-reserved"
  Notification Service: subscribes to "payment-success" → sends confirmation email
```

The result: services are loosely coupled. Each service only knows what events it cares about. Adding a new service (e.g., loyalty-points-service) requires no change to any existing service — just subscribe to "payment-success."

**EDA trade-offs:**

| Benefit | Cost |
|---|---|
| Loose coupling | Harder to trace a flow end-to-end |
| Independent scaling | Eventual consistency — state changes are async |
| Easy extensibility | Debugging requires distributed tracing |
| No synchronous bottleneck | Complex error handling and compensation |

---

## 5. Saga Pattern — Distributed Transactions

When a business transaction spans multiple services, you cannot use a single database transaction. The Saga pattern breaks the operation into a sequence of local transactions, each with a compensating action if something goes wrong.

```
Business operation: "Place an order"
  Step 1: Order Service → create order record   (compensate: cancel order)
  Step 2: Payment Service → charge card         (compensate: refund card)
  Step 3: Inventory Service → reserve stock     (compensate: release stock)
  Step 4: Notification Service → send email     (no compensation needed)

Failure at Step 3 (out of stock):
  → Run compensating action for Step 2: refund card
  → Run compensating action for Step 1: cancel order
  → System returns to consistent state
```

### Choreography vs Orchestration

**Choreography:** Each service listens for events and decides what to do next. No central coordinator.

```
Order Service:  places order → publishes "order-created"
Payment Service: hears "order-created" → charges card → publishes "payment-completed"
Inventory:       hears "payment-completed" → reserves stock → publishes "stock-reserved"
Notification:    hears "stock-reserved" → sends email

Compensations triggered by "payment-failed", "stock-unavailable" events.
```

**Orchestration:** A central saga orchestrator tells each service what to do next.

```
Orchestrator:
  → "Payment Service: charge card"
  Payment Service: charges, responds "success"
  → "Inventory Service: reserve stock"
  Inventory: fails, responds "out of stock"
  → "Payment Service: refund card" (compensation)
  → "Order Service: cancel order" (compensation)
```

| | Choreography | Orchestration |
|---|---|---|
| **Coordination** | Distributed (via events) | Centralised (orchestrator) |
| **Coupling** | Low — services don't know each other | Medium — services know orchestrator |
| **Visibility** | Hard to see the overall flow | Easy — orchestrator knows state |
| **Complexity** | Distributed complexity | Centralised complexity |
| **Best for** | Simple, short sagas | Complex, long-running sagas |

---

## 6. Event Sourcing

Instead of storing current state, store the **sequence of events** that led to that state. The current state is derived by replaying all events.

```
Traditional storage:
  Account table: {id: 123, balance: £350}
  (just the current value — history is lost)

Event Sourcing:
  Event log: [
    {account:123, type:"deposit",  amount:£500},
    {account:123, type:"withdraw", amount:£100},
    {account:123, type:"withdraw", amount:£50}
  ]
  Current balance = replay events: £500 - £100 - £50 = £350
```

**Benefits:**

| Benefit | Why It Matters |
|---|---|
| **Full audit trail** | Every change is recorded permanently |
| **Time travel** | Reconstruct state at any point in the past |
| **Event replay** | Rebuild any derived view from the same events |
| **Debugging** | See exactly what happened and in what order |

**Costs:**

| Cost | Why It Hurts |
|---|---|
| **Query complexity** | Current state requires replaying events; need snapshots |
| **Storage growth** | Event log grows forever |
| **Schema evolution** | Old events with old schemas must still be processable |
| **Conceptual shift** | Developers must think in events, not state |

> **Event sourcing is powerful but heavy.** Apply it where audit trail and history are genuinely valuable (financial systems, compliance-heavy domains). Don't apply it everywhere by default.

**Snapshots** solve the replay performance problem:

```
After every 1,000 events, store a snapshot of current state.
To get current state: load latest snapshot → replay only events since snapshot.
```

---

## 7. The Outbox Pattern — Atomic Write + Publish

A critical reliability problem: you update the database and then publish a message to the broker. What if the process crashes between the two?

```
BAD:
Step 1: Update database (success)
Step 2: Publish to Kafka (CRASH HERE)

Result: DB updated, no event published → other services never notified → inconsistency
```

The **Outbox Pattern** solves this by treating the message publish as part of the database transaction.

```
OUTBOX PATTERN:
Step 1: In a single DB transaction:
          - Update the main record
          - INSERT the message into an "outbox" table

Step 2: A separate background process:
          - Reads unsent messages from the outbox table
          - Publishes to the message broker
          - Marks message as sent

Result: DB update and message publishing are atomic.
         Crash anywhere → transaction rolls back OR message is republished.
```

```
DB Transaction (atomic):
  UPDATE orders SET status='paid' WHERE id=1001
  INSERT INTO outbox (topic, payload, sent=false)
         VALUES ('order-events', '{"order_id":1001,"event":"paid"}', false)

Background publisher:
  SELECT * FROM outbox WHERE sent=false
  → publish to Kafka
  → UPDATE outbox SET sent=true WHERE id=...
```

The outbox table is inside the same database as your business data, so both updates happen atomically. The background publisher guarantees at-least-once delivery (it may publish twice if it crashes after publishing but before marking sent — consumers must be idempotent).

---

## 8. Combining Patterns — A Real System

Real systems layer multiple patterns. Here's a payment platform:

```
User places order:
  Order Service → [Outbox Pattern] → publishes "order-placed"

Saga (Orchestrated):
  Orchestrator listens to "order-placed"
  → Pub/Sub: Payment Service charges card
  → Pub/Sub: Inventory reserves stock
  → Fan-out: Notification + Analytics + Loyalty all receive "order-confirmed"

If payment fails:
  → Saga compensation: cancel order
  → Publish "order-failed" event
  → Notification Service (subscribed) → sends failure email

Event Sourcing (within Payment Service):
  All payment events stored in event log
  Balance always computed from event replay
  Full audit trail for compliance
```

---

## 9. How Large Companies Apply These Patterns

| Company | Pattern | Application | Source |
|---|---|---|---|
| **Uber** | Saga + Choreography | Booking flow: driver assignment → payment → confirmation | Uber Eng Blog (public) |
| **Amazon** | Event-driven + Pub/Sub | Order pipeline: placed → payment → fulfilment → shipping | Public architecture talks |
| **Airbnb** | Saga + Orchestration | Reservation: booking → payment → host notification | Airbnb Eng Blog (public) |
| **Event sourcing** | Financial systems broadly | Ledger as immutable event log; full audit trail | Industry standard |

---

## 10. Best Practices

- **Use pub/sub to decouple services** — publishers should not know about their subscribers.
- **Use Saga for distributed transactions** — never use 2PC in microservices.
- **Use the Outbox Pattern** whenever you update a DB and need to publish a message — it's the only safe way.
- **Make all consumers idempotent** — essential for at-least-once delivery and Saga compensation.
- **Choose Saga choreography for simple flows; orchestration for complex ones.**
- **Apply Event Sourcing selectively** — only where audit trail and historical replay are genuine requirements.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Publishing event outside DB transaction | Crash between DB update and publish → inconsistency | Use Outbox Pattern |
| Saga without compensating actions | Failed step leaves system in partially-applied state | Define compensation for every saga step |
| Non-idempotent saga steps | Retried step causes duplicate side effects | Idempotency keys on all saga operations |
| Event sourcing everywhere | Massive complexity for simple CRUD data | Apply only where history/audit is genuinely needed |
| Choreography for complex sagas | Impossible to trace what state the saga is in | Use orchestration for multi-step, long-running sagas |

---

## 12. Interview Questions

1. What is pub/sub and how does it differ from a direct service call?
2. What is the difference between fan-out and competing consumers?
3. What is the Saga pattern and when do you use it instead of a database transaction?
4. Compare Saga choreography vs orchestration. When would you use each?
5. What is event sourcing? What problem does it solve and what does it cost?
6. What is the Outbox Pattern and why is it needed?
7. How would you design an order processing system using event-driven patterns?

---

## 13. Summary

| Pattern | Purpose | Key Trade-off |
|---|---|---|
| **Pub/Sub** | One event → many independent consumers | Eventual consistency; harder to trace |
| **Fan-out** | Every subscriber gets every message | All consumers must handle all messages |
| **Competing Consumers** | Scale throughput across workers | No global ordering; idempotency required |
| **Event-Driven** | Decouple services via events | Complex debugging; eventual consistency |
| **Saga** | Distributed transaction with compensation | Complex; no true atomicity |
| **Event Sourcing** | State as immutable event log | Storage cost; query complexity |
| **Outbox** | Atomic DB write + message publish | Extra table; background process needed |

---

## 14. Cross References

**Prerequisites:** 01-messaging-fundamentals.md · 02-message-queues.md · 03-event-streaming.md

**Related Topics:** Consistency (NFR #5) · Fault Tolerance (NFR #6) · Transactions & Isolation (DB #7)

**What to Learn Next:** 05-kafka-vs-rabbitmq.md

---

*System Design Engineering Handbook — 05-Messaging Series*