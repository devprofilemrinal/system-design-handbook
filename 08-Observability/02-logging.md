# 02 — Logging

> **08-Observability Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Logging?

A log is a timestamped record of a discrete event that occurred inside a system. Logs capture the narrative — what happened, when, to whom, and with what result.

Unlike metrics (aggregated numbers) or traces (request flows), logs are the raw record. When something goes wrong, logs are where you look to understand exactly what happened.

```
A user's checkout fails at 14:37:22.

Metrics tell you: error rate spiked at 14:37
Trace tells you: the request failed in the payment service after 2.3s
Log tells you: "Payment declined — card number invalid. User u-789.
                Stripe error: card_declined. Request ID: req-abc123."
```

Logs are the *evidence*. Metrics and traces point you to the right service and timeframe; logs tell you what actually happened.

---

## 2. Structured vs Unstructured Logging

The single most impactful logging decision is whether to use structured or plain-text logging.

### Unstructured (Plain Text) — Avoid

```
[2024-01-15 14:37:22] ERROR: Payment failed for user 789 - card declined after 234ms
[2024-01-15 14:37:23] INFO: Retry attempt 1 for user 789
[2024-01-15 14:37:25] ERROR: Payment failed for user 789 - card declined after 198ms
```

**Problems:**
- To find all errors for user 789, you need regex: fragile, slow, inconsistent
- You cannot group by error type, filter by duration, or aggregate across thousands of entries
- Each developer writes a different format — no consistency across services

### Structured (JSON) — Always

```json
{"ts":"2024-01-15T14:37:22Z","level":"ERROR","service":"payment-service",
 "version":"2.4.1","trace_id":"abc-123","span_id":"def-456",
 "user_id":"u-789","event":"payment_failed","provider":"stripe",
 "error_code":"card_declined","duration_ms":234,"attempt":1}

{"ts":"2024-01-15T14:37:23Z","level":"INFO","service":"payment-service",
 "version":"2.4.1","trace_id":"abc-123","span_id":"def-456",
 "user_id":"u-789","event":"payment_retry","attempt":2}
```

**Advantages:**
- Filter by any field: `service=payment-service AND error_code=card_declined`
- Group by: `count errors by error_code for last 1 hour`
- Join with traces by `trace_id`
- Machine-parseable — no regex needed
- Consistent across all services

> **Structured logging is non-negotiable in production systems.** Plain-text logs are unqueryable at scale.

---

## 3. Log Levels — What Each Means

Log levels define the severity of an event. Choosing the right level determines whether your logs are useful or noisy.

| Level | Meaning | Examples | Who Cares |
|---|---|---|---|
| **FATAL** | System cannot continue; imminent crash | Cannot connect to database on startup | Immediate human intervention |
| **ERROR** | Something failed; needs attention | Payment declined, DB query failed, API returned 5xx | Engineering on-call |
| **WARN** | Unexpected but handled; may indicate a problem | Retry succeeded on 3rd attempt, Slow query > 1s | Engineering (next business day) |
| **INFO** | Normal significant event | Request received, User logged in, Job completed | Operational awareness |
| **DEBUG** | Detailed diagnostic information | Function parameters, intermediate values | Developer debugging |
| **TRACE** | Most verbose; step-by-step execution | Every method call and parameter | Only during active debugging sessions |

**Production log level strategy:**

```
Production: INFO level by default
  → ERROR and WARN always emitted (alerts trigger on them)
  → INFO for significant business events
  → DEBUG and TRACE suppressed (too verbose; cost prohibitive at scale)

On-demand debug logging:
  → Dynamically increase to DEBUG for specific service/user/request
  → Return to INFO after investigation
  → Never leave DEBUG enabled globally in production
```

> **The most common logging mistake is treating everything as INFO.** When every line is equally important, nothing is important. ERROR means human intervention needed. INFO means normal operation. WARN means "watch this."

---

## 4. What to Log

The decision of what to log is as important as how to log it.

### Always Log

