# System Design Handbook

> A comprehensive, production-quality knowledge base for system design — built for software engineers preparing for interviews at top technology companies and for engineers who want to deepen their understanding of how large-scale systems are designed and operated.

---

## What This Is

This handbook is a structured, self-contained reference covering everything that appears in system design interviews and in real engineering work at scale. Every document is:

- **Language-agnostic** — concepts, diagrams, and trade-offs. No framework-specific code.
- **8–10 minute reads** — focused and dense. No padding, no repetition.
- **Diagram-first** — Mermaid diagrams and tables replace prose wherever possible.
- **Interview-ready** — every file ends with interview questions and a summary.

This is not a collection of blog posts. It is a structured engineering handbook — written to be read in order and referenced repeatedly.

---

## Who This Is For

- Software engineers preparing for **system design interviews** at FAANG, MAANG, and similar companies
- Engineers moving from mid-level to senior who need to develop architectural thinking
- Engineers who want a structured reference for production system design decisions

**Prerequisites:** Familiarity with basic data structures, SQL fundamentals, and HTTP. Everything else is built from first principles.

---

## How to Use This Handbook

### For Interview Preparation (Recommended Path)

```
1. Start with 00-Introduction
   → Understand FR, NFR, HLD, LLD

2. Read 01-NFR cover to cover
   → This is the vocabulary all other sections use

3. Read 02-Building-Blocks
   → Load balancers, CDN, API gateways — the components in every design

4. Read 11-Interview first
   → The framework, estimation cheatsheet, and trade-off vocabulary
   → Makes everything else more useful

5. Work through 10-Case-Studies
   → Apply everything you've learned to real systems
   → Study one per day; write your own design before reading

6. Use 03-07 as deep-dive references
   → When a case study uses Cassandra, go to 03-Databases/03-nosql.md
   → When it uses Kafka, go to 05-Messaging/03-event-streaming.md
```

### For a Specific Topic

Use the table of contents below to jump directly to any topic. Every file includes cross-references to related documents.

---

## Repository Structure

