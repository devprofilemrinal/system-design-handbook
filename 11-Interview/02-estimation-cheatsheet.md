# 02 — Estimation Cheatsheet

> **11-Interview Series** — Engineering Handbook
> Quick reference · Keep this open during practice

---

## 1. Powers of 2 — Storage Units

```
Power   | Exact Value          | Approx    | Name
--------|----------------------|-----------|----------
2^10    | 1,024                | 1 thousand | Kilobyte (KB)
2^20    | 1,048,576            | 1 million  | Megabyte (MB)
2^30    | 1,073,741,824        | 1 billion  | Gigabyte (GB)
2^40    | ~1.1 trillion        | 1 trillion | Terabyte (TB)
2^50    | ~1.1 quadrillion     | 1 thousand trillion | Petabyte (PB)
```

**In practice:**
```
1 KB  = 1,000 bytes     (use 10^3)
1 MB  = 1,000 KB        (use 10^6)
1 GB  = 1,000 MB        (use 10^9)
1 TB  = 1,000 GB        (use 10^12)
1 PB  = 1,000 TB        (use 10^15)
```

---

## 2. Time Units

```
1 minute    = 60 seconds
1 hour      = 3,600 seconds       ≈ 3.6 × 10^3
1 day       = 86,400 seconds      ≈ 10^5  (use 100,000)
1 month     = 2,592,000 seconds   ≈ 2.5 × 10^6
1 year      = 31,536,000 seconds  ≈ 3 × 10^7

Shortcut: 1 day ≈ 100,000 seconds (close enough for estimation)
```

---

## 3. Latency Reference Numbers

Memorise these orders of magnitude. They tell you why every design decision exists.

```
Operation                              Latency
──────────────────────────────────────────────────────
L1 cache reference                     ~1 ns
L2 cache reference                     ~4 ns
Main memory (RAM) reference            ~100 ns
Read 1 MB sequentially from memory     ~250 µs
SSD random read                        ~16 µs
SSD sequential read (1 MB)             ~1 ms
Network round trip (same datacenter)   ~500 µs
HDD seek + read                        ~2-10 ms
Network round trip (cross-region)      ~30-100 ms
Network round trip (cross-continent)   ~100-200 ms
```

**Key ratios to remember:**
```
RAM is ~1,000× faster than SSD
SSD is ~100× faster than HDD
Same datacenter is ~200× faster than cross-continent
Memory is ~100,000× faster than disk

→ This is why caching exists
→ This is why CDNs exist
→ This is why data locality matters
```

---

## 4. Common Data Sizes

```
Item                          Size
────────────────────────────────────────
ASCII character               1 byte
Unicode character (UTF-8)     1-4 bytes
Integer (32-bit)              4 bytes
Long integer (64-bit)         8 bytes
UUID                          16 bytes (128 bits)
Timestamp (Unix, 64-bit)      8 bytes
SHA-256 hash                  32 bytes
Average URL                   100 bytes
Short text message            100 bytes
Tweet (280 chars)             ~300 bytes with metadata
Average metadata row          ~1 KB
Thumbnail image               ~100 KB
Full HD image                 ~3 MB
1 minute of audio (MP3)       ~1 MB
1 minute of SD video          ~10 MB
1 minute of HD video          ~150 MB
1 minute of 4K video          ~600 MB
```

---

## 5. Traffic Estimation Formulas

```
Average RPS = DAU × requests_per_user_per_day / 86,400

Peak RPS = Average RPS × peak_multiplier
  (peak_multiplier is typically 2-5×; use 2× unless told otherwise)

Write RPS = DAU × writes_per_user / 86,400
Read RPS  = DAU × reads_per_user / 86,400
```

**Common DAU benchmarks (as of public figures):**
```
Small startup:          10,000 – 100,000 DAU
Mid-size product:       1 million – 10 million DAU
Large consumer app:     50 million – 200 million DAU
Tier-1 platform:        500 million – 2 billion DAU
```

---

## 6. Storage Estimation Formula

```
Daily storage = write RPS × 86,400 × record_size_bytes
5-year storage = daily × 365 × 5

With replication (factor of 3):
  Actual storage = 5-year storage × 3
```

**Example — Photo sharing:**
```
DAU = 100M
Each user uploads 2 photos/day on average
Photo size = 3 MB

Write RPS = 100M × 2 / 86,400 ≈ 2,315 photos/sec
Daily storage = 2,315 × 86,400 × 3 MB ≈ 600 TB/day
5-year storage = 600 TB × 365 × 5 ≈ 1,095 PB ≈ 1 exabyte

→ Need object storage (S3-like), not a database
→ Need tiered storage (hot/warm/cold)
→ Need aggressive compression and deduplication
```

---

## 7. Bandwidth Estimation Formula

```
Ingress bandwidth = write RPS × average_payload_size
Egress bandwidth  = read RPS × average_payload_size

Note: Egress is almost always larger than ingress
      (many readers per writer)
```

---

## 8. Worked Estimation Examples

