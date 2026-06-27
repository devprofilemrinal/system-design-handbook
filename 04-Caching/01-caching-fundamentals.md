# 01 — Caching Fundamentals

> **04-Caching Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Caching?

Caching is storing the result of an expensive operation in a faster, closer store so that future requests for the same result can be served without repeating that operation.

The word "expensive" almost always means one of three things:
- **Slow** — reading from disk, making a network call, querying a database
- **Computationally heavy** — rendering a page, running a recommendation algorithm
- **Costly** — calling a paid external API, running a big database aggregation

The cache sits between the requester and the slow source of truth. If the answer is already in the cache, return it immediately. If not, go get it, store it, then return it.

```
WITHOUT cache:
Every request → expensive operation → result       (always slow)

WITH cache:
Request → cache hit?
            YES → return immediately               (fast)
            NO  → expensive operation → store in cache → return
                                                   (slow first time, fast after)
```

> **The fundamental trade-off:** Caches trade memory for speed, and freshness for performance. You get faster responses but must manage the risk of serving data that is no longer accurate.

---

## 2. Why Caching Exists — The Hardware Reality

The reason caching works at all is a physical fact about computer hardware: different storage layers have radically different speeds and costs.

```
Storage Hierarchy (fastest → slowest):

Layer             | Latency      | Size         | Cost
──────────────────|──────────────|──────────────|──────────
CPU L1 Cache      | ~1 ns        | ~32 KB       | Very high
CPU L2 Cache      | ~4 ns        | ~256 KB      | High
CPU L3 Cache      | ~40 ns       | ~8 MB        | Medium-high
RAM (memory)      | ~100 ns      | ~GBs         | Medium
SSD               | ~16,000 ns   | ~TBs         | Low
HDD (disk)        | ~2,000,000 ns| ~TBs         | Very low
Network (same DC) | ~500,000 ns  | unlimited    | -
Network (internet)| ~100,000,000 ns | unlimited | -
```

Every layer of caching in software mirrors this hardware reality: move frequently-needed data closer to where it's consumed, into faster storage, so you pay the slow cost as rarely as possible.

> **Key insight:** RAM is ~160,000× faster than a network call across a data centre. Storing a database query result in Redis instead of re-querying is not a micro-optimisation — it's a fundamental architectural shift.

---

## 3. Cache Hit and Cache Miss

These two terms appear in every caching discussion. They describe what happens when a request arrives at the cache.

**Cache Hit:** The requested data is in the cache. Serve it immediately. Fast.

**Cache Miss:** The data is not in the cache. Must fetch from the source (database, API, computation). Slow. After fetching, store the result in the cache for next time.

```
Hit ratio = Cache hits / Total requests

Example:
  1000 requests, 850 served from cache, 150 from database
  Hit ratio = 850 / 1000 = 85%
```

**Hit ratio is the single most important metric for a cache.** A cache with a 50% hit ratio is barely helping. A cache with a 99% hit ratio is transformative — 99% of requests never touch the database.

What drives hit ratio:
- **Cache size:** Bigger cache = more data fits = more hits
- **TTL:** Too short = entries expire before they're reused = more misses
- **Data access pattern:** If users mostly request recent or popular content, hit ratio is naturally high (temporal and popularity locality)
- **Cache warming:** Pre-loading the cache on startup before real traffic arrives

---

## 4. The Cache Hierarchy in a Real System

In production, caching happens at multiple layers simultaneously. Each layer catches requests before they reach the next slower layer.

```
User's Browser
    │ (cache: HTML, CSS, JS, images — browser cache)
    ▼
CDN Edge Node
    │ (cache: static assets, public API responses)
    ▼
API Gateway / Reverse Proxy
    │ (cache: stable API responses)
    ▼
Application Server
    │ (cache: in-process / local cache for hot objects)
    ▼
Distributed Cache (Redis)
    │ (cache: DB query results, sessions, computed data)
    ▼
Database
    │ (cache: buffer pool — DB's own internal page cache)
    ▼
Disk
```

Each layer has a different scope, TTL policy, and purpose. Together they form a **defence in depth** against slow operations. A request that gets caught by the CDN never reaches the application. One caught by Redis never reaches the database.

---

## 5. What to Cache — and What Not To

Knowing what is safe and beneficial to cache is as important as knowing how to cache.

**Good candidates:**

| Data | Why |
|---|---|
| Database query results (hot queries) | Same query run by many users repeatedly |
| Rendered HTML pages (for anonymous users) | Expensive to generate; same for all visitors |
| API responses from external services | External calls are slow and often rate-limited |
| Session tokens | Read on every authenticated request |
| Configuration and feature flags | Read constantly; changes rarely |
| Computed aggregations (leaderboards, counts) | Expensive to compute; tolerate slight staleness |
| Static assets (images, CSS, JS) | Never change (versioned by filename) |

**Poor candidates:**

| Data | Why Not |
|---|---|
| User-specific private data (without isolation) | Risk of one user seeing another's data |
| Financial balances, inventory counts | Cannot tolerate stale reads |
| Real-time data (live prices, live scores) | Changes every second; cache is harmful |
| Unique per-request data | No reuse possible; cache just wastes memory |
| Very large objects | Single entry consumes disproportionate memory |

---

## 6. TTL — Time To Live

Every cache entry must have a TTL — a duration after which it automatically expires and is removed. On expiry, the next request is a miss and fetches fresh data.

