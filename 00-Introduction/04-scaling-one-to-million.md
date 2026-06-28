# 04 — Scaling from One to One Million Users

> **00-Introduction** — Engineering Handbook
> Language-agnostic · 8 min read

---

## 1. Why This Matters

Every large system started small. Twitter was a single Rails app. Instagram launched with one server. Uber began as a simple dispatch system.

The mistake engineers make is designing for a million users on day one. The other mistake is not understanding *how* to scale when the time comes.

This document walks through the exact progression — from a single server handling your first user to an architecture serving a million — and explains *why* each change is made at each stage. Every scaling decision is a reaction to a specific bottleneck. Understanding what breaks first at each stage is the foundation of scaling intuition.

> **The golden rule of scaling: never scale what isn't broken. Find the bottleneck, fix it, find the next one.**

---

## 2. Stage 0 — One Server, Everything Together

You have your first user. The simplest possible setup:

```
┌─────────────────────────────────────┐
│           Single Server             │
│                                     │
│  ┌──────────┐    ┌───────────────┐  │
│  │ Web App  │───▶│   Database    │  │
│  │ (code)   │    │  (same box)   │  │
│  └──────────┘    └───────────────┘  │
└─────────────────────────────────────┘
         ▲
         │
      Client
```

Everything — the web server, application code, and database — runs on a single machine.

**What works:** Everything is simple. One deployment. One log file. One server to SSH into.

**What breaks:** Everything goes down together. The database competes with the app for CPU and memory. You cannot scale the app without also scaling the database (and vice versa). One spike in traffic or one runaway query takes the entire system down.

**When to move on:** When you feel any pain at all — slow queries, high memory usage, traffic spikes causing timeouts.

---

## 3. Stage 1 — Separate the Database (~100 Users)

First separation: move the database to its own server. App and database no longer compete for resources.

```
                  Client
                    │
                    ▼
          ┌──────────────────┐
          │   Web / App      │
          │   Server         │
          └────────┬─────────┘
                   │  (network)
          ┌────────▼─────────┐
          │   Database       │
          │   Server         │
          └──────────────────┘
```

**What you gain:**
- App server and DB server scale independently
- Database gets dedicated memory — queries get faster
- A crash in one doesn't kill the other

**What you give up:**
- One more machine to manage
- Network latency between app and DB (usually < 1ms in same datacenter — acceptable)

**What still breaks:** Both servers are single points of failure. If either goes down, the system is down. You still can't handle a surge in traffic — the app server is a ceiling.

**When to move on:** When traffic begins to stress the single app server — slow response times, high CPU.

---

## 4. Stage 2 — Scale the App Horizontally (~1,000 Users)

Traffic is growing. One app server isn't enough. Add more app servers and a load balancer to distribute requests across them.

```
               Client
                 │
                 ▼
       ┌──────────────────┐
       │   Load Balancer  │
       └───┬──────────┬───┘
           │          │
  ┌────────▼───┐  ┌───▼────────┐
  │ App Server │  │ App Server │  ← can add more
  │     1      │  │     2      │
  └────────┬───┘  └───┬────────┘
           │          │
           └────┬─────┘
                ▼
       ┌──────────────────┐
       │    Database      │
       └──────────────────┘
```

**What you gain:**
- Traffic distributed — no single app server is a ceiling
- One app server can go down without the whole system going down
- Horizontal scaling is now possible — add more servers when needed

**The critical constraint this introduces:** App servers must be **stateless**. If Server 1 stores a user's session in local memory, and the next request goes to Server 2, the session is lost.

```
STATEFUL (broken with multiple servers):
  User logs in → session stored in Server 1's memory
  Next request routed to Server 2 → "Who are you? Log in again."

STATELESS (correct):
  User logs in → session stored in Redis (shared external store)
  Any server can handle any request → reads session from Redis
```

**Rule: App servers must store zero state.** All state lives in the database, cache, or session store.

**What still breaks:** The database is now the single bottleneck. All reads and writes hit one DB server. As read traffic grows, the DB becomes saturated.

