# 05 — Cache Invalidation

> **04-Caching Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Why Invalidation Is the Hard Problem

> *"There are only two hard things in computer science: cache invalidation and naming things."* — Phil Karlton

Caching is easy. The challenge is knowing **when cached data is wrong and removing it before anyone notices**.

The moment you cache a piece of data, a clock starts ticking. Somewhere, eventually, the source of truth (the database) will update. The cached copy is now stale. The question is: how long does the stale copy survive, and what damage does it do while it lives?

```
DB updated: product price changes from £29 to £25 at T=0

Without proper invalidation:
  User A reads cache at T=1 → sees £29  (wrong)
  User A reads cache at T=30 → sees £29 (still wrong)
  User A reads cache at T=3600 → sees £29 (TTL expires → finally correct)

With proper invalidation:
  Price changes → cache entry deleted immediately
  User A reads at T=1 → cache miss → DB read → sees £25 (correct)
```

The goal of cache invalidation is to collapse that window — from "stale until TTL expires" to "stale for milliseconds."

---

## 2. Invalidation Strategies

### 2.1 TTL Expiry (Time-Based)

Every cache entry has a TTL. When it expires, the entry is deleted. The next request fetches fresh data.

```
Cache entry: price:456 = £29  TTL=3600s

T=0:    price changes in DB to £25
T=1:    user reads → cache HIT → £29 (stale, but within TTL window)
T=3600: TTL expires → entry deleted
T=3601: user reads → cache MISS → DB read → £25 (fresh)
```

**Advantage:** Automatic, requires no coordination, always works.
**Disadvantage:** Maximum staleness = TTL duration. For a 1-hour TTL, users may see stale data for up to 1 hour.

> **TTL is the backstop, not the strategy.** It guarantees stale data eventually disappears. But if your TTL is 1 hour and a price changes, you have a 1-hour window of wrong prices. Explicit invalidation is needed to close that window.

### 2.2 Event-Driven Invalidation (Write Invalidation)

When the database is updated, explicitly delete the cache entry immediately.

```
App updates product price in DB:
  → database.update("UPDATE products SET price=25 WHERE id=456")
  → cache.delete("product:456")
  (cache entry gone immediately)

Next read: cache MISS → DB read → fresh data → re-cached
```

**Advantage:** Near-zero staleness window.
**Disadvantage:** Application code must always remember to invalidate. Easy to miss in complex code paths. Race conditions possible (see Section 4).

### 2.3 Write-Through (Combined Write)

Data is written to cache and DB simultaneously. No separate invalidation step — the cache always has the latest value.

Covered in detail in `02-caching-strategies.md`.

### 2.4 Cache-Aside with Delete on Write

The most common combination in production: cache-aside for reads, explicit delete on writes.

```
Reads:  App → cache → miss → DB → populate cache
Writes: App → DB first → delete cache entry
```

**Why delete, not update?**

```
Race condition if you UPDATE cache on write:

T=1: Thread A writes price=£25 to DB
T=2: Thread B writes price=£30 to DB
T=3: Thread B updates cache: price=£30  (most recent write wins so far — correct)
T=4: Thread A updates cache: price=£25  (Thread A's update arrives late — stale wins!)

Result: DB says £30, cache says £25. Cache is wrong.

If you DELETE instead:
T=1: Thread A writes price=£25 to DB, deletes cache
T=2: Thread B writes price=£30 to DB, deletes cache
T=3: Next read → cache miss → DB read → £30 (always correct)
```

Deletion is safe because the next read always fetches the current DB value. Updating is dangerous because out-of-order updates corrupt the cache.

---

## 3. The Cache Stampede (Thundering Herd)

A cache stampede occurs when a popular cache entry expires, and many requests arrive simultaneously — all experiencing a miss at the same time, all going to the database at once.

```
Popular product page: cached, accessed 10,000 times/second
TTL expires at T=0:

T=0.001: Request 1 → miss → goes to DB
T=0.002: Request 2 → miss → goes to DB (key still not in cache)
T=0.003: Request 3 → miss → goes to DB
...
T=0.100: 1,000 simultaneous DB queries for the same data

DB overwhelmed → latency spikes → cascading failure
```

