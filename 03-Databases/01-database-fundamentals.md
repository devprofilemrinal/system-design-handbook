# Database Fundamentals

> **Databases #1** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is a Database?

A database is an organised system for storing, retrieving, and managing data persistently. "Persistently" is the key word — data survives process restarts, crashes, and power failures, unlike data held in application memory.

Every production system needs a database because:
- Application memory is temporary — the moment the process dies, data is gone
- Multiple servers need to share the same data
- Data must survive failures
- Queries across large datasets need to be efficient

---

## 2. The Database Landscape

There is no single "best" database. The landscape has evolved because different problems require fundamentally different storage and retrieval strategies.

```
Databases
├── Relational (SQL)
│     Structured data, tables, relationships, ACID transactions
│     Examples: PostgreSQL, MySQL, Oracle, SQL Server
│
├── Key-Value
│     Fastest lookups by key; simplest model
│     Examples: Redis, DynamoDB (simple mode), Memcached
│
├── Document
│     JSON-like documents; flexible schema
│     Examples: MongoDB, Firestore, Couchbase
│
├── Wide-Column
│     Rows with flexible columns; massive write throughput
│     Examples: Cassandra, HBase, Bigtable
│
├── Graph
│     Nodes and edges; relationship-heavy queries
│     Examples: Neo4j, Amazon Neptune
│
├── Time-Series
│     Optimised for timestamped data streams
│     Examples: InfluxDB, Prometheus, TimescaleDB
│
└── Search Engine
      Full-text search and relevance ranking
      Examples: Elasticsearch, OpenSearch
```

> **The most important skill is choosing the right database for the access pattern — not defaulting to what you know best.**

---

## 3. ACID — The Gold Standard for Correctness

ACID is a set of four properties that guarantee a database transaction is processed reliably. Relational databases are built around these guarantees.

| Property | Meaning | Example |
|---|---|---|
| **Atomicity** | A transaction is all-or-nothing. Either every step completes, or none do. | Transfer $100: debit A AND credit B. If credit fails, debit is rolled back. |
| **Consistency** | A transaction moves the database from one valid state to another. All constraints remain satisfied. | A bank balance cannot go negative if a constraint forbids it. |
| **Isolation** | Concurrent transactions don't interfere with each other. Each sees a consistent snapshot. | Two users booking the last seat don't both succeed. |
| **Durability** | Once committed, data survives crashes, power loss, anything. | After "Payment confirmed," the record is safe even if the server dies. |

> **Atomicity is the most interview-critical property.** It is what prevents half-applied operations — the scenario where money leaves one account but never arrives in another.

---

## 4. BASE — The Alternative Philosophy

NoSQL databases often relax ACID guarantees in exchange for higher availability and scalability. They follow BASE instead.

| Property | Meaning |
|---|---|
| **Basically Available** | The system responds to every request, even if some data is stale or unavailable |
| **Soft state** | The system's state may change over time even without new input (due to replication catching up) |
| **Eventually consistent** | All replicas will converge to the same value — eventually |

```
ACID: "I guarantee correctness right now."
BASE: "I guarantee I'll respond now. Correctness will catch up soon."
```

Neither is universally better. ACID is non-negotiable for financial data. BASE is the right choice for social media feeds, view counters, shopping carts.

---

## 5. How Databases Store Data — The Basics

Understanding *how* databases store data explains *why* certain databases are fast for certain operations.

### Row-Oriented Storage (most relational DBs)

Stores all columns of a row together on disk. Great for reading or writing complete records.

```
Row 1: [id=1, name="Alice", age=30, city="London"]
Row 2: [id=2, name="Bob",   age=25, city="Paris" ]
Row 3: [id=3, name="Carol", age=35, city="London"]

Query: SELECT * FROM users WHERE id = 2
→ Read one row from disk. Fast.

Query: SELECT AVG(age) FROM users
→ Must read every row to get the age column. Slower for analytics.
```

### Column-Oriented Storage (data warehouses, analytics)

Stores all values of a column together. Great for aggregations across many rows.

```
id column:   [1, 2, 3, ...]
name column: ["Alice", "Bob", "Carol", ...]
age column:  [30, 25, 35, ...]

Query: SELECT AVG(age) FROM users
→ Read only the age column from disk. Very fast.
→ Highly compressible (similar values together).
```

### Log-Structured Storage (LSM Trees — used by Cassandra, RocksDB)

All writes go to an in-memory buffer first, then are flushed to disk in sorted batches. Optimised for extremely high write throughput.

```
Write → memory buffer (fast) → periodic flush to disk (sorted file)
Read  → check memory + merge multiple sorted files
```

> **Knowing which storage engine a database uses tells you its strengths and weaknesses before you test it.**

---

## 6. Indexes — How Databases Find Data Fast

Without an index, every query requires reading every row in the table — called a **full table scan**. For a table with 100 million rows, this is catastrophically slow.

An index is a separate data structure that allows the database to jump directly to the rows matching a query condition.

```
WITHOUT index:
Query: SELECT * FROM orders WHERE customer_id = 12345
→ Read all 100 million rows, check each one. Slow.

WITH index on customer_id:
→ Jump directly to matching rows. Fast.
```

The most common index structure is the **B-Tree** — a balanced tree that keeps data sorted and supports range queries efficiently. Full coverage in the Indexing document.

> **Missing indexes are the single most common cause of slow database queries in production.** Always think "what queries run against this table, and are they indexed?"

---

## 7. The Read/Write Trade-off

Every database design decision involves a fundamental trade-off between read performance and write performance.