---

## 5. Stage 3 — Add a Cache (~10,000 Users)

The database is under pressure. Most reads are for the same popular data — user profiles, product details, configuration. Fetching them from disk on every request is wasteful.

Add a cache layer (Redis or Memcached) between the app servers and the database.

```
               Client
                 │
                 ▼
       ┌──────────────────┐
       │   Load Balancer  │
       └───┬──────────┬───┘
           │          │
  ┌────────▼───┐  ┌───▼────────┐
  │ App Server │  │ App Server │
  └────────┬───┘  └───┬────────┘
           │          │
           └────┬─────┘
                │
       ┌────────▼─────────┐
       │   Redis Cache    │  ← check here first
       └────────┬─────────┘
                │ (cache miss)
       ┌────────▼─────────┐
       │    Database      │  ← only if cache misses
       └──────────────────┘
```

**Read path with cache (cache-aside pattern):**

```
1. App checks Redis: "Do you have user:123?"
2a. Cache HIT  → Redis returns data in ~0.5ms → done
2b. Cache MISS → App queries DB in ~10ms
                  → App stores result in Redis (TTL = 1 hour)
                  → Returns result to client
```

**What you gain:**
- Hot data served from memory in under 1ms
- DB reads dramatically reduced — 95%+ cache hit rate for popular data
- DB can now handle more write traffic (reads offloaded)

**What you give up:**
- Stale data risk — cache may serve old data until TTL expires
- Cache invalidation complexity — when data changes, cache must be updated or invalidated
- One more infrastructure component to manage and monitor

**What still breaks:** All writes still go to one database. Write-heavy workloads will saturate the DB even with a cache. A single DB is also still a single point of failure.

---

## 6. Stage 4 — Read Replicas (~50,000 Users)

Reads are handled by the cache but some queries can't be cached — reports, search, anything that needs fresh data. And if the DB goes down, the system goes down.

Add **read replicas** — copies of the primary database that handle read traffic.

```
                 App Servers
                 │        │
         writes  │        │ reads
                 ▼        ▼
       ┌──────────────┐  ┌────────────────┐
       │   Primary DB  │  │  Redis Cache   │
       │   (writes)    │  │                │
       └──────┬────────┘  └────────────────┘
              │ replication     (cache miss)
       ┌──────▼────────┐              │
       │   Replica 1   │◄─────────────┘
       │   (reads)     │
       └───────────────┘
       ┌───────────────┐
       │   Replica 2   │
       │   (reads)     │
       └───────────────┘
```

**Write path:** Always goes to the primary.
**Read path:** Goes to Redis first, then to a read replica if cache misses.

**What you gain:**
- Reads distributed across multiple servers — read capacity scales horizontally
- If primary fails, a replica can be promoted (reduced downtime)
- Reporting and analytical queries can run on replicas without impacting production writes

**What you give up:**
- **Replication lag** — replicas are slightly behind the primary (milliseconds to seconds). A write may not be immediately visible on a read from a replica.
- More operational complexity — you must route reads and writes to different servers

**The replication lag trade-off:** For most reads (viewing a social feed, browsing products), seeing data that is 500ms old is completely acceptable. For reads that must see the very latest write (payment confirmation, inventory check), read from the primary.

**What still breaks:** The primary database is still a write bottleneck. At very high write volumes, a single primary saturates. We haven't addressed this yet.

---

## 7. Stage 5 — Add a CDN (~100,000 Users)

Static assets — images, CSS, JavaScript, videos — are being served from your app servers. Each request travels to your data center, gets the file, and returns. This is slow for users who are geographically far from your servers, and it consumes server bandwidth for content that never changes.

Add a CDN. Static assets are served from edge nodes physically close to users.

```
User in Tokyo                 User in London
       │                             │
       ▼                             ▼
┌─────────────┐             ┌─────────────┐
│  CDN Edge   │             │  CDN Edge   │
│  Tokyo      │             │  London     │
└──────┬──────┘             └──────┬──────┘
       │ cache miss                │ cache miss
       └──────────────┬────────────┘
                      ▼
             ┌─────────────────┐
             │   Your Origin   │
             │   (data center) │
             └─────────────────┘
```

