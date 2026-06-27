# 01 — Observability Fundamentals

> **08-Observability Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Observability?

Observability is the ability to understand the internal state of a system purely from its external outputs — the data it produces. A system is observable if, when something goes wrong, you can answer "what happened, where, and why?" without deploying new code or attaching a debugger.

The term comes from control theory: a system is observable if its internal state can be inferred from outputs alone.

> **The practical test:** When your system behaves unexpectedly at 3 AM, can you figure out what's wrong using only the data already being collected? If yes — observable. If no — not observable.

Observability is not a tool you buy. It is a property you build into your system through disciplined instrumentation. The tools (Datadog, Prometheus, Jaeger) are the means; observability is the end.

---

## 2. Monitoring vs Observability — A Critical Distinction

These terms are used interchangeably but they are different.

| | Monitoring | Observability |
|---|---|---|
| **Question** | Is the system healthy? | Why is it unhealthy? |
| **Approach** | Check predefined metrics against thresholds | Explore any question about system behaviour |
| **Detects** | Failures you anticipated and instrumented | Novel failures you didn't predict |
| **Requires** | Dashboards, alerts on known failure modes | Rich telemetry that supports arbitrary queries |
| **Analogy** | A thermometer that alerts when temperature exceeds 38°C | A full medical panel that lets a doctor diagnose any condition |

**Why both are needed:**

Monitoring tells you *something is wrong*. Observability lets you discover *what and why* — including failures no one anticipated.

A system with monitoring but without observability gives you alerts that wake you up at 3 AM but leaves you unable to figure out the cause. A system with observability but no monitoring gives you rich data but no proactive notification of problems.

```
Monitoring: "Error rate exceeded 1% — alert fired"
Observability: "The 1% errors are all from users in the EU region,
                specifically on the checkout endpoint,
                specifically when the payment provider returns a 504,
                which started happening at 14:37 after the payment service
                deployed version 2.4.1"
```

---

## 3. The Three Pillars of Observability

Observability is built on three complementary data types. Each answers different questions. None alone is sufficient.

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   LOGS          METRICS          TRACES         │
│  (what          (how many/       (where did     │
│  happened)       how fast)        time go)      │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Logs — What Happened

Logs are timestamped records of discrete events. They capture the narrative of what the system did.

```
{"ts":"2024-01-15T14:37:22Z","level":"ERROR","service":"payment-service",
 "trace_id":"abc-123","user_id":"u-456","event":"payment_failed",
 "provider":"stripe","error_code":"card_declined","duration_ms":234}
```

**Logs answer:** What exactly happened at this timestamp? What was the error message? What was the user ID? What was the request payload?

### Metrics — How Many, How Fast, How Full

Metrics are numeric measurements aggregated over time. They capture the state and performance of the system in a form that can be graphed, alerted on, and compared.

```
http_request_duration_seconds{service="payment",endpoint="/charge",status="500"} = 0.234
http_requests_total{service="payment",status="200"} = 147293
db_connection_pool_active{service="payment"} = 47
```

**Metrics answer:** What is the current error rate? How has latency trended over the last hour? Are we approaching resource saturation?

### Traces — Where Did Time Go

Traces follow a single request as it flows through multiple services, showing how much time was spent in each service and each operation.

```
Request abc-123 (total: 450ms):
  └── API Gateway: 5ms
      └── Order Service: 420ms
          ├── DB query (SELECT order): 8ms
          ├── call to Payment Service: 380ms  ← bottleneck
          │   ├── Stripe API call: 340ms      ← root cause
          │   └── DB write: 12ms
          └── publish event: 15ms
```

**Traces answer:** Where did this slow request spend its time? Which service or operation is the bottleneck? What is the causal chain of calls?

---

## 4. Why You Need All Three

The pillars are complementary. You need all three because each has gaps the others fill.

