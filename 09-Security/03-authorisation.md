# 03 — Authorisation

> **09-Security Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Authorisation?

Authorisation (AuthZ) is the process of determining **what an authenticated user is allowed to do**. Authentication answers "who are you?" — authorisation answers "what can you do?"

```
AUTHENTICATION (file 02): Verify identity
  → "This request is from user Alice (user_id=u-789)"

AUTHORISATION (this file): Enforce permissions
  → "Can Alice read /orders/1001?"
  → "Can Alice delete /users/u-456?"
  → "Can Alice access the admin panel?"
```

**Why authorisation deserves its own deep treatment:** It is the most frequently broken security control in web applications. OWASP consistently ranks broken access control as the #1 vulnerability. Getting it wrong means authenticated users can access other users' data, escalate privileges, or perform actions they shouldn't.

---

## 2. RBAC — Role-Based Access Control

RBAC is the most widely used authorisation model. Permissions are assigned to **roles**, and users are assigned to roles.

```
Roles:
  admin    → can: create, read, update, delete all resources
  editor   → can: create, read, update own resources
  viewer   → can: read all public resources
  billing  → can: read, update billing information

Users:
  Alice → [admin]
  Bob   → [editor, viewer]
  Carol → [viewer]
  Dave  → [billing, viewer]
```

**Checking a permission:**

```
Alice requests DELETE /articles/123:
  1. Look up Alice's roles: [admin]
  2. Look up permissions for admin: {DELETE articles: true}
  3. Permission granted ✅

Carol requests DELETE /articles/123:
  1. Look up Carol's roles: [viewer]
  2. Look up permissions for viewer: {DELETE articles: false}
  3. Permission denied ❌ (403 Forbidden)
```

### RBAC Strengths and Weaknesses

**Strengths:**
- Simple to understand, implement, and audit
- Works well for most applications with a small number of distinct user types
- Easy to onboard users (assign role, done)

**Weaknesses:**
- Role explosion — complex systems end up with hundreds of roles
- Coarse-grained — "editor can edit all articles" but what if an editor should only edit *their own* articles?
- Static — roles don't change based on context (time, location, resource attributes)

---

## 3. ABAC — Attribute-Based Access Control

ABAC makes access decisions based on **attributes** of the user, the resource, and the environment — not just a role label.

```
ABAC policy example:
  "A user can edit a document IF:
    - user.department = document.department  (same department)
    - user.clearance_level >= document.classification  (sufficient clearance)
    - request.time is between 09:00 and 18:00  (business hours only)"

Alice's attributes:
  department = engineering, clearance_level = 3

Document attributes:
  department = engineering, classification = 2

Alice requests EDIT /documents/doc-456 at 14:00:
  - engineering = engineering ✅
  - 3 >= 2 ✅
  - 14:00 in business hours ✅
  → Access granted
```

### ABAC vs RBAC

| | RBAC | ABAC |
|---|---|---|
| **Decision based on** | User's roles | Attributes of user, resource, environment |
| **Granularity** | Coarse (role-level) | Fine (any attribute combination) |
| **Flexibility** | Low | Very high |
| **Complexity** | Low | High |
| **Auditability** | Easy | Harder (policy logic can be complex) |
| **Use when** | Distinct user types, simple permissions | Complex, context-sensitive access rules |

> **Start with RBAC. Move to ABAC when RBAC's coarseness causes real problems** — not as a default. ABAC is powerful but adds significant complexity to implementation and reasoning.

---

## 4. ACLs — Access Control Lists

An ACL is a list attached to a resource specifying which users or groups can perform which operations on it.

```
File: /reports/Q4-financial.xlsx

ACL:
  Alice   → READ, WRITE
  Bob     → READ
  finance → READ
  public  → (no access)
```

ACLs are resource-centric — you look at the resource to see who can access it. RBAC is user-centric — you look at the user to see what they can access.

ACLs work well for:
- File systems (Unix permissions are a simple ACL)
- Object storage (S3 bucket policies are ACLs)
- Network rules (firewall ACLs allow/deny traffic by IP)

