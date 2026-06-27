# 05 — Incident Response

> **08-Observability Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Incident Response?

Incident response is the structured process for detecting, investigating, resolving, and learning from system failures. It is the operational side of observability — where all the instrumentation you built actually gets used.

An incident is any unplanned event that degrades or disrupts service quality for users. This includes outages, degraded performance, data integrity issues, and security breaches.

> **Good incident response is what separates systems that recover fast from systems that collapse slowly.** The technical design of your system determines how often things break. Incident response determines how quickly and completely you recover.

---

## 2. Incident Severity Levels

Not every incident is equally urgent. Severity levels allow teams to calibrate their response — from an immediate all-hands page to a next-business-day ticket.

| Severity | Definition | Response | Examples |
|---|---|---|---|
| **SEV-1 (Critical)** | Complete outage or critical data loss. Many users cannot use core functionality. | Immediate response. All-hands if needed. 24/7. | Payment system down, authentication service unavailable, data loss |
| **SEV-2 (Major)** | Significant degradation. Core functionality impacted for many users. | Immediate response. On-call engineer. Business hours team. | Checkout 10× slower, 5% error rate on core API, key feature unavailable |
| **SEV-3 (Minor)** | Partial degradation. Workaround exists. Limited user impact. | Response within hours. Business hours. | Single region slow, non-critical feature broken, elevated error rate on edge case |
| **SEV-4 (Low)** | Cosmetic or minor issue. No user impact yet. | Next business day ticket. | Dashboard shows wrong colour, minor metric anomaly, log noise |

**Severity is determined by user impact, not technical cause.** A database running at 95% CPU is not an incident if users see no degradation. A 0.1% error rate on the payment API is a SEV-1 if it's causing users to lose money.

---

## 3. The Incident Response Lifecycle

Every incident follows this lifecycle, regardless of severity:

```
1. DETECTION
   Alert fires (automated) or user reports (manual)
   → How fast did we find out?

2. ACKNOWLEDGEMENT
   On-call engineer acknowledges the alert
   → Someone is working on it; stops escalation

3. TRIAGE
   Understand scope: what is broken, who is affected, how badly?
   → Assign severity level
   → Decide whether to escalate

4. INVESTIGATION
   Use observability tools to find root cause
   → Metrics: when did it start? What changed?
   → Traces: which service is the bottleneck?
   → Logs: what specific errors are occurring?

5. MITIGATION
   Stop the bleeding — reduce user impact NOW
   (may not fix root cause; buys time)
   → Rollback, disable feature flag, restart service, increase capacity

6. RESOLUTION
   Root cause fixed; service fully restored

7. POST-MORTEM
   Learn from it so it doesn't happen again
```

---

## 4. Detection — Finding Incidents Fast

The faster you detect an incident, the less time users are affected. Detection gaps are expensive.

**Sources of detection (fastest to slowest):**

```
Ideal: Synthetic monitoring / alerting fires before users notice
Good:  Internal alert fires within 1-2 minutes of incident start
Acceptable: User reports within 5-10 minutes
Poor: User complaint reaches support → support escalates → engineering notified (30+ min)
Terrible: Detected by data team during weekly review
```

**Proactive detection tools:**

| Tool | What It Detects |
|---|---|
| **Synthetic monitoring** | Simulated user transactions run every minute; fail before real users notice |
| **Metric alerts** | Error rate, latency, saturation threshold breaches |
| **Anomaly detection** | Statistical deviation from historical baseline (traffic drops, unusual error patterns) |
| **Dead man's switch** | Expects a heartbeat every N minutes; alerts if heartbeat stops (catches silent failures) |
| **Log-based alerts** | Fire when specific error patterns appear in logs |

**The dead man's switch** is particularly valuable. Regular metric alerts only fire when something goes above a threshold. If a service silently stops processing (traffic drops to zero, queue stops being consumed), a normal alert won't fire — the silence is the problem. A dead man's switch alerts when it *doesn't* receive expected events.

```
Dead man's switch:
  "Alert if no successful payment processed in the last 5 minutes"
  → Catches: service crash, queue failure, dependency outage
  → NOT caught by: error rate alert (zero errors because zero requests)
```

---

## 5. Triage — Establishing Scope Quickly

In the first minutes of an incident, the goal is not to find root cause — it's to understand scope and assign severity.

**The three triage questions:**

```
1. What is broken?
   (which service/endpoint/feature is affected)

2. Who is affected?
   (all users? specific region? specific customer segment?)

3. How bad is it?
   (complete outage? degraded performance? data integrity issue?)
```