```
✅ Request received (endpoint, method, user ID, trace ID)
✅ Request completed (status, duration, result)
✅ Every error with full context (what failed, why, what was the state)
✅ Security-relevant events (login, logout, failed auth, permission denied)
✅ State transitions (order status changed, payment processed, user created)
✅ External calls (to downstream services, to external APIs) with result and duration
✅ Configuration loaded on startup (what version, what environment)
✅ Significant business events (new user registered, payment completed)
```

### Never Log

```
❌ Passwords, PIN codes, security questions
❌ Credit card numbers, CVV, bank account numbers
❌ Personally identifiable information (PII) beyond what's operationally necessary
❌ Session tokens, JWT tokens, API keys
❌ Full request/response bodies containing sensitive data
❌ Health check endpoints (they fire hundreds of times per minute; pure noise)
❌ Static asset requests (/favicon.ico, /robots.txt)
```

**The PII problem:** Logs are often retained longer than user data and accessed by more people than production databases. Logging PII creates compliance risks (GDPR right to erasure — you can delete a DB record but log files are immutable).

**Solution for identifying users in logs without logging PII:**

```
Log the user ID (internal identifier), not name/email/address.
If you need to investigate a specific user, look up their ID in the user database.
The log says user_id=u-789; the database maps that to alice@example.com.
```

---

## 5. Contextual Logging — Making Logs Useful

A log entry in isolation is often useless. Its value comes from context — the information needed to understand what was happening when the event occurred.

**Minimum context every log line should carry:**

```json
{
  "ts": "2024-01-15T14:37:22Z",        // when
  "level": "ERROR",                    // severity
  "service": "payment-service",        // which service
  "version": "2.4.1",                  // which version (crucial for debugging deploys)
  "environment": "production",         // which environment
  "trace_id": "abc-123-def-456",       // correlates with distributed trace
  "span_id": "ghi-789",               // position in the trace
  "host": "payment-pod-7d8f9",        // which instance (for multi-instance debugging)
  // then the event-specific fields:
  "event": "payment_failed",
  "user_id": "u-789",
  "error_code": "card_declined",
  "duration_ms": 234
}
```

**Correlation ID / Trace ID** is the single most important context field. It links every log entry for a single user request across all services. Without it, tracing a failing request through 10 services means searching each service's logs separately and manually correlating timestamps.

---

## 6. Centralised Logging

In a distributed system, each service on each instance writes its own log files. You cannot SSH into 50 servers to search logs. You need centralised log aggregation.

```
Service instances                 Aggregation           Storage & Search
Pod 1 (payment v1) ──logs──→
Pod 2 (payment v1) ──logs──→ [Collector/Agent] → [Log Store] → [Search UI]
Pod 3 (payment v2) ──logs──→   (Fluentd/        (Elasticsearch  (Kibana/
Pod 4 (order v1)   ──logs──→    Logstash/        Loki/           Grafana)
Pod 5 (order v1)   ──logs──→    OTel Collector)  Splunk)
```

**Popular stacks:**

| Stack | Components | Notes |
|---|---|---|
| **ELK** | Elasticsearch + Logstash + Kibana | Powerful search; expensive at scale |
| **EFK** | Elasticsearch + Fluentd + Kibana | Fluentd is lighter than Logstash |
| **Grafana Loki** | Loki + Promtail + Grafana | Label-based (like Prometheus for logs); cheaper; less powerful search |
| **Splunk** | Splunk Enterprise | Enterprise standard; powerful; very expensive |
| **Datadog** | Managed service | Full-stack observability; expensive but simple |

> **Loki is increasingly preferred for greenfield systems** due to lower cost and tight Grafana integration. ELK remains the most feature-rich for complex log analysis.

---

## 7. Log Sampling — Managing Volume

High-traffic services can generate millions of log lines per second. Storing every line is expensive. Sampling reduces volume while preserving signal.

### Tail-Based Sampling

Log all requests that resulted in errors or high latency. Sample a percentage of successful fast requests.

```
Rules:
  HTTP 5xx → log 100% (every error is captured)
  P99+ latency → log 100% (every slow request captured)
  HTTP 2xx, normal latency → log 1% (sample to control volume)
```

This captures all interesting events (errors, slow requests) while reducing the volume of uninteresting events (successful fast requests that add no diagnostic value).

### Head-Based Sampling