```
system-design-handbook/
│
├── 00-Introduction/
│   └── 01-system-design-introduction.md
│
├── 01-NFR/                          ← Non-Functional Requirements (9 files)
│   ├── 01-latency-and-throughput.md
│   ├── 02-availability.md
│   ├── 03-scalability.md
│   ├── 04-reliability.md
│   ├── 05-consistency.md
│   ├── 06-fault-tolerance.md
│   ├── 07-security.md
│   ├── 08-observability-maintainability.md
│   └── 09-durability-disaster-recovery.md
│
├── 02-Building-Blocks/              ← Core Infrastructure Components (7 files)
│   ├── 01-load-balancers.md
│   ├── 02-api-gateway.md
│   ├── 03-reverse-proxy.md
│   ├── 04-cdn.md
│   ├── 05-rate-limiting.md
│   ├── 06-dns.md
│   └── 07-service-discovery.md
│
├── 03-Databases/                    ← Database Deep Dives (9 files)
│   ├── 01-database-fundamentals.md
│   ├── 02-sql-relational.md
│   ├── 03-nosql.md
│   ├── 04-indexing.md
│   ├── 05-replication.md
│   ├── 06-partitioning-sharding.md
│   ├── 07-transactions-isolation.md
│   ├── 08-caching.md
│   └── 09-database-selection.md
│
├── 04-Caching/                      ← Caching Strategies (5 files)
│   ├── 01-caching-fundamentals.md
│   ├── 02-caching-strategies.md
│   ├── 03-eviction-policies.md
│   ├── 04-distributed-caching.md
│   └── 05-cache-invalidation.md
│
├── 05-Messaging/                    ← Messaging & Event Streaming (5 files)
│   ├── 01-messaging-fundamentals.md
│   ├── 02-message-queues.md
│   ├── 03-event-streaming.md
│   ├── 04-messaging-patterns.md
│   └── 05-kafka-vs-rabbitmq.md
│
├── 06-Distributed-Systems/          ← Distributed Systems Theory (5 files)
│   ├── 01-distributed-systems-fundamentals.md
│   ├── 02-cap-theorem.md
│   ├── 03-consistency-and-clocks.md
│   ├── 04-consensus-and-leader-election.md
│   └── 05-consistent-hashing.md
│
├── 07-Microservices/                ← Microservices Architecture (5 files)
│   ├── 01-microservices-fundamentals.md
│   ├── 02-service-decomposition.md
│   ├── 03-inter-service-communication.md
│   ├── 04-microservices-patterns.md
│   └── 05-microservices-operational.md
│
├── 08-Observability/                ← Observability & Operations (5 files)
│   ├── 01-observability-fundamentals.md
│   ├── 02-logging.md
│   ├── 03-metrics-and-alerting.md
│   ├── 04-distributed-tracing.md
│   └── 05-incident-response.md
│
├── 09-Security/                     ← Security Engineering (7 files)
│   ├── 01-security-fundamentals.md
│   ├── 02-authentication.md
│   ├── 03-authorisation.md
│   ├── 04-transport-security.md
│   ├── 05-application-security.md
│   ├── 06-secrets-and-key-management.md
│   └── 07-api-security.md
│
├── 10-Case-Studies/                 ← 30 System Design Case Studies
│   ├── 01-url-shortener.md
│   ├── 02-rate-limiter.md
│   ├── 03-key-value-store.md
│   ├── 04-twitter-social-feed.md
│   ├── 05-whatsapp-chat-system.md
│   ├── 06-youtube-video-platform.md
│   ├── 07-uber-ride-sharing.md
│   ├── 08-google-search.md
│   ├── 09-netflix.md
│   ├── 10-payment-system.md
│   ├── 11-notification-system.md
│   ├── 12-web-crawler.md
│   ├── 13-distributed-cache.md
│   ├── 14-typeahead-autocomplete.md
│   ├── 15-distributed-message-queue.md
│   ├── 16-instagram-photo-sharing.md
│   ├── 17-google-drive-dropbox.md
│   ├── 18-ticketmaster-booking.md
│   ├── 19-slack-team-messaging.md
│   ├── 20-zoom-video-conferencing.md
│   ├── 21-amazon-ecommerce.md
│   ├── 22-google-maps.md
│   ├── 23-leaderboard-gaming.md
│   ├── 24-pastebin-text-storage.md
│   ├── 25-distributed-job-scheduler.md
│   ├── 26-ad-click-aggregation.md
│   ├── 27-live-streaming-twitch.md
│   ├── 28-fraud-detection.md
│   ├── 29-search-autocomplete-api.md
│   └── 30-metrics-monitoring-system.md
│
├── 11-Interview/                    ← Interview Preparation (4 files)
│   ├── 01-how-to-approach-system-design.md
│   ├── 02-estimation-cheatsheet.md
│   ├── 03-common-mistakes.md
│   └── 04-trade-off-vocabulary.md
│
├── assets/                          ← Diagrams and images
├── glossary/                        ← Term definitions
├── templates/                       ← Document templates
├── .gitignore
├── LICENSE
└── README.md
```

---

## What Each Section Covers

### 00 — Introduction
The foundational framework for system design: Functional Requirements, Non-Functional Requirements, High-Level Design, and Low-Level Design. What they are, how they relate, and how to use them together.

### 01 — Non-Functional Requirements
The nine quality attributes that define how well a system works. Each file covers one NFR: what it is, how to measure it, and what architectural decisions it drives.

