# 03 — Inter-Service Communication

> **07-Microservices Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. The Core Problem

In a monolith, services call each other via in-process function calls — fast, reliable, no network involved. In microservices, every inter-service call crosses a network. This fundamentally changes the reliability and performance characteristics.

```
MONOLITH: OrderService.getUser(userId)
  → in-process call → returns in microseconds → never fails

MICROSERVICES: HTTP GET /users/123
  → network call → takes 1-50ms → can fail, time out, be delayed
```

Every design decision in inter-service communication is about managing this gap: how to get the reliability and performance you need despite an unreliable network between services.

---

## 2. Synchronous vs Asynchronous — The Primary Choice

Every inter-service call is either synchronous (the caller waits for a response) or asynchronous (the caller fires and moves on). This choice has deep consequences for your system's reliability and coupling.

| | Synchronous | Asynchronous |
|---|---|---|
| **Caller blocks** | Yes — waits for response | No — continues immediately |
| **Latency** | Compounds across call chains | Decoupled; absorbed by queue |
| **Coupling** | Both services must be available | Producer and consumer independent |
| **Error handling** | Immediate; caller handles failure | Complex; needs DLQ and retry |
| **Use when** | Caller needs the result to continue | Caller doesn't need an immediate result |

> **Default to asynchronous for anything that doesn't require an immediate response.** Sending a confirmation email, updating analytics, processing images, reserving inventory — none of these need to block the user's request. Making them synchronous couples your services unnecessarily.

---

## 3. Synchronous Communication — REST and gRPC

### REST (HTTP + JSON)

The most common choice. Simple, universal, human-readable.

```
Request:
  GET /users/123 HTTP/1.1
  Host: user-service
  Authorization: Bearer <token>

Response:
  200 OK
  {"id": 123, "name": "Alice", "email": "alice@mail.com"}
```

**Strengths:** Every language supports HTTP. Easy to debug (curl, browser). Widely understood. Works through firewalls and proxies.

**Weaknesses:** JSON serialisation overhead. HTTP/1.1 is text-based (verbose). No built-in schema enforcement (unless OpenAPI is added). Not suited for streaming or bidirectional communication.

**Use REST for:** Public APIs, external partners, teams that need maximum interoperability, services where human readability matters.

### gRPC (HTTP/2 + Protocol Buffers)

A high-performance RPC framework from Google. Uses a binary protocol and strongly-typed schema definitions.

```
// Service definition (language-agnostic schema)
service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc ListUsers (ListUsersRequest) returns (stream User);
}

message GetUserRequest { int64 user_id = 1; }
message User { int64 id = 1; string name = 2; string email = 3; }
```

The schema generates client and server code in any supported language. The binary Protocol Buffer encoding is 3–10× smaller than equivalent JSON.

**Strengths:** Fast (binary, HTTP/2 multiplexing). Strongly typed schema with backward-compatibility rules. Supports streaming (server-side, client-side, bidirectional). Built-in code generation eliminates manual serialisation.

**Weaknesses:** Harder to debug (binary is not human-readable). Less browser-friendly. Requires schema management tooling.

**Use gRPC for:** Internal high-throughput service-to-service calls, streaming data, polyglot environments where type safety across languages matters.

| | REST | gRPC |
|---|---|---|
| **Protocol** | HTTP/1.1 + JSON | HTTP/2 + Protobuf |
| **Performance** | Good | Excellent |
| **Schema** | Optional (OpenAPI) | Mandatory |
| **Streaming** | Limited | Native |
| **Debugging** | Easy (curl, browser) | Hard (binary) |
| **Browser support** | Full | Limited (needs proxy) |
| **Use for** | Public APIs, external | Internal high-perf calls |

---

## 4. Handling Synchronous Failures

Every synchronous call must be wrapped with reliability patterns. These are not optional — they are the minimum viable resilience for any production service.

### Timeout

Every outbound call must have a timeout. A call with no timeout can block a thread forever.

```
Rule: Set timeout at 2-3× the P99 latency of the callee

If User Service P99 = 50ms:
  Timeout = 100-150ms
  → Catches real failures
  → Doesn't trigger on legitimate slow calls
```