Decide at the start of a request whether to log it (before the outcome is known). Simple but may miss important slow or failed requests if they are randomly not selected.

> **Tail-based sampling is strongly preferred for production systems.** It guarantees all errors and anomalies are captured, which is what you actually need for debugging.

---

## 8. Log Retention and Cost

Logs are expensive to store and search. Define a retention policy based on business and compliance needs.

| Log Category | Typical Retention | Reason |
|---|---|---|
| Debug/Info | 7–14 days | Operational debugging; short value |
| Error/Warn | 30–90 days | Incident investigation; trend analysis |
| Security/Audit | 1–7 years | Compliance (GDPR, PCI-DSS, SOX) |
| Access logs | 90 days – 1 year | Security investigation, compliance |

**Cost management strategies:**
- Store hot logs (last 7 days) in fast/expensive storage for quick search
- Archive older logs to cheap cold storage (S3 Glacier) for compliance
- Drop DEBUG logs in production; don't generate what you don't need

---

## 9. How Large Companies Handle Logging

| Company | Approach | Source |
|---|---|---|
| **Netflix** | Keystone pipeline processes billions of log events per day; structured JSON throughout | Netflix Tech Blog (public) |
| **LinkedIn** | Kafka as the central log transport; structured logs feed multiple downstream consumers | LinkedIn Eng Blog (public) |
| **Cloudflare** | Processes hundreds of billions of log events daily; custom log pipeline | Cloudflare Blog (public) |
| **Uber** | Centralised logging with Kafka transport; logs treated as events | Uber Eng Blog (public) |

---

## 10. Best Practices

- **Always use structured (JSON) logging** — unstructured logs are unqueryable at scale.
- **Include trace ID on every log line** — it's the thread connecting logs, metrics, and traces.
- **Never log sensitive data** — PII, passwords, tokens, card numbers.
- **Use the right log level** — ERROR means needs attention; INFO means normal; debug turned off in production.
- **Centralise logs** — never rely on SSH + grep for production debugging.
- **Use tail-based sampling** — capture 100% of errors; sample successful requests.
- **Define retention policies** — don't pay to store debug logs for a year.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Unstructured plain-text logs | Unsearchable at scale; brittle regex required | Structured JSON always |
| Logging PII or secrets | Compliance violation; data breach risk | Log user IDs, not names/emails/cards |
| No trace ID in logs | Can't correlate a log entry to a request or trace | Inject trace ID on every request; propagate it |
| DEBUG level in production | Massive log volume; expensive; noise drowns signal | INFO in production; DEBUG on-demand only |
| Everything at INFO level | No signal in the noise; alerts trigger on wrong things | Use levels deliberately |
| No centralised logging | SSH + grep across 50 servers is not operations | Centralised aggregation from day one |
| No retention policy | Storage costs grow unboundedly | Define and enforce retention per log category |

---

## 12. Interview Questions

1. What is the difference between structured and unstructured logging?
2. What should every log line include as minimum context?
3. Why should you never log passwords or PII?
4. What is tail-based log sampling and why is it preferred?
5. What are the ELK and Loki stacks? What are the trade-offs?
6. How does a trace ID make logs more useful?
7. What log level would you use for a payment failure? A retried request that ultimately succeeded?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Structured logging** | JSON format. Queryable, filterable, machine-parseable. Non-negotiable. |
| **Log levels** | ERROR = needs attention. WARN = unexpected but handled. INFO = normal. Debug off in prod. |
| **What to log** | State transitions, errors, external calls, security events. |
| **What not to log** | PII, passwords, tokens, card numbers. Log user IDs only. |
| **Trace ID** | Single most important context field. Correlates all pillars. |
| **Centralised logging** | ELK, Loki, or managed service. SSH + grep is not operations. |
| **Sampling** | Tail-based: capture 100% errors, sample successes. |
| **Retention** | Define per category. Debug = days. Audit = years. |

---

## 14. Cross References

**Prerequisites:** 01-observability-fundamentals.md

**Related Topics:** 03-metrics-and-alerting.md · 04-distributed-tracing.md · Security (NFR #7)

**What to Learn Next:** 03-metrics-and-alerting.md

---

*System Design Engineering Handbook — 08-Observability Series*