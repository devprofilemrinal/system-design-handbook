# 03 — Eviction Policies

> **04-Caching Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Eviction?

A cache lives in memory. Memory is finite. When the cache reaches its size limit and a new entry needs to be stored, the cache must remove an existing entry to make room. This process is **eviction**.

The **eviction policy** determines *which entry to remove*. The goal is always the same: evict the entry whose removal will cause the fewest future cache misses — in other words, remove the data you're least likely to need again.

```
Cache is full (capacity: 1000 entries)

New entry arrives → must evict one to make room

Which one?

Policy A: Remove the one that was accessed longest ago (LRU)
Policy B: Remove the one accessed least often (LFU)
Policy C: Remove the oldest entry regardless of access (FIFO)
Policy D: Remove an entry at random (Random)
Policy E: Remove the entry that expires soonest (TTL-based)
```

> **The eviction policy is one of the most consequential cache configuration decisions.** The wrong policy evicts hot data, increasing miss rates and crushing performance.

---

## 2. LRU — Least Recently Used

The most widely used eviction policy. When the cache is full, remove the entry that was accessed least recently — the "coldest" entry.

### The Intuition

If you haven't accessed something in a while, you probably won't access it soon. Things you've used recently, you're likely to use again soon. This is called **temporal locality** — a fundamental pattern in real-world access behaviour.

```
Cache capacity: 4 entries
Current state: [A, B, C, D]  (A = most recently used, D = least recently used)

New entry E arrives:
  → D is least recently used → evict D
  → Cache: [E, A, B, C]

Request for B arrives:
  → B moves to front (most recently used)
  → Cache: [B, E, A, C]
```

### How LRU Is Implemented

Classic LRU uses a **doubly linked list** + **hash map**:
- Hash map: O(1) lookup by key
- Linked list: maintains access order (head = most recent, tail = least recent)
- On access: move node to head (O(1) via linked list pointer manipulation)
- On eviction: remove tail node (O(1))

```
Hash map: {A → nodeA, B → nodeB, C → nodeC, D → nodeD}
Linked list: HEAD → [A] ↔ [B] ↔ [C] ↔ [D] ← TAIL (evict from here)

Access B:
  Hash map lookup → nodeB
  Move nodeB to HEAD
  Linked list: HEAD → [B] ↔ [A] ↔ [C] ↔ [D] ← TAIL
```

### When LRU Falls Short

**The "one-time scan" problem:** A large scan query (e.g. "export all users") accesses millions of rows once, flooding the cache with entries that will never be accessed again — evicting genuinely hot data.

**Solution:** Some systems use a variant called **LRU-K** (only promote to "hot" zone after K accesses) or segment the cache into hot and cold regions to protect truly hot data.

---

## 3. LFU — Least Frequently Used

Remove the entry that has been accessed the fewest times overall. The idea: if something has barely been accessed, it's probably not important.

```
Cache state:
  [A: 100 accesses, B: 50 accesses, C: 3 accesses, D: 1 access]

New entry E arrives → evict D (1 access = least frequent)
```

### When LFU Is Better Than LRU

If your access pattern is popularity-based (some items are always hot, most items are accessed once and never again), LFU outperforms LRU.

```
News article site:
  Top 10 articles: accessed millions of times (always hot)
  Long-tail articles: accessed once, never again

LRU: long-tail articles evict top articles because a recent scan just hit them
LFU: long-tail articles (1 access) evict before top articles (millions)
→ LFU wins here
```

### LFU's Weakness — The "Cache Pollution" Problem

Old items that were popular long ago accumulate high frequency counts. New but hot items start with frequency=1 and get evicted immediately — even if they're becoming the new hot data.

```
Old item X: accessed 10,000 times last month, rarely touched now
New item Y: just published, accessed 5,000 times today
LFU evicts Y (count=5000) before X (count=10000) → wrong choice
```

**Solution:** Use **time-decayed LFU** — access counts decay over time, so old counts fade. This is more complex but avoids the stale-popularity problem.

---

## 4. FIFO — First In, First Out

Remove the entry that has been in the cache the longest — regardless of how many times it's been accessed.

```
Cache state (in order of insertion):
  [A (inserted 10min ago), B (9min ago), C (5min ago), D (1min ago)]

New entry E arrives → evict A (oldest insertion)
```

### When FIFO Is Appropriate

FIFO works when you need strict ordering guarantees or when all cached items have similar expected lifetimes. It's simple, predictable, and fair.

**Problem:** FIFO ignores actual usage. A very popular item (A) might be evicted simply because it was inserted first — even if it's accessed constantly. This leads to poor hit rates for access patterns where some items are consistently more popular.

> **FIFO is rarely the right choice for general caching.** It's useful for queues and log buffers, not for caches where access frequency matters.

---

## 5. Random Replacement

When the cache is full, evict a randomly selected entry.

Surprisingly effective in practice — at scale, random eviction approximates LRU in many workloads, at much lower implementation complexity. No bookkeeping required.

**Used by:** Some CPU caches, simple in-memory caches, situations where implementation simplicity matters.

**Not appropriate when:** Your data has strong hotspots — random eviction will occasionally remove a very hot entry.

---

## 6. TTL-Based Eviction

Entries are evicted when their TTL (Time To Live) expires, regardless of access recency or frequency.

This is not a standalone eviction policy — it's a **correctness mechanism** that runs in parallel with whichever eviction policy you choose. Expired entries are removed first; if more room is needed, the primary policy (LRU, LFU, etc.) kicks in.