```
SCENARIO: User reports checkout is slow

Metrics alone: "P99 latency is 3.2s — elevated"
  → Tells you there's a problem. Not where or why.

Logs alone: "Found error logs from payment service"
  → Tells you something failed. Hard to correlate with specific requests.
  → Can't aggregate to see patterns across 10,000 requests.

Traces alone: "This specific slow request spent 2.8s in payment service"
  → Tells you where. But what's causing it across all requests?

All three together:
  Metrics: P99 is elevated since 14:37 — correlates with a deployment
  Trace: specific slow requests all bottleneck in the Stripe API call
  Logs: Stripe API returning 429 Too Many Requests since 14:37
  → Root cause: payment service deployed without noticing Stripe rate limits
```

---

## 5. SLIs, SLOs, and SLAs — Operationalising Observability

These three terms define how you measure and commit to service quality. They were introduced in the NFR section at a concept level — here is how they work operationally.

### SLI — Service Level Indicator

A quantitative measure of the service's behaviour. The *what you actually measure*.

```
SLI examples:
  Request success rate:  successful_requests / total_requests
  Request latency P99:   99th percentile of response time
  Error rate:            5xx_responses / total_responses
  Availability:          minutes_system_available / total_minutes
```

An SLI is a ratio over a rolling time window (typically 30 days).

### SLO — Service Level Objective

The target value for an SLI. The *internal goal* engineering commits to.

```
SLO examples:
  "99.9% of requests succeed (measured over 30 days)"
  "P99 latency < 200ms for 99% of 5-minute windows"
  "Availability ≥ 99.95%"
```

SLOs are stricter than SLAs. If your SLA promises 99.9%, your SLO target is 99.95% — leaving a buffer before breaching the external contract.

### SLA — Service Level Agreement

The contractual commitment to customers, with financial or legal consequences if breached.

```
SLA: "If availability drops below 99.9% in any calendar month,
      customers receive a 10% service credit"
```

### Error Budget

Error budget = 100% - SLO. The amount of unreliability you are *allowed* to have.

```
SLO: 99.9% availability
Error budget: 0.1% = 43.8 minutes/month of allowed downtime

If you've used 20 minutes this month:
  → 23.8 minutes remaining
  → Budget is healthy → ship features aggressively

If you've used 43 minutes (budget nearly exhausted):
  → Freeze risky changes → focus on reliability
  → Do not deploy until budget recovers next month
```

**Why error budgets change engineering culture:** They turn reliability into a shared currency between product (wants features) and SRE (wants stability). Both teams agree: features ship fast when the budget is healthy; reliability work takes priority when it's depleted.

---

## 6. The Observability Stack

A complete observability stack has five layers:

```
┌────────────────────────────────────────────────────┐
│  5. ALERTING & INCIDENT RESPONSE                   │
│     (PagerDuty, OpsGenie — who gets woken up)      │
├────────────────────────────────────────────────────┤
│  4. VISUALISATION & DASHBOARDS                     │
│     (Grafana — make data understandable)           │
├────────────────────────────────────────────────────┤
│  3. STORAGE & QUERY                                │
│     Logs → Elasticsearch/Loki                      │
│     Metrics → Prometheus/Datadog/InfluxDB          │
│     Traces → Jaeger/Zipkin/Tempo                   │
├────────────────────────────────────────────────────┤
│  2. COLLECTION & AGGREGATION                       │
│     (OpenTelemetry Collector, Fluentd, Logstash)   │
├────────────────────────────────────────────────────┤
│  1. INSTRUMENTATION (in your services)             │
│     Structured logs, metric counters, trace spans  │
└────────────────────────────────────────────────────┘
```

**Layer 1 is the most important and the most neglected.** No amount of sophisticated tooling compensates for poor instrumentation. If the service doesn't emit useful data, the rest of the stack has nothing to work with.

---

## 7. OpenTelemetry — The Standard

OpenTelemetry (OTel) is the open-source standard for instrumentation. It provides:
- A vendor-neutral API for emitting logs, metrics, and traces
- SDKs for all major languages
- A collector that receives, processes, and exports telemetry to any backend

```
Service → [OTel SDK] → [OTel Collector] → Prometheus (metrics)
                                        → Jaeger (traces)
                                        → Elasticsearch (logs)
```

