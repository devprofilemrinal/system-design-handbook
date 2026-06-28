# 02 — Availability

> **NFR Deep Dive #2** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Availability?

Availability is the fraction of time a system is operational and able to serve requests. It is the most visible non-functional requirement — when availability fails, users cannot use the product at all.

```
Availability = Uptime / (Uptime + Downtime)

Example:
  A system that is down for 8.76 hours in a year:
  Availability = (8,751.24 / 8,760) = 99.9%
```

Availability is expressed as a percentage — specifically, as "nines."

---

## 2. The Nines Framework

Every additional nine represents a 10× improvement in uptime — and a roughly 10× increase in engineering investment to achieve it.

| Availability | Annual Downtime | Monthly Downtime | Weekly Downtime | Common Name |
|---|---|---|---|---|
| 90% | 36.5 days | 72 hours | 16.8 hours | One nine |
| 99% | 3.65 days | 7.2 hours | 1.68 hours | Two nines |
| 99.9% | 8.76 hours | 43.8 minutes | 10.1 minutes | Three nines |
| 99.99% | 52.56 minutes | 4.38 minutes | 1.01 minutes | Four nines |
| 99.999% | 5.26 minutes | 26.3 seconds | 6.05 seconds | Five nines |
| 99.9999% | 31.5 seconds | 2.63 seconds | < 1 second | Six nines |

**Industry reference points (based on public SLAs):**
- AWS EC2 SLA: 99.99%
- Google Cloud Compute: 99.99%
- Most SaaS products: 99.9% (three nines)
- Payment processors (Stripe, Visa): target 99.999%

> **Key insight:** The difference between 99.9% and 99.99% is 48 minutes of downtime per month vs 4 minutes. That difference determines whether a payment outage costs you one customer support call or front-page news coverage.

---

## 3. Why Availability Is Hard

Availability is not just "don't let the server crash." A system is unavailable from a user's perspective whenever it fails to serve a request successfully — for any reason:

```
Causes of unavailability:
  Hardware failure     — disk dies, network card fails, datacenter power outage
  Software bugs        — memory leak, deadlock, unhandled exception
  Deployment errors    — bad config pushed to production, failed migration
  Dependency failure   — database is slow, third-party API is down
  Traffic overload     — more requests than the system can handle
  Operational errors   — someone deletes the wrong data, misconfigures a firewall
  Planned maintenance  — deployments, upgrades, database migrations
```

Every one of these is a real production failure mode. High availability means designing for all of them simultaneously.

---

## 4. Measuring Availability: SLI, SLO, and SLA

These three terms define how availability is measured, targeted, and contracted. They are used at every major tech company and appear in every SRE conversation.

### SLI — Service Level Indicator

The actual metric you measure. The raw number.

```
Common availability SLIs:
  Request success rate = successful_requests / total_requests
  Uptime percentage    = minutes_available / total_minutes
  Error rate           = failed_requests / total_requests
```

An SLI is always a ratio over a time window (usually 30 days rolling).

### SLO — Service Level Objective

The internal target you set for an SLI. The goal engineering commits to.

```
Examples:
  "99.9% of requests succeed over any 30-day window"
  "The service is available 99.99% of the time"
  "Error rate stays below 0.1%"
```

SLOs are internal. They are stricter than SLAs to give you a buffer before breaching external commitments.

### SLA — Service Level Agreement

The contractual commitment to customers, with defined consequences for breach.

```
Example SLA:
  "If monthly availability falls below 99.9%, customers receive a 10% service credit.
   If it falls below 99.0%, customers receive a 25% credit."
```

### The Relationship

```
SLI (what you measure) → compared against → SLO (your internal target)
                                                  ↓ (stricter than)
                                             SLA (what you promise customers)

If SLI breaches SLO → engineering alert, incident response
If SLI breaches SLA → customer credits, contractual consequences
```

---

## 5. Error Budgets

The error budget is the single most useful concept for balancing reliability work with feature development. It converts availability into a shared engineering currency.

```
Error budget = 100% - SLO

If SLO = 99.9%:
  Error budget = 0.1%
  = 43.8 minutes of allowable downtime per month
  = 8.76 hours per year

If SLO = 99.99%:
  Error budget = 0.01%
  = 4.38 minutes of allowable downtime per month
```