| File | Topic |
|---|---|
| 01 | Latency & Throughput — percentiles, Little's Law, queuing theory |
| 02 | Availability — nines, SLA/SLO/SLI, error budgets |
| 03 | Scalability — vertical vs horizontal, sharding, statelessness |
| 04 | Reliability — MTBF/MTTR, idempotency, durability |
| 05 | Consistency — CAP theorem, consistency models, ACID vs BASE |
| 06 | Fault Tolerance — circuit breakers, cascading failures, resilience patterns |
| 07 | Security — CIA triad, AuthN/AuthZ, encryption, common attacks |
| 08 | Observability & Maintainability — three pillars, golden signals, deployability |
| 09 | Durability & Disaster Recovery — RPO/RTO, backup strategies, DR patterns |

### 02 — Building Blocks
The infrastructure components that appear in every system design. Understanding these means you can explain *why* each component exists, not just draw a box for it.

| File | Topic |
|---|---|
| 01 | Load Balancers — L4 vs L7, algorithms, health checks, SSL termination |
| 02 | API Gateway — routing, auth, rate limiting, BFF pattern |
| 03 | Reverse Proxy — server protection, caching, compression, canary deploys |
| 04 | CDN — edge caching, TTL, cache busting, push vs pull |
| 05 | Rate Limiting — token bucket, leaky bucket, distributed counters |
| 06 | DNS — resolution, TTL trade-offs, GeoDNS, failover |
| 07 | Service Discovery — registries, client vs server-side, health checking |

### 03 — Databases
From fundamentals to selection. The most critical section for interview performance — database choice is the highest-signal decision in any system design.

| File | Topic |
|---|---|
| 01 | Fundamentals — ACID, BASE, storage engines, read/write trade-offs |
| 02 | SQL & Relational — normalisation, joins, transactions, isolation levels |
| 03 | NoSQL — key-value, document, wide-column, graph — when each fits |
| 04 | Indexing — B-trees, composite indexes, covering indexes, cardinality |
| 05 | Replication — leader-follower, sync vs async, replication lag |
| 06 | Partitioning & Sharding — range vs hash, consistent hashing, hotspots |
| 07 | Transactions & Isolation — MVCC, 2PC, Saga pattern |
| 08 | Caching — cache-aside, write-through, TTL, invalidation |
| 09 | Database Selection — decision framework, polyglot persistence, interview scenarios |

### 04 — Caching
The complete caching playbook — from why caching exists to cache stampede prevention.

### 05 — Messaging
Queues, streams, and the patterns built on them. Kafka vs RabbitMQ. The Outbox Pattern. Saga choreography vs orchestration.

### 06 — Distributed Systems
The theory underpinning every large-scale system. The 8 Fallacies, CAP theorem, vector clocks, Raft consensus, consistent hashing.

### 07 — Microservices
When to use microservices (and when not to). Domain-driven decomposition. Inter-service communication. Strangler Fig, BFF, CQRS, Bulkhead patterns.

### 08 — Observability
Structured logging, the four golden signals, distributed tracing, incident response, and post-mortems.

### 09 — Security
Authentication, authorisation, TLS/mTLS, OWASP Top 10, secrets management, and API security. Seven files covering security end-to-end.

### 10 — Case Studies
30 complete system design case studies. Each follows the same structure:
- Problem statement and scope
- Functional and non-functional requirements
- Scale estimation with back-of-envelope math
- High-level design with architecture diagrams
- Deep dives into the 2–3 hardest problems
- Trade-off explanations for every major decision
- Follow-up interview questions

**Beginner:** URL Shortener · Rate Limiter · Key-Value Store · Leaderboard · Pastebin

**Intermediate:** Twitter Feed · WhatsApp · YouTube · Uber · Notification System · Web Crawler · Distributed Cache · Typeahead · Instagram · Google Drive · Ticketmaster · Slack · Amazon E-commerce · Search Autocomplete

**Advanced:** Google Search · Netflix · Payment System · Distributed Message Queue · Zoom · Google Maps · Distributed Job Scheduler · Ad Click Aggregation · Live Streaming · Fraud Detection · Metrics & Monitoring

