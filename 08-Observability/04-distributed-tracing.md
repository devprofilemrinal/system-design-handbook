# 04 — Distributed Tracing

> **08-Observability Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. The Problem Tracing Solves

In a monolith, debugging a slow request is straightforward: look at the code path, add some timing, find the slow function. In a microservices system with 20+ services, a single user request may touch 8 different services across 15 network hops. When that request is slow or fails, where do you look?

```
User reports: "My checkout took 8 seconds"

Without tracing:
  → Check Order Service logs: nothing obvious
  → Check Payment Service logs: found an error!
  → But what happened before? Was Order Service slow before calling Payment?
  → Check Inventory Service logs: did it get called? When?
  → SSH into 8 servers, grep through logs, manually correlate by timestamp
  → 2 hours to find root cause

With tracing:
  → Search trace_id for this user's request
  → Single view: Order Service (20ms) → Payment Service (7.8s) → Stripe API (7.6s)
  → Root cause: Stripe API is slow
  → 2 minutes to find root cause
```

Distributed tracing creates a single, unified view of a request's journey across all services — showing exactly where time was spent and where errors occurred.

---

## 2. Core Concepts

### Trace

A trace represents the complete journey of a single request through the entire system, from the moment the user initiates it to the moment the response is returned. Every trace has a unique **trace ID**.

```
Trace ID: abc-123-def-456
Total duration: 450ms

This trace contains everything that happened to handle the user's request.
```

### Span

A span represents a single unit of work within a trace — one operation in one service. A trace is composed of many spans, each representing one step in the processing chain.

```
Trace abc-123 contains:
  Span 1: API Gateway receives request            [0ms → 5ms]
  Span 2: Order Service processes request         [5ms → 420ms]
    Span 2a: Order Service queries DB             [8ms → 16ms]
    Span 2b: Order Service calls Payment Service  [20ms → 410ms]
      Span 3: Payment Service processes           [25ms → 405ms]
        Span 3a: Payment Service calls Stripe     [30ms → 400ms]
        Span 3b: Payment Service writes DB        [402ms → 412ms]
  Span 4: API Gateway sends response             [420ms → 450ms]
```

Spans have:
- **Span ID:** Unique identifier for this span
- **Parent Span ID:** The span that initiated this one (creates the tree structure)
- **Start time and duration**
- **Service name and operation name**
- **Tags/attributes:** Key-value metadata (HTTP method, status code, DB query)
- **Events:** Timestamped log entries attached to the span
- **Status:** OK, Error, or Unset

### Parent-Child Relationship

Spans are related hierarchically. When Service A calls Service B, the call creates a child span whose parent is A's current span.

```
API Gateway span (root)
└── Order Service span (child of Gateway)
    ├── DB query span (child of Order Service)
    └── Payment Service span (child of Order Service)
        ├── Stripe API call span (child of Payment Service)
        └── DB write span (child of Payment Service)
```

This tree structure is called the **span tree** or **trace tree**. Visualised as a flame graph, it shows the causal chain and where time was spent at each level.

---

## 3. Context Propagation — How Traces Cross Service Boundaries

The most important mechanism in distributed tracing. When Service A calls Service B, it must pass the trace context (trace ID, parent span ID) to B so B can attach its spans to the same trace.

```
Service A handles a request:
  → Creates new span (span_id=span-A, parent=none)
  → Injects context into outbound HTTP headers:
    traceparent: 00-abc123def456-spanA-01
    (trace_id=abc123def456, parent_span_id=spanA, flags=01)

Service B receives the call:
  → Reads traceparent header
  → Creates new span (span_id=span-B, parent=spanA)
  → Attaches span to the existing trace abc123def456

Service B calls Service C:
  → Injects updated context: parent_span_id=spanB
  → Service C creates span with parent=spanB

Result: All spans share trace_id=abc123def456 → one unified trace
```

**W3C TraceContext** is the standard HTTP header format for context propagation. OpenTelemetry uses and enforces it.

**What happens if propagation breaks?**

```
Service A → calls → Service B (but forgets to inject headers)
                                ↓
                  Service B creates a NEW trace with a new trace_id
                  B's work is invisible to A's trace

→ The trace is broken. You see A's half and B's half as two separate traces.
→ You cannot tell they were part of the same user request.
```

