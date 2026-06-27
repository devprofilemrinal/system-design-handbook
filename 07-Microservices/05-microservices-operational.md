# 05 — Microservices Operational Concerns

> **07-Microservices Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. The Operational Reality

Microservices shift complexity from the application (where a monolith has it) to the infrastructure and operations layer. A monolith has one deployment pipeline, one log stream, one binary to monitor. A system of 50 microservices has 50 pipelines, 50 log streams, 50 binaries — each with its own failure modes, version, and lifecycle.

This is the microservices tax paid in operations. Managing it well is what separates a working microservices system from a chaotic one.

> **The rule:** if you can't operate microservices well, you don't have microservices — you have a distributed monolith that's hard to debug and painful to deploy.

---

## 2. Independent Deployability — The Core Requirement

The primary benefit of microservices is that each service can be deployed independently. If this property is lost, the system becomes a distributed monolith — all the complexity of microservices with none of the deployment benefits.

**Independent deployability requires:**

- **No shared databases** — if Service A and B share a database, a schema change in one affects the other; they cannot deploy independently
- **Backward-compatible API changes** — deploying a new version of Service A must not break existing consumers of A
- **No distributed big-bang deployments** — no "we must deploy services A, B, C, and D simultaneously for this feature to work"

```
INDEPENDENT DEPLOYMENT (correct):
  Team A deploys User Service v2.3 on Monday
  Team B deploys Order Service v4.1 on Tuesday
  (no coordination needed between teams)

COUPLED DEPLOYMENT (wrong — distributed monolith):
  "We need to deploy User v2, Order v3, and Payment v5 all at the same time"
  (defeats the purpose of microservices entirely)
```

---

## 3. Deployment Strategies

Each service goes through its own deployment pipeline. Safe deployment strategies minimise risk for each individual service.

### Rolling Deployment

Replace instances one at a time. At any point during the deployment, both old and new versions are serving traffic.

```
Initial:  [v1] [v1] [v1] [v1]
Step 1:   [v2] [v1] [v1] [v1]
Step 2:   [v2] [v2] [v1] [v1]
Step 3:   [v2] [v2] [v2] [v1]
Final:    [v2] [v2] [v2] [v2]
```

**Key requirement:** v1 and v2 must be compatible during the transition — the same requests will hit both versions simultaneously.

### Blue-Green Deployment

Maintain two identical environments. Deploy to the idle one (green). Switch traffic instantly when ready. If something breaks, switch back immediately.

```
Blue (live):  [v1] [v1] [v1] ← all traffic here
Green (idle): [v2] [v2] [v2] ← deploy here first

Test green → switch traffic → green is now live, blue is standby
If problem: switch back to blue instantly
```

**Cost:** Requires 2× infrastructure during the transition. Worth it for services where instant rollback is critical.

### Canary Deployment

Route a small percentage of traffic to the new version. Monitor. If metrics are healthy, gradually increase the percentage.

```
Start: 100% → v1
Canary phase 1: 95% → v1, 5% → v2 (monitor for 30 min)
Canary phase 2: 80% → v1, 20% → v2 (monitor for 1 hour)
Canary phase 3: 50% → v1, 50% → v2 (monitor for 1 hour)
Full rollout: 100% → v2
Problem detected: 100% → v1 (immediate rollback)
```

**Canary is the safest strategy for high-traffic services.** Problems affect only a fraction of users before they're caught. Most large companies use canary as their default.

### Feature Flags

Decouple deployment from release. Deploy code with new features disabled. Enable them without deploying.

```
Deploy Service v2 with flag "new-checkout-flow" = OFF
  → All users see old checkout flow

Enable flag for 1% of users (internal beta)
  → 1% see new flow; 99% see old

Ramp to 100% when confident
  → Instant global enable; no deployment needed

Problem detected: toggle OFF instantly
  → No rollback deployment needed
```

**Feature flags** give you the fastest rollback of any deployment strategy — a configuration change with no deployment. They also enable A/B testing and gradual rollouts at the user level.

---

## 4. API Versioning

Because services are consumed by other services and external clients, changing an API without breaking consumers requires versioning.

### Versioning Strategies

**URL versioning** — version in the path:

```
/api/v1/users/123    (old contract)
/api/v2/users/123    (new contract with breaking changes)
```

Simple, visible, easy to route. Both versions run simultaneously until v1 consumers migrate.