### Example A: Chat System

```
Given: 500M DAU, 40 messages/user/day, message size = 100 bytes

Write RPS  = 500M × 40 / 100,000 = 200,000 writes/sec
             (using 100,000 ≈ seconds in a day)
Peak write = 200,000 × 2 = 400,000 writes/sec

Read RPS   = much higher (each message read by recipient(s))
             Assume 1 message → 2 readers on average
             = 400,000 reads/sec average

Daily storage = 200,000 × 100,000 × 100 bytes
              = 200,000 × 100,000 × 100
              = 2 × 10^12 bytes = 2 TB/day

5-year storage = 2 TB × 365 × 5 = 3,650 TB ≈ 3.6 PB

Conclusions:
  400K writes/sec → must use write-optimised NoSQL (Cassandra)
  3.6 PB total → need tiered storage; cold messages to object store
  High read RPS → caching for recent messages critical
```

### Example B: URL Shortener

```
Given: 100M new URLs/day, 10:1 read-to-write ratio

Write RPS = 100M / 100,000 = 1,000 writes/sec
Read RPS  = 1,000 × 10 = 10,000 reads/sec

URL record size = ~500 bytes (URL + metadata)
Daily storage   = 1,000 × 100,000 × 500 bytes = 50 GB/day
5-year storage  = 50 GB × 365 × 5 ≈ 91 TB

Conclusions:
  1,000 writes/sec → single relational DB handles this fine
  10,000 reads/sec → need caching (95% of reads are hot URLs)
  91 TB → manageable with standard relational DB + archival
```

### Example C: Video Platform

```
Given: 50M DAU, 5 videos watched/day, average 5 min = 300 MB/video

Read bandwidth = 50M × 5 × 300 MB / 86,400
              = 75,000,000,000 MB / 86,400
              ≈ 868,000 MB/sec ≈ 868 GB/sec ≈ 7 Tbps egress

Conclusions:
  7 Tbps is impossible from a single origin
  → CDN is not optional; it's the entire delivery strategy
  → Multi-CDN for this scale
  → Video must be transcoded into multiple resolutions at upload time
```

---

## 9. Capacity — Single Machine Limits

Useful for knowing when you've outgrown a single node.

```
Component              Approximate Single-Node Limit
────────────────────────────────────────────────────
Web server (RPS)       ~10,000–50,000 RPS
SQL DB (reads/sec)     ~10,000–50,000 QPS (with indexing)
SQL DB (writes/sec)    ~5,000–10,000 TPS
NoSQL (Cassandra)      ~100,000–500,000 ops/sec per node
Redis                  ~100,000–1,000,000 ops/sec
Network bandwidth      ~10 Gbps (1.25 GB/sec)
SSD throughput         ~1–3 GB/sec sequential
RAM                    ~128 GB–1 TB (high-end server)
```

> These are approximate. Use them to reason about when you need to scale, not as precise limits. "Can a single DB handle 500K writes/sec?" → No. "Can it handle 5,000?" → Yes, probably.

---

## 10. Replication Factor

When estimating storage, always multiply by your replication factor.

```
Standard:   3× replication (tolerate 2 node failures)
High-value: 5× replication (extra durability)
CDN assets: distributed across hundreds of edge nodes globally

Effective storage = raw storage × replication_factor

Example: 100 TB of user data × 3 = 300 TB of actual disk provisioned
```

---

## 11. Quick Number Sense

```
1 million users:
  If each does 10 actions/day → 100 RPS average

100 million users:
  If each does 10 actions/day → 10,000 RPS average

1 billion users:
  If each does 10 actions/day → 100,000 RPS average

100,000 RPS × 1 KB per request = 100 MB/sec = 0.8 Gbps network
```

---

## 12. The Estimation Conversation

In an interview, estimation should be a conversation, not a silent calculation.

```
"Let me do some quick back-of-envelope math.
 With 500 million DAU and assuming each user sends about 40 messages per day...
 that's 20 billion messages per day.
 Dividing by 86,400 seconds... roughly 230,000 messages per second average.
 Peak would be around 2× that, so about 460,000 per second.
 This immediately tells me a single database won't work for writes —
 we need horizontal partitioning or a write-optimised NoSQL store."
```

Show your reasoning. The math matters less than what you conclude from it.

---

## 13. Summary Reference Card

```
TRAFFIC
  Avg RPS   = DAU × ops/user / 100,000
  Peak RPS  = Avg × 2

STORAGE
  Daily     = Write_RPS × 100,000 × record_size
  5-year    = Daily × 1,825
  With rep  = 5-year × 3

LATENCY HIERARCHY
  Memory < SSD < Network(local) < Disk < Network(remote)
  1ns     16µs   500µs           10ms    100ms

SINGLE NODE LIMITS (approximate)
  Web: ~50K RPS
  SQL writes: ~10K TPS
  SQL reads:  ~50K QPS
  Redis:      ~1M ops/sec
```

---

*System Design Engineering Handbook — 11-Interview Series*