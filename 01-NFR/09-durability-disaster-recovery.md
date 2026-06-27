# Durability & Disaster Recovery

> **NFR Deep Dive #9** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Overview

**Durability** guarantees that once data is committed, it survives any failure — crashes, power loss, disk death. **Disaster Recovery (DR)** is the plan and capability to restore service after a *major* failure: a datacenter outage, region loss, or catastrophic data corruption.

Durability is about *never losing data*. DR is about *getting the whole system back* after the worst happens.

> **Core premise:** Disasters are not "if" but "when." Regions fail, fiber gets cut, someone runs `DELETE` without a `WHERE`. A system without a tested recovery plan is one bad day away from permanent loss.

---

## 2. Durability — Don't Lose Committed Data

When a system acknowledges a write, durability promises that write is safe even if the node dies one millisecond later.

| Technique | How It Protects Data |
|---|---|
| **Write-Ahead Log (WAL)** | Record the change to a durable log *before* applying it; replay on crash recovery |
| **Replication** | Keep copies on multiple nodes; one disk dies, data survives |
| **Quorum writes** | Acknowledge only after N replicas confirm — survives node loss |
| **Backups & snapshots** | Periodic copies for recovery from corruption or deletion |
| **Checksums** | Detect silent disk corruption (bit rot) |
| **fsync to disk** | Force data out of volatile memory onto persistent storage |

### Durability is measured in nines (like availability)

```
Cloud object stores commonly advertise "eleven nines": 99.999999999%
→ statistically, you'd lose one object every ~10 million years.
Achieved by replicating each object across many devices and facilities.
```

> **The durability-vs-latency trade:** Waiting for more replicas to confirm a write = more durable but slower. `fsync` on every write = safest but kills throughput. Systems tune this deliberately.

---

## 3. The Two DR Metrics: RPO & RTO

These are the two numbers that define a disaster recovery plan. Memorize them.

| Metric | Full Name | Question | Determines |
|---|---|---|---|
| **RPO** | Recovery Point Objective | How much *data* can we afford to lose? | Backup frequency |
| **RTO** | Recovery Time Objective | How much *downtime* can we tolerate? | Recovery architecture |

```
        ← RPO →              ← RTO →
  ┌──────────────┬──────────┬──────────────┐
  │ last backup  │ DISASTER │  recovered   │
  └──────────────┴──────────┴──────────────┘
   data since last      time to restore
   backup is LOST       service

RPO = how far back in time you recover to (data loss window)
RTO = how long it takes to get back up (downtime window)
```

**Example:**
- RPO = 1 hour → you back up at least hourly; you may lose up to 1 hour of data.
- RTO = 15 minutes → your architecture must restore service within 15 minutes.

> **Lower RPO/RTO = higher cost.** Near-zero RPO needs continuous replication; near-zero RTO needs hot standby infrastructure running 24/7. Match the targets to business impact, don't gold-plate.

---

## 4. Backup Strategies

| Type | What | Trade-off |
|---|---|---|
| **Full** | Complete copy of all data | Simple restore; large, slow, costly |
| **Incremental** | Only changes since last backup | Small/fast; slower restore (chain replay) |
| **Differential** | Changes since last *full* backup | Middle ground |
| **Continuous (CDC)** | Stream every change in real time | Lowest RPO; most complex |

### The 3-2-1 rule (industry standard)

```
3 copies of your data
2 different storage media/types
1 copy off-site (different region)
```

> **Critical:** A backup you have never restored is not a backup — it's a hope. Test restores regularly. Backups silently fail (corruption, missing data, wrong config) and you only find out during a real disaster.

---

## 5. DR Strategies — The Cost/Speed Spectrum

Four standard approaches, trading cost against recovery speed (RTO).

| Strategy | How It Works | RTO | Cost |
|---|---|---|---|
| **Backup & Restore** | Restore from backups into new infra after disaster | Hours–days | Lowest |
| **Pilot Light** | Minimal core (e.g. DB replica) always running; scale up on disaster | ~10s of min | Low |
| **Warm Standby** | Scaled-down but full copy running; scale up to take traffic | Minutes | Medium |
| **Hot Standby / Multi-Site** | Full duplicate running live; instant failover (often active-active) | Seconds–zero | Highest |

```
COST  ←──────────────────────────────────────→  COST
LOW                                              HIGH
Backup/Restore → Pilot Light → Warm → Hot/Multi-Site
SLOW ←─────────────── RTO ──────────────────→ FAST
```

