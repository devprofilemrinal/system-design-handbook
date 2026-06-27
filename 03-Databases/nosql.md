# NoSQL Databases

> **Databases #3** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is NoSQL?

NoSQL ("Not Only SQL") is an umbrella term for databases that don't use the traditional relational table model. They emerged to solve problems that relational databases struggle with at scale: extremely high write throughput, flexible schemas, massive datasets distributed across many nodes, and access patterns that don't fit neatly into tables and joins.

> **Common misconception:** NoSQL doesn't mean "no SQL syntax" or "no structure." It means "a different data model." Many NoSQL databases have their own query languages and impose structure — just not the relational kind.

---

## 2. Why NoSQL Exists — The Problems It Solves

Relational databases were designed in an era of single, powerful servers. The internet changed the scale of data and traffic:

| Challenge | Relational DB Problem | NoSQL Solution |
|---|---|---|
| Schema changes every sprint | ALTER TABLE locks tables; painful migrations | Schema-flexible document model |
| 1M writes/second | Single primary write node saturates | Distributed write architecture |
| Petabytes of data | Single server can't hold it | Horizontal sharding built in |
| Global users, low latency | Centralised DB → cross-region latency | Multi-region replication by design |
| Access by single key (cache-like) | Overkill to use a full SQL engine | Key-value: O(1) hash lookup |

---

## 3. The Four Types of NoSQL Databases

### 3.1 Key-Value Stores

The simplest database model. Every piece of data is stored as a value associated with a unique key. Think of it as a giant hash map.

```
SET user:1001:session = "eyJhbGciOi..."   (store)
GET user:1001:session                      (retrieve) → O(1)
DEL user:1001:session                      (delete)
```

**Characteristics:**
- Fastest possible read/write — O(1) hash lookup
- No query capability beyond "get by key"
- Values are opaque — the DB doesn't know or care what's inside

**Best for:** Session storage, caching, feature flags, rate limiting counters, leaderboards.

**Not suitable for:** Complex queries, relationships, filtering by non-key attributes.

**Examples:** Redis, DynamoDB (simple mode), Memcached

> **Note:** Redis is often used as a cache in front of other databases. Full caching patterns are covered in the `04-Caching` section.

---

### 3.2 Document Databases

Stores data as **documents** — typically JSON or BSON (binary JSON). Each document is self-contained and can have a different structure. No fixed schema required.

```json
// Document 1
{
  "id": "user_001",
  "name": "Alice",
  "email": "alice@mail.com",
  "address": {
    "street": "123 Main St",
    "city": "London"
  },
  "tags": ["premium", "newsletter"]
}

// Document 2 — different structure, same collection
{
  "id": "user_002",
  "name": "Bob",
  "email": "bob@mail.com"
  // no address, no tags — that's fine in a document DB
}
```

**Characteristics:**
- Schema-flexible — each document can differ
- Nested data (arrays, sub-objects) stored together
- Query by any field, including nested fields
- No joins — related data is typically embedded or referenced by ID

**Best for:** User profiles, product catalogues, content management, any entity with variable attributes.

**Not suitable for:** Complex multi-entity transactions, highly relational data, data that changes structure frequently across documents you need to query uniformly.

**Examples:** MongoDB, Firestore, Couchbase, DynamoDB (document mode)

---

### 3.3 Wide-Column Stores

Data is stored in tables with rows and columns — but unlike relational, each row can have a completely different set of columns, and columns are grouped into **column families**.

The key insight: data is stored and retrieved **column family by column family**, not row by row. This is optimised for reading and writing specific column groups at very high speed.

```
Row key: "user_001"
  Column family: profile  → {name: "Alice", email: "alice@mail.com"}
  Column family: activity → {last_login: "2024-01-15", login_count: 142}

Row key: "user_002"
  Column family: profile  → {name: "Bob", phone: "555-1234"}  ← different columns, that's fine
  Column family: activity → {last_login: "2024-01-10"}
```

**The partition key and clustering key are critical:**
- **Partition key:** Determines which node holds this data. All rows with the same partition key are stored together.
- **Clustering key:** Determines the sort order of rows within a partition.

This schema is designed **around your queries**, not around your entities.

```
Use case: "Get all messages in channel #general, newest first"
Partition key: channel_id  → all messages for a channel on one node
Clustering key: timestamp DESC → sorted newest first
→ One read, one node, extremely fast
```

**Best for:** Time-series data, event logs, messaging, IoT sensor data, anything with very high write throughput.

**Not suitable for:** Complex queries across partition keys, multi-row transactions, data with many relationships.

**Examples:** Cassandra, HBase, Google Bigtable, ScyllaDB

---

### 3.4 Graph Databases

Stores data as **nodes** (entities) and **edges** (relationships). Relationships are first-class citizens — stored and traversed directly, not derived through JOINs.

```
(Alice) --[FOLLOWS]--> (Bob)
(Alice) --[FOLLOWS]--> (Carol)
(Bob)   --[FOLLOWS]--> (Dave)

Query: "Who does Alice follow who also follows Dave?"
→ Graph traversal: Alice → Bob → Dave ✅ (Bob qualifies)
→ In a relational DB, this requires expensive multi-level self-joins
```

**Characteristics:**
- Relationship traversal is O(1) per hop — follow an edge directly
- Relational multi-level JOIN is O(n^k) for k levels of relationship
- Schema is flexible — nodes and edges can have any properties
- Not suited for bulk data operations

**Best for:** Social networks, recommendation engines, fraud detection (follow money through entities), knowledge graphs, access control (role hierarchies).