```
Cache state:
  [A: TTL expires in 5s, B: TTL expires in 3600s, C: expired 10s ago]

New entry arrives:
  Step 1: Remove expired entries (C is gone) → is there room? If yes, done.
  Step 2: If still full → apply LRU/LFU to remaining entries
```

> **TTL-based expiry is mandatory regardless of policy.** It ensures stale data doesn't survive indefinitely. Your eviction policy handles the memory-pressure case; TTL handles the freshness case.

---

## 7. Side-by-Side Comparison

| Policy | Evicts | Strength | Weakness | Use When |
|---|---|---|---|---|
| **LRU** | Least recently accessed | Great for temporal locality; most access patterns | Vulnerable to one-time scans polluting the cache | Default; general-purpose |
| **LFU** | Least frequently accessed | Great for popularity-based patterns | Penalises new hot items; stale popularity | Content platforms, media libraries |
| **FIFO** | Oldest insertion | Simple; predictable | Ignores access patterns; poor hit rate | Rare — mostly queues |
| **Random** | Random entry | Simple; no bookkeeping | No intelligence; occasional hot eviction | Simple systems; large, uniform datasets |
| **TTL-based** | Expired entries | Ensures freshness; mandatory correctness layer | Not a complete eviction policy alone | Always — paired with above |

---

## 8. Redis Eviction Policies in Practice

Redis is the most widely used caching tool. It provides 8 eviction policies, selectable per deployment:

| Redis Policy | Behaviour |
|---|---|
| `noeviction` | Return error when memory full; no eviction (default) |
| `allkeys-lru` | LRU across all keys |
| `volatile-lru` | LRU only on keys with TTL set |
| `allkeys-lfu` | LFU across all keys |
| `volatile-lfu` | LFU only on keys with TTL set |
| `allkeys-random` | Random across all keys |
| `volatile-random` | Random only on keys with TTL set |
| `volatile-ttl` | Evict key with the shortest remaining TTL |

**Common choices:**
- `allkeys-lru` — general-purpose cache; all keys eligible for eviction
- `allkeys-lfu` — popularity-based content; media, articles, products
- `volatile-lru` — mixed Redis usage (cache + persistent data); only TTL'd keys evicted

> **If Redis is used purely as a cache, use `allkeys-lru` or `allkeys-lfu`.** If Redis stores both persistent data and cache data, use `volatile-*` to protect the persistent keys.

---

## 9. Cache Size and Eviction Rate

Eviction rate is a health signal for your cache:

```
High eviction rate → cache is too small for the working set
                  → hot data is being evicted → miss rate rises
                  → database absorbs the missed load

Low/zero eviction → cache is large enough for the working set
                  → hot data stays in cache → high hit rate
```

**The working set** is the set of data actively accessed in a time window. If your cache is smaller than your working set, evictions are constant and hit rate is low no matter the policy.

```
Working set: 50GB of hot data
Cache size: 10GB → 80% of hot data cannot be held → constant eviction

Fix options:
  1. Increase cache size (more RAM)
  2. Reduce working set (application-level filtering of what gets cached)
  3. Accept the miss rate and ensure DB can handle it
```

---

## 10. Best Practices

- **Use LRU as the default** — it works well for the majority of real-world access patterns.
- **Use LFU for popularity-based content** (articles, products, media) where some items are always hot.
- **Always combine with TTL** — eviction handles memory pressure; TTL handles freshness.
- **Monitor eviction rate** — a rising eviction rate signals the cache is undersized.
- **In Redis, choose `allkeys-lru`** for a pure cache deployment.
- **Don't set cache size too tight** — aim for the cache to comfortably hold the working set with some headroom.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Using FIFO for a general cache | Poor hit rate; frequently accessed items evicted arbitrarily | Use LRU or LFU |
| No TTL alongside eviction policy | Stale data lives until evicted by memory pressure (could be hours) | Always set TTL |
| Cache too small for working set | High eviction rate; poor hit rate; DB overloaded | Increase cache size or narrow the working set |
| LRU when scan queries are common | One large scan evicts all hot data | Use LRU-K or segment into hot/cold regions |
| Not monitoring eviction rate | Silent degradation of cache effectiveness | Alert on rising eviction rate |

---

## 12. Interview Questions

1. What is cache eviction and why is it necessary?
2. How does LRU work? What data structures implement it efficiently?
3. What is the "one-time scan" problem with LRU?
4. When would you choose LFU over LRU?
5. What is LFU's weakness with newly popular items?
6. What does a high eviction rate indicate about your cache?
7. In Redis, which eviction policy would you choose for a pure cache deployment?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Eviction** | Remove entries when cache is full to make room for new ones |
| **LRU** | Evict least recently used. Best default. Temporal locality. |
| **LFU** | Evict least frequently used. Better for popularity patterns. |
| **FIFO** | Evict oldest insertion. Simple but ignores access patterns. |
| **TTL** | Mandatory freshness layer. Always pair with a primary policy. |
| **Eviction rate** | Health signal. High rate = cache too small for working set. |
| **Redis** | `allkeys-lru` for pure cache; `volatile-lru` for mixed use. |

---

## 14. Cross References

**Prerequisites:** 01-caching-fundamentals.md · 02-caching-strategies.md

**Related Topics:** 04-distributed-caching.md · 05-cache-invalidation.md

**What to Learn Next:** 04-distributed-caching.md

---

*System Design Engineering Handbook — 04-Caching Series*