Context propagation must work correctly across every service boundary. One broken link in the chain breaks the entire trace.

---

## 4. Trace Anatomy — A Real Example

```
Trace: abc-123 | User: checkout request | Total: 450ms
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

API Gateway             ████████████████████████████████░  [0–5ms]
  Order Service           ████████████████████████████░   [5–420ms]
    SELECT order items       ██░                           [8–16ms]
    Call Payment Service       ████████████████████████░  [20–410ms]
      Payment logic              █░                        [25–30ms]
      Call Stripe API              ████████████████████░   [30–400ms]
      ← Stripe returns             response               [400ms]
      Write payment record             ██░                 [402–412ms]
  Return to API Gateway               ░                   [420ms]
API Gateway sends response                             ░  [420–450ms]

Bottleneck identified: Stripe API call = 370ms (82% of total time)
```

This flame graph view makes the bottleneck immediately obvious — no log grepping required.

---

## 5. Sampling — Managing Trace Volume

A high-traffic service handles thousands of requests per second. Storing a complete trace for every request would be prohibitively expensive. Sampling reduces volume.

### Head-Based Sampling

The decision to sample is made at the **start** of the trace, before any processing happens.

```
Sampling rate: 10%
→ 10% of all requests generate a full trace
→ 90% are dropped immediately (no trace data stored)
```

**Problem:** You don't know at the start whether this request will be interesting (slow, error). A 10% sample may miss the very slow requests that caused your P99 to spike.

### Tail-Based Sampling

The full trace is collected for every request. The decision to keep or discard is made **after** the trace completes, based on what happened.

```
Rules:
  Keep 100%: any trace with an error
  Keep 100%: any trace with P99+ latency
  Keep 1%:   all other traces (successful, normal latency)
```

This guarantees every error and every anomaly is captured while keeping storage costs manageable.

**Tail-based sampling requires a collector that can buffer complete traces** before making the keep/discard decision.

> **Tail-based sampling is the correct approach for production systems.** Head-based sampling is simpler to implement but may miss the exact requests you need for debugging.

### Sampling Strategy in Practice

```
Low-volume service (<100 RPS): sample 100% — cost is manageable
Medium-volume (100–10K RPS): tail-based — keep all errors + 10% of successes
High-volume (>10K RPS): tail-based — keep all errors + 1% of successes
Very high-volume: adaptive sampling — dynamically adjust rate based on current volume
```

---

## 6. OpenTelemetry — The Standard

OpenTelemetry is the open-source framework for distributed tracing (and all three observability pillars). It provides:

- **API:** Vendor-neutral interfaces for creating spans, counters, gauges
- **SDK:** Language-specific implementations of the API
- **Collector:** Receives telemetry from services, processes/samples it, exports to backends
- **Semantic conventions:** Standardised attribute names (e.g., `http.method`, `db.system`)

```
Service code                    OTel Collector          Backends
──────────────                  ──────────────          ────────
[OTel SDK]                      ┌──────────────┐
  creates spans  →   export  →  │   Receive    │ → Jaeger (traces)
  records metrics               │   Filter     │ → Prometheus (metrics)
  emits logs                    │   Sample     │ → Elasticsearch (logs)
                                │   Export     │
                                └──────────────┘
```

**Why OpenTelemetry matters:**

Before OTel, every tracing vendor had its own SDK (Jaeger client, Zipkin client, Datadog agent). Switching vendors meant changing instrumentation code throughout the codebase. With OTel, you instrument once against the standard API and export to any backend by changing collector configuration.

> **Instrument with OpenTelemetry from day one.** It is now the industry standard, CNCF graduated, and supported by every major observability vendor.

---

## 7. Tracing Tools

| Tool | Type | Notes |
|---|---|---|
| **Jaeger** | Open-source | CNCF project; open-sourced by Uber; widely adopted |
| **Zipkin** | Open-source | Original Twitter project; simpler than Jaeger |
| **Tempo (Grafana)** | Open-source | Integrates natively with Grafana and Loki; cost-efficient |
| **Datadog APM** | Managed | Full-stack; expensive; excellent UX |
| **Honeycomb** | Managed | Designed for high-cardinality trace analysis |
| **AWS X-Ray** | Managed | AWS-native; integrates with AWS services |

