# Caching — Overview & Database Context

> **Databases #8** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Caching?

Caching is storing the result of an expensive operation in a faster, temporary store so that subsequent requests for the same result can be served almost instantly — without repeating the expensive work.

The expensive operation in the database context is almost always: **going to disk**. Disk reads are thousands of times slower than memory reads. Caching moves hot data from disk into memory.

```
WITHOUT cache:
  Request → Application → Database (disk) → response   ~10–100ms

WITH cache:
  Request → Application → Cache (memory) → response    ~0.1–1ms
                               │
                               └── (on miss) → Database → cache it → response
```

> **This document covers caching specifically in its relationship to databases — where it sits, which patterns to use, and what to watch out for. The full treatment of caching patterns, eviction policies, and distributed caching tools lives in the `04-Caching` section of this handbook.**

---

## 2. Why Caching Matters for Databases

Most production databases are read-heavy. A typical web application is 90–99% reads. Without caching, every page load translates into multiple database queries. At scale, this means:

- Database CPU and disk I/O saturate long before application servers do
- Latency grows as the database queues up requests
- Scaling the database vertically is expensive; horizontally is complex

Caching absorbs this read pressure before it reaches the database.

```
Without cache: 10,000 RPS → 10,000 DB queries/second → DB overloaded
With cache (90% hit rate): 10,000 RPS → 1,000 DB queries/second → DB comfortable
```

---

## 3. What Lives in a Cache vs What Lives in a Database

This is the foundational decision. Getting it wrong either defeats the cache or compromises data integrity.

| Data Characteristic | In Cache? | Reason |
|---|---|---|
| Frequently read, rarely changes | ✅ Yes | High hit rate; low invalidation overhead |
| Same for many users | ✅ Yes | One cached entry serves many users |
| Computationally expensive to generate | ✅ Yes | Cache the result; avoid recomputation |
| User-specific, sensitive | ⚠️ Sometimes | Cache per-user, with strict TTL |
| Changes every second | ❌ No | Stale immediately; cache is harmful |
| Requires strong consistency | ❌ No | Cache may serve outdated value |
| Transactional (financial records) | ❌ No | Cache has no ACID guarantees |
| Source of truth | ❌ No | Database is always the source of truth |

---

## 4. Where the Cache Sits — Cache Topology

### In-Process Cache (Local Cache)
The cache lives inside the application process itself. Zero network latency.

```
Application Process:
  [Business Logic] ←→ [Local HashMap / Caffeine cache] ←→ [Database]
```

**Problem:** Each application server has its own cache. If you have 10 servers, the same data is cached 10 times — inconsistently. If one server updates an item and invalidates its cache, the other 9 still serve stale data.

**Use for:** Immutable or very slowly-changing reference data (country codes, configuration). Small datasets.

### Distributed Cache (External Cache)
A separate caching service (Redis, Memcached) shared by all application servers.

```
App Server 1 ──┐
App Server 2 ──┼──→ Redis Cluster ←→ Database
App Server 3 ──┘
```

**All servers share the same view of the cache.** A write on Server 1 that invalidates a cache entry is immediately reflected for Servers 2 and 3. This is the standard approach for production systems.

---

## 5. Cache-Aside — The Most Important Pattern

Also called **lazy loading**. The application manages the cache explicitly. This is the most widely used pattern.

```
Read flow:
  1. Application checks cache for key
  2a. Cache HIT  → return cached value directly
  2b. Cache MISS → query database → store result in cache → return value

Write flow:
  1. Write to database
  2. Invalidate (delete) the cache entry for that key
     → Next read will be a miss → fresh value fetched from DB
```

```
READ:                              WRITE:
App → Cache: GET user:123          App → DB: UPDATE user 123
        │                                  │
        ├── HIT → return ✅                └── Cache: DEL user:123
        │
        └── MISS → DB → data
                    │
                    └── Cache: SET user:123 = data, TTL=300s
                    └── return data
```

**Advantages:**
- Only requested data is cached (no wasted memory on never-accessed data)
- Cache failures are non-fatal — app falls back to database
- Works with any database

**Disadvantage:** First request for any key is always a cache miss (cold start). Can be mitigated with cache warming on startup.

---

## 6. Write-Through — Keeping Cache Fresh

Every write goes to both the cache and the database simultaneously. Reads are always served from cache because it's always up-to-date.

```
Write flow:
  1. Application writes to Cache
  2. Cache synchronously writes to Database
  3. Both updated → respond to application

Read flow:
  1. Application reads from Cache
  2. Always a HIT (cache always has the latest data)
```

**Advantage:** Cache is always fresh. No stale reads.

**Disadvantage:** Write latency increases (must write two places). Every write pollutes the cache with data that may never be read.

**Best for:** Read-heavy workloads where you can tolerate slightly higher write latency and want zero stale reads.

---

## 7. TTL — Time To Live

Every cache entry has a TTL — a time after which it expires and is deleted. On expiry, the next request is a miss and fetches fresh data from the database.

```
Cache: SET product:456 = {...} TTL=3600   (expires in 1 hour)

Requests within 1 hour → cache hit → fast
After 1 hour → cache miss → fresh DB read → re-cached
```

