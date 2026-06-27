# Indexing

> **Databases #4** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is an Index?

An index is a separate data structure that the database maintains alongside your table. It stores a small, organised subset of your data in a way that makes specific lookups dramatically faster — at the cost of extra storage and slower writes.

The analogy is exact: a book's index at the back lists topics alphabetically with page numbers. Instead of reading every page to find "consistency," you look it up in the index and jump straight to page 47. The database does the same.

```
WITHOUT index:
Query: SELECT * FROM orders WHERE customer_id = 12345
→ Database reads every row in orders table (full table scan)
→ 100 million rows → could take seconds

WITH index on customer_id:
→ Database looks up 12345 in index → finds row locations instantly
→ Jumps directly to those rows
→ Returns in milliseconds
```

> **Missing indexes are the #1 cause of slow queries in production.** Before adding hardware or rewriting queries, check your indexes.

---

## 2. How the Most Common Index Works — B-Tree

The **B-Tree (Balanced Tree)** is the default index structure in almost every relational database. Understanding it intuitively explains most indexing behaviour.

A B-Tree keeps data sorted and balanced. Every lookup traverses from root to leaf — always the same number of steps regardless of table size.

```
B-Tree on customer_id:

              [500]
             /     \
        [250]       [750]
       /     \     /     \
   [100]  [350] [600]  [900]
    ...    ...   ...    ...
  (leaf nodes point to actual table rows)

Query: customer_id = 350
→ Start at root: 350 < 500 → go left
→ At 250: 350 > 250 → go right
→ At 350: found → pointer to table row → fetch row
→ 3 steps for millions of rows. Always O(log n).
```

**B-Tree properties:**
- Data is kept **sorted** — supports range queries efficiently (`WHERE date BETWEEN x AND y`)
- Always **balanced** — same depth for every leaf, so lookup time is consistent
- **O(log n)** for lookup, insert, and delete — extremely efficient

---

## 3. Types of Indexes

### Primary Index
Automatically created on the primary key. Every table has exactly one. In clustered primary indexes (InnoDB/MySQL), table rows are physically sorted by the primary key on disk.

### Secondary Index
Any index you create beyond the primary. Points to the primary key, which then points to the row.

```
Secondary index on email:
"alice@mail.com" → primary key 1 → fetch row with id=1
```

### Composite Index (Multi-column)
An index on two or more columns together. The order of columns matters significantly.

```
Index on (last_name, first_name)

Query: WHERE last_name = 'Smith'              → uses index ✅
Query: WHERE last_name = 'Smith' AND first_name = 'John' → uses index ✅
Query: WHERE first_name = 'John'              → does NOT use index ❌
       (index is sorted by last_name first; can't skip to first_name)
```

> **Left-prefix rule:** A composite index `(A, B, C)` can be used by queries that filter on `A`, `A+B`, or `A+B+C` — but not on `B` alone or `C` alone. Always put the most selective column first.

### Covering Index
An index that contains all the columns a query needs. The database satisfies the query entirely from the index without touching the actual table rows.

```
Index on (customer_id, order_date, total)

Query: SELECT order_date, total FROM orders WHERE customer_id = 12345
→ All needed columns are in the index → table never accessed → very fast
```

### Hash Index
Uses a hash function instead of a tree. O(1) lookup for exact matches — faster than B-Tree for equality. But cannot support range queries at all.

```
Hash index on customer_id:
customer_id = 12345 → hash → direct bucket → row pointer (O(1)) ✅
customer_id > 12000 → hash has no concept of "greater than" → full scan ❌
```

Use hash indexes only for exact-match lookups (session tokens, API keys).

### Partial Index
An index that only covers a subset of rows — those matching a condition.

```
CREATE INDEX idx_active_users ON users (email) WHERE status = 'ACTIVE';

→ Smaller index (only active users) → faster lookups → less write overhead
→ Only helps queries that also filter WHERE status = 'ACTIVE'
```

---

## 4. How Indexes Affect Writes

Every index must be updated on every INSERT, UPDATE, or DELETE. This is the fundamental cost.

```
Table with 5 indexes:
INSERT one row → update table + update 5 indexes = 6 write operations
DELETE one row → update table + update 5 indexes = 6 write operations

Table with no indexes:
INSERT one row → update table only = 1 write operation
```

> **More indexes = slower writes.** A table with 10 indexes can be 10x slower to write to than the same table with 1. Index strategically — not everything that might be queried.

---

## 5. Index Selectivity

**Selectivity** is the ratio of distinct values to total rows. High selectivity = few rows per value = index is effective. Low selectivity = many rows per value = index barely helps.

```
Table: 10 million users

Index on gender (values: M/F/Other):
  Each value matches ~3 million rows
  → Low selectivity → index barely helps → DB may ignore it

Index on email (values: unique per user):
  Each value matches exactly 1 row
  → High selectivity → index is extremely effective
```

> **Rule:** Index high-selectivity columns. Indexing a boolean column (`is_active`) on a table where 99% of rows are active is nearly useless — the DB will prefer a full scan.

