# 01 — Security Fundamentals

> **09-Security Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. The Security Mindset

Security is fundamentally different from other engineering disciplines. In performance, reliability, or scalability, you optimise against neutral forces — physics, probability, load. In security, you are designing against an **active adversary** who is intelligent, motivated, and constantly looking for your weakest point.

This changes everything:
- A system that works correctly 99.99% of the time may still be completely broken from a security perspective
- The adversary only needs to find *one* way in; you must defend *all* of them
- Security is never "done" — the threat landscape evolves continuously

> **The security mindset:** assume breach. Design not just to prevent attacks, but to detect them, limit their impact, and recover from them. A system that assumes it will never be breached will be catastrophically unprepared when it is.

---

## 2. The CIA Triad — The Three Security Goals

Every security control serves one or more of these three foundational goals. When evaluating any security decision, ask which property you are protecting.

| Property | Meaning | Threat | Example Control |
|---|---|---|---|
| **Confidentiality** | Only authorised parties can read data | Data breach, eavesdropping, insider threat | Encryption, access control, least privilege |
| **Integrity** | Data is not tampered with or corrupted | Man-in-the-middle, SQL injection, supply chain attack | Digital signatures, checksums, parameterised queries |
| **Availability** | Authorised users can access when needed | DDoS, ransomware, resource exhaustion | Rate limiting, redundancy, circuit breakers |

These three properties conflict and require balance. Maximum confidentiality (encrypt everything, restrict all access) harms availability. Maximum availability (no restrictions, always accessible) destroys confidentiality.

---

## 3. Threat Modelling

Threat modelling is the structured process of identifying what could go wrong in a system and designing controls to prevent or mitigate it. It should happen **before** building, not after.

### The Four Questions (STRIDE-inspired)

```
1. What are we building?
   → Draw a data flow diagram: where does data enter, where does it go,
     who can touch it?

2. What can go wrong?
   → For each component and data flow, identify threats

3. What are we going to do about it?
   → For each threat: mitigate, accept, transfer, or eliminate

4. Did we do a good enough job?
   → Review the threat model against completed implementation
```

### STRIDE — Threat Categories

STRIDE is a mnemonic for the six categories of security threats. Applying it systematically to your data flow diagram surfaces threats you might otherwise miss.

| Threat | Violates | Example |
|---|---|---|
| **S**poofing | Authentication | Attacker impersonates a legitimate user or service |
| **T**ampering | Integrity | Attacker modifies data in transit or at rest |
| **R**epudiation | Non-repudiation | User denies performing an action; no audit trail |
| **I**nformation disclosure | Confidentiality | Sensitive data exposed to unauthorised parties |
| **D**enial of Service | Availability | System overwhelmed; legitimate users cannot access |
| **E**levation of privilege | Authorisation | User gains more access than they should have |

**Practical threat modelling process:**

```
Step 1: Draw the system — services, databases, APIs, external dependencies
Step 2: Mark trust boundaries — where does data cross from trusted to less trusted?
Step 3: For each data flow crossing a trust boundary: apply STRIDE
Step 4: For each identified threat: what is the risk? what is the control?
Step 5: Document decisions — which threats are accepted vs. mitigated
```

---

## 4. Attack Surface

The attack surface is the sum of all paths through which an attacker can interact with your system. Reducing the attack surface reduces risk.

```
Attack surface includes:
  External APIs (public endpoints)
  Admin interfaces (internal, but still accessible)
  Third-party dependencies (libraries, services)
  User-supplied input (every field, header, URL parameter)
  Network interfaces (every port listening for connections)
  Employees (social engineering, phishing)
  Supply chain (build tools, CI/CD, package registries)
```

**Attack surface reduction principles:**

```
Principle of minimal exposure:
  Disable unused ports and services
  Remove unused dependencies (each is a potential vulnerability)
  Shut down debug/admin endpoints in production
  Use private networks for internal services (not public internet)

Principle of minimal footprint:
  Run services with least required permissions
  Use read-only file systems where possible
  Containers with minimal base images
```

---

## 5. Defence in Depth

Defence in depth is the strategy of layering multiple independent security controls so that defeating one does not compromise the system. Named after the military concept of layered defensive positions.

```
PERIMETER LAYER:
  WAF, DDoS protection, IP allowlists, TLS
  → Block most attacks before they reach services

NETWORK LAYER:
  Network segmentation, VPC, firewall rules, private subnets
  → Limit lateral movement if perimeter is breached

APPLICATION LAYER:
  Input validation, parameterised queries, auth checks
  → Stop application-level attacks

DATA LAYER:
  Encryption at rest, field-level encryption, access controls
  → Protect data even if application is compromised

DETECTION LAYER:
  Logging, monitoring, alerting, anomaly detection
  → Detect breaches that penetrate other layers

RECOVERY LAYER:
  Backups, incident response, disaster recovery
  → Recover when everything else fails
```

> **No single control is sufficient.** Defence in depth means an attacker who bypasses the WAF still faces application-level validation; one who finds an injection vulnerability still can't read encrypted data; one who reads data still triggers anomaly detection.

---

## 6. Zero Trust Architecture

