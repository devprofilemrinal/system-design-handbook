# Observability & Maintainability

> **NFR Deep Dive #8** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Overview

**Observability** is the ability to understand a system's internal state from its external outputs — logs, metrics, and traces. **Maintainability** is how easily a system can be changed, debugged, and operated over time.

They're paired because they share a goal: **keeping a system healthy and changeable in production over its lifetime.** Most of a system's life is spent being operated and modified, not built — these NFRs govern that majority.

> **Core idea:** *You cannot fix, operate, or improve what you cannot see.* Observability is what turns "the system is slow, somewhere" into "the payment service's database query P99 spiked at 14:32."

---

## 2. Monitoring vs Observability

A subtle but important distinction.

| | Monitoring | Observability |
|---|---|---|
| **Answers** | "Is the system healthy?" (known questions) | "*Why* is it unhealthy?" (unknown questions) |
| **Approach** | Predefined dashboards & alerts | Explore arbitrary, ad-hoc questions |
| **Detects** | Failures you anticipated | Novel failures you didn't predict |

> Monitoring tells you *something is wrong*. Observability lets you discover *what and why* — including failures no one foresaw. You need both.

---

## 3. The Three Pillars of Observability

```
        ┌─────────────────────────────────────┐
        │   LOGS      METRICS      TRACES      │
        │  (events)  (numbers)  (request path) │
        └─────────────────────────────────────┘
```

| Pillar | What It Is | Answers | Cost |
|---|---|---|---|
| **Logs** | Timestamped records of discrete events | "What exactly happened at 14:32?" | High volume; expensive at scale |
| **Metrics** | Numeric measurements over time (aggregates) | "What's the error rate / latency trend?" | Cheap; efficient to store |
| **Traces** | The path of one request across all services | "Where did this request spend its time?" | Medium; needs instrumentation |

### 3.1 Logs

Discrete event records. Use **structured logging** (JSON key-value) — not plain text — so logs are searchable and machine-parseable.

```
❌ Plain:      "User 123 login failed at 14:32"
✅ Structured: {"event":"login_failed","user_id":123,"ts":"14:32","ip":"…"}
```

| Level | Use For |
|---|---|
| ERROR | Something failed and needs attention |
| WARN | Unexpected but handled |
| INFO | Normal significant events |
| DEBUG | Detailed diagnostic info (usually off in prod) |

> Centralize logs (ELK, Loki, Splunk) — searching across nodes by SSH-ing into each is not viable at scale.

### 3.2 Metrics

Numeric, aggregatable, cheap to store. The four "golden signals" (Google SRE) cover most needs:

