# 04 — Trade-off Vocabulary

> **11-Interview Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Why Trade-off Language Matters

The ability to articulate trade-offs clearly is the single biggest differentiator between candidates who pass senior system design interviews and those who don't. Interviewers are not looking for perfect answers — they are looking for engineers who reason clearly about imperfect choices.

Every design decision in a real system sacrifices something. The question is never "what is the best solution?" — it is always "what are we trading, and why is that trade acceptable given our constraints?"

This document gives you the vocabulary, frameworks, and sentence patterns to communicate trade-offs precisely and confidently.

---

## 2. The Core Trade-off Pairs

These are the fundamental tensions in system design. Every major architectural decision maps to one or more of these pairs.

### Consistency vs Availability

```
Choosing strong consistency:
  "I'm choosing strong consistency here because [this data] cannot be stale.
   A stale [balance/inventory/booking] directly causes [financial loss/oversell/
   double-booking]. The cost is that during a partition, the system will
   refuse requests rather than serve potentially wrong data — brief
   unavailability is preferable to incorrect results."

Choosing availability / eventual consistency:
  "I'm choosing eventual consistency here because [this data] — a feed post
   or a like count — can tolerate being slightly stale. The user experience
   cost of seeing a post 500ms late is negligible. The benefit is that the
   system always responds, and writes are faster because we don't need to
   wait for quorum acknowledgement."
```

### Latency vs Throughput

```
Optimising for latency:
  "I'm optimising for latency on this path because the user is waiting
   for a response. Every millisecond of added latency increases drop-off.
   The cost is lower throughput — we process each request immediately
   rather than batching."

Optimising for throughput:
  "I'm optimising for throughput on this pipeline because it's a background
   job that processes analytics events. The user doesn't see the result in
   real time. Batching 1,000 events together before writing to the database
   reduces the per-event overhead by 1,000× at the cost of up to 5 seconds
   of additional latency per event."
```

### Read Performance vs Write Performance

```
Optimising reads:
  "I'm adding this cache and denormalising this data to speed up reads.
   The cost is write amplification — every write must now update the primary
   store and invalidate or update the cache — and the risk of serving
   slightly stale data. Given the 99:1 read/write ratio we estimated,
   this trade-off strongly favours the read optimisation."

Optimising writes:
  "I'm removing secondary indexes from this table because at 200K writes/sec,
   index maintenance is becoming the bottleneck. The cost is slower reads —
   queries that relied on those indexes now require full table scans. Given
   that this is a write-heavy event log that is rarely queried directly,
   that's an acceptable cost."
```

### Simplicity vs Flexibility

```
Choosing simplicity:
  "I'm choosing a monolith over microservices here because the team is
   small, the domain boundaries aren't fully understood yet, and the
   scale doesn't justify the operational overhead. The cost is that
   scaling specific components independently is harder. We can extract
   services later when we have evidence that specific components need
   independent scaling."

Choosing flexibility:
  "I'm separating these two services even though it adds complexity
   because they have genuinely different scaling characteristics — one is
   read-heavy, one is write-heavy — and different teams own them. The
   coordination overhead is justified by the deployment independence."
```

### Strong Durability vs Performance

```
Choosing durability:
  "I'm using synchronous replication here with acks=all in Kafka.
   This means every write waits for all replicas to confirm before
   acknowledging. The cost is additional write latency (~5-10ms per
   network round trip to replicas). For payment events, data loss is
   not acceptable — the latency cost is worth it."

Choosing performance:
  "I'm using async replication here with acks=1. The leader acknowledges
   as soon as it writes locally — replicas catch up asynchronously.
   The risk is that if the leader fails before replication completes,
   we lose those writes. For analytics events, losing a few view counts
   in a crash scenario is acceptable — the write throughput improvement
   is significant."
```

---

## 3. The Decision Sentence Patterns

These sentence patterns give you a consistent structure for expressing trade-offs. Use them until the structure becomes natural.

### Pattern 1 — The "Because / At the cost of" Pattern

```
"I'm choosing [X] because [benefit]. At the cost of [trade-off].
 Given [constraint], this trade-off is acceptable/preferable because [reason]."

Example:
  "I'm choosing Cassandra for message storage because it's write-optimised
   and the wide-column model maps perfectly to our access pattern of fetching
   messages by channel ordered by timestamp. At the cost of limited query
   flexibility — we can't do arbitrary joins. Given that our only access
   pattern is 'get last N messages for channel X', this trade-off is
   acceptable — we'll never need those joins."
```

