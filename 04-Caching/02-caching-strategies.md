# 02 — Caching Strategies

> **04-Caching Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Overview

A caching strategy defines the rules for **when data enters the cache**, **when it is updated**, and **when it is removed**. Choosing the wrong strategy for a use case leads to stale data, poor hit ratios, data loss, or unnecessary complexity.

There are four primary strategies. Each makes a different trade-off between read performance, write performance, consistency, and complexity.

| Strategy | Data Enters Cache | Write Goes To | Consistency | Best For |
|---|---|---|---|---|
| **Cache-Aside** | On read miss | DB directly | Eventual | General-purpose reads |
| **Read-Through** | On read miss (via cache) | DB directly | Eventual | Simplifying read logic |
| **Write-Through** | On write | Cache + DB together | Strong | Read-heavy, freshness critical |
| **Write-Behind** | On write | Cache first, DB async | Weak | Write-heavy, throughput critical |

---

## 2. Cache-Aside (Lazy Loading)

The most widely used strategy. The application manages the cache explicitly. The cache is a side data structure — hence the name.

### How It Works

```
READ:
  1. App checks cache for key
  2a. HIT  → return value from cache
  2b. MISS → App queries database
            → App stores result in cache with TTL
            → App returns result to caller

WRITE:
  1. App writes new value to database
  2. App deletes (invalidates) the cache entry for that key
     → Next read will be a miss → fresh value loaded from DB
```

```
App ──GET user:123──→ Cache
                         │
                  HIT ───┘ return immediately
                         │
                  MISS ──→ DB ──→ value
                              ←── App stores in cache
                              ←── return to caller
```

### Why Delete on Write (Not Update)?

When a write happens, you delete the cache entry rather than update it. This avoids a subtle race condition:

```
Thread A: writes new value to DB, about to update cache
Thread B: writes a different value to DB, updates cache first
Thread A: updates cache with its (now stale) value
→ Cache now holds stale data until TTL expires

Solution: delete the key on write → next read fetches fresh from DB
```

Deletion is safer. The cost is one extra DB read on the next cache miss, which is acceptable.

### Advantages

- Only requested data is cached — no memory wasted on data that's never read
- Cache failures are non-fatal — app reads directly from DB
- Works with any existing database; no special integration needed
- Simple to implement and reason about

### Disadvantages

- First request for any key is always a miss (cold start per key)
- Application code is responsible for both cache and DB logic — coupling
- Race condition between delete and read must be handled carefully

> **Use cache-aside as your default.** It's the most battle-tested, flexible, and failure-tolerant caching pattern.

---

## 3. Read-Through

Similar to cache-aside, but the **cache itself** (not the application) is responsible for loading data from the database on a miss. The application only ever talks to the cache.

### How It Works

```
READ:
  1. App requests key from cache
  2a. HIT  → cache returns value immediately
  2b. MISS → cache fetches from DB automatically
           → cache stores result
           → cache returns value to app

App never talks to DB directly for reads.
```

```
App ──GET product:456──→ Cache
                              │
                       HIT ───┘ return
                              │
                       MISS ──→ (cache loads from DB itself)
                              ←── return to App
```

### Difference from Cache-Aside

| | Cache-Aside | Read-Through |
|---|---|---|
| Who fetches on miss | Application | Cache layer |
| App knows about DB | Yes | No (for reads) |
| Simplifies app code | No | Yes |
| Cache flexibility | High | Requires cache to know how to load data |

### Advantages

- Application code is simpler — only talks to cache
- Cache and DB loading logic in one place

### Disadvantages

- Cache must be configured with a data loader — couples cache to DB schema
- Less flexible when loading logic is complex or data comes from multiple sources
- Cold start problem still exists

> **Read-through is common in ORM caching layers and frameworks** where the cache sits transparently between the ORM and the DB.

---

## 4. Write-Through