| Golden Signal | Measures |
|---|---|
| **Latency** | How long requests take (track P50/P99, see NFR #1) |
| **Traffic** | Demand on the system (RPS) |
| **Errors** | Rate of failed requests |
| **Saturation** | How "full" resources are (CPU, memory, disk) |

> If you track only four things, track these. They catch the vast majority of problems.

### 3.3 Traces

A trace follows a single request across every service it touches, via a propagated **trace ID**. Essential for microservices, where one request fans out across many services.

```
Request abc-123:
  API Gateway      [2ms]
    └→ Auth Service    [15ms]
    └→ Order Service   [120ms]  ← the bottleneck is here
         └→ Database      [110ms]  ← actually here

Without tracing: "the request was slow."
With tracing: "110ms was spent in the order DB query."
```

---

## 4. Alerting

Metrics are only useful if someone is notified when they go wrong.

| Principle | Why |
|---|---|
| **Alert on symptoms, not causes** | Alert "users see errors," not "CPU is 80%" (high CPU may be fine) |
| **Alert on SLO violations** | Tie alerts to user-facing impact and error budgets |
| **Make alerts actionable** | Every alert should require a human action; otherwise it's noise |
| **Avoid alert fatigue** | Too many alerts → people ignore them → real ones get missed |

> **Page a human only for things a human must act on now.** Everything else is a dashboard or a ticket, not a 3 AM page.

---

## 5. Maintainability — Operability & Evolvability

Maintainability has two faces: how easily you can **operate** the system, and how easily you can **change** it.

| Aspect | Means | Enabled By |
|---|---|---|
| **Operability** | Easy to run, deploy, recover | Automation, good observability, runbooks |
| **Evolvability** | Easy to change safely | Modularity, loose coupling, tests, clear interfaces |
| **Simplicity** | Easy to understand | Avoiding accidental complexity, good abstractions |

### Key practices

| Practice | Benefit |
|---|---|
| **CI/CD** | Automated, repeatable, low-risk deployments |
| **Infrastructure as Code** | Reproducible environments; no manual drift |
| **Modularity / loose coupling** | Change one part without breaking others |
| **Automated testing** | Change confidently; catch regressions early |
| **Documentation & runbooks** | New engineers onboard; on-call can respond fast |
| **Zero-downtime deploys** | Ship without outages (blue-green, canary, rolling) |

---

## 6. Safe Deployment Strategies

Since most outages come from deployments (see Availability NFR), *how* you ship matters as much as what.

| Strategy | How It Works | Benefit |
|---|---|---|
| **Rolling** | Replace instances gradually | No downtime; slow rollback |
| **Blue-Green** | Two full environments; switch traffic instantly | Instant rollback; doubles infra cost |
| **Canary** | Route a small % of traffic to the new version first | Catch problems before full rollout |
| **Feature flags** | Toggle features on/off without redeploying | Decouple deploy from release; instant kill switch |

> **Canary + feature flags are the safest combo:** ship to 1% of users, watch the metrics, then ramp up — and kill instantly if something breaks.

---

## 7. How Large Companies Apply This

| Company | Application | Source |
|---|---|---|
| **Google** | Originated the "four golden signals"; SRE model ties alerts to SLOs/error budgets | *Google SRE Book* (public) |
| **Netflix** | Heavy canary analysis; automated rollback on bad metrics | Netflix Tech Blog (public) |
| **Most orgs** | OpenTelemetry as the standard for traces/metrics; Prometheus + Grafana dashboards | Public/open-source standards |

> **Inferred:** Internal tooling varies; the practices (golden signals, distributed tracing, canary) are publicly documented and widely adopted.

---

## 8. Best Practices

- **Instrument all three pillars** — logs, metrics, and traces are complementary, not redundant.
- **Use structured logging** so logs are searchable at scale.
- **Track the four golden signals** as a baseline for every service.
- **Propagate a trace ID** through every service for end-to-end visibility.
- **Alert on user-facing symptoms** tied to SLOs, not raw resource numbers.
- **Ship safely** — canary or blue-green with automated rollback.
- **Automate everything operational** — CI/CD, IaC, self-healing.
- **Write runbooks** so on-call can act without tribal knowledge.

---

## 9. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Logs only, no metrics/traces | Can't see trends or cross-service flow | Instrument all three pillars |
| Unstructured plain-text logs | Unsearchable; useless at scale | Structured (JSON) logging, centralized |
| Alerting on causes not symptoms | Noisy, ignorable alerts | Alert on user impact / SLO breaches |
| Alert fatigue | Real alerts missed in the noise | Make every alert actionable |
| No distributed tracing in microservices | "It's slow somewhere" — can't pinpoint | Propagate trace IDs |
| Manual, risky deployments | Outages from bad releases | CI/CD + canary + rollback |
| No runbooks | Slow incident response, high MTTR | Document common failures and responses |

---

## 10. Interview Questions

1. What are the three pillars of observability?
2. Difference between monitoring and observability?
3. What are the four golden signals?
4. Why is distributed tracing essential in a microservices architecture?
5. Why alert on symptoms rather than causes?
6. Compare blue-green, canary, and rolling deployments.
7. How do feature flags improve maintainability and safety?
8. Why is structured logging better than plain-text logging?

---

## 11. Summary

| Concept | Key Takeaway |
|---|---|
| **Observability** | Understand internal state from outputs. Can't fix what you can't see. |
| **3 Pillars** | Logs (events), Metrics (numbers), Traces (request path). |
| **Golden Signals** | Latency, Traffic, Errors, Saturation. |
| **Monitoring vs Obs.** | Known questions vs exploring unknown ones. |
| **Alerting** | On symptoms/SLOs, actionable, avoid fatigue. |
| **Maintainability** | Operability + evolvability; automate, modularize, test. |
| **Safe deploys** | Canary + feature flags + automated rollback. |

---

## 12. Cross References

**Prerequisites:** System Design Fundamentals · Availability (NFR #2) · Latency & Throughput (NFR #1)

**Related Topics:** SLA/SLO/SLI · CI/CD · Microservices · Distributed Tracing

**What to Learn Next:** Disaster Recovery (NFR #9)

---

*System Design Engineering Handbook — NFR Series*