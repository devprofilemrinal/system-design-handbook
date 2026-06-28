# 03 — High-Level Design & Low-Level Design

> **00-Introduction** — Engineering Handbook
> Language-agnostic · 8 min read

---

## 1. The Two Layers of Design

Once requirements are defined, the design work happens in two distinct layers. They answer different questions and operate at different levels of abstraction.

```
HIGH-LEVEL DESIGN (HLD)
  "What are the major components and how do they connect?"

  Components, services, databases, APIs, data flows.
  The blueprint. The 30,000-foot view.
  A competent engineer reading the HLD knows what the system does
  and how it is structured — without knowing the implementation details.

LOW-LEVEL DESIGN (LLD)
  "What exactly is inside each component?"

  Database schemas, detailed API contracts, data models,
  specific algorithms for the hard problems, sequence flows
  within a component.
  The construction drawings. The ground-level detail.
```

Neither is more important than the other — they serve different purposes at different stages. In a 45-minute system design interview, you spend most time on HLD and go deep on LLD only for the 2-3 hardest problems.

---

## 2. High-Level Design — The Blueprint

HLD answers six questions. Every HLD, for any system, covers these six things.

### Question 1: What Are the Major Components?

Name and describe each major component — services, databases, caches, queues, CDNs, gateways. Every box in your diagram needs a name and a one-line purpose.

```
For a URL shortener:

Components:
  API Gateway          — receives all inbound requests, handles auth
  URL Shortening Svc   — generates short codes from long URLs
  Redirect Service     — resolves short codes to long URLs (hot path)
  PostgreSQL           — source of truth for all URL mappings
  Redis Cache          — in-memory store for hot short codes
  CDN                  — caches redirect responses at the edge
```

### Question 2: How Do They Connect?

Show data flow between components. Which components talk to which? Synchronous (direct call) or asynchronous (via queue)?

```
HLD data flow for URL shortener:

Create flow:    Client → API Gateway → Shortening Svc → PostgreSQL
                                                       → Redis (write-through)

Redirect flow:  Client → CDN (cache hit → done)
                Client → CDN (miss) → API Gateway → Redirect Svc → Redis → PostgreSQL
```

### Question 3: What APIs Does the System Expose?

Define the external interface. Not every field — the endpoint, the method, the key inputs and outputs, and why the contract is designed that way.

```
POST /urls
  Input:  { long_url, custom_alias?, expires_at? }
  Output: { short_url, short_code }
  Why: POST because we're creating a resource

GET /{short_code}
  Output: 302 redirect → long_url
  Why: 302 not 301 — preserves analytics and allows destination updates
```

### Question 4: What Databases Are Used and Why?

Choose the database for each component based on access pattern, scale, and consistency requirements. Justify the choice — this is one of the highest-signal decisions in any HLD.

```
Decision framework:

  What is the access pattern?
    Key-value lookups → Redis, DynamoDB
    Time-ordered writes per entity → Cassandra
    Complex queries with joins → PostgreSQL, MySQL
    Full-text search → Elasticsearch

  What is the consistency requirement?
    Strong (financial, inventory) → RDBMS with ACID
    Eventual (social feeds, analytics) → NoSQL with replication

  What is the write throughput?
    < 10,000 writes/sec → single relational DB handles it
    > 100,000 writes/sec → sharded NoSQL or distributed DB

  What is the data shape?
    Structured, relational → SQL
    Document-like, flexible schema → MongoDB, DynamoDB
    Wide rows per entity, time-ordered → Cassandra
    Graph relationships → Neo4j
```

### Question 5: What Are the Critical Data Flows?

Walk through the most important user journeys step by step, tracing the request through every component. This is often drawn as a sequence diagram.

```
For WhatsApp message delivery:
  1. Sender's app sends message to Chat Server via WebSocket
  2. Chat Server writes message to Cassandra
  3. Chat Server publishes event to Kafka
  4. Kafka routes event to recipient's Chat Server
  5. Recipient's Chat Server pushes message via WebSocket (if online)
  6. If offline: Push Notification Service delivers via APNs/FCM
```

### Question 6: Where Are the Non-Functional Concerns Addressed?

Show explicitly where each NFR is handled in the architecture.