ACLs become impractical when you have millions of resources (you'd need millions of ACLs) or when permissions change across many resources simultaneously.

---

## 5. The Most Critical AuthZ Vulnerability — IDOR

**Insecure Direct Object Reference (IDOR)** is when an attacker manipulates a resource identifier to access resources they don't own. It is the most common authorisation failure and is consistently in the OWASP Top 10.

```
Legitimate request:
  GET /orders/1001   ← Alice's order
  Server returns Alice's order details ✅

IDOR attack:
  GET /orders/1002   ← Bob's order (Alice changed 1001 to 1002)
  Server returns Bob's order details ❌ (should have been denied)

The server checked: "Is the user authenticated?" ✅
The server failed to check: "Does this user own order 1002?" ❌
```

**Why IDOR happens:** Developers add authentication (is this a logged-in user?) but forget to add ownership checks (does this user own *this* resource?).

**Prevention:**

```
❌ Wrong:
  function getOrder(orderId):
    return db.query("SELECT * FROM orders WHERE id = ?", orderId)

✅ Correct:
  function getOrder(orderId, currentUserId):
    order = db.query("SELECT * FROM orders WHERE id = ?", orderId)
    if order.userId != currentUserId:
      throw ForbiddenError("Access denied")
    return order
```

**Alternative — use non-sequential, non-guessable IDs:**

```
❌ Sequential:    /orders/1001, /orders/1002 (easily enumerable)
✅ UUID:          /orders/550e8400-e29b-41d4-a716-446655440000 (not guessable)
✅ Scoped routes: /users/me/orders/1001 (URL structure enforces ownership)
```

Even with UUIDs, always check ownership — security by obscurity is not access control.

---

## 6. Scopes — Fine-Grained OAuth Permissions

In OAuth 2.0 and API contexts, **scopes** define which specific actions a token is permitted to perform. They are a form of least-privilege authorisation for tokens.

```
OAuth scopes:
  read:profile       → can read user profile
  write:profile      → can update user profile
  read:orders        → can read orders
  write:orders       → can create/update orders
  admin              → full access

A mobile app might request: ["read:profile", "read:orders"]
A third-party integration might request: ["read:orders"]
An admin tool might request: ["admin"]

If a token has scope ["read:profile"], it CANNOT:
  → Write to the profile (403 on write endpoints)
  → Read orders (403 on order endpoints)
```

Scopes are evaluated at the API level: the server checks that the token's scope includes permission for the requested operation.

---

## 7. Policy Engines — Externalising AuthZ Logic

In complex systems, authorisation logic distributed across every service becomes hard to maintain, audit, and change consistently. A **policy engine** centralises authorisation decisions.

**The Pattern: Policy Decision Point (PDP)**

```
Service receives request
  → Service calls Policy Engine: "Can user u-789 DELETE /orders/1001?"
  → Policy Engine evaluates policies against user attributes, resource, action
  → Policy Engine returns: ALLOW or DENY
  → Service enforces the decision
```

**Open Policy Agent (OPA)** is the most widely adopted open-source policy engine.

```
OPA policy example (Rego language — conceptual):
  allow {
    input.user.role == "admin"
  }

  allow {
    input.action == "read"
    input.user.id == input.resource.owner_id
  }

  allow {
    input.user.department == input.resource.department
    input.action != "delete"
  }
```

**Benefits of externalising authorisation:**
- Policies managed in one place; consistent across all services
- Policy changes don't require service deployments
- Policies can be tested independently
- Full audit trail of every access decision

**When to use a policy engine:**
- Complex authorisation logic used across many services
- Compliance requirements (GDPR, HIPAA) requiring auditable access decisions
- Multi-tenant systems with complex per-tenant permission models

---

## 8. Implementing AuthZ Correctly — Practical Checklist

```
✅ Every endpoint checks authorisation — not just authentication
✅ Never trust client-provided identifiers — verify ownership server-side
✅ Use non-sequential resource IDs (UUIDs) — but don't rely on obscurity alone
✅ Apply least privilege to service accounts — not just users
✅ Log all authorisation decisions — especially denials
✅ Test for IDOR — enumerate IDs, change user IDs, test horizontal escalation
✅ Test for privilege escalation — can a viewer perform admin actions?
✅ Review authorisation on every API change — new endpoints need new checks
```

---

## 9. How Large Companies Handle AuthZ

| Company | Approach | Source |
|---|---|---|
| **Google** | Zanzibar — a globally consistent authorisation system used across all Google products | Google Research paper (public) |
| **Netflix** | Attribute-based access control for content licensing rules (region, device, subscription) | Netflix Tech Blog (public) |
| **Airbnb** | Fine-grained authorisation service; host can only manage their own listings | Airbnb Eng Blog (public) |
| **Cloudflare** | OPA-based policy engine for internal API authorisation | Cloudflare Blog (public) |

---

## 10. Best Practices

- **Check authorisation on every request, not just at login.**
- **Check both authentication and ownership** — IDOR is always an ownership check failure.
- **Use non-sequential IDs** (UUIDs) — reduces guessability but is not a substitute for ownership checks.
- **Apply RBAC first** — move to ABAC only when role-level granularity is genuinely insufficient.
- **Externalise complex policy** to a policy engine (OPA) — keeps service code simple and policy consistent.
- **Log all denials** — a user repeatedly hitting 403s is a signal worth investigating.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Checking auth at login only | Session-level but not resource-level access control | Check ownership on every resource request |
| IDOR via sequential IDs | User iterates IDs, reads other users' data | UUID + ownership check |
| Overly broad roles | Users have access to more than needed | Fine-grained roles; least privilege |
| No audit logging of denials | Cannot detect probing or escalation attempts | Log all 403 responses with user ID and resource |
| AuthZ logic in every service | Inconsistent; hard to change; audit nightmare | Centralise in a policy engine for complex rules |

---

## 12. Interview Questions

1. What is the difference between RBAC and ABAC? When would you choose each?
2. What is an IDOR vulnerability? Give an example and explain how to prevent it.
3. What are OAuth scopes and how do they relate to least privilege?
4. What is a policy engine (OPA) and when would you use one?
5. Why is it insufficient to check authentication without also checking authorisation?
6. How would you prevent a user from accessing another user's order in an API?
7. What HTTP status code should a successful auth but failed authZ return?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **AuthZ** | What can you do? Always follows AuthN. |
| **RBAC** | Permissions via roles. Simple, widely used. Coarse-grained. |
| **ABAC** | Permissions via attributes. Flexible but complex. |
| **ACL** | Resource-centric list of who can do what. Good for files and objects. |
| **IDOR** | #1 AuthZ failure. Always check ownership, not just authentication. |
| **Scopes** | Token-level least privilege in OAuth and APIs. |
| **Policy engine** | Centralised, auditable, testable authorisation logic. OPA is the standard. |

---

## 14. Cross References

**Prerequisites:** 02-authentication.md · 01-security-fundamentals.md

**Related Topics:** 07-api-security.md · 05-application-security.md · Security (NFR #7)

**What to Learn Next:** 04-transport-security.md

---

*System Design Engineering Handbook — 09-Security Series*