This is the thundering herd problem: a herd of requests stampedes toward the database the moment a popular cache entry falls.

### Solutions

**1. Probabilistic Early Expiration**

Before the TTL expires, start randomly refreshing the entry early. Each request has a small probability of refreshing based on how close the TTL is to expiring.

```
TTL=3600s. At T=3580s (20s remaining), each request has a small chance of refreshing.
By T=3600s, the entry has already been refreshed silently by one of the requests.
No stampede — the cache never actually expires for users.
```

**2. Mutex / Lock on Miss**

When a cache miss occurs, only one request fetches from the database. All others wait for that result.

```
Request 1 → miss → acquires lock → fetches from DB → populates cache → releases lock
Requests 2–1000 → miss → wait for lock → read from cache (already populated)

Result: 1 DB query instead of 1,000.
```

**Downside:** Adds latency for all waiting requests during the fetch window.

**3. Stale-While-Revalidate**

Serve the stale value immediately while a background process fetches the fresh value. No request waits.

```
Cache entry TTL expires:
  → Serve stale value to the user immediately (no miss penalty)
  → Background: fetch fresh value from DB → update cache

User sees a briefly stale value; no DB stampede; no added latency.
```

This is the best UX approach for non-critical data where brief staleness is acceptable.

**4. Refresh-Ahead / Pre-warming**

Proactively refresh entries before they expire. No expiry = no stampede.

```
TTL=3600s. At T=3000s (83% elapsed):
  Background job re-fetches and resets TTL to 3600s.
  Entry never actually expires → no stampede window.
```

---

## 4. Race Conditions in Invalidation

Even with explicit invalidation, race conditions can lead to stale data surviving longer than expected.

### The Classic Race

```
T=1: Thread A: reads DB (price=£25), about to write to cache
T=2: Thread B: updates DB (price=£30), deletes cache key
T=3: Thread A: writes £25 to cache ← stale! Thread A's read was before the update.

Result: DB=£30, cache=£25. Wrong.
```

**Solutions:**

**Option 1: Short TTL as safety net**
Even if the race occurs, TTL ensures the stale entry expires soon. Acceptable when brief staleness is tolerable.

**Option 2: Version-based caching**
Each DB update increments a version number. Cache entry includes the version. On write, only delete the entry if the cached version matches.

```
Read: cache key = "product:456:v7"  (includes version)
Write: DB version → 8
  → delete "product:456:v7"
  → next read fetches fresh, stores as "product:456:v8"
  → old "product:456:v7" entry in cache (if any) is now a different key → harmless
```

**Option 3: Compare-and-set on cache write**
Only write to cache if the key doesn't already exist (set NX — set if not exists). Prevents a late-arriving stale write from overwriting a fresh one.

---

## 5. Bulk Invalidation

Sometimes you need to invalidate many cache entries at once — all entries for a user, all entries for a product category, all entries belonging to a tenant.

```
User 123 changes their subscription plan:
  → Need to invalidate: user:123:profile, user:123:plan, user:123:features,
                        user:123:limits, user:123:dashboard
```

**Approaches:**

| Approach | How | Trade-off |
|---|---|---|
| **Delete each key explicitly** | List all affected keys and delete them | Simple but must maintain the full key list |
| **Namespace versioning** | Cache key includes a version: `user:123:v4:*`. Increment v to 5 → all old entries become unreachable | Entries aren't deleted — they expire via TTL. Wastes memory until TTL. |
| **Tag-based invalidation** | Tag cache entries; delete by tag | Powerful but complex; requires tag index |

**Namespace versioning (most practical):**

```
User version stored in DB: user:123:cache_version = 4
All cache keys: "user:123:v4:profile", "user:123:v4:plan", etc.

User changes plan:
  → Increment version: user:123:cache_version = 5
  → All old v4 keys are now "orphaned" — they'll be fetched from DB next request
     (because the key includes v4, not v5)
  → Old v4 entries eventually evicted by TTL or LRU
```