Answering these three questions takes 2-5 minutes and determines everything else: severity level, who to page, what to communicate to stakeholders.

**Scope signals to check immediately:**

```
Error rate dashboard: spike = wide impact; normal = narrow impact
Traffic dashboard: drop = upstream problem or silent failure
Trace error sampling: which endpoints? which user segments?
Recent deployments: did anything go out in the last hour?
External status pages: is it us or a dependency? (Stripe, AWS, Cloudflare)
```

---

## 6. The On-Call Role

On-call engineers are the first responders for incidents outside business hours. They are alerted by PagerDuty, OpsGenie, or similar tools when a threshold is breached.

**The on-call contract:**

```
On-call engineer responsibilities:
  → Acknowledge alert within 5 minutes (SEV-1/2)
  → Assess severity and scope within 15 minutes
  → Communicate status to stakeholders within 30 minutes
  → Escalate if unable to resolve within defined timeframe
  → Document actions taken in the incident timeline

Team responsibilities to on-call:
  → Runbooks for every common failure mode
  → Meaningful alerts (not noise)
  → Access to all necessary tools and dashboards
  → Escalation path clearly defined
  → Rotation that doesn't burn engineers out
```

**On-call rotation best practices:**
- Minimum 2-week rotation (1 week too short to learn; months too long to sustain)
- Handoff includes open issues and any ongoing degradation
- Compensation (time off or pay) for after-hours incidents
- Regular rotation review: if on-call is constantly painful, the system has an observability or reliability problem

---

## 7. Runbooks — The On-Call's Guide

A runbook is step-by-step documentation for diagnosing and resolving specific failure modes. Runbooks transform incident response from heroic improvisation to repeatable procedure.

**Structure of a good runbook:**

```
Runbook: Payment Service High Error Rate

Symptoms:
  - Alert: "payment_error_rate > 1% for 5 minutes"
  - Users unable to complete checkout

Immediate checks:
  1. Check Stripe status page (status.stripe.com) — is this a provider outage?
  2. Check payment service error logs (Kibana query: service=payment AND level=ERROR)
  3. Check recent deployments (deployment dashboard — last 2 hours)

If Stripe is down:
  → Enable "maintenance mode" feature flag on checkout
  → Page product manager to draft user communication
  → Monitor Stripe status; disable flag when resolved

If recent deployment:
  → Rollback payment-service to previous version
  → Command: kubectl rollout undo deployment/payment-service
  → Monitor error rate for 5 minutes

If neither:
  → Check payment DB connection pool (dashboard: payment-db-connections)
  → If pool exhausted: restart payment-service pods
  → Escalate to payment team lead if unresolved after 15 minutes

Escalation: #incident-channel → @payment-team-lead → VP Engineering
```

**Key properties of a good runbook:**
- Actionable in the first 5 minutes, even for an unfamiliar engineer
- Links directly to dashboards, logs, and commands — no searching required
- Includes escalation path with names and contacts
- Updated after every incident that required improvisation

---

## 8. Incident Communication

Clear communication during an incident reduces panic, sets expectations, and builds trust.

**Internal communication (during the incident):**

```
#incidents Slack channel — running timeline:
  14:37 INCIDENT DECLARED: Payment service error rate >5%
  14:42 INVESTIGATING: Checking Stripe status and recent deployments
  14:48 IDENTIFIED: Stripe API returning 503s; provider incident
  14:50 MITIGATING: Feature flag enabled; checkout showing maintenance message
  15:20 RESOLVED: Stripe recovered; feature flag disabled; monitoring for 30 min
  15:50 ALL CLEAR: Error rate nominal; incident closed
```

**External communication (for user-facing incidents):**
- Status page update within 15 minutes of declaration (even just "investigating")
- Be factual: "We are experiencing elevated error rates on checkout" — not vague ("some users may be experiencing issues")
- Update every 30 minutes even if there's nothing new ("still investigating; no ETA yet")
- Resolution notice with brief explanation

---

## 9. Post-Mortem — Learning From Failures

A post-mortem is a blameless analysis of an incident, conducted after resolution, to understand what happened and prevent recurrence.

**"Blameless" is the most important word.** If engineers fear blame, they hide mistakes, underreport incidents, and don't share what they learned. A blameless culture surfaces more information and produces better outcomes.

**Post-mortem structure:**