### 11 — Interview
Four focused files that turn knowledge into interview performance:
1. **How to approach system design** — the 5-phase framework with time allocation
2. **Estimation cheatsheet** — latency numbers, storage units, worked examples
3. **Common mistakes** — 15 failure patterns with corrections
4. **Trade-off vocabulary** — sentence patterns, vocabulary, and responses to common challenges

---

## Case Study Format

Every case study is written as a complete tutorial. The structure is consistent across all 30:

```
1. Problem Statement
   What are we building? Why does it exist? What makes it hard?

2. Requirements
   Functional: what the system must do (explicitly scoped)
   Non-functional: how well it must do it (with numbers)
   Out of scope: what we are deliberately not designing today

3. Scale Estimation
   Back-of-envelope math for traffic, storage, and bandwidth
   Conclusions that directly drive architectural decisions

4. High-Level Design
   API design with request/response examples
   Architecture diagram (Mermaid)
   Component-by-component explanation

5. Deep Dives
   The 2-3 hardest problems, explained in full
   Multiple approaches considered, with trade-offs

6. Trade-offs
   Every major decision with: what we chose, what we gave up, why

7. Follow-up Questions
   What interviewers ask after the main design
   Common extensions and harder variants
```

---

## Topic Quick Reference

Looking for a specific concept? Find it here.