**Not suitable for:** High write throughput, large flat datasets, use cases without meaningful relationships.

**Examples:** Neo4j, Amazon Neptune, ArangoDB

---

## 4. Side-by-Side Comparison

| | Key-Value | Document | Wide-Column | Graph |
|---|---|---|---|---|
| **Data model** | key → value | JSON documents | Rows with flexible columns | Nodes + edges |
| **Query** | By key only | By any field | By partition + clustering key | Relationship traversal |
| **Schema** | None | Flexible | Semi-structured | Flexible |
| **Relationships** | None | Reference by ID | None | First-class |
| **Write throughput** | Highest | High | Very high | Medium |
| **Strong consistency** | Sometimes | Sometimes | Tunable | Usually |
| **Best for** | Cache, session | Profile, catalogue | Events, time-series | Social, fraud, graph |

---

## 5. Consistency in NoSQL — What You Trade

Most NoSQL databases are AP systems under the CAP theorem — they choose availability and partition tolerance over strict consistency. This means:

- Writes are acknowledged before all replicas confirm
- Reads may return stale data briefly
- All replicas eventually converge (**eventual consistency**)

Some NoSQL databases (Cassandra, DynamoDB) offer **tunable consistency** — you can choose per-query:

```
Write quorum: QUORUM  → wait for majority of replicas to confirm
Read quorum:  QUORUM  → read from majority → guaranteed to see latest write

Write quorum: ONE     → fastest; just one node confirms
Read quorum:  ONE     → fastest; may return stale data
```

> **The right setting depends on the use case.** View counts → eventual is fine. Inventory levels → strong consistency required.

---

## 6. NoSQL Data Modelling — Think in Queries

The most important mindset shift when moving from relational to NoSQL:

```
Relational: design around entities and relationships → queries follow
NoSQL:      design around your queries → data model follows
```

In Cassandra, your table schema is literally defined by the query you need to answer. If you need to query messages by channel AND by user, you create **two separate tables** — one optimised for each query.

```
Table 1: messages_by_channel
  Partition key: channel_id
  → query: "get messages in channel X"

Table 2: messages_by_user
  Partition key: user_id
  → query: "get messages sent by user Y"

Same underlying data, two different physical tables.
Duplication is intentional and accepted.
```

This is the opposite of normalisation — deliberately denormalise to match access patterns.

---

## 7. How Large Companies Use NoSQL

| Company | Database | Use Case | Source |
|---|---|---|---|
| **Discord** | Cassandra | Message storage — billions of messages, time-ordered per channel | Discord Eng Blog (public) |
| **Netflix** | Cassandra | User viewing history, recommendations data | Netflix Tech Blog (public) |
| **MongoDB Atlas** | MongoDB | Widely used for product catalogues, user profiles across industries | Public |
| **LinkedIn** | Various | Graph traversal for connections; document for profiles | LinkedIn Eng Blog (public) |
| **Twitter/X** | Redis | Timeline caching, rate limiting counters | Public talks |

---

## 8. Best Practices

- **Choose NoSQL for a specific, proven reason** — not because it's "more scalable" (relational scales very far with proper design).
- **Design your NoSQL schema around your queries** — know your access patterns before writing a single table definition.
- **Accept and plan for eventual consistency** — build your application to handle briefly stale data.
- **Don't try to do joins** — if you need joins, you're using the wrong tool. Either embed the data or use a relational DB.
- **Be deliberate about duplication** — in wide-column and document DBs, controlled duplication is often the right choice.

---

## 9. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Choosing NoSQL to "be scalable" without a reason | All the complexity; none of the benefit | Start relational; move to NoSQL for proven specific need |
| Trying to do relational joins in NoSQL | Impossible or extremely slow | Embed data or change the data model |
| Not thinking about partition keys | Hot partition — one node gets all the traffic | Choose high-cardinality partition keys that distribute evenly |
| Expecting strong consistency everywhere | Stale reads cause bugs | Understand and handle eventual consistency in application code |
| Ignoring the schema entirely (document DB) | Query performance degrades; inconsistent data across documents | Define field conventions; use indexes on queried fields |

---

## 10. Interview Questions

1. What is NoSQL and why was it created?
2. Explain the four types of NoSQL databases and give a use case for each.
3. Why does Discord use Cassandra for message storage? What makes it the right fit?
4. What does "design around queries" mean in NoSQL vs "design around entities" in relational?
5. What is a partition key in Cassandra and why does choosing the right one matter?
6. When would you use a graph database instead of adding a JOIN to a relational query?
7. What consistency trade-offs do most NoSQL databases make and why?

---

## 11. Summary

| Type | Model | Best For |
|---|---|---|
| **Key-Value** | key → value | Session, cache, counters |
| **Document** | JSON documents | Profiles, catalogues, variable-schema entities |
| **Wide-Column** | Rows + column families | Events, time-series, high-write throughput |
| **Graph** | Nodes + edges | Social graphs, fraud, recommendations |

| Concept | Key Takeaway |
|---|---|
| **NoSQL exists because** | Relational struggles with extreme scale, flexible schema, and certain access patterns |
| **CAP trade-off** | Most NoSQL is AP — eventually consistent |
| **Design principle** | Design around queries, not entities |
| **When to use** | Specific proven need, not default |

---

## 12. Cross References

**Prerequisites:** Database Fundamentals (DB #1) · SQL & Relational (DB #2)

**Related Topics:** Consistency (NFR #5) · Scalability (NFR #3) · Partitioning & Sharding (DB #6)

**What to Learn Next:** Indexing (DB #4) · Replication (DB #5)

---

*System Design Engineering Handbook — Databases Series*