No explicit delete needed. Just increment a version.

---

## 6. Event-Driven Invalidation at Scale

In a microservices architecture, the service that writes data may not be the same service that owns the cache entry. You need a mechanism to propagate invalidation across service boundaries.

```
Order Service writes: order confirmed → deduct inventory
Inventory Service has a cache: product:456:stock = 100

How does Inventory Service's cache know to invalidate?
```

**Solution: publish an event on write; cache consumers subscribe**

```
Order Service:
  → DB: inventory decremented
  → Publishes event: {type: "inventory_changed", product_id: 456}

Inventory Service cache manager:
  → Subscribes to "inventory_changed" events
  → On event: cache.delete("product:456:stock")
  → Next read: fresh stock count from DB
```

This is **event-driven cache invalidation** — decoupled, scalable, and appropriate for microservices.

---

## 7. How Large Companies Handle Invalidation

| Company | Approach | Source |
|---|---|---|
| **Facebook** | Lease-based invalidation on Memcached; prevents thundering herd via leases | Facebook Eng Blog (public) |
| **CDNs globally** | TTL + manual purge API for urgent invalidation (Cloudflare, CloudFront) | Public CDN docs |
| **Most distributed systems** | Cache-aside + delete on write + short TTL as safety net | Industry standard |
| **Event-driven systems** | Kafka events trigger cache invalidation across services | Common microservices pattern |

---

## 8. Best Practices

- **Use TTL as the safety net, explicit delete as the primary strategy** — don't rely on TTL alone for consistency.
- **Always delete, never update on write** — prevents out-of-order write races.
- **Use stale-while-revalidate** for popular non-critical entries to prevent thundering herd.
- **Use namespace versioning** for bulk invalidation without expensive delete operations.
- **Use event-driven invalidation** in microservices to propagate cache invalidation across service boundaries.
- **Monitor stale read rate** — if your invalidation strategy is working, stale reads should be rare.

---

## 9. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Relying on TTL alone | Stale data lives for the full TTL duration | Explicit delete on write |
| Updating cache on write (not deleting) | Race conditions cause stale cache entries | Always delete, never update |
| No stampede protection for hot entries | DB overwhelmed on popular entry expiry | Stale-while-revalidate or mutex |
| Not invalidating across service boundaries | Cache and DB diverge silently | Event-driven invalidation via message bus |
| Deleting too broadly (cache flush) | Entire cache cleared → cold start stampede | Targeted invalidation by key or namespace |

---

## 10. Interview Questions

1. What is cache invalidation and why is it considered the hard problem?
2. Why should you delete a cache entry on write rather than updating it?
3. What is a cache stampede (Thundering Herd)? How do you prevent it?
4. What is stale-while-revalidate and when would you use it?
5. How do you perform bulk cache invalidation efficiently?
6. What is namespace versioning for caches?
7. In a microservices system where service A writes data and service B caches it, how does service B know to invalidate?

---

## 11. Summary

| Concept | Key Takeaway |
|---|---|
| **TTL** | Safety net. Caps staleness. Not a complete strategy alone. |
| **Delete on write** | Primary invalidation. Always delete, never update. Prevents race conditions. |
| **Stampede** | Many concurrent misses → DB overwhelmed. Prevent with stale-while-revalidate or mutex. |
| **Stale-while-revalidate** | Serve stale; refresh in background. Best UX for non-critical data. |
| **Namespace versioning** | Increment a version → old keys orphaned. Efficient bulk invalidation. |
| **Event-driven** | Publish write events → cache consumers invalidate. Microservices standard. |

---

## 12. Cross References

**Prerequisites:** 01-caching-fundamentals.md · 02-caching-strategies.md · 04-distributed-caching.md

**Related Topics:** Consistency (NFR #5) · Fault Tolerance (NFR #6) · Messaging Systems

**What to Learn Next:** 05-Messaging section · 06-Distributed-Systems section

---

*System Design Engineering Handbook — 04-Caching Series*

---

> **04-Caching series complete.**
> Covered: Fundamentals · Strategies · Eviction Policies · Distributed Caching · Cache Invalidation