**Header versioning** — version in request header:

```
GET /users/123
Accept: application/vnd.company.v2+json
```

Cleaner URLs, but less visible and harder to test in a browser.

**Backward-compatible changes (no version bump needed):**
- Adding new optional fields to a response
- Adding new optional request parameters
- Adding new endpoints

**Breaking changes (require a new version):**
- Removing a field from a response
- Changing a field's type or name
- Changing required parameters

### The Expand-Contract Pattern

For API changes that would break consumers, use a two-phase approach:

```
Phase 1 (Expand):
  Add new field alongside the old one
  old_field: "2024-01-15"        (keep for compatibility)
  new_field: "2024-01-15T00:00:00Z"  (add the new format)

Phase 2 (consumers migrate):
  All consumers update to use new_field

Phase 3 (Contract):
  Remove old_field from the API
  (only safe after all consumers have migrated)
```

This prevents the need for a hard cutover — consumers can migrate at their own pace.

---

## 5. Health Checks and Readiness

Deployment pipelines and load balancers need to know whether a service instance is healthy and ready to receive traffic.

### Liveness vs Readiness

| Check | Question | Failure action |
|---|---|---|
| **Liveness** | Is the process alive? (not crashed, not deadlocked) | Restart the container |
| **Readiness** | Is the service ready to receive traffic? | Remove from load balancer rotation |

```
/health/live   → "yes, I'm running" (simple; just checks the process responds)
/health/ready  → "yes, I'm ready for traffic" (checks DB connection, cache, dependencies)
```

A service may be **live** (not crashed) but not **ready** (database connection not yet established during startup). The readiness check prevents traffic from being sent to instances that aren't ready to handle it.

**Deep readiness check:**

```
/health/ready checks:
  ✅ Database connection pool: connected
  ✅ Redis cache: connected
  ✅ Critical config: loaded
  ❌ Downstream payment service: unavailable
  → Returns 503 Service Unavailable
  → Load balancer removes this instance from rotation
  → Traffic rerouted to healthy instances
```

---

## 6. Configuration Management

Microservices must not have configuration hardcoded. Different environments (dev, staging, production) need different values. Configuration must be changeable without redeployment.

### Approaches

| Approach | How | When |
|---|---|---|
| **Environment variables** | Injected at runtime by orchestrator | Simple config; twelve-factor app standard |
| **Config files** | Mounted as files into container | Structured config; can be versioned |
| **Config service** | Centralised config store (Consul, AWS Parameter Store) | Dynamic config that changes at runtime |
| **Secret manager** | Encrypted store for sensitive values (Vault, AWS Secrets Manager) | Passwords, API keys, certificates — never in env vars for production |

> **Never store secrets in code, config files committed to git, or plain environment variables in production.** Use a dedicated secret manager with access control and rotation policies.

---

## 7. Service-to-Service Authentication

In a microservices system, services call each other. How does Service B know that an inbound request is genuinely from Service A and not from an attacker who gained network access?

### Mutual TLS (mTLS)

Both the client and server present certificates. Each verifies the other's identity before the connection is established.

```
Service A calls Service B:
  A presents its certificate: "I am Service A"
  B verifies A's certificate against trusted CA
  B presents its certificate: "I am Service B"
  A verifies B's certificate
  Encrypted, mutually authenticated connection established
```

mTLS ensures:
- Only authorised services can call each other
- Traffic is encrypted in transit
- No service can impersonate another

Managing certificates across hundreds of services is complex — this is another reason service meshes (Istio) are valuable; they handle mTLS automatically.

### JWT Service Tokens

Services exchange signed JWT tokens to authenticate.

```
Service A calls Service B:
  A's request includes header: Authorization: Bearer <service-jwt>
  B validates: "Is this JWT signed by our trusted key? Is it for Service A?"
  B checks: "Is Service A allowed to call this endpoint?"
  → Allow or reject
```

---

## 8. Operational Observability in Microservices

Observability is covered in depth in `01-NFR/observability-maintainability.md`. The specific microservices challenge is scale: 50 services × multiple instances each = thousands of log streams and metric sources to correlate.

**The minimum viable observability stack for microservices:**