### Pattern 2 — The "Approach A vs Approach B" Pattern

```
"There are two approaches here. Approach A gives us [benefit] but [cost].
 Approach B gives us [benefit] but [cost]. Given [specific constraint
 from our requirements], I'd choose [X] because [reason]."

Example:
  "For the URL redirect, there are two approaches. Approach A is to
   look up the URL in the database on every request — simple, always
   fresh, but the database becomes a bottleneck at 100K reads/sec.
   Approach B is to cache the URL mapping in Redis — sub-millisecond
   reads, scales horizontally, but introduces stale data risk if a
   URL is updated. Given that URL mappings are effectively immutable
   — they're rarely changed after creation — I'd choose caching with
   a 24-hour TTL. The stale data risk is negligible for this use case."
```

### Pattern 3 — The "Right now / Later" Pattern

```
"Right now, [simpler approach] is sufficient because [current scale/state].
 As we grow and [trigger condition], we would migrate to [more complex approach]
 because [reason]. I'd design the [component] with that migration in mind by [X]."

Example:
  "Right now, a single PostgreSQL primary with read replicas is sufficient
   because we estimated 5,000 writes/sec — well within a single node's
   capacity. As we grow beyond 50,000 writes/sec, we would migrate to
   sharding by user_id because a single primary would saturate. I'd
   design the user_id as our primary key and keep all user data together
   from day one, so the sharding migration changes only the routing
   layer, not the data model."
```

---

## 4. The Core Trade-off Vocabulary

Words and phrases that signal sophisticated trade-off reasoning to interviewers.

### For Consistency

```
"strongly consistent"        — every read sees the latest write
"eventually consistent"      — replicas converge; brief staleness possible
"read-your-own-writes"       — you see your own changes; others may not
"linearisable"               — strongest guarantee; atomic operations
"causal consistency"         — related events ordered; unrelated may differ
"stale reads"                — reading data that isn't the most recent value
"replication lag"            — the delay before replicas catch up
"quorum read/write"          — R+W>N ensures overlap; tunable consistency
```

### For Availability

```
"high availability (HA)"           — designed to survive component failures
"single point of failure (SPOF)"   — one component whose failure = outage
"active-passive failover"          — standby takes over on primary failure
"active-active"                    — multiple instances serve simultaneously
"graceful degradation"             — serve reduced functionality vs full failure
"circuit breaker"                  — fail fast when downstream is broken
"error budget"                     — quantified allowable unreliability
"nines" (99.9%, 99.99%)            — availability expressed in nines
```

### For Scale

```
"horizontal scaling"         — add more machines
"vertical scaling"           — bigger machine
"stateless"                  — no server-side session state; freely scalable
"shard key"                  — attribute that determines data partition
"hot partition"              — one shard getting disproportionate load
"fan-out"                    — one event delivered to many consumers
"backpressure"               — signal upstream to slow down when overwhelmed
"competing consumers"        — multiple workers pull from shared queue
"cache hit rate"             — % of reads served from cache vs database
```

### For Performance

```
"P99 latency"                — 99th percentile; the tail experience
"tail latency amplification" — fan-out makes P99 the common case
"read/write ratio"           — proportion of reads to writes
"working set"                — data actively accessed; should fit in cache
"hot path"                   — the critical request path that must be fast
"cold path"                  — background processing; latency-insensitive
"batching"                   — group operations for throughput at cost of latency
"amortise"                   — spread fixed costs across many operations
```

### For Data

```
"normalised"                 — each fact stored once; good for writes
"denormalised"               — data duplicated for read performance
"polyglot persistence"       — different databases for different use cases
"schema-on-read"             — structure applied when reading (NoSQL)
"schema-on-write"            — structure enforced at write time (SQL)
"idempotent"                 — safe to apply multiple times; same result
"at-least-once"              — delivery guaranteed; duplicates possible
"exactly-once"               — no loss, no duplicates; hardest guarantee
"write amplification"        — one logical write causes multiple physical writes
```

---

## 5. Responding to Common Interview Challenges