### How Error Budgets Change Engineering Decisions

```
Scenario A — Budget healthy (used < 20% this month):
  → Ship features aggressively
  → Deploy frequently
  → Try riskier experiments
  → The budget absorbs the risk

Scenario B — Budget nearly exhausted:
  → Freeze risky releases
  → Focus engineering on reliability
  → Do not deploy until next month resets the budget
  → Identify and fix the root cause of recent incidents

Scenario C — Budget exceeded:
  → No new feature deployments
  → All engineering focuses on reliability
  → Post-mortem required before returning to normal cadence
```

> **Why this matters:** Error budgets turn reliability from a subjective argument ("we should be more stable") into an objective one ("we've used 87% of our budget in week 1 — we cannot afford another deployment this month"). Both product and engineering agree on the same number.

---

## 6. Availability at the System Level

Individual component availability combines to produce system availability. Understanding this math is critical for designing reliable systems.

### Sequential Dependencies (The Multiplication Rule)

When a request must pass through multiple components in sequence, system availability is the product of each component's availability.

```
System: Client → Load Balancer → Service → Database

Load Balancer availability: 99.99%
Service availability:       99.99%
Database availability:      99.99%

System availability = 0.9999 × 0.9999 × 0.9999
                    = 0.9997
                    = 99.97%

Three components each at four nines → system is only at three nines.
```

**The lesson:** Every component you add to the critical path degrades overall availability. This is why:
- Non-critical work is moved off the critical path (async queues)
- Caches sit in front of databases (reduce dependency on slow components)
- Circuit breakers fail fast rather than waiting (cap the blast radius)

### Parallel Redundancy (Improving Availability)

When multiple instances handle the same work, the system is only unavailable if ALL instances fail simultaneously.

```
Two load balancer instances, each at 99.9% availability:

P(both fail simultaneously) = (1 - 0.999) × (1 - 0.999)
                             = 0.001 × 0.001
                             = 0.000001
                             = 0.0001%

System availability = 1 - 0.0001% = 99.9999%

Two instances at 99.9% → system at 99.9999% (six nines!)
```

This is why redundancy is the primary tool for achieving high availability. Run multiple instances. If one fails, the others continue serving.

---

## 7. RPO and RTO — Recovery Objectives

When unavailability does occur, two metrics define acceptable recovery:

| Metric | Full Name | Definition | Question It Answers |
|---|---|---|---|
| **RTO** | Recovery Time Objective | Maximum acceptable time to restore service after failure | How long can we be down? |
| **RPO** | Recovery Point Objective | Maximum acceptable data loss measured in time | How much data can we lose? |

```
Timeline of a failure:

Last backup     System fails    System restored
     │               │                │
     ▼               ▼                ▼
─────●───────────────●────────────────●────▶ time
     │←── RPO ──────►│←───── RTO ────►│

RPO: How old can the restored data be?
     (gap between last backup and the failure)

RTO: How long until the service is back?
     (gap between failure and recovery)
```

### RPO and RTO by System Type

| System | Acceptable RTO | Acceptable RPO | Why |
|---|---|---|---|
| Payment processing | < 5 minutes | < 1 minute | Every minute down is direct revenue loss |
| E-commerce checkout | < 15 minutes | < 5 minutes | Users abandon carts; revenue impact |
| Social media feed | < 30 minutes | < 1 hour | Inconvenient but not catastrophic |
| Email system | < 1 hour | < 15 minutes | Delayed, not lost |
| Internal analytics | < 4 hours | < 24 hours | Business reporting; not user-facing |
| Archival system | < 24 hours | < 24 hours | Accessed rarely; long recovery acceptable |

**RPO drives backup strategy:** If RPO = 1 minute, you need near-continuous replication. If RPO = 24 hours, daily backups may suffice.

**RTO drives failover strategy:** If RTO = 5 minutes, you need hot standby (running, ready to switch). If RTO = 4 hours, warm standby (stopped, needs startup) may be acceptable.

---

## 8. The Four Pillars of High Availability

Every high-availability system is built on the same four foundations. Skipping any one of them creates gaps that incidents will find.

### Pillar 1 — Redundancy

No single point of failure. Every critical component has at least one backup.