```
Every service emits:
  Logs:    structured JSON with trace_id, service_name, version, environment
  Metrics: request_count, error_rate, latency_p50/p99, saturation
  Traces:  distributed trace with parent/child span relationships

Central platform collects:
  Logs    → Centralised log store (ELK, Loki)
  Metrics → Time-series DB (Prometheus, Datadog)
  Traces  → Distributed trace store (Jaeger, Zipkin)

Dashboards: Per-service AND end-to-end request view
Alerts:     On SLO breaches, not raw metrics
```

**The trace ID is the thread that ties it all together.** Without it, correlating a slow user request across 10 services is nearly impossible.

---

## 9. The Twelve-Factor App

The Twelve-Factor App methodology (Heroku, 2011) defines best practices for building services that are portable, scalable, and operationally manageable. It is widely treated as the standard for microservice implementation.

| Factor | Principle |
|---|---|
| **Codebase** | One codebase per service, tracked in version control |
| **Dependencies** | Explicitly declare all dependencies; no implicit system-level dependencies |
| **Config** | Config in environment variables, not code |
| **Backing services** | Treat databases, caches, queues as attached resources (replaceable) |
| **Build/Release/Run** | Strict separation of build, release, and run stages |
| **Processes** | Stateless processes; state in backing services |
| **Port binding** | Services are self-contained; expose via port binding |
| **Concurrency** | Scale by adding processes, not threads |
| **Disposability** | Fast startup; graceful shutdown |
| **Dev/prod parity** | Keep development and production as similar as possible |
| **Logs** | Treat logs as event streams; don't manage log files |
| **Admin processes** | Run admin tasks as one-off processes |

> **Factor 6 (stateless processes) is the most important for microservices.** A stateless service can be freely scaled, replaced, or killed — no session data is lost because there is no session data in the process. State lives in Redis, in the database, in the message queue.

---

## 10. Best Practices

- **Canary deploys as the default** — the safest deployment strategy for live services.
- **Feature flags for instant rollback** — decouples deployment from release.
- **Liveness and readiness checks on every service** — let orchestrators manage health automatically.
- **Config in environment; secrets in a secret manager** — never hardcode either.
- **mTLS for service-to-service authentication** — especially in a service mesh.
- **Structured logging with trace IDs** — the foundation of debuggability in microservices.
- **Stateless services** — all state in backing stores; services can be killed and replaced freely.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Coupled deployments ("deploy A, B, C together") | No independence; distributed monolith | Redesign service boundaries; enforce backward compatibility |
| Breaking API changes without versioning | Consumers break on service deployment | Versioned APIs; expand-contract pattern |
| Hardcoded config or secrets | Different environment configs require code changes; secret exposure risk | Environment variables + secret manager |
| No readiness check | Traffic sent to instances not yet ready | Liveness + readiness endpoints on every service |
| Stateful service instances | Can't scale or replace freely | All state to external stores |
| No distributed tracing | Debugging cross-service requests is impossible | Trace ID propagation from day one |

---

## 12. Interview Questions

1. What is independent deployability and why does losing it create a distributed monolith?
2. Compare rolling, blue-green, and canary deployment strategies.
3. What is the difference between a liveness and readiness health check?
4. What is the expand-contract pattern for API versioning?
5. Why should services be stateless? Where does the state go?
6. What is mTLS and why is it used for service-to-service authentication?
7. What are the minimum observability requirements for a production microservice?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Independent deployability** | The core property. Lost = distributed monolith. |
| **Canary deployment** | Gradual traffic shift. Safest for production services. |
| **Feature flags** | Decouple deploy from release. Fastest rollback. |
| **Liveness vs readiness** | Alive = not crashed. Ready = can serve traffic. Both needed. |
| **API versioning** | Expand-contract for backward compatibility. Never break consumers. |
| **Stateless services** | State in backing stores. Enables free scaling and replacement. |
| **Secrets** | Secret manager always. Never in code or plain env vars. |
| **Observability** | Structured logs + metrics + traces with trace ID propagation. |

---

## 14. Cross References

**Prerequisites:** 01-microservices-fundamentals.md · 03-inter-service-communication.md · 04-microservices-patterns.md

**Related Topics:** Observability & Maintainability (NFR #8) · Security (NFR #7) · Availability (NFR #2)

**What to Learn Next:** 08-Observability section · 09-Security section · 10-System-Design-Patterns

---

*System Design Engineering Handbook — 07-Microservices Series*

---

> **07-Microservices series complete.**
> Covered: Fundamentals · Service Decomposition · Inter-Service Communication · Microservices Patterns · Operational Concerns