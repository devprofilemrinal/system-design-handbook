# Transactions & Isolation

> **Databases #7** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is a Transaction?

A transaction is a group of database operations that are treated as a single, indivisible unit of work. The key guarantee: either **all operations succeed**, or **none of them are applied**. There is no in-between.

```
Bank transfer: move £100 from Account A to Account B

Step 1: Debit  Account A by £100
Step 2: Credit Account B by £100

WITHOUT transactions:
  Step 1 succeeds. Server crashes. Step 2 never runs.
  → £100 has vanished from A and never appeared in B. Money gone.

WITH transactions:
  Step 1 and Step 2 run together.
  Server crashes after Step 1 → database rolls back Step 1 on recovery.
  → Both accounts unchanged. Retry the transfer.
```

> **Transactions are what separate a reliable system from a system that silently corrupts data under failure.**

---

## 2. ACID — Revisited in Depth

Introduced in Database Fundamentals. Here we go deeper on each property.

### Atomicity — All or Nothing

The transaction either commits fully or rolls back completely. Partial application is impossible.

**Implementation:** Databases use a **Write-Ahead Log (WAL)**. Every change is written to the log before being applied to the actual data. On crash recovery, the database replays the log for committed transactions and discards entries for uncommitted ones.

```
WAL:
  BEGIN txn-001
  DEBIT account A, £100
  CREDIT account B, £100
  COMMIT txn-001          ← crash here

Recovery:
  txn-001 has COMMIT in log → replay DEBIT and CREDIT → apply them
  (if crash before COMMIT → no COMMIT in log → discard → rollback)
```

### Consistency — Valid State to Valid State

Transactions must respect all database constraints (foreign keys, unique constraints, check constraints). A transaction that would violate a constraint is rejected.

```
Constraint: account balance cannot go below 0
Transaction: debit £200 from account with £150 balance
→ Database rejects the transaction → constraint preserved
```

### Isolation — Transactions Don't Interfere

Concurrent transactions behave as if they ran sequentially. This is the most complex property. Isolation is configurable — see Section 3.

### Durability — Committed Data Survives

Once the database says "committed," that data survives crashes, power loss, and hardware failure. Implemented via WAL flushed to durable storage (disk/SSD) before acknowledging commit.

---

## 3. Isolation Levels — The Core of This Document

Isolation is expensive. Achieving perfect isolation (every transaction as if it ran alone) requires significant locking that kills concurrency. In practice, databases offer configurable isolation levels — you choose how much isolation you need and pay the corresponding concurrency cost.

### The Three Anomalies You're Protecting Against

**Dirty Read:** You read a value written by another transaction that hasn't committed yet. If that transaction rolls back, you read data that never officially existed.

```
Transaction A: writes price = £50  (not yet committed)
Transaction B: reads price = £50   ← dirty read
Transaction A: rolls back → price is actually still £45
Transaction B is now working with a ghost value
```

**Non-Repeatable Read:** You read the same row twice within one transaction and get different values because another transaction modified and committed between your reads.

```
Transaction A: reads stock = 10
Transaction B: updates stock = 0, commits
Transaction A: reads stock = 0  ← different value! Not repeatable.
```

**Phantom Read:** You run the same query twice and get different *sets* of rows because another transaction inserted or deleted rows matching your filter.

```
Transaction A: SELECT * FROM orders WHERE total > 100  → 5 rows
Transaction B: INSERT new order with total = 150, commits
Transaction A: SELECT * FROM orders WHERE total > 100  → 6 rows  ← phantom
```

---

### The Four Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read | Performance |
|---|---|---|---|---|
| **Read Uncommitted** | ✅ Possible | ✅ Possible | ✅ Possible | Fastest |
| **Read Committed** | ❌ Prevented | ✅ Possible | ✅ Possible | Fast |
| **Repeatable Read** | ❌ Prevented | ❌ Prevented | ✅ Possible | Moderate |
| **Serializable** | ❌ Prevented | ❌ Prevented | ❌ Prevented | Slowest |

**Read Uncommitted:** No protection at all. Transactions can see each other's uncommitted changes. Almost never used in practice — the risks are too high.

**Read Committed:** The practical default for most databases (PostgreSQL, Oracle). You only see committed data. Prevents dirty reads. Non-repeatable reads are possible.

**Repeatable Read:** Once you read a row, no other transaction can modify it until you commit. Prevents non-repeatable reads. MySQL's InnoDB default. Still allows phantom reads in theory (though InnoDB's implementation prevents them too).

**Serializable:** The highest level. Transactions behave as if they ran one at a time. No anomalies possible. Implemented via strict locking or serializable snapshot isolation (SSI). Significant performance impact.

> **Practical guidance:** Use Read Committed for most applications. Use Serializable for financial transactions, inventory bookings, or any case where phantom reads would cause business damage (double-booking a seat, overselling inventory).

---

## 4. How Isolation Is Implemented — Locking vs MVCC

Two fundamentally different approaches:

### Pessimistic Locking

Assume conflicts will happen. Acquire locks before accessing data.

```
Transaction A wants to read + update row 42:
  → Acquires lock on row 42
  → Does its work
  → Releases lock

Transaction B wants the same row:
  → Waits for A to release the lock
  → Gets the lock → proceeds
```

Simple but reduces concurrency — transactions queue up waiting for locks. Risk of **deadlock**: A holds lock on row 1 waiting for row 2; B holds lock on row 2 waiting for row 1. Neither can proceed. Databases detect and resolve deadlocks by aborting one transaction.

### Optimistic Locking (MVCC — Multi-Version Concurrency Control)

Assume conflicts are rare. Don't lock — keep multiple versions of data.