```
Availability → Redundant services behind load balancer;
               DB leader-follower with automatic failover

Latency      → Redis cache on the hot read path;
               CDN for cacheable responses

Throughput   → Horizontal scaling of stateless services;
               Kafka to decouple write spikes

Consistency  → PostgreSQL with ACID for financial data;
               Eventual consistency acceptable for feed data

Durability   → Cassandra with RF=3, acks=all for messages;
               S3 for media with cross-region replication
```

---

## 3. The HLD Diagram

A good HLD diagram is the centrepiece of the design. Every component is a box. Every relationship is an arrow. Every arrow has a direction.

### What to Include

```
✓ Client entry points (browser, mobile, API consumer)
✓ Load balancer / API Gateway
✓ Core services (named by their function, not technology)
✓ Databases (with the type — relational, NoSQL, cache, object store)
✓ Async layer (queue, stream) where needed
✓ External services (payment provider, SMS gateway, CDN)
✓ CDN if content delivery is relevant

✗ Internal implementation details of each service
✗ Deployment topology (Kubernetes, Docker — not needed at HLD)
✗ Monitoring, logging pipelines (unless Observability is a core requirement)
✗ Every edge case handled — show the happy path first
```

### A Typical HLD Diagram Structure

```
                              ┌─────────────────┐
                              │     Client       │
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │   API Gateway   │  ← TLS, auth, rate limit
                              └────────┬────────┘
                                       │
               ┌───────────────────────┼───────────────────────┐
               │                       │                       │
      ┌────────▼───────┐    ┌──────────▼──────┐    ┌──────────▼──────┐
      │  Service A     │    │   Service B     │    │   Service C     │
      └────────┬───────┘    └──────────┬──────┘    └─────────────────┘
               │                       │
      ┌────────▼───────┐    ┌──────────▼──────┐
      │   Primary DB   │    │     Cache       │
      │  (PostgreSQL)  │    │    (Redis)      │
      └────────┬───────┘    └─────────────────┘
               │
      ┌────────▼───────┐
      │   Replica DB   │
      └────────────────┘
```

---

## 4. Low-Level Design — Inside the Components

LLD zooms into the internals of each component. In system design context (not OOP), this means three things:

### 1. Database Schema

Define the tables (or collections, or wide-column structures) that store the system's data. Include the columns, data types, primary keys, indexes, and the reasoning for each design decision.

```
-- For a messaging system

CREATE TABLE messages (
    conversation_id  UUID,
    message_id       BIGINT,      -- Snowflake ID (embeds timestamp, sortable)
    sender_id        UUID,
    content          TEXT,
    message_type     VARCHAR(10), -- 'text', 'image', 'voice'
    created_at       TIMESTAMP,
    PRIMARY KEY (conversation_id, message_id)
) WITH CLUSTERING ORDER BY (message_id DESC);

Why Cassandra:
  Write throughput: 231,000 writes/sec (estimated)
  Access pattern: always "get recent messages for conversation X"
  Partition key: conversation_id (all messages for one conversation together)
  Clustering key: message_id DESC (newest first, efficient for pagination)
```

### 2. API Contracts

Define the detailed interface of each endpoint — not just the method and URL, but the full request and response structure, error codes, and edge case behaviour.

```
POST /messages
  Headers:
    Authorization: Bearer <token>
    Idempotency-Key: <uuid>        ← required; prevents duplicate messages

  Body:
    {
      conversation_id: "uuid",
      content: "Hello!",
      message_type: "text"
    }

  Success: 201 Created
    { message_id: "12345678", created_at: "2024-01-15T14:37:22Z" }

  Errors:
    400 Bad Request    — content is empty or > 500 chars
    401 Unauthorized   — invalid or missing token
    403 Forbidden      — sender is not a member of this conversation
    409 Conflict       — duplicate Idempotency-Key (message already sent)
    429 Too Many Requests — rate limit exceeded
```

### 3. Sequence Diagrams for Complex Flows

For any flow involving multiple services, timed operations, or error handling, a sequence diagram shows the exact order of operations.

```
Message delivery sequence (simplified):

Sender App       Chat Server      Cassandra        Kafka        Receiver App
     │                │                │              │               │
     │──SEND msg──→   │                │              │               │
     │                │──INSERT──────→ │              │               │
     │                │ ←──OK──────── │              │               │
     │  ←──ACK sent── │                │              │               │
     │                │──PUBLISH────────────────────→ │               │
     │                │                │              │──DELIVER────→ │
     │                │                │              │               │
```