**Short TTL (30–60s):** Data stays fresh; more DB traffic; less stale risk.
**Long TTL (hours–days):** Less DB traffic; risk of serving stale data longer.

> **TTL is your backstop.** Even if explicit invalidation fails, TTL ensures stale data doesn't live forever. Always set a TTL — a cache entry with no TTL is a memory leak waiting to happen.

---

## 8. Cache Invalidation — The Hard Problem

> *"There are only two hard things in computer science: cache invalidation and naming things."* — Phil Karlton

When data in the database changes, the corresponding cache entry must be removed or updated. Getting this wrong means users see stale data.

**Invalidation strategies:**

| Strategy | How | Best For |
|---|---|---|
| **TTL expiry** | Entry expires automatically after N seconds | Tolerable staleness; simple implementation |
| **Write-through invalidation** | Invalidate/update cache on every DB write | When staleness is not tolerable |
| **Event-driven** | DB change event triggers cache invalidation | Complex systems; eventual consistency acceptable |

**The race condition problem:**

```
T1: App reads DB → gets value V1
T2: Another process updates DB to V2, deletes cache entry
T3: App writes V1 to cache ← STALE! V1 just overwrote the (now correct) empty cache
```

This race is rare but real. Solutions: use short TTLs to cap the damage window; use atomic compare-and-set when populating cache.

---

## 9. Cache Eviction — When the Cache Is Full

When the cache reaches its memory limit, it must evict (remove) entries to make room for new ones. The eviction policy determines which entries to remove.

| Policy | Evicts | Best For |
|---|---|---|
| **LRU (Least Recently Used)** | Entry not accessed for the longest time | General purpose — most commonly used |
| **LFU (Least Frequently Used)** | Entry accessed fewest times | When access frequency matters more than recency |
| **TTL-based** | Expired entries first | When freshness is the primary concern |
| **FIFO** | Oldest entry regardless of access | Simple; rarely optimal |

> **LRU is the default for most production caches.** It naturally evicts "cold" data (not accessed recently) and keeps "hot" data in memory.

> **For the full deep dive** — Redis data structures, cache stampede, Thundering Herd problem, distributed cache architecture, cache warming strategies — see the **`04-Caching`** section of this handbook.

---

## 10. How Large Companies Use Caching with Databases

| Company | Application | Source |
|---|---|---|
| **Facebook** | Memcached as the primary caching layer in front of MySQL; billions of cached objects | Facebook Eng Blog (public) |
| **Twitter/X** | Redis for timeline caching; avoids DB reads for most feed requests | Public talks |
| **Stack Overflow** | Aggressive in-process caching; serves millions of requests with relatively few DB servers | Stack Overflow Blog (public) |
| **Netflix** | EVCache (Memcached-based) for personalisation and metadata | Netflix Tech Blog (public) |

---

## 11. Best Practices

- **Cache-aside is the safest default** — simple, failure-tolerant, flexible.
- **Always set a TTL** — prevents stale data from living forever; prevents memory leaks.
- **Invalidate on write** — delete the cache entry when the underlying data changes.
- **Cache at the right granularity** — cache the final response when possible, not intermediate objects.
- **Use a distributed cache** (Redis) for multi-server deployments.
- **Don't cache the database — cache the result** — cache what the user needs, not raw DB rows.

---

## 12. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No TTL on cache entries | Stale data served indefinitely; memory grows unbounded | Always set TTL |
| Caching user-specific private data in a shared key | One user's data served to another | Include user ID in cache key |
| Not invalidating on write | Stale reads persist until TTL | Explicitly invalidate on every write |
| Caching failures (caching error responses) | Bad data spreads; normal requests get error responses | Never cache error responses |
| Stampede on cold start | All servers miss at once → DB overwhelmed | Cache warming; probabilistic early expiration |

---

## 13. Interview Questions

1. What is a cache and why does it exist in front of a database?
2. Explain the cache-aside pattern. What is a cache miss?
3. What is the difference between cache-aside and write-through?
4. What is a TTL and why must every cache entry have one?
5. What is cache invalidation? Why is it considered hard?
6. What is LRU eviction? When does eviction happen?
7. Why is a distributed cache (Redis) preferred over in-process caches for multi-server systems?

---

## 14. Summary

| Concept | Key Takeaway |
|---|---|
| **Why cache** | Memory is ~100,000× faster than disk. Absorb reads before they hit DB. |
| **What to cache** | Frequently-read, rarely-changed, non-sensitive, non-transactional data. |
| **Cache-aside** | Check cache → hit returns; miss fetches DB and populates cache. Default pattern. |
| **Write-through** | Writes update cache and DB together. No stale reads. Slower writes. |
| **TTL** | Expiry time. Always set one. The backstop against stale data. |
| **Invalidation** | Delete cache entry on DB write. The hard part. |
| **LRU eviction** | Evict least-recently-used when memory is full. Default policy. |
| **Deep dive** | Full caching coverage in `04-Caching` section. |

---

## 15. Cross References

**Prerequisites:** Database Fundamentals (DB #1) · Latency & Throughput (NFR #1) · Scalability (NFR #3)

**Related Topics:** Full caching deep dive → `04-Caching` section · Replication (DB #5)

**What to Learn Next:** Database Selection (DB #9) · `04-Caching` section

---

*System Design Engineering Handbook — Databases Series*