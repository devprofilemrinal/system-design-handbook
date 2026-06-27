# 03 — Metrics & Alerting

> **08-Observability Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Are Metrics?

Metrics are numeric measurements collected over time that describe the behaviour, performance, and health of a system. Unlike logs (which record what happened) and traces (which follow individual requests), metrics aggregate behaviour across thousands of requests into numbers you can graph, compare, and alert on.

```
Log entry: "GET /checkout took 1,247ms — status 200"
Metric:    http_request_duration_p99{endpoint="/checkout"} = 1,230ms
           (represents the P99 of thousands of requests, not one)
```

Metrics answer questions like:
- How many requests are we handling per second right now?
- What is the 99th percentile latency for the checkout endpoint?
- How full is the database connection pool?
- What fraction of payment requests are failing?

---

## 2. Metric Types

Different types of measurements require different metric types. Using the wrong type produces misleading data.

### Counter

A counter only goes up. It tracks cumulative counts of events. You derive rate from counters by comparing values over time.

```
http_requests_total = 1,482,394  (total since service started)

Rate of requests per second = (value_now - value_1_min_ago) / 60
→ Rate is more useful than the raw counter
```

**Use for:** Total requests, total errors, total bytes sent, total jobs processed. Anything you want to measure as "how many happened in the last N seconds."

### Gauge

A gauge can go up or down. It represents a current value at a point in time.

```
db_connection_pool_active = 47     (currently 47 connections in use)
memory_usage_bytes = 2,147,483,648 (current memory usage)
queue_depth = 1,203                (current messages in queue)
```

**Use for:** Current queue depth, current active connections, current CPU/memory usage, temperature — anything representing a current level.

### Histogram

A histogram samples observations and counts them into configurable buckets. It allows you to calculate percentiles (P50, P95, P99) across a population of observations.

```
http_request_duration_seconds_bucket{le="0.05"}  = 14,231  (requests ≤ 50ms)
http_request_duration_seconds_bucket{le="0.1"}   = 28,442  (requests ≤ 100ms)
http_request_duration_seconds_bucket{le="0.5"}   = 51,094  (requests ≤ 500ms)
http_request_duration_seconds_bucket{le="1.0"}   = 54,211  (requests ≤ 1000ms)
http_request_duration_seconds_bucket{le="+Inf"}  = 54,892  (all requests)

P99 ≈ 500ms (99% of 54,892 requests completed in ≤ 500ms)
```

**Use for:** Request durations, response sizes, queue wait times — anything where you need percentile distribution, not just an average.

> **Never use averages for latency. Always use histograms for percentiles.** An average of 50ms can hide a P99 of 5,000ms. A histogram reveals the full distribution.

### Summary