---

## 8. What to Add to Spans — Practical Guidance

A span's value depends on the attributes attached to it. Too few attributes make the trace useless for debugging; too many add overhead.

**HTTP service spans — minimum attributes:**

```
http.method = "POST"
http.url = "/api/checkout"
http.status_code = 500
http.request_content_length = 1247
user.id = "u-789"             (not email — just the ID)
error = true
error.message = "Payment declined"
```

**Database spans — minimum attributes:**

```
db.system = "postgresql"
db.name = "orders"
db.operation = "SELECT"
db.statement = "SELECT * FROM orders WHERE user_id = ?"  (parameterised — no values)
```

**External API spans:**

```
peer.service = "stripe"
http.url = "https://api.stripe.com/v1/charges"
http.method = "POST"
http.status_code = 429
```

---

## 9. How Large Companies Use Distributed Tracing

| Company | Tool | Application | Source |
|---|---|---|---|
| **Uber** | Jaeger (open-sourced it) | Traces millions of rides/requests per day; critical for microservices debugging | Uber Eng Blog (public) |
| **Netflix** | Edgar (internal, built on Zipkin) | Traces streaming session quality, API latency | Netflix Tech Blog (public) |
| **Twitter/Zipkin** | Zipkin (open-sourced it) | Original distributed tracing system; traces across hundreds of services | Twitter/X Eng Blog (public) |
| **Cloudflare** | OpenTelemetry + Tempo | All services instrumented with OTel; traces exported to Tempo | Cloudflare Blog (public) |

---

## 10. Best Practices

- **Propagate context across every service boundary** — one broken link breaks the entire trace.
- **Use tail-based sampling** — capture all errors and anomalies, sample successes.
- **Instrument with OpenTelemetry** — vendor-neutral; switch backends without code changes.
- **Add meaningful attributes to spans** — operation name alone is insufficient for debugging.
- **Never log sensitive data in span attributes** — trace data often has broader access than application logs.
- **Correlate traces with logs** — ensure trace ID appears in both trace spans and log entries.
- **Build trace-based dashboards** — P99 latency by service, error rate by trace root cause.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Broken context propagation | Traces split across services; cannot see end-to-end | Test context propagation at every service boundary |
| Head-based sampling only | Miss the slow/failing requests you need most | Tail-based sampling for production |
| No attributes on spans | Traces show structure but no diagnostic information | Add HTTP method, status, user ID, error details |
| Not correlating with logs | Trace points to Payment Service; logs have no trace ID | Inject trace ID into every log line |
| Logging sensitive data in spans | Security/compliance risk | Log IDs, not PII or secrets |
| Manual instrumentation only | Inconsistent; tedious; often incomplete | Auto-instrumentation via OTel SDK + manual for business logic |

---

## 12. Interview Questions

1. What is a trace and what is a span? How do they relate?
2. What is context propagation and what happens if it breaks?
3. What is the difference between head-based and tail-based sampling?
4. Why is tail-based sampling preferred for production systems?
5. What is OpenTelemetry and why is it important?
6. How does distributed tracing help debug a slow request across 10 services?
7. What attributes should a database span include?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Trace** | Complete journey of one request. Identified by trace ID. |
| **Span** | One unit of work in one service. Spans form a tree. |
| **Context propagation** | Passing trace ID across service boundaries. One broken link = broken trace. |
| **Head sampling** | Decision at start. Simple but misses anomalies. |
| **Tail sampling** | Decision after completion. Captures all errors. Production standard. |
| **OpenTelemetry** | The standard. Instrument once; export anywhere. |
| **Flame graph** | Visualises span tree. Makes bottleneck instantly visible. |

---

## 14. Cross References

**Prerequisites:** 01-observability-fundamentals.md · 03-metrics-and-alerting.md · Microservices Operational (MS #5)

**Related Topics:** 02-logging.md · Consistency & Clocks (DS #3) · Fault Tolerance (NFR #6)

**What to Learn Next:** 05-incident-response.md

---

*System Design Engineering Handbook — 08-Observability Series*