```
Row 42 has version 5 (current value: price = £45)

Transaction A reads row 42 at version 5 (price = £45)
Transaction B reads row 42 at version 5 (price = £45)

Transaction A writes: price = £50, creates version 6
Transaction B writes: price = £40, checks: "is version still 5?" → NO (it's 6 now)
Transaction B must retry with the new version
```

**MVCC is what most modern databases use** (PostgreSQL, MySQL InnoDB, Oracle). It allows readers and writers to not block each other — readers see a consistent snapshot of the past, writers create new versions. This is why `SELECT` in PostgreSQL doesn't block `UPDATE` and vice versa.

---

## 5. Distributed Transactions

So far we've discussed transactions within a single database. When an operation spans multiple services or databases, maintaining atomicity becomes much harder.

```
Order Service: writes order to Order DB
Payment Service: charges card via Payment DB
Inventory Service: decrements stock in Inventory DB

If Payment succeeds but Inventory fails → inconsistent state
You can't use a database transaction across three different databases.
```

### Two-Phase Commit (2PC)

A coordinator asks all participants to prepare (guarantee they can commit), then tells everyone to commit or all to abort.

```
Phase 1 — Prepare:
  Coordinator → "Can you commit?" → Order DB, Payment DB, Inventory DB
  All respond "YES"

Phase 2 — Commit:
  Coordinator → "Commit!" → all three
  All commit.

If any respond "NO" in Phase 1:
  Coordinator → "Abort!" → all roll back
```

**Problem:** If the coordinator crashes between Phase 1 and Phase 2, participants are stuck holding locks, waiting indefinitely. 2PC is **blocking** — a coordinator failure blocks the whole system. Used in traditional enterprise systems but avoided in high-scale distributed systems.

### Saga Pattern

Break the distributed transaction into a sequence of local transactions, each with a **compensating action** (the undo operation) if something goes wrong.

```
1. Order Service: create order         → compensate: cancel order
2. Payment Service: charge card        → compensate: refund card
3. Inventory Service: reserve stock    → compensate: release stock

If step 3 fails:
  → Run compensating action for step 2 (refund)
  → Run compensating action for step 1 (cancel order)
  → System back to consistent state
```

Two styles:
- **Choreography:** Each service publishes events; the next service listens and reacts. No central coordinator.
- **Orchestration:** A central saga orchestrator tells each service what to do next.

**Saga is the preferred pattern for distributed transactions in microservices.** It's not a true atomic transaction — there's a window where the system is partially applied — but it achieves eventual consistency with full availability.

---

## 6. How Large Companies Apply This

| Company | Application | Source |
|---|---|---|
| **Banks globally** | Serializable isolation for ledger transactions; 2PC within a single DB cluster | Industry standard |
| **Stripe** | Sagas for payment flows across services; idempotency keys for safe retries | Stripe Eng Blog (public) |
| **Uber** | Saga orchestration for booking flows (driver reservation, payment, notification) | Uber Eng Blog (public) |
| **Amazon** | Avoids 2PC; uses eventual consistency + compensation heavily | Werner Vogels public talks |

---

## 7. Best Practices

- **Use transactions for any multi-step write** that must be atomic.
- **Use Read Committed** as your default isolation level.
- **Use Serializable** when double-booking or phantom reads would cause real damage.
- **Prefer MVCC databases** (PostgreSQL) — readers don't block writers.
- **Use Saga pattern** for distributed operations across microservices.
- **Use idempotency keys** on all compensating and retried operations to make them safe.
- **Keep transactions short** — long-running transactions hold locks or MVCC snapshots, hurting concurrency.

---

## 8. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No transactions on multi-step writes | Partial updates corrupt data on failure | Wrap in a transaction |
| Long-running transactions | Lock contention; snapshot size grows; other queries slow down | Keep transactions as short as possible |
| Wrong isolation level | Phantom reads cause double-booking; dirty reads cause decisions on ghost data | Choose isolation based on anomaly cost, not default |
| 2PC in microservices | Coordinator failure blocks the whole system | Use Saga pattern instead |
| Saga without idempotency | Compensating actions run multiple times; incorrect rollback | All saga steps and compensations must be idempotent |

---

## 9. Interview Questions

1. What is a database transaction? Why does it exist?
2. Explain each of the four ACID properties with a concrete example.
3. What is a dirty read? What isolation level prevents it?
4. What is the difference between pessimistic and optimistic locking?
5. What is MVCC and why do modern databases prefer it?
6. Why is Two-Phase Commit (2PC) problematic in distributed systems?
7. What is the Saga pattern? What is a compensating transaction?
8. What isolation level would you use for a seat-booking system? Why?

---

## 10. Summary

| Concept | Key Takeaway |
|---|---|
| **Transaction** | All-or-nothing group of operations. |
| **ACID** | Atomicity (all/nothing), Consistency (valid state), Isolation (no interference), Durability (survives crashes) |
| **Isolation levels** | Read Committed (default). Serializable (strictest, slowest). |
| **Dirty/NR/Phantom** | Three anomalies; each higher isolation level prevents more. |
| **MVCC** | Multiple versions; readers don't block writers. Modern default. |
| **2PC** | Coordinator-based distributed atomicity. Blocking on failure. |
| **Saga** | Distributed eventual consistency with compensating actions. Microservices standard. |

---

## 11. Cross References

**Prerequisites:** Database Fundamentals (DB #1) · SQL & Relational (DB #2) · Consistency (NFR #5)

**Related Topics:** Replication (DB #5) · Reliability (NFR #4) · Fault Tolerance (NFR #6)

**What to Learn Next:** Caching — `04-Caching` section · Database Selection (DB #9)

---

*System Design Engineering Handbook — Databases Series*