Every write goes to the cache and the database simultaneously, in the same operation. The cache is always up to date.

### How It Works

```
WRITE:
  1. App writes value to cache
  2. Cache synchronously writes to DB (or app writes to both)
  3. Both updated → acknowledge to caller

READ:
  1. App checks cache → always a HIT (cache has latest data)
```

```
App ──WRITE price=£25──→ Cache ──synchronously──→ DB
                          (both written before ACK)

App ──READ price──→ Cache → HIT (always fresh)
```

### Advantages

- Cache always has fresh data — reads never serve stale values
- No cache invalidation needed — data is always current in cache

### Disadvantages

- **Write latency is higher** — every write must update two systems
- **Write amplification** — every written key is cached, even if it's never read. Wastes memory.
- Write throughput may be limited by cache write speed

### When To Use

Write-through suits systems that are read-heavy with strict freshness requirements, where the extra write latency is acceptable. Product catalogues, configuration systems, reference data.

```
Write frequency: low–medium
Read frequency: very high
Staleness tolerance: very low
→ Write-through is a good fit
```

---

## 5. Write-Behind (Write-Back)

The application writes to the cache only. The cache asynchronously flushes to the database in the background. The DB is eventually consistent with the cache.

### How It Works

```
WRITE:
  1. App writes to cache → acknowledged immediately (fast)
  2. Cache queues the write
  3. Background process flushes queue to DB asynchronously

READ:
  1. App reads from cache → HIT (always latest, even before DB is updated)
```

```
App ──WRITE──→ Cache (ACK immediately) ──async later──→ DB
App ──READ──→ Cache → latest value (even if DB hasn't been updated yet)
```

### Advantages

- **Lowest write latency** — caller doesn't wait for DB write
- **Highest write throughput** — multiple writes to the same key can be coalesced into one DB write
- Write batching reduces DB load significantly

### Disadvantages

- **Risk of data loss** — if the cache crashes before flushing to DB, the buffered writes are lost
- **Complexity** — the async flush mechanism must be reliable and ordered
- **Inconsistency** — DB is behind cache; any direct DB reader sees stale data

### When To Use

```
Scenario: 1,000 writes/second to a counter (view count, like count)
Write-behind: update cache counter → flush to DB every 5 seconds
→ DB gets 1 write per 5 seconds instead of 1,000
→ 99.9% reduction in DB write load
→ Acceptable: losing a few view counts on crash is not a business problem
```

Write-behind is appropriate when:
- Data loss of buffered writes is acceptable (analytics, counters, non-financial data)
- Write throughput exceeds what the DB can handle directly
- Writes to the same key are frequent and can be coalesced

**Never use write-behind for** financial data, orders, or anything where losing a write causes a real-world problem.

---

## 6. Refresh-Ahead (Pre-fetching)

A proactive strategy. The cache predicts which entries will be needed soon and refreshes them before they expire — avoiding a miss when the TTL runs out.

### How It Works

```
Entry has TTL=60 seconds.
Refresh-ahead threshold: 80% of TTL elapsed (48 seconds).

At T=48s: cache detects entry is 80% through its TTL
         → proactively fetches fresh value from DB in background
         → replaces the old entry before it expires

At T=60s: entry would have expired, but it's already been refreshed.
          → No miss. No added latency for the caller.
```

### Advantages

- Eliminates miss latency for hot, predictable data — callers never wait
- Works well when access patterns are predictable (popular products, dashboards)

### Disadvantages

- Pre-fetches data that may not be requested → wasted DB queries
- Complex to implement correctly
- Only effective for truly "hot" data with predictable access

---

## 7. Choosing the Right Strategy