---

## 5. HLD vs LLD — What Goes Where

A common confusion: what belongs in HLD and what in LLD?

| Question | HLD | LLD |
|---|---|---|
| What are the services? | ✅ | |
| What DB technology? | ✅ | |
| What is the API endpoint? | ✅ (name + purpose) | ✅ (full contract) |
| What are the DB tables? | (naming + purpose) | ✅ (full schema) |
| How does the data flow? | ✅ (component-level) | ✅ (operation-level) |
| What algorithm solves the hard problem? | (identify the problem) | ✅ (the algorithm) |
| Where does the cache sit? | ✅ | |
| What is the cache TTL? | | ✅ |
| What consistency model? | ✅ | |
| How is the partition key chosen? | | ✅ |

HLD answers "what" and "why." LLD answers "how" — the specific implementation decisions that make it work.

---

## 6. In a 45-Minute Interview

Time is limited. Here is how to allocate HLD and LLD time.

```
Minutes 0-10:   Requirements (FR + NFR + estimation)
Minutes 10-25:  HLD — all six components:
                  ✓ Draw the diagram (all components and connections)
                  ✓ Walk through the core data flow
                  ✓ Name the databases and justify the choices
                  ✓ Show the APIs at a high level

Minutes 25-40:  LLD — go deep on 2-3 hard problems:
                  ✓ The interviewer will direct you, or you pick the hardest
                  ✓ Schema for the most complex entity
                  ✓ The algorithm for the most interesting problem
                  ✓ The sequence diagram for the trickiest flow

Minutes 40-45:  Wrap-up
                  ✓ Summarise trade-offs
                  ✓ Identify what's not yet addressed
                  ✓ Invite questions
```

**The most common time allocation mistake:** spending too long on the HLD diagram and never getting to the LLD deep dive. The LLD deep dive is where senior candidates differentiate themselves. Draw the HLD quickly, state the components, move to depth.

---

## 7. The Trade-offs Section

Every good HLD and LLD ends with a trade-offs discussion. This is not optional — it is where you demonstrate that you understand what you built.

```
Format for each trade-off:

Decision:   [what you chose]
Over:       [what you didn't choose]
Gives you:  [the benefit]
Costs you:  [what you gave up]
Acceptable because: [why the cost is worth it given your requirements]

Example:

Decision:   Cassandra for message storage
Over:       PostgreSQL
Gives you:  231,000 writes/sec; horizontal scalability; time-ordered reads by conversation
Costs you:  No ACID transactions; limited query flexibility; no joins
Acceptable because: Access pattern is purely "insert + fetch by conversation ID ordered by time"
                    We never need cross-conversation queries. ACID not required for messages.
```

---

## 8. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Drawing HLD without justifying component choices | Looks like a guess | Every component earns its place with one sentence of justification |
| LLD that is just OOP class diagrams | Wrong level of abstraction for system design | LLD = schema, API contracts, sequence diagrams, algorithms |
| No trade-offs discussion | Design looks like it has no weaknesses | Every choice sacrifices something — name it |
| HLD that doesn't address NFRs | Architecture disconnected from requirements | Show explicitly where each NFR is handled |
| Too much time on HLD, no LLD depth | Junior signal — can't go deep | Time-box HLD; get to the hard problems |
| Schema without justification of key choices | Looks like a database exercise | Explain why each primary key and index was chosen |

---

## 9. Summary

| Layer | Question | Contains |
|---|---|---|
| **HLD** | What are the components and how do they connect? | Services, databases, APIs (names), data flow, NFR mapping |
| **LLD** | What exactly is inside each component? | DB schema, API contracts, sequence diagrams, algorithms |
| **Trade-offs** | What did you give up and why? | One entry per major decision |

**The relationship between them:** HLD identifies the hard problems. LLD solves them. Trade-offs justify the solutions.

A system design is complete when you can answer: what are all the components, how do they connect, what does the data look like, how does the hard flow work, and what did each major decision cost you?

---

## 10. Cross References

**Prerequisites:** `02-functional-and-non-functional-requirements.md`

**Next:** `01-NFR/` — deep dives on each non-functional requirement

**For interviews:** `11-Interview/01-how-to-approach-system-design.md`

**See HLD and LLD in practice:** Any of the 30 case studies in `10-Case-Studies/`

---

*System Design Engineering Handbook — 00-Introduction*