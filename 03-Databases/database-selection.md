# Database Selection & Trade-offs

> **Databases #9** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Why Selection Matters

Database selection is one of the most consequential and hardest-to-reverse decisions in system design. The wrong choice leads to:

- Rewrites at scale (Discord migrated MongoDB → Cassandra; painful and expensive)
- Performance ceilings you can't engineer around
- Consistency bugs that corrupt data
- Operational complexity that slows down every future change

The right choice comes from matching **access patterns and NFRs** to database strengths — not from defaulting to what the team knows or what's currently popular.

> **Rule: the database should fit the query, not the other way around.**

---

## 2. The Decision Framework — Ask These Questions First

Before selecting a database, answer these questions. The answers eliminate most candidates immediately.

```
Q1: What is the read/write ratio?
    → 99% reads: optimise for reads (indexes, replicas, caching)
    → 50% writes: optimise for writes (LSM storage, fewer indexes, NoSQL)

Q2: What consistency level is required?
    → Financial / inventory: strong consistency → ACID → relational or CP NoSQL
    → Feed / analytics: eventual consistency → AP NoSQL

Q3: Is the schema fixed or flexible?
    → Fixed, well-understood: relational
    → Evolving, nested, variable per record: document

Q4: What are the access patterns? (the most important question)
    → Complex queries, joins: relational
    → Lookup by a single key: key-value
    → Time-ordered data per entity: wide-column
    → Relationship traversal: graph
    → Full-text search: search engine

Q5: What are the scale requirements?
    → Moderate scale (<10M rows, <10K TPS): relational scales fine
    → Massive write throughput (>100K TPS): NoSQL
    → Global distribution, multi-region: NoSQL built for this

Q6: Is ACID across multiple entities required?
    → Yes: relational (or NewSQL)
    → No: NoSQL simplifies significantly
```

---

## 3. Database Types — Quick Selection Reference

| Database Type | Choose When | Avoid When | Examples |
|---|---|---|---|
| **Relational (RDBMS)** | Structured data, complex queries, ACID transactions, relationships between entities | Write throughput > 100K TPS, schema evolving constantly, petabyte-scale | PostgreSQL, MySQL, Oracle |
| **Key-Value** | Fastest possible lookup by a single key, caching, sessions, counters | Complex queries, relationships, filtering by anything other than key | Redis, DynamoDB, Memcached |
| **Document** | Schema-flexible entities, nested data, varying structure per record | Complex multi-entity joins, strict ACID, large-scale time-series | MongoDB, Firestore |
| **Wide-Column** | Very high write throughput, time-ordered data per entity, massive datasets | Complex queries, strong consistency, multi-row transactions | Cassandra, HBase, Bigtable |
| **Graph** | Relationship traversal, social graphs, fraud detection, recommendation | High write throughput, large flat datasets | Neo4j, Neptune |
| **Time-Series** | Metrics, monitoring, IoT, events with timestamps | General-purpose storage, complex joins | InfluxDB, Prometheus, TimescaleDB |
| **Search Engine** | Full-text search, relevance ranking, faceted filtering | Primary data store, ACID transactions | Elasticsearch, OpenSearch |
| **Object Store** | Binary files, images, video, backups, large documents | Low-latency random access, structured queries | S3, GCS, Azure Blob |

---

## 4. The Access Pattern → Database Mapping

The most reliable way to choose: map your actual queries to their optimal storage model.

| Access Pattern | Example Query | Optimal Database |
|---|---|---|
| Lookup by unique ID | `GET user by user_id` | Key-Value or Relational (PK lookup) |
| Complex filter + join | `Orders WHERE status='paid' JOIN users` | Relational |
| Flexible nested schema | User profile with variable attributes | Document |
| Time-ordered events per entity | "Last 50 messages in channel X" | Wide-Column (partition=channel, cluster=time) |
| Relationship traversal | "Friends of friends within 3 hops" | Graph |
| Full-text search | "Find articles about distributed systems" | Search Engine |
| Aggregation over time | "CPU usage for host X over last 24h" | Time-Series |
| Binary file serving | Profile photos, video content | Object Store |

---

## 5. Read Your NFRs Into the Decision

Access patterns tell you *what* the database must do. NFRs tell you *how well* it must do it.

| NFR | Implication for Database Choice |
|---|---|
| **Low latency (P99 < 10ms)** | In-memory key-value (Redis) or wide-column with good partition key |
| **High write throughput (>100K TPS)** | Wide-column (Cassandra) or key-value; relational struggles here |
| **Strong consistency** | Relational (ACID) or CP NoSQL (etcd, Zookeeper); avoid AP NoSQL |
| **High availability (99.99%+)** | Multi-AZ replication; AP NoSQL (Cassandra) or replicated relational |
| **Low RPO (< 1 min data loss)** | Synchronous replication; multi-AZ |
| **Flexible schema** | Document DB; avoid relational if schema changes weekly |
| **Global distribution** | Cassandra (multi-region native), DynamoDB Global Tables, CockroachDB |
| **GDPR data residency** | Sharding or DB config that restricts EU data to EU nodes |
| **Cost sensitivity** | Open-source PostgreSQL/MySQL over managed/proprietary; object store for cold data |

---

## 6. Polyglot Persistence — Using Multiple Databases

Real systems almost always use more than one database. Each service picks the best database for its specific needs. This is **polyglot persistence**.