---

## 6. When the Database Ignores Your Index

The query optimiser may skip your index if:

| Scenario | Why Index Is Skipped |
|---|---|
| Low selectivity | Full scan is faster than many index lookups |
| Function applied to indexed column | `WHERE UPPER(email) = 'ALICE@...'` breaks the index |
| Leading column not in query | Composite index `(A, B)` not used if only `B` is filtered |
| Very small table | Full scan of 100 rows is faster than an index lookup |
| Implicit type conversion | `WHERE customer_id = '12345'` (string vs int) |

> **Always use `EXPLAIN` or `QUERY PLAN` to verify your index is being used.** Never assume.

---

## 7. Indexing Strategy for System Design

A practical framework for deciding what to index:

```
Step 1: Identify your read queries (the most frequent ones)
Step 2: For each query, find the WHERE, JOIN, ORDER BY, GROUP BY columns
Step 3: Index those columns
Step 4: Consider composite indexes for multi-column filters
Step 5: Consider covering indexes for high-frequency, read-critical queries
Step 6: Profile write performance — if too slow, evaluate which indexes to drop
```

**Schema and index example for an orders table:**

```
Table: orders (id, customer_id, product_id, status, created_at, total)

Likely queries:
  "Get orders for customer X"         → index on customer_id
  "Get orders by status"              → index on status (low sel. — maybe skip)
  "Get recent orders for customer X"  → composite index (customer_id, created_at DESC)
  "Order details page"                → primary key id lookup (already indexed)

Result:
  PRIMARY KEY (id)                         ← automatic
  INDEX (customer_id)                      ← for single-customer queries
  INDEX (customer_id, created_at DESC)     ← for "customer's recent orders"
  INDEX (status, created_at DESC)          ← only if status filter is selective
```

---

## 8. Indexes in NoSQL

NoSQL databases also support indexes, though with more constraints:

| Database | Primary Index | Secondary Index | Notes |
|---|---|---|---|
| **Cassandra** | Partition + clustering key | Limited — expensive at scale | Schema design replaces most indexing needs |
| **MongoDB** | `_id` field | Any field, including nested | Rich index support similar to relational |
| **DynamoDB** | Partition + sort key | GSI (Global Secondary Index) | Pre-define access patterns; indexes cost money |
| **Redis** | Key itself | Sorted sets for range queries | No traditional indexes; data structure is the index |

> **In wide-column stores (Cassandra), the schema IS the index.** You design your partition and clustering keys to answer your specific queries — secondary indexes on Cassandra are generally avoided at scale.

---

## 9. Best Practices

- **Index every foreign key** — unindexed FK columns cause full scans on JOINs.
- **Put the most selective column first** in a composite index.
- **Use covering indexes** for your highest-frequency, most critical reads.
- **Avoid indexing low-selectivity columns** (booleans, status codes with few values).
- **Don't apply functions to indexed columns in queries** — it breaks the index.
- **Use partial indexes** to keep indexes small and fast when you only query a subset.
- **Verify with EXPLAIN / QUERY PLAN** — don't assume the index is being used.

---

## 10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No index on foreign keys | JOINs become full scans; catastrophic at scale | Index every FK column |
| Over-indexing | Write performance degrades significantly | Index based on actual query needs |
| Wrong column order in composite index | Index not used for expected queries | Most selective / most filtered column first |
| Function on indexed column in WHERE | Index bypassed; full scan | Index computed column or rewrite query |
| Ignoring query plan | Assuming index is used when it isn't | Always EXPLAIN critical queries |

---

## 11. Interview Questions

1. What is an index and what problem does it solve?
2. How does a B-Tree index work? What is its time complexity?
3. What is the left-prefix rule for composite indexes?
4. What is a covering index and when would you use one?
5. Why might a database ignore an index you've created?
6. What is the cost of indexes? What do you trade for faster reads?
7. Given a table with `user_id`, `created_at`, and `status`, what indexes would you add for the query "get active orders for user X ordered by date"?

---

## 12. Summary

| Concept | Key Takeaway |
|---|---|
| **Purpose** | Jump to data instead of scanning. Turns O(n) into O(log n). |
| **B-Tree** | Default type. Sorted, balanced, supports ranges. O(log n). |
| **Hash index** | O(1) exact match only. No ranges. |
| **Composite** | Multi-column. Left-prefix rule — column order matters. |
| **Covering** | All needed columns in index. Table never touched. |
| **Partial** | Index a subset of rows. Smaller, faster. |
| **Cost** | Every index slows writes. Index what you actually query. |
| **Selectivity** | High = effective. Low = database may ignore. |

---

## 13. Cross References

**Prerequisites:** Database Fundamentals (DB #1) · SQL & Relational (DB #2)

**Related Topics:** SQL & Relational (DB #2) · Partitioning & Sharding (DB #6) · Scalability (NFR #3)

**What to Learn Next:** Replication (DB #5) · Partitioning & Sharding (DB #6)

---

*System Design Engineering Handbook — Databases Series*