### Retry with Exponential Backoff and Jitter

Retry transient failures. Never retry immediately — the service may still be recovering.

```
Attempt 1: fails → wait 100ms + random jitter
Attempt 2: fails → wait 200ms + random jitter
Attempt 3: fails → wait 400ms + random jitter
Max retries: 3 → return error or trigger fallback
```

Only retry **idempotent** operations. Retrying a non-idempotent write causes duplicate side effects.

### Circuit Breaker

After N consecutive failures, stop calling the service entirely. Fail fast. After a recovery window, send a test request — if it succeeds, resume normal calls.

```
CLOSED (normal): calls pass through
  → N failures → OPEN

OPEN (failing): return error immediately without calling
  → timeout elapsed → HALF-OPEN

HALF-OPEN (testing): send one request
  → success → CLOSED
  → failure → OPEN again
```

**Why circuit breakers matter:** Without them, a failing downstream service holds your threads open, exhausting your thread pool and making your own service fail too — a cascading failure.

### Fallback

When a service is unavailable, return something useful instead of an error.

```
Recommendation service is down:
  → Fallback: return the top 10 most popular items (from cache)
  → User sees "Recommended for you" with generic popular items
  → No error page; degraded but functional

vs without fallback:
  → Return 503 Service Unavailable to the user
  → Entire page fails to load
```

---

## 5. Asynchronous Communication — Events and Queues

Asynchronous communication uses a message broker (Kafka, RabbitMQ, SQS) as an intermediary. Services are completely decoupled — they don't know about each other, only about the events they publish and consume.

```
Order Service places order:
  → publishes "order-placed" event to broker
  → continues immediately (does NOT wait)

Payment Service (independently):
  → consumes "order-placed"
  → charges card
  → publishes "payment-completed"

Inventory Service (independently):
  → consumes "order-placed"
  → reserves stock
  → publishes "stock-reserved"
```

All three services operate independently. Payment Service could be down for an hour — Order Service doesn't care; the event waits in the queue.

Full coverage of async patterns in `05-Messaging/04-messaging-patterns.md`. The key point here: **use async when the operation doesn't need to block the user's response.**

---

## 6. Service Mesh — Moving Communication Logic to Infrastructure

In a large microservices architecture, every service implements the same patterns: timeouts, retries, circuit breakers, mutual TLS, distributed tracing. This means the same code (or configuration) repeated in every service. A **service mesh** moves this logic out of the application and into the infrastructure.

### How It Works

A lightweight **sidecar proxy** is deployed alongside every service instance. All network traffic goes through the sidecar. The sidecar handles:

```
Service A code → sidecar A → network → sidecar B → Service B code

Sidecar handles:
  - mTLS (mutual TLS between services)
  - Retry logic
  - Circuit breaking
  - Load balancing
  - Distributed tracing (inject trace headers)
  - Traffic splitting (canary deployments)
  - Rate limiting
```

Service code becomes dramatically simpler — it just makes a plain network call, and the sidecar handles all resilience concerns automatically.

```
WITHOUT service mesh:
  Every service implements:
    - timeout configuration
    - retry logic
    - circuit breaker
    - mTLS certificate management
    - trace header injection
  → Same code in every service → maintenance burden → inconsistency

WITH service mesh:
  Every service just calls its dependency
  Sidecar handles everything else
  → Consistent policy across all services
  → Observability built in
  → No application code change needed
```

### Service Mesh Trade-offs

| Benefit | Cost |
|---|---|
| Consistent resilience across all services | Operational complexity (another layer to manage) |
| mTLS with zero application code | Sidecar adds latency to every call (~1ms) |
| Centralized policy control | Learning curve; new failure modes |
| Built-in distributed tracing | Only worth it at significant service count |

> **A service mesh is worth it at 20+ services.** Below that, implement resilience patterns directly in the service. The operational overhead of a mesh outweighs the benefit for small systems.

**Common service meshes:** Istio, Linkerd, Consul Connect.

---

## 7. API Gateway vs Service Mesh

These are complementary, not alternatives. They operate at different layers.