**What goes on the CDN:** Everything that doesn't change per-user — images, videos, JavaScript bundles, CSS, HTML templates, fonts.

**What stays on your servers:** Anything personalised or dynamic — user feed, account data, search results.

**What you gain:**
- Global users get fast asset delivery (< 20ms from edge vs 200ms from origin)
- Origin server bandwidth dramatically reduced
- DDoS mitigation at the edge — volumetric attacks absorbed before reaching your infrastructure

**What you give up:**
- Cache invalidation complexity — when you deploy new JS/CSS, CDN must be told to clear stale cached versions (cache busting with content hashes)
- Cost — CDN bandwidth is paid per GB

---

## 8. Stage 6 — Shard the Database (~500,000 Users)

Write volume has grown beyond what a single primary database can handle. At this point, you need to split the data across multiple database servers — **sharding**.

**Sharding** partitions data across multiple database instances so that each instance holds a subset of the data and handles a fraction of the total write and read load.

```
App Servers
     │
     ▼
┌───────────────────┐
│   Shard Router    │  ← routes to correct shard based on key
└─┬───────────┬─────┘
  │           │
  ▼           ▼
┌──────┐   ┌──────┐
│Shard │   │Shard │   ← can add more shards
│  A   │   │  B   │
│user  │   │user  │
│0-4M  │   │4M-8M │
└──────┘   └──────┘
```

**Choosing a shard key:**

```
The shard key determines which shard stores a given record.

Good shard key: user_id
  → All data for one user on one shard
  → Operations that touch one user = single-shard query (fast)
  → Hotspot risk: if one user has vastly more data than others

Bad shard key: created_at (timestamp)
  → All new data goes to the latest shard (hotspot)
  → One shard gets all writes; others are idle

Good distribution: hash(user_id) % num_shards
  → Even spread across shards
  → But: a user's data could cross shards on a reshard (use consistent hashing)
```

**What you gain:**
- Write capacity scales horizontally — add shards to add write throughput
- Each shard holds less data — queries faster

**What you give up:**
- Cross-shard queries are expensive (or impossible) — "find all users who posted in the last hour" requires querying every shard and merging results
- Resharding (adding shards) requires migrating data — complex and risky
- Transactions across shards require distributed transaction protocols (2PC)

**The trade-off is real:** Sharding is powerful but costly. Delay it as long as possible. Read replicas, caching, and query optimisation can often push the sharding decision much further than you'd expect.

---

## 9. Stage 7 — Decouple with Message Queues (~1,000,000 Users)

At scale, some operations don't need to happen synchronously in the user's request path. Sending a welcome email, generating a thumbnail, updating a search index, notifying followers — none of these need to complete before the user sees a response.

Doing them synchronously makes the request slow and ties the reliability of the main flow to the reliability of every downstream service.

**Decouple with a message queue (Kafka, SQS, RabbitMQ):**

```
Without queue (synchronous, coupled):
  User uploads photo
  → App saves photo to DB
  → App calls Thumbnail Service (waits)
  → App calls Notification Service (waits)
  → App updates Search Index (waits)
  → Response to user
  Total: 2,000ms

With queue (async, decoupled):
  User uploads photo
  → App saves photo to DB
  → App publishes "photo_uploaded" event to queue
  → Response to user (instant)
  Total: 50ms

  (In background, independently)
  Queue → Thumbnail Service processes event
  Queue → Notification Service processes event
  Queue → Search Index updates
```

**What you gain:**
- User gets a faster response — only the critical work is synchronous
- Downstream services can fail without affecting the main flow (events queue up and are processed when the service recovers)
- Each service scales independently based on its own queue depth
- Natural rate limiting — services consume from the queue at their own pace

**What you give up:**
- Work is done asynchronously — there's a delay between upload and thumbnail appearing
- More complex debugging — tracing a request across queues and workers requires distributed tracing
- At-least-once delivery means consumers must handle duplicate events