```
Single point of failure (SPOF) examples:
  One load balancer → the entire system goes down if it fails
  One database primary → all writes fail if it fails
  One availability zone → regional failure takes down everything

Redundancy removes SPOFs:
  Two load balancers → if one fails, the other continues
  Primary + replica(s) → automatic failover on primary failure
  Multi-AZ deployment → regional failure only affects part of the system
```

The minimum for any production system: two instances of every stateless service, primary + replica for databases, multi-AZ deployment.

### Pillar 2 — Health Checks and Automatic Failover

Redundancy only helps if unhealthy instances are automatically removed from traffic. Humans cannot respond fast enough — failover must be automatic.

```
Health check types:
  Liveness:   "Is the process alive?" → restart if not
  Readiness:  "Is the service ready to handle requests?" → remove from rotation if not

Health check flow:
  Load balancer sends GET /health every 10 seconds
  If 3 consecutive failures → mark instance as unhealthy → stop sending traffic
  If instance recovers → mark healthy → resume sending traffic

The RTO of automatic failover: seconds.
The RTO of manual human intervention: minutes to hours.
```

### Pillar 3 — Graceful Degradation

When part of the system fails, the rest continues working — possibly with reduced functionality, but not a complete outage.

```
Examples of graceful degradation:
  Recommendation service is down:
    → Show popular items instead (static fallback)
    → User still completes their purchase

  Search index is slow:
    → Return cached results from the last successful query
    → User sees slightly stale results but continues browsing

  Payment provider has elevated errors:
    → Queue the payment and retry asynchronously
    → Show user "processing" instead of error page
```

The goal: the system degrades to a reduced but still useful state, rather than failing completely.

### Pillar 4 — Observability and Fast Detection

You cannot fix what you cannot see. High availability requires detecting failures as quickly as possible — ideally before users notice.

```
Detection methods (fastest to slowest):
  Synthetic monitoring:   Simulated user transactions every 60 seconds
  Metric alerts:          Error rate threshold breach → page on-call
  Log-based alerts:       Specific error pattern in logs → alert
  User reports:           Support tickets → human escalation (slow)

Target detection time for 99.99% availability:
  With 4.38 minutes/month budget, each incident must be detected and
  resolved within minutes. Relying on user reports means you've
  already breached your budget before you even know there's a problem.
```

---

## 9. Planned vs Unplanned Downtime

Availability targets apply to both planned and unplanned downtime combined. Many teams focus only on incidents, forgetting that deployments, maintenance, and migrations also consume the budget.

### Zero-Downtime Deployments

Every deployment is a potential availability event. Modern deployment strategies eliminate this:

**Rolling deployment:**
```
Deploy new version to instances one at a time.
Traffic continues flowing to healthy instances.
If the new version is bad, stop rollout before all instances are updated.
→ Zero downtime, gradual rollout, easy rollback
```

**Blue-Green deployment:**
```
Run two identical environments: Blue (live) and Green (idle).
Deploy new version to Green.
Test Green. Switch traffic atomically.
If problems: switch back to Blue instantly.
→ Zero downtime, instant rollback capability
```

**Canary deployment:**
```
Route 1% of traffic to the new version.
Monitor for errors and latency regressions.
Gradually increase to 5%, 10%, 50%, 100%.
If problems at any stage: roll back.
→ Zero downtime, real production validation, minimal blast radius
```

### Database Migrations

Schema migrations on live databases are one of the most common causes of planned outages. Zero-downtime database migrations use the expand-contract pattern:

```
Phase 1 (Expand): Add the new column/table alongside the old
  → Both old and new code can work simultaneously
  → No outage required

Phase 2 (Migrate): Backfill data into the new structure
  → Background job, no traffic impact

Phase 3 (Contract): Remove the old column/table once all code uses new
  → Another background job, no outage required
```

---

## 10. Multi-Region Availability

For 99.99% and above, single-region deployment is often insufficient. A regional infrastructure failure (network issue, datacenter problem) would exceed the monthly downtime budget in hours.

```
Single region deployment:
  Regional failure → complete outage
  RTO = time to bring up another region (hours)
  Availability ceiling ≈ 99.9% (limited by regional failure rates)

Multi-region active-active:
  Traffic distributed across multiple regions
  Regional failure → traffic routes to remaining regions automatically
  RTO = seconds (DNS failover or load balancer routing)
  Availability ceiling ≈ 99.99%+ (multiple independent failure domains)
```