| Optimised For | Technique | Cost |
|---|---|---|
| **Faster reads** | Add indexes | Writes become slower (must update each index) |
| **Faster reads** | Denormalise data | More storage; risk of inconsistency |
| **Faster reads** | Add read replicas | Replication lag; stale reads possible |
| **Faster writes** | Remove indexes | Reads become slower |
| **Faster writes** | Write to memory first (LSM) | More complex reads (merge multiple files) |
| **Faster writes** | Async replication | Risk of data loss on crash |

> **Before designing a schema or choosing a database, always establish the read/write ratio.** A system that is 95% reads (product catalogue) is designed very differently from one that is 95% writes (IoT sensor data).

---

## 8. Choosing the Right Database

This decision is one of the most impactful you make in system design. The wrong choice is expensive to reverse.

| If you need... | Use... | Because... |
|---|---|---|
| ACID transactions, complex queries, relationships | Relational (PostgreSQL, MySQL) | Built for exactly this |
| Fastest possible lookup by a single key | Key-Value (Redis, DynamoDB) | O(1) hash lookup |
| Flexible schema, nested documents | Document (MongoDB) | Schema-free JSON storage |
| Massive write throughput, time-ordered data | Wide-Column (Cassandra) | Append-optimised LSM storage |
| Relationship traversal (friends of friends) | Graph (Neo4j) | Adjacency list native; JOINs in relational are O(n²) |
| Full-text search, relevance ranking | Search (Elasticsearch) | Inverted index built for text |
| Timestamped metrics, sensor streams | Time-Series (InfluxDB) | Optimised compression and downsampling |
| Files, images, videos | Object Store (S3) | Not a database — but often confused with one |

> **Polyglot persistence:** Large systems use multiple databases — each chosen for a specific use case. User profiles → Postgres. Session tokens → Redis. Search → Elasticsearch. This is normal and correct, not over-engineering.

---

## 9. A Note on Caching

Caching is the practice of storing frequently-accessed data in a fast, in-memory store (like Redis or Memcached) so the database doesn't need to be queried every time.

```
Without cache: Every request → Database (10–100ms)
With cache:    Hot requests  → Cache (< 1ms) → database only on miss
```

Caching sits in front of your database and absorbs the majority of read traffic. It is one of the highest-leverage performance improvements in any system.

> **Caching is a large topic with its own patterns, trade-offs, and failure modes — it has a dedicated section in this handbook (`04-Caching`).** The key point here: before reaching for database scaling (replicas, sharding), consider whether caching can absorb the load first. It is almost always cheaper and simpler.

---

## 10. Scaling Databases

When a single database can no longer handle the load, the options are:

| Strategy | What It Does | When |
|---|---|---|
| **Read Replicas** | Copy data to read-only nodes; distribute reads | Read-heavy workload (>80% reads) |
| **Caching** | Absorb reads before they hit the DB | Hot data accessed repeatedly |
| **Vertical Scaling** | Bigger machine | Quick fix; has a ceiling |
| **Sharding** | Split data across multiple database instances | Write throughput or data size exceeds single node |
| **CQRS** | Separate read and write data models entirely | Complex domains with very different read/write patterns |

> Scaling strategy is covered in depth in the Replication and Partitioning/Sharding documents.

---

## 11. Best Practices

- **Choose based on access patterns**, not familiarity — the right database for the query pattern, not the most popular one.
- **Default to relational** for new systems — battle-tested, rich tooling, ACID guarantees. Move to specialist DBs only when you have a specific, proven need.
- **Establish read/write ratio early** — it dictates schema design, indexing strategy, and scaling approach.
- **Index what you query** — every WHERE, JOIN, and ORDER BY should have an index.
- **Use caching before scaling the DB** — it is almost always cheaper.
- **Never store files in a database** — use an object store (S3) and store only the reference URL.

---

## 12. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Using one database for everything | Wrong trade-offs for some use cases | Polyglot persistence — match DB to access pattern |
| Missing indexes on queried columns | Full table scans; catastrophic slowdown at scale | Index every column used in WHERE/JOIN/ORDER BY |
| Storing binary files in DB rows | Massive bloat; slow backups; poor performance | Object store for files; DB stores the URL |
| Not establishing read/write ratio | Optimised for the wrong direction | Profile first; design for the dominant pattern |
| Premature sharding | Enormous complexity before it's needed | Exhaust vertical scaling + replicas + caching first |

---

## 13. Interview Questions

1. What does ACID stand for and why does it matter?
2. What is the difference between ACID and BASE? Give an example of when you'd choose each.
3. What is a full table scan and why is it a problem?
4. What is the difference between row-oriented and column-oriented storage?
5. A system has 1 billion rows and reads are slow. Walk through your options in order.
6. When would you choose a document database over a relational one?
7. What is polyglot persistence?

---

## 14. Summary

| Concept | Key Takeaway |
|---|---|
| **Databases** | Persistent, organised storage. Survives crashes. |
| **ACID** | Atomicity, Consistency, Isolation, Durability. Correctness first. |
| **BASE** | Basically Available, Soft state, Eventually consistent. Availability first. |
| **Storage engines** | Row (OLTP), Column (analytics), LSM (high writes). |
| **Indexes** | Jump to data instead of scanning. Essential. |
| **Read/write ratio** | The first question in any DB design. |
| **Caching** | Absorb reads before they hit the DB. Covered in `04-Caching`. |
| **Scaling** | Replicas → Caching → Sharding. In that order. |

---

## 15. Cross References

**Prerequisites:** System Design Fundamentals (FR, HLD, LLD)

**Related Topics:** Scalability (NFR #3) · Reliability (NFR #4) · Consistency (NFR #5)

**What to Learn Next:** SQL & Relational Databases (DB #2) · NoSQL Deep Dive (DB #3)

---

*System Design Engineering Handbook — Databases Series*