---

## 10. The Full Picture at One Million Users

```
                         Users (global)
                              │
                    ┌─────────▼──────────┐
                    │     CDN / Edge     │ ← static assets, popular responses
                    └─────────┬──────────┘
                              │
                    ┌─────────▼──────────┐
                    │   Load Balancer    │
                    └──┬──────────────┬──┘
                       │              │
              ┌────────▼──┐    ┌──────▼─────┐
              │App Server │    │ App Server │  ... (autoscale)
              │  (stateless)   │ (stateless)│
              └────────┬──┘    └──────┬─────┘
                       │              │
              ┌────────▼──────────────▼────┐
              │          Redis             │ ← sessions + hot cache
              └────────────────────────────┘
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
     ┌─────────┐  ┌─────────┐  ┌─────────┐
     │ Shard A │  │ Shard B │  │ Shard C │ ← sharded primary DBs
     └────┬────┘  └────┬────┘  └────┬────┘
          │            │            │
     ┌────▼────┐  ┌────▼────┐  ┌────▼────┐
     │Replica  │  │Replica  │  │Replica  │ ← read replicas per shard
     └─────────┘  └─────────┘  └─────────┘
                       │
              ┌────────▼──────────┐
              │   Message Queue   │ ← async processing
              │   (Kafka / SQS)   │
              └──┬────────────┬───┘
                 │            │
        ┌────────▼───┐  ┌─────▼──────┐
        │  Thumbnail │  │Notification│  ... (worker services)
        │  Service   │  │  Service   │
        └────────────┘  └────────────┘
```

---

## 11. The Scaling Stages at a Glance

| Stage | Scale | Change | Bottleneck Solved |
|---|---|---|---|
| **0** | 1 user | Everything on one server | Starting point |
| **1** | ~100 users | Separate DB server | App and DB competing for resources |
| **2** | ~1,000 users | Multiple app servers + load balancer | Single app server ceiling |
| **3** | ~10,000 users | Cache (Redis) | DB read saturation |
| **4** | ~50,000 users | Read replicas | DB read capacity + single point of failure |
| **5** | ~100,000 users | CDN | Static asset delivery latency + origin bandwidth |
| **6** | ~500,000 users | Database sharding | DB write saturation |
| **7** | ~1,000,000 users | Message queues | Synchronous coupling + request path latency |

---

## 12. Principles Extracted

**1. Separate concerns early.** App server and DB server should be separate as soon as you feel any pain.

**2. Make app servers stateless.** This is the prerequisite for horizontal scaling. Without it, you cannot add servers freely.

**3. Cache reads aggressively.** Memory is 10,000× faster than disk. Most reads are for the same small set of hot data. Cache it.

**4. Scale reads before writes.** Read replicas are simpler than sharding. Exhaust the read scaling options before introducing write sharding.

**5. Push work off the critical path.** If the user doesn't need the result immediately, use a queue. The request is faster and the system more resilient.

**6. Scale in response to bottlenecks, not in anticipation of them.** Each stage introduces real complexity. Only accept that complexity when the bottleneck is real and measurable.

**7. There is no final architecture.** A million users is not the end. The same progression continues — to 10 million, 100 million, a billion. The principles remain the same. The components multiply.

---

## 13. Cross References

**Prerequisites:** `01-system-design-introduction.md` · `02-functional-and-non-functional-requirements.md`

**Deep dives on each component introduced:**
- Load Balancers → `02-Building-Blocks/01-load-balancers.md`
- Caching → `04-Caching/01-caching-fundamentals.md`
- Database Replication → `03-Databases/05-replication.md`
- Sharding → `03-Databases/06-partitioning-sharding.md`
- CDN → `02-Building-Blocks/04-cdn.md`
- Message Queues → `05-Messaging/02-message-queues.md`

**See this in practice:** Every case study in `10-Case-Studies/` shows a fully scaled version of a real system.

---

*System Design Engineering Handbook — 00-Introduction*