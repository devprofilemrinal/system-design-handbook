# SQL & Relational Databases

> **Databases #2** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is a Relational Database?

A relational database organises data into **tables** (also called relations). Each table has rows (records) and columns (fields). Tables can be linked to each other through **relationships** using keys.

The term "relational" comes from the mathematical theory of relations — not from "the tables are related to each other," though that is also true.

```
users table:                    orders table:
┌────┬───────┬───────────────┐  ┌─────────┬─────────┬────────┐
│ id │ name  │ email         │  │ id      │ user_id │ total  │
├────┼───────┼───────────────┤  ├─────────┼─────────┼────────┤
│  1 │ Alice │ alice@mail.com│  │ 1001    │ 1       │ £49.99 │
│  2 │ Bob   │ bob@mail.com  │  │ 1002    │ 1       │ £12.00 │
│  3 │ Carol │ carol@mail.com│  │ 1003    │ 2       │ £99.00 │
└────┴───────┴───────────────┘  └─────────┴─────────┴────────┘
                                 user_id refers back to users.id (foreign key)
```

The power of the relational model: you can ask complex questions that span multiple tables — "give me all orders placed by users who signed up in the last 30 days" — and the database handles the complexity efficiently.

---

## 2. Core Concepts

### Primary Key
A column (or set of columns) that uniquely identifies each row in a table. No two rows can have the same primary key. Every table should have one.

```
users.id is the primary key → no two users can have the same id
```

### Foreign Key
A column in one table that references the primary key of another table. This creates and enforces a relationship between tables.

```
orders.user_id is a foreign key → it references users.id
→ you cannot insert an order with user_id = 99 if no user with id=99 exists
→ this constraint is enforced by the database automatically
```

### Normalisation
The process of organising a database to reduce redundancy. The goal: store each piece of information exactly once.

```
BAD (denormalised — user name duplicated in every order):
orders: [id=1001, user_name="Alice", total=£49.99]
        [id=1002, user_name="Alice", total=£12.00]
        If Alice changes her name → must update every order row

GOOD (normalised — user name stored once):
users:  [id=1, name="Alice"]
orders: [id=1001, user_id=1, total=£49.99]
        [id=1002, user_id=1, total=£12.00]
        Alice changes her name → update one row in users
```

**Normal Forms:**

| Form | Rule | Eliminates |
|---|---|---|
| **1NF** | Each column holds atomic (single) values; no repeating groups | Duplicate columns |
| **2NF** | 1NF + every non-key column depends on the entire primary key | Partial dependencies |
| **3NF** | 2NF + no non-key column depends on another non-key column | Transitive dependencies |

> **Target 3NF for transactional systems.** Denormalise deliberately only when query performance requires it and you accept the consistency trade-off.

---

## 3. Joins — Querying Across Tables

Joins combine rows from multiple tables based on a related column.

```
SELECT users.name, orders.total
FROM orders
JOIN users ON orders.user_id = users.id
WHERE orders.total > 50

→ Returns: Alice, £99.00 (Bob's order is £99 — Carol has no orders)
```

| Join Type | Returns |
|---|---|
| **INNER JOIN** | Only rows where a match exists in both tables |
| **LEFT JOIN** | All rows from left table; nulls where no match in right |
| **RIGHT JOIN** | All rows from right table; nulls where no match in left |
| **FULL JOIN** | All rows from both tables; nulls where no match on either side |

> **INNER JOIN is the most common.** If you're designing a schema and you find yourself needing many LEFT JOINs, consider whether your data model is missing a relationship.

---

## 4. Transactions and Isolation Levels

A **transaction** groups multiple operations into one atomic unit. Either all succeed or all are rolled back.

```
BEGIN TRANSACTION
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A'
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B'
COMMIT   ← both succeed → changes saved
         ← if any step fails → ROLLBACK → neither change applied
```

### Isolation Levels

When multiple transactions run concurrently, isolation levels control how much they interfere with each other. Higher isolation = more correct, but slower (more locking).

| Level | Problem Prevented | Still Possible | Performance |
|---|---|---|---|
| **Read Uncommitted** | Nothing | Dirty reads, phantom reads | Fastest |
| **Read Committed** | Dirty reads | Non-repeatable reads, phantom reads | Fast |
| **Repeatable Read** | Dirty + non-repeatable reads | Phantom reads | Moderate |
| **Serializable** | All anomalies | Nothing | Slowest |

**The three problems:**
- **Dirty read:** You read data another transaction wrote but hasn't committed yet — you might read something that gets rolled back.
- **Non-repeatable read:** You read a row twice in the same transaction and get different values because another transaction modified it between your reads.
- **Phantom read:** You run the same query twice and get different *sets* of rows because another transaction inserted or deleted rows between your reads.

> **Practical default: Read Committed.** Most databases (PostgreSQL, Oracle) use this. Use Serializable only when absolute correctness is required (financial systems, inventory); it has significant performance cost.

Full coverage of isolation levels in the Transactions & Isolation document.

---

## 5. Indexes in Relational Databases

Indexes are the primary performance tool in relational databases. An index on a column allows the database to find matching rows without scanning the entire table.