Similar to histogram but pre-calculates percentiles at the client side. Less flexible than histograms (can't aggregate across instances) but lower overhead.

| Type | Direction | Use For |
|---|---|---|
| **Counter** | Up only | Event counts, totals |
| **Gauge** | Up and down | Current levels, resource usage |
| **Histogram** | Bucketed observations | Latency percentiles, size distributions |

---

## 3. The Four Golden Signals

Google's SRE Book identified four signals that, when monitored together, cover the majority of problems in any production service. If you monitor nothing else, monitor these.

### Signal 1 — Latency

How long does it take to serve a request?

```
Measure:
  P50 (median): what's the typical experience?
  P99: what's the worst experience most users see?
  P99.9: what's the absolute worst case?

Alert on: P99 > SLO threshold
Watch for: sudden increase in P99 (often signals a slow dependency or DB issue)

Critical distinction: measure latency of successful AND failed requests separately.
  A service returning instant errors looks "fast" in latency metrics — misleading.
```

### Signal 2 — Traffic

How much demand is the system handling?

```
Measure:
  Requests per second (RPS) for each endpoint
  Messages per second for queues
  Bytes per second for data pipelines

Alert on: traffic drops to zero (service stopped receiving requests — silent failure)
Watch for: traffic spike exceeding capacity plan
```

Traffic is the denominator for error rate calculations and helps interpret other signals. A 10% error rate during 100 RPS is very different from 10% during 100,000 RPS.

### Signal 3 — Errors

What fraction of requests are failing?

```
Measure:
  HTTP 5xx rate: 5xx_count / total_requests
  Business error rate: payment_failed / total_payment_attempts
  Explicit errors: not all errors are HTTP 5xx (a 200 with "success: false" is still an error)

Alert on: error rate > SLO threshold (e.g., > 0.1% of requests failing)
Watch for: sudden increase in a specific error code
```

**Explicit errors are often missed.** An API that returns `{"status": "ok", "data": {"error": "user not found"}}` with HTTP 200 is returning an error. Instrument for business outcomes, not just HTTP codes.

### Signal 4 — Saturation

How full is the system? How close to its maximum capacity?

```
Measure:
  CPU utilisation (%)
  Memory utilisation (%)
  DB connection pool usage (active / max)
  Thread pool usage (active / max)
  Disk usage (%)
  Queue depth (messages waiting)

Alert on: any resource > 80% utilisation sustained (approaching saturation cliff)
Watch for: steadily growing queue depth (consumers not keeping up)
```

Saturation predicts future failures before they occur. A DB connection pool at 95% capacity will soon reject connections entirely. Alerting at 80% gives you time to act.

> **Saturation is the only leading indicator among the four signals.** The other three are lagging — they tell you something has already gone wrong. Saturation warns you before it does.

---

## 4. RED and USE — Complementary Frameworks

Two additional frameworks complement the golden signals for specific contexts.

### RED — Rate, Errors, Duration

Specifically for services (microservices, APIs):

```
Rate:     How many requests per second?
Errors:   What fraction are failing?
Duration: How long do they take (P99)?
```

RED is simpler than four golden signals and sufficient for service-level dashboards.

### USE — Utilisation, Saturation, Errors

Specifically for infrastructure resources (servers, databases, queues):

```
Utilisation:  What % of the resource is in use?
Saturation:   How much work is waiting (queue depth, wait time)?
Errors:       What is the error rate for this resource?
```

USE applied to every system resource catches infrastructure bottlenecks that don't appear in service-level metrics.

---

## 5. Designing Alerts That Don't Cause Alert Fatigue

Alert fatigue is when there are so many alerts — too many false positives, too many low-priority alerts — that engineers start ignoring them. When real alerts fire in the noise, they get missed.

### Alert on Symptoms, Not Causes

```
❌ Alert: "CPU > 80%"
   (CPU might be high and everything is fine; doesn't tell you if users are impacted)

✅ Alert: "Error rate > 0.5% for 5 minutes"
   (Users are experiencing errors — unambiguously a problem)

❌ Alert: "Replication lag > 1 second"
   (Lag might be acceptable; users may not be impacted)

✅ Alert: "Payment API P99 > 2 seconds for 10 minutes"
   (User-visible impact — checkout is slow)
```

### Alert on SLO Violations, Not Thresholds

Tie alerts to your SLO burn rate. Alert when your error budget is being consumed faster than sustainable.

```
SLO: 99.9% success rate (30-day window)
Error budget: 43.8 minutes of downtime equivalent per month

Burn rate alert:
  "If we keep failing at this rate, we'll exhaust our monthly budget in 1 hour"
  → Fire immediately for fast burns
  "If we keep failing at this rate, we'll exhaust our monthly budget in 3 days"
  → Fire as a ticket (not a page) for slow burns
```

### The Alerting Properties

Every alert should have all four:

| Property | Meaning |
|---|---|
| **Actionable** | The on-call engineer knows what to do when it fires |
| **Accurate** | High precision — few false positives |
| **Timely** | Fires with enough time to act before user impact |
| **Relevant** | Represents actual or imminent user impact |

If an alert lacks any of these, it should be downgraded to a ticket or removed entirely.

---

## 6. Prometheus — The Metrics Standard

Prometheus is the dominant open-source metrics system. Understanding it is essential for system design discussions.

### How Prometheus Works

Services expose a `/metrics` endpoint. Prometheus **scrapes** (polls) this endpoint on a schedule (typically every 15 seconds) and stores the time-series data.

```
Service /metrics endpoint:
  http_requests_total{method="GET",status="200"} 14823
  http_requests_total{method="GET",status="500"} 42
  http_request_duration_seconds_bucket{le="0.1"} 13941
  http_request_duration_seconds_bucket{le="0.5"} 14801

Prometheus scrapes every 15 seconds → stores as time-series → queryable via PromQL
```

### Labels — The Power of Prometheus

Labels are key-value pairs that add dimensions to metrics. They allow you to slice and dice the same metric along multiple axes.

```
http_requests_total{service="payment",endpoint="/charge",status="500",region="eu-west"}

Filter by service: service="payment"
Filter by error:   status="500"
Filter by region:  region="eu-west"
Group by:          sum by (endpoint) rate(http_requests_total[5m])
```

**Label cardinality warning:** Each unique combination of label values creates a separate time series. High-cardinality labels (user_id, request_id) can create millions of time series and make Prometheus unsustainably expensive.

```
❌ High cardinality (avoid):
  http_requests_total{user_id="u-123456"} ← one series per user — millions of series

✅ Low cardinality (safe):
  http_requests_total{service="payment",status="500"} ← bounded set of combinations
```

### Grafana — Visualisation Layer

Prometheus is queried via PromQL. Grafana visualises these queries as dashboards.

```
Prometheus (stores time-series data) → Grafana (visualises as graphs/dashboards)

Typical dashboard panels:
  - RPS by endpoint (line graph)
  - Error rate % by service (gauge + threshold lines)
  - P99 latency by endpoint (line graph with SLO line)
  - DB connection pool usage % (gauge)
  - Active alerts (table)
```

---

## 7. How Large Companies Apply This

| Company | Approach | Source |
|---|---|---|
| **Google** | Originated four golden signals; Borgmon (internal) inspired Prometheus design | *Google SRE Book* (public) |
| **Netflix** | Atlas — custom time-series system built for Netflix's scale; billions of metrics | Netflix Tech Blog (public) |
| **Uber** | M3 — open-sourced distributed metrics platform designed for extreme scale | Uber Eng Blog (public) |
| **Cloudflare** | Prometheus + Grafana across all services; custom exporters per infrastructure component | Cloudflare Blog (public) |

---

## 8. Best Practices

- **Monitor the four golden signals for every service** — they cover the majority of production problems.
- **Use histograms for latency, not averages** — averages hide tail latency.
- **Alert on symptoms** (user impact), not causes (CPU, memory).
- **Tie alerts to SLO burn rates** — not arbitrary thresholds.
- **Avoid high-cardinality labels** — they make Prometheus unsustainably expensive.
- **Every alert must be actionable** — if you don't know what to do when it fires, it shouldn't be an alert.
- **Build per-service dashboards** — each team owns and understands their service's signals.

---

## 9. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Alerting on averages instead of percentiles | Tail latency hidden; real user impact missed | Histogram P99 for all latency alerts |
| Too many alerts | Alert fatigue; real alerts missed in noise | Every alert must be actionable; prune regularly |
| Alerting on causes not symptoms | Pages for non-impacting events | Alert on error rate, latency SLO breach |
| High-cardinality labels | Prometheus performance degrades; costs explode | Keep label values bounded |
| No dashboards per service | On-call has no starting point for investigation | Standard service dashboard for every team |
| Using averages for capacity planning | Under-provision for peak loads | Plan based on P99 and peak traffic |

---

## 10. Interview Questions

1. What are the four metric types? When would you use a histogram instead of a counter?
2. What are the four golden signals? What does each measure?
3. Why should you alert on symptoms rather than causes?
4. What is alert fatigue and how do you prevent it?
5. What is label cardinality in Prometheus and why is high cardinality a problem?
6. What is the difference between RED and USE frameworks?
7. How would you design an alert for a payment API that must maintain 99.9% success rate?

---

## 11. Summary

| Concept | Key Takeaway |
|---|---|
| **Counter** | Cumulative, goes up only. Derive rate from it. |
| **Gauge** | Current value. Goes up and down. |
| **Histogram** | Distribution in buckets. Calculate P99. Never use averages for latency. |
| **Four Golden Signals** | Latency, Traffic, Errors, Saturation. Monitor all four for every service. |
| **Saturation** | The leading indicator. Warns before failure occurs. |
| **Alerting** | On symptoms and SLO burn rates. Actionable. No false positives. |
| **Prometheus** | Scrape-based metrics. PromQL queries. Labels add dimensions — keep cardinality low. |

---

## 12. Cross References

**Prerequisites:** 01-observability-fundamentals.md · Latency & Throughput (NFR #1)

**Related Topics:** 02-logging.md · 04-distributed-tracing.md · Availability (NFR #2)

**What to Learn Next:** 04-distributed-tracing.md

---

*System Design Engineering Handbook — 08-Observability Series*