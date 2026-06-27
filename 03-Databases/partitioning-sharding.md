# Partitioning & Sharding

> **Databases #6** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Partitioning?

Partitioning is the practice of splitting a large dataset into smaller, more manageable pieces called **partitions**. Each partition holds a subset of the data, and the system knows exactly which partition to look in for any given piece of data.

There are two dimensions to partition along:
- **Horizontal partitioning (sharding):** Split by *rows* — different rows go to different partitions/nodes.
- **Vertical partitioning:** Split by *columns* — different columns go to different tables/stores.

> **Sharding and horizontal partitioning mean the same thing.** Sharding is simply the term used when those horizontal partitions live on *separate database nodes* — each partition is an independent database instance.

---

## 2. Why Partition?

A single database node has hard limits: a ceiling on storage, RAM, and write throughput. When your data grows beyond what one node can handle, you must split it.

```
Single node limits (approximate):
  Storage: ~tens of TB on SSD
  Write throughput: ~tens of thousands of writes/second
  RAM for hot data: ~hundreds of GB

When you exceed any of these → you partition
```

Replication (DB #5) helps with reads and availability, but it doesn't help with writes or storage — every replica holds the full dataset. **Sharding is the solution for write throughput and storage that exceeds a single node.**

---

## 3. Horizontal Partitioning (Sharding)

Each shard is a separate database instance holding a subset of rows.

```
Full dataset: 1 billion users

Shard 1: users 0       → 249,999,999   (250M rows)
Shard 2: users 250,000,000 → 499,999,999
Shard 3: users 500,000,000 → 749,999,999
Shard 4: users 750,000,000 → 999,999,999

Write for user 600,000,000 → goes to Shard 3 only
Read for user 100,000,000  → goes to Shard 1 only
```

Each shard operates as an independent database — it can have its own replicas, its own backups, and its own failover.

---

## 4. Sharding Strategies — How to Decide Which Shard

The **shard key** (also called partition key) is the column used to determine which shard a row belongs to. Choosing it correctly is the most important and hardest-to-reverse decision in sharding.

### Range-Based Sharding

Divide the key space into ranges. Each shard owns a range.

```
user_id 0–999         → Shard A
user_id 1000–1999     → Shard B
user_id 2000+         → Shard C
```

**Advantage:** Simple. Range queries are efficient (all January orders on one shard).
**Problem:** Hotspots. If most writes go to new user IDs and IDs are sequential, Shard C gets all the writes. The others sit idle.

### Hash-Based Sharding

Apply a hash function to the key. The hash result determines the shard.

```
shard = hash(user_id) % number_of_shards

user_id 1001 → hash → 2 → Shard 2
user_id 1002 → hash → 0 → Shard 0
user_id 1003 → hash → 2 → Shard 2
```

**Advantage:** Even distribution — hash functions scatter data uniformly.
**Problem:** Range queries are now impossible (sequential IDs are not on sequential shards). Resharding is painful — changing the number of shards changes the hash result for almost every key.

### Consistent Hashing

A smarter version of hash sharding that minimises data movement when you add or remove shards.

```
Imagine a ring of hash values (0 to 2^32):
  Shards are placed at positions around the ring.
  Each key hashes to a position; it belongs to the next shard clockwise.

Add a new shard:
  Only the keys that were "between" the new shard's position and its predecessor move.
  ~1/N of data moves instead of almost all of it.
```

**Used by:** Cassandra, DynamoDB, many distributed caches. Essential for systems where shard count changes frequently.

### Geographic / Directory-Based Sharding

Route based on an attribute like region or tenant.

```
EU users   → EU Shard  (data residency compliance)
US users   → US Shard
APAC users → APAC Shard

OR

Tenant A (enterprise) → Dedicated Shard
Tenant B–Z (SMBs)     → Shared Shard Pool
```

**Advantage:** Data residency compliance, natural isolation per tenant.
**Problem:** Uneven load if tenants or regions differ significantly in size.

---

## 5. Choosing the Right Shard Key

The shard key decision is permanent (or extremely expensive to change). Get it right upfront.

| Property | Why It Matters | Bad Example | Good Example |
|---|---|---|---|
| **High cardinality** | Low cardinality = few shards get all data | `status` (3 values) | `user_id` (billions of values) |
| **Even distribution** | Uneven = hotspot on one shard | `created_at` if inserts are bursty | `user_id` with hash |
| **Aligns with access** | Cross-shard queries are expensive | Shard by `country` if queries are per-user | Shard by `user_id` |
| **Immutable** | Changing shard key = moving row to different shard | `status` (changes frequently) | `user_id` (never changes) |

> **The hotspot problem is the most common sharding mistake.** A celebrity's account on a social network, or a super-active seller on an e-commerce platform, can concentrate all traffic on one shard. The shard key must not allow single values to dominate.

---

## 6. Vertical Partitioning

Split a table by columns — move rarely-accessed or large columns to a separate table or store.

```
Original users table (one row per user):
  id, name, email, password_hash, bio (text), avatar_blob (binary, 500KB)

Problem: Every query loads the avatar blob even if you only want the name.

Vertical partition:
  users_core:   id, name, email, password_hash  (small, hot)
  users_profile: user_id, bio, avatar_url        (larger, cold)
  Avatar binary: stored in object store (S3)

→ Common queries hit users_core only → much smaller rows → faster scans, better caching
```

---

## 7. The Challenges of Sharding

Sharding solves the capacity problem but introduces significant complexity:

| Challenge | Description | Mitigation |
|---|---|---|
| **Cross-shard queries** | "Get all orders over £100" requires querying every shard and aggregating | Design shard key to co-locate data needed together |
| **Cross-shard transactions** | Updating rows on two different shards atomically requires distributed transactions | Avoid cross-shard writes; use eventual consistency |
| **Resharding** | Adding/removing shards requires moving data | Consistent hashing minimises movement |
| **Hotspots** | One shard receives disproportionate traffic | Choose shard key carefully; add shard-level caching |
| **Operational complexity** | More nodes = more to monitor, back up, fail over | Managed sharding (Vitess, managed cloud DBs) |

---

## 8. When to Shard

Sharding is a last resort, not a first choice. Exhaust simpler options first:

```
Step 1: Optimise queries and add indexes           (often solves 80% of problems)
Step 2: Vertical scaling (bigger machine)           (quick, no code changes)
Step 3: Caching (absorb reads)                      (see 04-Caching section)
Step 4: Read replicas (scale reads)                 (see Replication, DB #5)
Step 5: Vertical partitioning (split columns)       (simpler than sharding)
Step 6: Shard (horizontal partition)                (last resort)
```

> **Sharding before you need it is one of the most expensive engineering mistakes.** The complexity is constant; the benefits only appear at scale. Shard when a single node genuinely cannot keep up.

---

## 9. How Large Companies Apply This

| Company | Application | Source |
|---|---|---|
| **Shopify** | Shards MySQL by `shop_id` — each shop's data lives on one shard; all queries for a shop are local | Shopify Eng Blog (public) |
| **Instagram** | Shards Postgres by `user_id` using a custom sharding framework | Instagram Eng Blog (public) |
| **Cassandra** | Built-in horizontal partitioning; partition key is the shard key; consistent hashing used internally | Apache Cassandra docs |
| **Vitess** | Open-source sharding layer for MySQL; used by YouTube, Slack | Vitess public docs |

> **Inferred:** Specific shard counts and configurations are not public.

---

## 10. Best Practices

- **Exhaust simpler options** before sharding — indexing, caching, replicas, vertical scale.
- **Choose shard key carefully** — high cardinality, even distribution, immutable, aligned with access patterns.
- **Use consistent hashing** if shard count will change.
- **Co-locate related data** on the same shard to avoid cross-shard queries.
- **Never shard on a key that can be a hotspot** — celebrity accounts, popular products, today's date.
- **Consider a managed sharding solution** (Vitess, CockroachDB, managed cloud databases) before building your own.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Sharding too early | Massive complexity with no benefit | Profile first; shard only when single node is the proven bottleneck |
| Bad shard key (low cardinality) | Hotspot — one shard gets all traffic | Choose high-cardinality, evenly-distributed key |
| Mutable shard key | Every update moves the row to a new shard | Choose an immutable attribute as shard key |
| Ignoring cross-shard query cost | Queries that touch all shards are slow | Design shard key to co-locate frequently joined data |
| No replication within shards | Each shard is a SPOF | Every shard should have its own replicas |

---

## 12. Interview Questions

1. What is the difference between replication and sharding?
2. What is a shard key and what properties make a good one?
3. Explain range vs hash sharding. What is the main problem with each?
4. What is consistent hashing and why does it matter?
5. What are the challenges of sharding that don't exist with a single database?
6. A user posts a viral tweet and all reads/writes for that tweet hit one shard. What is this problem called and how do you solve it?
7. When would you choose to shard vs adding more read replicas?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Sharding** | Split rows across multiple DB nodes. Solves write throughput and storage limits. |
| **Shard key** | Determines which shard a row lives on. Most important, hardest-to-change decision. |
| **Range sharding** | Simple; efficient range queries. Risk: hotspots. |
| **Hash sharding** | Even distribution. Kills range queries; resharding is painful. |
| **Consistent hashing** | Minimises data movement when adding/removing shards. |
| **Hotspot** | One shard overloaded. Bad shard key. The most common sharding mistake. |
| **When to shard** | Last resort. Exhaust indexes → cache → replicas → vertical scale first. |

---

## 14. Cross References

**Prerequisites:** Database Fundamentals (DB #1) · Replication (DB #5) · Scalability (NFR #3)

**Related Topics:** Consistency (NFR #5) · NoSQL (DB #3) · Consistent Hashing

**What to Learn Next:** Transactions & Isolation (DB #7) · Database Selection (DB #9)

---

*System Design Engineering Handbook — Databases Series*