```
Is the data read frequently by many users?
    YES → Cache-aside or Read-through
    NO  → Don't cache it

Is write latency a critical concern?
    YES → Write-behind (if data loss on crash is acceptable)
    NO  → Continue

Do reads need to always see the latest value (no staleness)?
    YES → Write-through
    NO  → Cache-aside with appropriate TTL

Is the system read-heavy (>90% reads)?
    YES → Write-through or Cache-aside
    NO (write-heavy) → Write-behind

Is data access predictable and hot?
    YES → Add Refresh-ahead on top of chosen strategy
```

| Use Case | Recommended Strategy |
|---|---|
| Social media profiles | Cache-aside (read-heavy; some staleness acceptable) |
| Product catalogue | Write-through (reads must see latest price) |
| Analytics counters (views, likes) | Write-behind (high write volume; tolerate loss) |
| Session tokens | Cache-aside with short TTL |
| Configuration / feature flags | Write-through or refresh-ahead |
| Financial balances | Do not cache (or cache-aside with very short TTL + read-through on critical reads) |

---

## 8. How Large Companies Apply These Strategies

| Company | Strategy | Application | Source |
|---|---|---|---|
| **Facebook** | Cache-aside | Social graph data; application manages Memcached explicitly | Facebook Eng Blog (public) |
| **Netflix** | Read-through + refresh-ahead | Metadata caching; proactively refreshes popular content metadata | Netflix Tech Blog (public) |
| **Redis use cases** | Write-behind | Counters (likes, views) — async flush to persistent store | Redis public docs |
| **Most web apps** | Cache-aside | Standard for database query caching | Industry standard |

---

## 9. Best Practices

- **Default to cache-aside** — most flexible, failure-safe, and easiest to reason about.
- **Use write-through** only when reads must always be fresh and write latency increase is acceptable.
- **Use write-behind** only for non-critical write-heavy data (counters, analytics) where losing buffered writes is acceptable.
- **Never use write-behind for financial or transactional data.**
- **Combine strategies** — cache-aside for most data; write-behind for high-frequency counters.
- **Delete cache entries on write** (don't update) in cache-aside to avoid race conditions.

---

## 10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Updating cache on write (not deleting) | Race condition leaves stale value in cache | Delete the key; let next read fetch fresh |
| Write-behind for financial data | Data loss on crash = money lost | Write-through or skip cache for financial writes |
| Cache-aside without TTL | Stale data lives forever | Always set TTL |
| Write-through for write-heavy workload | Write latency too high; throughput collapses | Use write-behind or remove cache from write path |
| No cache invalidation on write | Reads serve stale data indefinitely | Invalidate (delete) cache on every relevant write |

---

## 11. Interview Questions

1. What is cache-aside and how does it differ from read-through?
2. Why do you delete a cache entry on write rather than updating it?
3. What is write-through? What does it trade off compared to cache-aside?
4. When is write-behind the right strategy? What is its risk?
5. For a product catalogue (high reads, low writes, price must be current): which strategy?
6. For a "likes" counter updated thousands of times per second: which strategy?
7. What is the race condition in cache-aside and how do you mitigate it?

---

## 12. Summary

| Strategy | Write Path | Read Path | Best For | Risk |
|---|---|---|---|---|
| **Cache-Aside** | DB only; delete cache | Check cache → miss → DB | General reads | Cold start; race on write |
| **Read-Through** | DB only | Cache loads from DB on miss | Simplified reads | Cache-DB coupling |
| **Write-Through** | Cache + DB together | Always cache (fresh) | Read-heavy, freshness critical | Higher write latency |
| **Write-Behind** | Cache only; async DB | Always cache | Write-heavy, throughput critical | Data loss on crash |
| **Refresh-Ahead** | Any of the above | Proactive refresh before expiry | Predictable hot data | Wasted prefetches |

---

## 13. Cross References

**Prerequisites:** 01-caching-fundamentals.md

**Related Topics:** 03-eviction-policies.md · 05-cache-invalidation.md · Consistency (NFR #5)

**What to Learn Next:** 03-eviction-policies.md

---

*System Design Engineering Handbook — 04-Caching Series*