> **Pick based on RTO/RPO targets.** A blog can live with Backup & Restore. A payment system needs Hot Standby. Most production systems land on Pilot Light or Warm Standby.

---

## 6. Multi-Region for DR

The strongest DR comes from running across geographically separate regions, so an entire region can fail without taking you down.

| Pattern | Behaviour |
|---|---|
| **Active-Passive** | One region serves; another stands by for failover |
| **Active-Active** | Multiple regions serve simultaneously; lose one, others absorb load |

**Trade-offs:** multi-region adds cost, cross-region latency, and data-consistency challenges (replicating data across regions reintroduces the CAP trade-off — see Consistency NFR).

> **DNS failover** is the common switch mechanism: health checks detect a dead region and repoint DNS to a healthy one. Note DNS TTL caching can delay this — factor it into RTO.

---

## 7. How Large Companies Apply This

| Company | Application | Source |
|---|---|---|
| **Amazon S3** | 11 nines durability via replication across multiple facilities | AWS public docs |
| **Cloud providers** | Region + AZ design; cross-region replication for DR | Public cloud docs |
| **Netflix** | Multi-region active-active; regularly evacuates a whole region as a drill | Netflix Tech Blog (public) |
| **Banks** | Strict low RPO/RTO; synchronous replication; tested DR plans for compliance | Industry standard |

> **Inferred:** Exact internal RPO/RTO figures are rarely public; the strategies (multi-region, cross-region replication, region-failover drills) are documented publicly.

---

## 8. Best Practices

- **Set explicit RPO and RTO** per system, driven by business impact — not guesses.
- **Follow 3-2-1** for backups; keep at least one copy off-site/off-region.
- **Test restores regularly** — an untested backup is not a backup.
- **Run DR drills** — actually fail over; don't just plan on paper.
- **Replicate across regions** for systems that can't tolerate region loss.
- **Match DR strategy to RTO** — don't pay for hot standby if backup/restore suffices.
- **Automate failover** with health checks — manual failover is slow and error-prone.
- **Use WAL + replication** for write durability at the database layer.

---

## 9. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Never testing backups | Discover they're broken during a real disaster | Scheduled restore tests |
| Backups in the same region as data | A region outage destroys both | Off-region/off-site copies (3-2-1) |
| No defined RPO/RTO | No way to design or measure DR adequacy | Set explicit targets per system |
| Untested failover | Failover fails when it's actually needed | Regular DR game days |
| Single copy of data | Permanent loss on disk failure | Replicate + back up |
| Ignoring DNS TTL in RTO | Failover slower than promised | Account for TTL; use low TTLs for failover records |

---

## 10. Interview Questions

1. Define RPO and RTO. How does each drive design decisions?
2. What's the difference between durability and availability?
3. Explain the 3-2-1 backup rule.
4. Compare Backup-and-Restore, Pilot Light, Warm, and Hot Standby DR strategies.
5. How is "eleven nines" of durability achieved?
6. Why is an untested backup considered worthless?
7. How does multi-region active-active improve DR, and what does it cost?
8. What role does DNS play in regional failover?

---

## 11. Summary

| Concept | Key Takeaway |
|---|---|
| **Durability** | Committed data survives any failure. Measured in nines. |
| **WAL + replication** | The core mechanisms of write durability. |
| **RPO** | Acceptable *data loss* window → sets backup frequency. |
| **RTO** | Acceptable *downtime* window → sets recovery architecture. |
| **3-2-1** | 3 copies, 2 media, 1 off-site. |
| **DR strategies** | Backup→Restore (cheap/slow) … Hot Standby (costly/instant). |
| **Test it** | Untested backups and failovers fail when you need them. |

---

## 12. Cross References

**Prerequisites:** System Design Fundamentals · Availability (NFR #2) · Reliability (NFR #4)

**Related Topics:** Replication · Consistency (NFR #5) · Multi-Region Architecture · Load Balancing (DNS failover)

**What to Learn Next:** Databases series (Replication, Partitioning) · CAP Theorem

---

*System Design Engineering Handbook — NFR Series*

---

> **NFR series complete.** Covered: Latency & Throughput · Availability · Scalability · Reliability · Consistency · Fault Tolerance · Security · Observability & Maintainability · Durability & Disaster Recovery.