**Why OTel matters:** Before OTel, each observability vendor had its own SDK. Switching vendors meant changing instrumentation code throughout the codebase. With OTel, you instrument once against the standard API and switch backends by changing collector configuration.

> **Instrument with OpenTelemetry from day one.** It is now the industry standard and is supported by every major observability vendor.

---

## 8. What Good Observability Looks Like

A well-observed system allows you to answer any of these questions using existing telemetry — without adding new instrumentation:

```
Questions you should always be able to answer:

Performance:
  "What is the P99 latency of the checkout endpoint right now?"
  "Which endpoint is slowest for users in Germany?"

Errors:
  "What percentage of payment requests are failing?"
  "What are the top 3 error types in the last hour?"

Dependencies:
  "Is the database slower than usual?"
  "Which downstream service is causing the most timeout errors?"

Causality:
  "Show me the trace for this specific slow request"
  "What changed at 14:37 that caused error rate to spike?"

Capacity:
  "How close is the DB connection pool to saturation?"
  "At current growth rate, when will we need to scale?"
```

If any of these questions can't be answered, that's a gap in observability to close.

---

## 9. How Large Companies Apply This

| Company | Approach | Source |
|---|---|---|
| **Google** | Originated SRE model with SLIs/SLOs/error budgets; Dapper (distributed tracing precursor) | *Google SRE Book* (public) |
| **Netflix** | Per-service dashboards; automated canary analysis using metrics; Atlas for metrics at scale | Netflix Tech Blog (public) |
| **Uber** | Jaeger for distributed tracing (open-sourced); M3 for metrics at scale | Uber Eng Blog (public) |
| **Cloudflare** | OpenTelemetry adoption across all services; custom metrics pipeline | Cloudflare Blog (public) |

---

## 10. Best Practices

- **Instrument with OpenTelemetry** — vendor-neutral from day one.
- **Correlate all three pillars with a trace ID** — without it you can't connect a log entry to its trace or the metrics spike to its request.
- **Define SLOs before you need them** — not after an incident.
- **Use error budgets** to make reliability vs. feature velocity trade-offs explicit and objective.
- **Alert on symptoms (user impact), not causes (CPU spikes).**
- **Make dashboards per team** — every team should own their service's observability.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Logging without structure | Logs unsearchable at scale | Structured JSON logging always |
| No trace ID propagation | Can't correlate logs and traces for a single request | Inject and propagate trace ID on every request |
| Monitoring without observability | Alerts fire but root cause can't be found | Add rich telemetry alongside threshold-based monitoring |
| No SLOs defined | Reliability is subjective; no basis for prioritisation | Define SLOs early; tie alerts to them |
| Alerting on causes not symptoms | Alert fatigue; irrelevant pages | Alert on user-facing SLO breaches only |

---

## 12. Interview Questions

1. What is the difference between monitoring and observability?
2. What are the three pillars of observability and what does each answer?
3. Why do you need all three pillars? What gaps does each fill?
4. What is an SLI, SLO, and SLA? How do they relate?
5. What is an error budget and how does it affect engineering decisions?
6. What is OpenTelemetry and why does it matter?
7. What questions should a well-observed system always be able to answer?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Observability** | Understand internal state from external outputs. Property of the system, not a tool. |
| **vs Monitoring** | Monitoring = known questions. Observability = any question. Need both. |
| **Three pillars** | Logs (what), Metrics (how many/fast), Traces (where time went). All three needed. |
| **SLI** | What you measure (e.g. success rate). |
| **SLO** | Internal target (e.g. 99.95%). Stricter than SLA. |
| **Error budget** | 100% - SLO. Currency balancing features vs reliability. |
| **OpenTelemetry** | The standard. Instrument once; switch backends freely. |

---

## 14. Cross References

**Prerequisites:** Observability & Maintainability (NFR #8) · Microservices Operational (MS #5)

**Related Topics:** Distributed Tracing (DS series) · Fault Tolerance (NFR #6) · Availability (NFR #2)

**What to Learn Next:** 02-logging.md · 03-metrics-and-alerting.md

---

*System Design Engineering Handbook — 08-Observability Series*