```
E-commerce platform:

User Service       → PostgreSQL (profiles, addresses — relational, ACID)
Product Catalogue  → MongoDB (flexible schema — category A has 5 attrs; B has 50)
Search             → Elasticsearch (full-text, faceted filtering)
Sessions/Cache     → Redis (key-value, sub-millisecond)
Order Service      → PostgreSQL (ACID transactions — must not lose orders)
Analytics          → Cassandra or BigQuery (high write throughput / column scan)
Product Images     → S3 (object store — never put files in a DB)
Recommendations    → Neo4j or DynamoDB (graph traversal / flexible)
```

> **This is not over-engineering.** Each database in the list above is the right tool for that specific job. Using PostgreSQL for everything would mean: slow full-text search, painful flexible schemas, graph queries as exponential JOINs, and serving images from database rows. The polyglot approach is the correct one.

---

## 7. Common System Design Interview Scenarios

### "Design a URL shortener"

| Component | Database | Why |
|---|---|---|
| URL mappings | Key-Value (Redis) + persistent store (PostgreSQL) | Redis for O(1) redirect; Postgres for durability |
| Analytics/click events | Cassandra | High write throughput, time-ordered per short code |

### "Design a chat system (WhatsApp)"

| Component | Database | Why |
|---|---|---|
| Messages | Cassandra | Partition=conversation, cluster=timestamp; massive write throughput |
| User profiles | PostgreSQL | Relational, ACID, not high-write |
| Online status | Redis | Fast in-memory key-value; TTL for auto-expiry |

### "Design Twitter/social feed"

| Component | Database | Why |
|---|---|---|
| Tweets | Cassandra | Write-heavy; time-ordered per user |
| User profiles/follows | PostgreSQL | Relational; graph of follows small enough for relational |
| Search | Elasticsearch | Full-text search over tweets |
| Timeline cache | Redis | Pre-computed feeds cached; fast reads |

### "Design a ride-sharing system (Uber)"

| Component | Database | Why |
|---|---|---|
| Driver locations | Redis (geo index) | Sub-millisecond proximity queries; location changes rapidly |
| Trip records | PostgreSQL | ACID transactions; financial data |
| Driver/rider profiles | PostgreSQL | Structured, relational |
| Analytics / history | Cassandra | High volume; time-ordered |

---

## 8. When to Reconsider Your Choice

A database choice that was right at launch may need revisiting. Warning signs:

| Signal | Likely Problem | Possible Fix |
|---|---|---|
| Query timeouts at scale | Missing indexes or wrong database for access pattern | Indexing strategy review; consider different DB |
| Write latency climbing | Single primary at capacity | Sharding; switch to write-optimised NoSQL |
| Schema migrations taking hours | Relational table too large; frequent schema change | Consider document DB for that service |
| JOIN queries becoming complex | Highly connected data in relational | Graph DB for relationship-heavy queries |
| Storage costs exploding | Keeping warm data in relational; files in DB | Cold data to object store; archive strategy |

---

## 9. Best Practices

- **Start with relational for new systems** — strong tooling, ACID, well-understood. Switch when you have a proven specific need.
- **Identify access patterns before choosing** — run queries on paper against each candidate.
- **Read your NFRs into the decision** — consistency, latency, throughput constraints eliminate candidates.
- **Use polyglot persistence** — match each service's database to its access pattern.
- **Never store files in a database** — use an object store; store the URL reference.
- **Plan for growth** — choose a database that can scale to your 18-month projection, not just today.
- **Benchmark with realistic data** — a database that performs well with 1,000 rows may behave differently with 100 million.

---

## 10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Choosing database by popularity | Wrong tool for the access pattern | Match database to queries, not trends |
| One database for all services | Each service makes wrong trade-offs | Polyglot persistence |
| Premature NoSQL adoption | Complexity without benefit at small scale | Start relational; migrate when proven needed |
| Storing images/video in DB | Massive row sizes; slow backup; expensive | Object store (S3) for all binary files |
| Ignoring consistency requirements | Data corruption in financial or inventory systems | Match consistency model to correctness requirement |
| Not prototyping queries | Assumptions about performance prove wrong | Benchmark with realistic query + data volume |

---

## 11. Interview Questions

1. How do you choose between a relational and a NoSQL database?
2. What questions do you ask before selecting a database?
3. A system needs to store 100 billion time-ordered events and query the last 50 by entity ID. Which database and why?
4. What is polyglot persistence? Give an example.
5. You're designing a social network. Which databases would you use for posts, friendships, search, and sessions?
6. When would you choose Cassandra over PostgreSQL for storing messages?
7. Why should files never be stored in a relational database?

---

## 12. Summary

| Question | Directs You To |
|---|---|
| ACID transactions required? | Relational or CP NoSQL |
| Single-key lookup at any scale? | Key-Value |
| Flexible, nested, evolving schema? | Document |
| Very high write throughput + time ordering? | Wide-Column |
| Deep relationship traversal? | Graph |
| Full-text search? | Search Engine |
| Metrics / timestamps? | Time-Series |
| Binary files? | Object Store (never a DB) |

| Principle | Key Takeaway |
|---|---|
| **Start relational** | Default until you have a specific reason not to |
| **Match access patterns** | The query defines the database, not the other way |
| **Use polyglot persistence** | Different services, different databases |
| **NFRs eliminate candidates** | Consistency, latency, throughput constraints narrow the field |

---

## 13. Cross References

**Prerequisites:** All previous Database documents (DB #1–#8) · NFR series

**Related Topics:** Scalability (NFR #3) · Consistency (NFR #5) · System Design Fundamentals

**What to Learn Next:** `04-Caching` section · Messaging Systems · Distributed Systems

---

*System Design Engineering Handbook — Databases Series*

---

> **Databases series complete.**
> Covered: Fundamentals · SQL/Relational · NoSQL · Indexing · Replication · Partitioning & Sharding · Transactions & Isolation · Caching Overview · Database Selection