```
Table: 100 million orders
Query: SELECT * FROM orders WHERE customer_id = 12345

Without index: scan all 100M rows → seconds
With index on customer_id: jump to matching rows → milliseconds
```

**What to always index:**
- Primary keys (automatic)
- Foreign keys
- Columns in WHERE clauses
- Columns in ORDER BY and GROUP BY
- Columns frequently used in JOINs

> Indexes have a cost: every INSERT, UPDATE, and DELETE must also update each index. Don't index everything — index what your actual queries need. Full coverage in the Indexing document.

---

## 6. Schema Design Patterns

### One-to-Many
One user has many orders. The "many" side holds the foreign key.

```
users (1) ──────────── orders (many)
              orders.user_id → users.id
```

### Many-to-Many
A student can enrol in many courses; a course has many students. Requires a **junction table**.

```
students ──── enrolments ──── courses
              (junction table holds both foreign keys)
              enrolments.student_id → students.id
              enrolments.course_id  → courses.id
```

### Soft Delete
Instead of actually deleting rows (which is irreversible and removes history), mark them as deleted.

```
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

"Delete" user: UPDATE users SET deleted_at = NOW() WHERE id = 123
Query active users: SELECT * FROM users WHERE deleted_at IS NULL
```

---

## 7. When Relational Databases Struggle

Relational databases are the right default for most systems, but they have limits.

| Scenario | Problem | Alternative |
|---|---|---|
| Schema changes very frequently | ALTER TABLE on large tables is painful (locks rows) | Document DB |
| Write throughput > ~100K TPS | Single primary becomes a bottleneck | Wide-column (Cassandra) |
| Highly connected data (social graph) | Multi-level JOINs become exponentially expensive | Graph DB |
| Full-text search with relevance ranking | SQL LIKE is not production-grade search | Elasticsearch |
| Columns vary wildly per row (sparse) | Many NULL columns; wasteful | Document or Wide-Column DB |

> **These limits don't mean you should avoid relational DBs. They mean you should know when to reach for something else.**

---

## 8. How Large Companies Use Relational Databases

| Company | Application | Source |
|---|---|---|
| **GitHub** | Uses MySQL for core repository metadata, user data, issues, PRs | GitHub Eng Blog (public) |
| **Shopify** | MySQL at massive scale; extensive sharding by shop ID | Shopify Eng Blog (public) |
| **Instagram** | PostgreSQL for user data, posts metadata, relationships | Instagram Eng Blog (public) |
| **Airbnb** | MySQL for listings, bookings, users — core transactional data | Airbnb Eng Blog (public) |

> **Observation:** The largest internet companies in the world still use relational databases for their core transactional data. NoSQL is a complement, not a replacement.

---

## 9. Best Practices

- **Use relational as your default.** Switch to specialist DBs only when you have a demonstrated need.
- **Always define a primary key** on every table.
- **Always add foreign key constraints** — let the database enforce referential integrity.
- **Target 3NF** for transactional schemas; denormalise only when profiling shows a need.
- **Index foreign keys** — JOINs on unindexed columns are full scans.
- **Use transactions** for any multi-step operation that must be atomic.
- **Use appropriate isolation level** — Read Committed for most cases; Serializable when correctness is critical.

---

## 10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No primary key | Can't uniquely identify rows; duplicates accumulate | Every table gets a primary key |
| No foreign key constraints | Invalid references accumulate silently; data corruption | Enforce referential integrity at the database layer |
| Not indexing foreign keys | JOINs perform full scans as data grows | Index every foreign key column |
| Too much normalisation | Queries require 8 JOINs for simple data | Balance normalisation with query practicality |
| Too little normalisation | Update anomalies; inconsistent data | Apply 3NF to transactional data |
| SELECT * in application code | Fetches unnecessary columns; fragile to schema changes | Select only the columns you need |

---

## 11. Interview Questions

1. What is a primary key vs a foreign key?
2. What are the three normal forms? Why do we normalise?
3. Explain the four ACID properties with a banking example.
4. What are the four SQL isolation levels and what anomaly does each prevent?
5. When would you use a LEFT JOIN vs an INNER JOIN?
6. How do indexes improve query performance and what is their cost?
7. When would you choose a relational database over NoSQL?

---

## 12. Summary

| Concept | Key Takeaway |
|---|---|
| **Relational model** | Tables, rows, columns, relationships via keys |
| **Primary key** | Unique identifier per row — every table needs one |
| **Foreign key** | Links tables; database enforces referential integrity |
| **Normalisation** | Store each fact once; 3NF is the target |
| **Joins** | Combine tables; INNER JOIN is most common |
| **Transactions** | Atomic, all-or-nothing groups of operations |
| **Isolation** | Read Committed default; Serializable for critical correctness |
| **Indexes** | Essential for query performance; cost is slower writes |

---

## 13. Cross References

**Prerequisites:** Database Fundamentals (DB #1)

**Related Topics:** Indexing (DB #4) · Transactions & Isolation (DB #7) · Consistency (NFR #5)

**What to Learn Next:** NoSQL Deep Dive (DB #3) · Indexing (DB #4)

---

*System Design Engineering Handbook — Databases Series*