**The trade-off:** Multi-region introduces data consistency challenges. A write in region A must replicate to region B before region B can read it. This is the CAP theorem in practice — covered in depth in `06-Distributed-Systems/02-cap-theorem.md`.

---

## 11. How Large Companies Apply This

| Company | Approach | Source |
|---|---|---|
| **Google** | Originated SRE model with SLIs/SLOs/error budgets; treats reliability as a quantified engineering problem | *Google SRE Book* (public) |
| **AWS** | 99.99% SLA on most compute services; multi-AZ architecture recommended for all production workloads | AWS documentation (public) |
| **Netflix** | Chaos Monkey deliberately kills production instances to verify the system degrades gracefully; multi-region active-active | Netflix Tech Blog (public) |
| **Stripe** | Five nines target for payment APIs; extensive runbooks and automated failover for every failure mode | Stripe engineering blog (public) |

> **Inferred:** Exact internal SLO values are not public. The architectural approaches above are described in public engineering material.

---

## 12. Best Practices

- **Set SLOs before incidents happen** — not reactively after things break.
- **Use error budgets** to make reliability vs. feature velocity trade-offs objective and shared.
- **Eliminate every SPOF** — start with load balancers, databases, and network egress.
- **Automate health checks and failover** — human response time is too slow for four-nines targets.
- **Design for graceful degradation** — define the reduced-functionality state for every possible dependency failure.
- **Test failover regularly** — an untested failover path is not a reliable failover path (Chaos Engineering).
- **Count planned downtime in your budget** — deployments and maintenance consume the same budget as incidents.

---

## 13. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Targeting 100% availability | Impossible; drives over-engineering; no error budget for deployments | Set a realistic target based on business need |
| Ignoring planned downtime | Deployments consume the budget before incidents happen | Use zero-downtime deployment strategies |
| SPOF in the critical path | One component failure = complete outage | Audit every component; add redundancy to SPOFs |
| Manual failover only | Humans too slow; RTO measured in hours not seconds | Automate health checks and failover |
| Untested failover | Works in theory, fails in practice | Regular DR drills; Chaos Engineering |
| SLO stricter than the dependencies allow | You cannot have 99.99% if your database vendor SLA is 99.9% | SLO must be consistent with dependency SLAs |
| No error budget policy | Reliability vs. features argument is subjective | Define what happens when the budget is consumed |

---

## 14. Interview Questions

1. What is the difference between an SLI, SLO, and SLA? Give an example of each.
2. A service has three components in sequence, each at 99.9% availability. What is the system availability?
3. What is an error budget and how does it change engineering decisions?
4. What is the difference between RTO and RPO? Give an example where they have very different targets.
5. What are the four pillars of high availability?
6. Why does running at high CPU utilisation hurt availability, not just performance?
7. How would you design a deployment strategy that contributes zero downtime to the availability budget?

---

## 15. Summary

| Concept | Key Takeaway |
|---|---|
| **Availability** | Fraction of time the system serves requests. Expressed in nines. |
| **Nines** | Each additional nine = 10× better but ~10× more investment. |
| **SLI → SLO → SLA** | Measure → internal target → external contract. SLO stricter than SLA. |
| **Error budget** | 100% - SLO. Shared currency between features and reliability. |
| **Sequential components** | System availability = product of component availabilities. Always lower. |
| **Redundancy** | Primary tool for improving availability. Removes single points of failure. |
| **RTO / RPO** | Recovery Time Objective (how long down). Recovery Point Objective (how much data lost). |
| **Graceful degradation** | Partial failure → reduced functionality, not complete outage. |

---

## 16. Cross References

**Prerequisites:** 01-latency-and-throughput.md · 00-Introduction/02-functional-and-non-functional-requirements.md

**Related Topics:**
- Resiliency patterns (circuit breaker, bulkhead, retry) → `07-Microservices/03-inter-service-communication.md` and `07-Microservices/04-microservices-patterns.md`
- Availability vs Consistency trade-off → `06-Distributed-Systems/02-cap-theorem.md`
- Fault tolerance deep dive → `01-NFR/06-fault-tolerance.md`
- Incident response → `08-Observability/05-incident-response.md`

**What to Learn Next:** 03-scalability.md · 04-reliability.md

---

*System Design Engineering Handbook — NFR Series*