| Concept | Location |
|---|---|
| ACID transactions | 03-Databases/01-database-fundamentals.md |
| Alert design | 08-Observability/03-metrics-and-alerting.md |
| API Gateway | 02-Building-Blocks/02-api-gateway.md |
| Authentication (JWT, OAuth) | 09-Security/02-authentication.md |
| Authorisation (RBAC, ABAC, IDOR) | 09-Security/03-authorisation.md |
| Availability nines | 01-NFR/02-availability.md |
| B-tree indexes | 03-Databases/04-indexing.md |
| BFF pattern | 07-Microservices/04-microservices-patterns.md |
| Bulkhead pattern | 07-Microservices/04-microservices-patterns.md |
| CAP theorem | 06-Distributed-Systems/02-cap-theorem.md |
| Cache invalidation | 04-Caching/05-cache-invalidation.md |
| Cache stampede / Thundering Herd | 04-Caching/05-cache-invalidation.md |
| Cache-aside pattern | 04-Caching/02-caching-strategies.md |
| Cassandra | 03-Databases/03-nosql.md |
| CDN | 02-Building-Blocks/04-cdn.md |
| Circuit breaker | 07-Microservices/03-inter-service-communication.md |
| Consistent hashing | 06-Distributed-Systems/05-consistent-hashing.md |
| CQRS | 07-Microservices/04-microservices-patterns.md |
| Dead letter queue | 05-Messaging/02-message-queues.md |
| Distributed tracing | 08-Observability/04-distributed-tracing.md |
| DNS | 02-Building-Blocks/06-dns.md |
| Error budget | 01-NFR/02-availability.md |
| Estimation formulas | 11-Interview/02-estimation-cheatsheet.md |
| Event sourcing | 05-Messaging/04-messaging-patterns.md |
| Eventual consistency | 01-NFR/05-consistency.md |
| Fault tolerance patterns | 01-NFR/06-fault-tolerance.md |
| Four golden signals | 08-Observability/03-metrics-and-alerting.md |
| GeoDNS | 02-Building-Blocks/06-dns.md |
| gRPC | 07-Microservices/03-inter-service-communication.md |
| HSTS | 09-Security/04-transport-security.md |
| Idempotency | 01-NFR/04-reliability.md |
| Incident response | 08-Observability/05-incident-response.md |
| Indexing strategy | 03-Databases/04-indexing.md |
| JWT | 09-Security/02-authentication.md |
| Kafka | 05-Messaging/03-event-streaming.md |
| Kafka vs RabbitMQ | 05-Messaging/05-kafka-vs-rabbitmq.md |
| Lamport / Vector clocks | 06-Distributed-Systems/03-consistency-and-clocks.md |
| Latency percentiles | 01-NFR/01-latency-and-throughput.md |
| Leader election | 06-Distributed-Systems/04-consensus-and-leader-election.md |
| Little's Law | 01-NFR/01-latency-and-throughput.md |
| Load balancers | 02-Building-Blocks/01-load-balancers.md |
| Logging (structured) | 08-Observability/02-logging.md |
| LRU eviction | 04-Caching/03-eviction-policies.md |
| Message queue | 05-Messaging/02-message-queues.md |
| Microservices decomposition | 07-Microservices/02-service-decomposition.md |
| Monolith vs microservices | 07-Microservices/01-microservices-fundamentals.md |
| mTLS | 09-Security/04-transport-security.md |
| MVCC | 03-Databases/07-transactions-isolation.md |
| NoSQL types | 03-Databases/03-nosql.md |
| OAuth 2.0 | 09-Security/02-authentication.md |
| OpenTelemetry | 08-Observability/04-distributed-tracing.md |
| Outbox pattern | 05-Messaging/04-messaging-patterns.md |
| PACELC | 06-Distributed-Systems/02-cap-theorem.md |
| Partitioning / Sharding | 03-Databases/06-partitioning-sharding.md |
| Post-mortem | 08-Observability/05-incident-response.md |
| Pub/Sub pattern | 05-Messaging/04-messaging-patterns.md |
| Raft consensus | 06-Distributed-Systems/04-consensus-and-leader-election.md |
| Rate limiting algorithms | 02-Building-Blocks/05-rate-limiting.md |
| Read replicas | 03-Databases/05-replication.md |
| Replication (DB) | 03-Databases/05-replication.md |
| Reverse proxy | 02-Building-Blocks/03-reverse-proxy.md |
| RPO / RTO | 01-NFR/09-durability-disaster-recovery.md |
| Saga pattern | 05-Messaging/04-messaging-patterns.md |
| Secrets management | 09-Security/06-secrets-and-key-management.md |
| Service discovery | 02-Building-Blocks/07-service-discovery.md |
| Service mesh | 07-Microservices/03-inter-service-communication.md |
| SLA / SLO / SLI | 08-Observability/01-observability-fundamentals.md |
| SQL injection | 09-Security/05-application-security.md |
| Strangler Fig | 07-Microservices/04-microservices-patterns.md |
| TLS | 09-Security/04-transport-security.md |
| Token bucket | 02-Building-Blocks/05-rate-limiting.md |
| Trade-off language | 11-Interview/04-trade-off-vocabulary.md |
| Transactions & isolation | 03-Databases/07-transactions-isolation.md |
| Two Generals Problem | 06-Distributed-Systems/01-distributed-systems-fundamentals.md |
| Vector clocks | 06-Distributed-Systems/03-consistency-and-clocks.md |
| Write-through / Write-behind | 04-Caching/02-caching-strategies.md |
| XSS / CSRF | 09-Security/05-application-security.md |
| Zero trust | 09-Security/01-security-fundamentals.md |

---

## Contributing

This handbook is a living document. If you find an error, a gap, or a concept that could be explained more clearly:

1. Open an issue describing the problem
2. Reference the specific file and section
3. If contributing a fix, keep the same format: language-agnostic, diagram-first, 8–10 minute read target

---

## License

MIT License — see `LICENSE` for details. Use freely for personal study and non-commercial purposes.

---

## Acknowledgements

Content draws on publicly available engineering blogs, research papers, and documentation from:
Netflix Tech Blog · Uber Engineering Blog · LinkedIn Engineering Blog · Discord Engineering Blog · Google SRE Book · Martin Fowler's Architecture Guides · Apache Kafka Documentation · Amazon AWS Documentation · Designing Data-Intensive Applications (Kleppmann)

Where company-specific implementation details are described, the source is noted. Details described as "inferred" are reasonable extrapolations from public information and are not confirmed internal documentation.

---

*Built for engineers who want to understand systems deeply, not just pass interviews.*