### "Why not just use X?"

```
Interviewer: "Why not just use a relational database here?"

Good response:
  "A relational database would work at this scale, and if we were at
   10K writes/sec I'd probably start there. But we estimated 400K writes/sec
   at peak. A single PostgreSQL primary saturates at around 10K TPS under
   heavy write load. We could shard it, but then we lose the joins and
   transactions that make relational databases valuable. Given that our
   access pattern is purely 'insert event, read by partition key ordered
   by time', Cassandra's write-optimised LSM storage and native partitioning
   is a better fit at this scale."
```

### "What if the cache goes down?"

```
Interviewer: "What happens if Redis goes down?"

Good response:
  "If Redis goes down, all cache reads become misses — 100% of traffic
   hits the database. At our read volume, that would likely overwhelm
   the database. So we need to design for this. First, Redis itself
   should run in cluster mode with replicas — this prevents single
   node failure. Second, the application should have a circuit breaker
   so a slow Redis doesn't block requests — fall through to the database.
   Third, we'd want Redis persistence (AOF) so cache survives restarts
   with data intact. Even with all of this, a full cache failure is a
   'degrade gracefully' scenario: the system works, just slower."
```

### "How does this scale to 10× the load?"

```
Interviewer: "How would this design handle 10× the traffic?"

Good response:
  "Good question. Let me trace through each layer.
   
   The application tier: stateless services, so we just add instances
   behind the load balancer. No architectural change.
   
   The cache: Redis Cluster handles horizontal scaling — add nodes,
   consistent hashing redistributes the key space. Minimal disruption.
   
   The database: this is where it gets interesting. At 10× our write
   estimate, we're at 2 million writes/sec — beyond what even Cassandra
   handles comfortably on a single cluster without significant node count.
   We'd need to shard by a secondary dimension or increase the cluster
   size significantly. I'd also consider offloading writes through a
   write-behind cache to batch database writes.
   
   The bottleneck at 10× is definitely the database write path."
```

---

## 6. The Trade-off Matrix — Quick Reference

When you need to justify a choice quickly, use this mental matrix.

| Situation | Choose | Trade-off You Accept |
|---|---|---|
| Financial / inventory data | Strong consistency | Some availability loss during partition |
| Social feed / recommendations | Eventual consistency | Briefly stale reads |
| 99:1 read/write ratio | Aggressive caching | Stale data risk; invalidation complexity |
| Very high write throughput | NoSQL (Cassandra) | Limited query flexibility |
| Complex queries / joins | Relational DB | Write throughput ceiling |
| Highly variable traffic | Async queues | Processing delay; added complexity |
| Need immediate response | Synchronous calls | Latency compounds; cascading failures |
| Background processing | Async + queue | Eventually processed; not real-time |
| Small team, new product | Modular monolith | Scaling individual components is harder |
| Large teams, clear domains | Microservices | Operational complexity; distributed transactions |

---

## 7. Phrases That Signal Maturity

Use these naturally, not performatively. They tell interviewers you've thought about these problems before.

```
"The hard problem here is..."
"This is where the interesting trade-off appears..."
"I'm making an assumption that [X] — is that reasonable?"
"The risk I'm accepting with this choice is..."
"If our requirements change to [Y], I'd revisit this decision because..."
"The failure mode I'm most concerned about is..."
"This works well until [scale threshold], at which point we'd need to..."
"I'd want to validate this with load testing before committing to it."
"The simpler approach is [A], and I'd start there — moving to [B] only
 when we have evidence that [specific problem] has actually appeared."
```

---

## 8. Summary

Trade-off reasoning is not a skill you develop by studying — it's a habit you develop by practicing. For every case study in this handbook:

1. Identify the two or three decisions with the most significant trade-offs
2. Write out the trade-off using the "Because / At the cost of" pattern
3. Practice saying it aloud until it sounds natural and confident

The goal is not to have memorised trade-offs. The goal is to have internalised the habit of thinking "what am I gaining, what am I giving up, and why is that acceptable here?" for every decision you make.

---

## 9. Cross References

**Read First:** 01-how-to-approach-system-design.md · 03-common-mistakes.md

**Apply in:** Every case study in 10-Case-Studies

---

*System Design Engineering Handbook — 11-Interview Series*