```
Cache entry: product:456 = {name: "Widget", price: £29.99}
TTL: 3600 seconds (1 hour)

Requests in next hour: cache HIT → fast
After 1 hour: entry expires → next request is a MISS → fresh fetch → re-cached
```

**Choosing TTL:**

| TTL Length | Effect | Use When |
|---|---|---|
| Very short (1–60s) | Frequently refreshed; nearly always fresh; more DB traffic | Data changes often; low staleness tolerance |
| Medium (5–60 min) | Good balance for most use cases | Product data, user profiles, listings |
| Long (hours–days) | Very few DB hits; risk of stale data lingering | Configuration, rarely-changed reference data |
| No TTL (permanent) | Never expires; must be manually invalidated | Immutable data (completed events, archived records) |

> **Always set a TTL.** An entry with no TTL is a memory leak — it grows forever. TTL is also your safety net against failed invalidation — even if your invalidation logic has a bug, the entry will eventually expire.

---

## 7. Cold Start Problem

When a system starts fresh (deployment, restart, new region), the cache is empty. Every request is a miss until the cache warms up. This can be dangerous:

```
Normal operation: 95% cache hit rate → 5% of requests hit database
Cold start: 0% hit rate → 100% of requests hit database simultaneously
→ Database overwhelmed → latency spikes → possible outage
```

**Mitigations:**

| Approach | How |
|---|---|
| **Cache warming** | Pre-load hot data into cache before routing real traffic |
| **Gradual traffic shift** | Send small % of traffic first; let cache warm before full cutover |
| **Persistent cache** | Redis persistence (RDB/AOF) survives restarts with data intact |
| **Background pre-fetching** | Proactively load likely-needed data before requests arrive |

---

## 8. Caching and Consistency

Every cache introduces a consistency gap — the window between when data changes in the database and when the cache reflects that change. Understanding this gap is critical.

```
Database updated at T=0: product price changed to £25
Cache still holds: £29.99 (cached 30 minutes ago, TTL=60min)
User reads at T=10: sees £29.99 ← stale, but cache doesn't know yet
Cache expires at T=30: next request fetches fresh £25
```

This gap is acceptable for many use cases (product descriptions, blog posts) and unacceptable for others (inventory, financial balances).

> **The question is not "how do I eliminate stale data?" but "how much stale data can my use case tolerate?"** Design TTLs and invalidation strategies around that tolerance.

---

## 9. How Large Companies Use Caching

| Company | Application | Source |
|---|---|---|
| **Facebook** | Memcached as primary read cache for social graph data; billions of requests per second served from cache | Facebook Eng Blog (public) |
| **Stack Overflow** | Extreme in-process caching; serves millions of requests with a small number of servers | Stack Overflow Blog (public) |
| **Netflix** | EVCache (distributed Memcached) for personalisation metadata; very high hit rates across regions | Netflix Tech Blog (public) |
| **Twitter/X** | Redis for pre-computed timelines; the fan-out problem solved by caching user feeds | Public talks |

---

## 10. Best Practices

- **Measure hit ratio first** — if you don't know your hit ratio, you don't know if your cache is working.
- **Set a TTL on every entry** — no exceptions; prevents memory leaks and caps staleness.
- **Cache at the right layer** — CDN for public static content; Redis for database results; in-process for immutable config.
- **Don't cache errors** — a cached error response means valid users get error pages long after the problem is fixed.
- **Warm the cache before routing traffic** — prevent cold-start database overload.
- **Include version/user identifiers in cache keys** — prevents cross-user data leaks.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No TTL on entries | Memory grows forever; stale data never refreshed | Always set TTL |
| Caching without measuring hit ratio | Cache may be doing nothing useful | Monitor hit/miss ratio; optimise toward >80% |
| Using same cache key for different users | User A sees User B's private data | Include user_id in cache key for user-specific data |
| Caching error responses | Errors cached and served to valid users | Never cache non-2xx responses |
| Cold start without warming | Database overwhelmed on restart | Implement cache warming or gradual traffic routing |
| Caching at only one layer | Misses load reduction opportunities | Layer caches: CDN → app → Redis → DB |

---

## 12. Interview Questions

1. What is caching and what problem does it solve?
2. What is a cache hit ratio and why does it matter?
3. What types of data are good candidates for caching? What types are not?
4. What is TTL and what happens if you don't set one?
5. What is the cold start problem and how do you mitigate it?
6. How does caching introduce a consistency gap? When is that acceptable?
7. Name the layers where caching can occur in a system and what each layer caches.

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Purpose** | Trade memory for speed; avoid repeating expensive operations |
| **Hardware reality** | RAM is ~160,000× faster than a network call; caching exploits this |
| **Hit ratio** | The key metric. Target >80%. |
| **TTL** | Always set one. Safety net against stale data and memory leaks. |
| **What to cache** | Frequently-read, shared, tolerable-staleness data |
| **What not to cache** | Financial data, real-time data, private data without key isolation |
| **Cold start** | Empty cache on boot → warm before routing traffic |
| **Consistency gap** | Inevitable. Accept and design around tolerable staleness. |

---

## 14. Cross References

**Prerequisites:** System Design Fundamentals · Latency & Throughput (NFR #1) · Scalability (NFR #3)

**Related Topics:** DB Caching Overview (DB #8) · CDN (Building Blocks #4)

**What to Learn Next:** 02-caching-strategies.md · 03-eviction-policies.md

---

*System Design Engineering Handbook — 04-Caching Series*