Traditional security assumed a hard perimeter: inside the network = trusted; outside = untrusted. Zero trust abandons this assumption entirely.

**Traditional perimeter model:**

```
Internet (untrusted) → Firewall → Internal network (trusted)
                                  → All services trust each other
                                  → Any internal actor has broad access
```

**Problem:** Once an attacker is inside the perimeter (via phishing, malware, a compromised employee, or a supply chain attack), they have broad access. The inside of the network trusts everything inside it.

**Zero trust model:**

```
"Never trust, always verify"

Every request — regardless of where it comes from (internal or external) —
must be authenticated, authorised, and encrypted.

Internal service A calling internal service B:
  → B requires A to authenticate (mTLS certificate)
  → B checks if A is authorised for this specific operation
  → Communication is encrypted
  → Even if A is compromised, it can only do what it's authorised to do
```

**Zero trust pillars:**

| Pillar | Implementation |
|---|---|
| **Verify explicitly** | Always authenticate and authorise based on identity, not network location |
| **Use least privilege** | Grant minimum access needed; time-limited where possible |
| **Assume breach** | Minimise blast radius; end-to-end encryption; monitor for anomalies |

**Practical zero trust elements:**
- Service-to-service mTLS (each service proves its identity)
- Short-lived credentials (tokens expire; no long-lived passwords between services)
- Per-request authorisation (not "logged in = can do everything")
- Network micro-segmentation (services can only reach what they need)

---

## 7. Principle of Least Privilege

Every user, service, and process should have the minimum access necessary to perform its function — nothing more.

```
Database access example:
  ❌ Wrong: Application uses a DB user with full admin rights
  ✅ Right: Application uses a DB user with SELECT on specific tables,
            INSERT/UPDATE on specific tables, no DROP/ALTER/DELETE rights

Service account example:
  ❌ Wrong: Service can read from any S3 bucket in the account
  ✅ Right: Service can read from exactly one specific S3 bucket path

Human access example:
  ❌ Wrong: All engineers have production database read access by default
  ✅ Right: Production DB access is time-limited, request-based,
            with full audit logging of every query
```

**Why least privilege matters even for internal systems:**

When a service or account is compromised, least privilege limits what the attacker can access. A compromised API service that only has read access to one table cannot be used to delete the entire database or access payment data.

---

## 8. Security by Design vs Security Bolted On

Security added as an afterthought is consistently more expensive and less effective than security designed in from the start.

| Approach | Cost | Effectiveness |
|---|---|---|
| Security from day one | Low — built into normal development | High — controls are native to the design |
| Security retrofit | High — requires refactoring existing systems | Medium — often incomplete; workarounds instead of proper controls |
| Security by penetration test only | Low to build; high when findings require remediation | Low — only finds known attack patterns |

**Secure SDLC (Software Development Lifecycle):**

```
Requirements:   Include security requirements alongside functional ones
Design:         Threat model before building
Development:    Secure coding guidelines; peer review for security
Testing:        SAST (static analysis), DAST (dynamic testing), dependency scanning
Deployment:     Secrets management; minimal container permissions; IaC scanning
Operations:     Runtime monitoring; vulnerability management; incident response
```

---

## 9. Common Security Mistakes in System Design

| Mistake | Risk | Correct Approach |
|---|---|---|
| Trusting internal network by default | Lateral movement after breach | Zero trust: authenticate everything |
| Single point of auth failure | Auth service down = all services down | Distributed auth with token validation |
| Over-privileged service accounts | Breach of one service = access to everything | Least privilege per service |
| No threat model | Build without knowing what you're defending against | Threat model before development starts |
| Security only at the perimeter | Attacker inside perimeter has free reign | Defence in depth at every layer |
| Audit logging as an afterthought | Cannot investigate breaches; no forensic trail | Immutable audit log from day one |

---

## 10. Interview Questions

1. What is the CIA triad? Give an example of a control for each property.
2. What is threat modelling and what are the STRIDE categories?
3. What is defence in depth? Give an example with three layers.
4. What is zero trust and how does it differ from a perimeter security model?
5. What is the principle of least privilege? Give a concrete example.
6. How does "assume breach" change how you design a system?
7. What is an attack surface and how do you reduce it?

---

## 11. Summary

| Concept | Key Takeaway |
|---|---|
| **Security mindset** | Active adversary. Assume breach. Detect, limit, recover. |
| **CIA triad** | Confidentiality + Integrity + Availability. Every control serves one or more. |
| **Threat modelling** | Identify threats before building. STRIDE categories. |
| **Attack surface** | Everything an attacker can interact with. Minimise it. |
| **Defence in depth** | Multiple independent layers. One failure doesn't compromise all. |
| **Zero trust** | Never trust, always verify. Network location proves nothing. |
| **Least privilege** | Minimum access needed. Limits blast radius on breach. |

---

## 12. Cross References

**Prerequisites:** Security (NFR #7) · System Design Fundamentals

**Related Topics:** 02-authentication.md · 04-transport-security.md · Observability (05-incident-response.md)

**What to Learn Next:** 02-authentication.md

---

*System Design Engineering Handbook — 09-Security Series*