| | API Gateway | Service Mesh |
|---|---|---|
| **Handles** | North-South traffic (external → internal) | East-West traffic (service → service) |
| **Clients** | External users, mobile apps, partners | Internal services |
| **Concerns** | Auth, rate limiting, routing, SSL termination | mTLS, retries, circuit breaking, tracing |
| **Position** | Edge of the system | Between every pair of services |

```
External client → [API Gateway] → Service A → [Service Mesh] → Service B
                  (north-south)               (east-west)
```

---

## 8. Distributed Tracing

When a request flows through multiple services, a single log in one service tells you almost nothing. You need to correlate logs and traces across all services that handled the request.

**Trace ID propagation:**

```
Client request arrives at API Gateway:
  Gateway generates trace_id = "abc-123"
  Gateway passes trace_id in header to Service A

Service A:
  Logs: [trace_id=abc-123] received request
  Passes trace_id to Service B in header

Service B:
  Logs: [trace_id=abc-123] DB query took 110ms

Reconstruction:
  Search for trace_id=abc-123 across all logs:
  → API Gateway: 2ms
  → Service A: 15ms (including call to B)
  → Service B: 110ms (DB query)
  → Total: 127ms   ← now you know where the time went
```

**Tools:** Jaeger, Zipkin, OpenTelemetry (the standard for instrumentation). Without distributed tracing, debugging a slow or failing request in microservices is extremely painful.

---

## 9. Best Practices

- **Every synchronous call needs: timeout + retry (with backoff + jitter) + circuit breaker.**
- **Default to async** for operations that don't need to block the user.
- **Propagate trace IDs** through every inter-service call — debugging without them is impossible.
- **Use gRPC for high-throughput internal calls**; REST for external or public APIs.
- **Consider a service mesh** when you have 20+ services and want consistent resilience policy.
- **Never call a downstream service without a timeout** — thread pool exhaustion is a silent killer.

---

## 10. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No timeouts on inter-service calls | Thread pool exhaustion → cascading failure | Timeout on every outbound call |
| Synchronous calls for non-blocking work | Tight coupling; upstream service slows down with downstream | Use async messaging for fire-and-forget operations |
| No circuit breaker | One failing service takes down callers | Circuit breaker on every synchronous dependency |
| No distributed tracing | Debugging a slow request across 10 services is impossible | OpenTelemetry + Jaeger/Zipkin from day one |
| Retrying non-idempotent operations | Duplicate side effects | Make operations idempotent before adding retries |
| Service mesh too early | Operational overhead outweighs benefit | Add service mesh at 20+ services |

---

## 11. Interview Questions

1. What is the difference between REST and gRPC? When would you use each?
2. What is a circuit breaker? What are its three states?
3. Why should retries use exponential backoff with jitter?
4. What is a service mesh and what problem does it solve?
5. How does distributed tracing work across microservices?
6. What is the difference between an API Gateway and a service mesh?
7. When should inter-service communication be asynchronous rather than synchronous?

---

## 12. Summary

| Concept | Key Takeaway |
|---|---|
| **Sync vs Async** | Sync when you need the result; async when you don't. Default to async. |
| **REST** | Universal, human-readable. Best for external/public APIs. |
| **gRPC** | Binary, fast, typed. Best for internal high-throughput calls. |
| **Resilience** | Every sync call needs timeout + retry + circuit breaker. Non-negotiable. |
| **Circuit breaker** | Fail fast when downstream is broken. Prevents cascading failure. |
| **Service mesh** | Moves resilience logic to infrastructure sidecar. Worth it at 20+ services. |
| **Distributed tracing** | Trace ID propagation across all services. Essential for debugging. |

---

## 13. Cross References

**Prerequisites:** 01-microservices-fundamentals.md · 02-service-decomposition.md · Messaging Fundamentals (MS #1)

**Related Topics:** API Gateway (BB #2) · Fault Tolerance (NFR #6) · Observability (NFR #8) · Kafka vs RabbitMQ (MS #5)

**What to Learn Next:** 04-microservices-patterns.md

---

*System Design Engineering Handbook — 07-Microservices Series*