```
Incident Summary
  What happened? When? How long? Severity? User impact?

Timeline
  Minute-by-minute: when was it detected, when were key discoveries made,
  when was it mitigated, when was it resolved?

Root Cause Analysis
  Why did this happen? Use the "5 Whys" technique to get to the real root cause.

  5 Whys example:
    Q: Why did users see errors?
    A: Payment service was returning 503s
    Q: Why was payment service returning 503s?
    A: DB connection pool was exhausted
    Q: Why was the pool exhausted?
    A: A new query introduced in v2.4.1 was not using the connection pool correctly
    Q: Why was this not caught?
    A: No load test was run before deployment
    Q: Why was no load test run?
    A: We have no automated load testing in the deployment pipeline
    Root cause: No automated load testing

Contributing Factors
  What made this worse or harder to detect?
  (Missing alert, unclear runbook, deploy at 5pm on a Friday)

Impact
  Users affected, revenue impact, SLO error budget consumed

What Went Well
  Blameless means also crediting good responses

Action Items
  Specific, assigned, with due dates
  NOT vague ("improve monitoring") but concrete ("Add P99 alert for payment DB query time — Owner: Alice — Due: 2024-02-01")
```

**Action items are what make post-mortems valuable.** A post-mortem with no action items is a history lesson. A post-mortem with concrete follow-through is a reliability investment.

---

## 10. How Large Companies Apply This

| Company | Practice | Source |
|---|---|---|
| **Google** | Pioneered blameless post-mortems in SRE; error budgets create shared accountability | *Google SRE Book* (public) |
| **PagerDuty** | Documented incident response practices; runbook-driven response | PagerDuty public docs |
| **Atlassian** | Incident.io and Opsgenie practices; post-mortem templates | Atlassian public docs |
| **Netflix** | Chaos engineering as proactive incident simulation; blameless culture | Netflix Tech Blog (public) |

---

## 11. Best Practices

- **Define severity levels and response times** — and enforce them consistently.
- **Write runbooks for every alert** — if there's no runbook, there shouldn't be an alert.
- **Conduct blameless post-mortems** for every SEV-1 and SEV-2 incident.
- **Track action items from post-mortems** — post-mortems without follow-through are theatre.
- **Use synthetic monitoring** — detect problems before users report them.
- **Add dead man's switch alerts** — catch silent failures that don't trigger threshold alerts.
- **Rotate on-call fairly** — a burned-out on-call is an unreliable on-call.

---

## 12. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No runbooks | On-call improvises; longer MTTR; knowledge stays in one person's head | Runbook for every alert before the alert goes live |
| Blame culture in post-mortems | Engineers hide mistakes; fewer insights; same incidents recur | Blameless post-mortems; frame as system failures not human failures |
| No severity levels | Everything is treated as maximum urgency → engineer burnout | Define severity levels; calibrate response |
| Post-mortems with no action items | Same incident recurs | Every post-mortem ends with concrete, assigned, dated actions |
| Alerts without runbooks | On-call doesn't know what to do; slow response | No alert ships without a runbook |
| Poor external communication | User trust erodes; support overwhelmed | Status page update within 15 minutes; regular updates |

---

## 13. Interview Questions

1. What is an incident and how do you define severity levels?
2. What is the incident response lifecycle?
3. What is a runbook and what makes a good one?
4. What is a blameless post-mortem and why is "blameless" important?
5. What is a dead man's switch alert and what does it catch?
6. How should you communicate externally during a user-facing incident?
7. What is the "5 Whys" technique and how does it apply to root cause analysis?

---

## 14. Summary

| Concept | Key Takeaway |
|---|---|
| **Severity levels** | Calibrate response to user impact. SEV-1 = all hands now. SEV-4 = next week ticket. |
| **Lifecycle** | Detect → Triage → Investigate → Mitigate → Resolve → Post-mortem. |
| **MTTR** | Mean Time To Resolution. Observability, runbooks, and practice reduce it. |
| **Runbook** | Step-by-step guide for specific failure modes. Required for every alert. |
| **Blameless post-mortem** | System analysis, not blame. Required for SEV-1/2. |
| **Action items** | What post-mortems produce. Specific, assigned, dated. |
| **Dead man's switch** | Alerts when expected events *stop* happening. Catches silent failures. |

---

## 15. Cross References

**Prerequisites:** 01-observability-fundamentals.md · 03-metrics-and-alerting.md · Availability (NFR #2)

**Related Topics:** Fault Tolerance (NFR #6) · Durability & DR (NFR #9) · Reliability (NFR #4)

**What to Learn Next:** 09-Security section · 10-System-Design-Patterns

---

*System Design Engineering Handbook — 08-Observability Series*

---

> **08-Observability series complete.**
> Covered: Fundamentals · Logging · Metrics & Alerting · Distributed Tracing · Incident Response