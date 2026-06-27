# 07 — API Security

> **09-Security Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. Why API Security Deserves Its Own Treatment

APIs are the primary attack surface of modern applications. Where traditional web apps served HTML to browsers, today's systems expose REST, GraphQL, and gRPC APIs consumed by mobile apps, SPAs, partners, and automated clients. Every endpoint is reachable, programmatically explorable, and potentially exploitable.

OWASP publishes a separate **API Security Top 10** alongside the general web Top 10, recognising that APIs introduce distinct risks: they expose business logic directly, they often handle sensitive data at high volume, and they are consumed by non-human clients that can probe them systematically.

> **APIs are different from web pages in one critical way:** a user who can't see a button on a web page can still call the underlying API endpoint directly. The UI is not a security boundary.

---

## 2. Input Validation — Trust Nothing

Every input arriving at an API endpoint must be validated before use. User-supplied data includes URL parameters, query strings, request headers, request bodies, uploaded files, and even data fetched from external URLs.

### Validate at Every Layer

```
Validation should happen at:
  1. API Gateway: schema validation, size limits, format checks
  2. Application layer: business rule validation, type checking
  3. Database layer: constraints (NOT NULL, UNIQUE, CHECK)

Each layer catches different things. Relying on one layer alone is insufficient.
```

### What to Validate

```
Type:         Is this an integer? A valid UUID? An ISO date?
Format:       Is this a valid email format? A valid URL?
Range:        Is this number within 1–100? Is this date in the future?
Length:       String < 255 chars? Array < 1000 elements?
Allowed values: Is this status one of [PENDING, ACTIVE, CANCELLED]?
Content:      For uploaded files: is this actually a PDF, not a PHP script?
```

**Allowlist vs Denylist:**

```
❌ Denylist approach: "reject these specific bad values"
   → Attackers find values you didn't think to block
   → Constantly needs updating as new attack vectors emerge

✅ Allowlist approach: "only accept these specific good values"
   → Anything not explicitly permitted is rejected
   → More restrictive; more secure
```

---

## 3. Authentication at the API Level

APIs use different authentication mechanisms than browser sessions. Token-based auth dominates.

### API Keys

Simple opaque tokens issued to developers/clients for programmatic access.

```
Request: GET /api/data
         Authorization: Bearer sk-prod-abc123def456

Server: look up 'sk-prod-abc123def456' in API key store
        → who does this belong to? what are their permissions? are they active?
```

**API key best practices:**

```
Generation:
  ✅ Cryptographically random (32+ bytes)
  ✅ Environment-specific (prod keys ≠ staging keys)
  ✅ Include a prefix for identification: sk-prod-xxx, sk-test-xxx

Storage:
  ✅ Hash before storing in DB (same as passwords)
  ✅ Only show the key to the user once (at creation)
  ✅ Never log in plaintext

Scoping:
  ✅ Keys have scopes (read-only vs write access)
  ✅ Keys can be scoped to specific IP ranges
  ✅ Keys have expiry dates

Rotation:
  ✅ Allow users to create new keys and revoke old ones
  ✅ Alert on suspicious usage patterns
```

### JWT at the API Level

Covered in Authentication (file 02). Key API-specific considerations:

```
Verify:
  ✅ Algorithm is what you expect (reject 'none' algorithm)
  ✅ Signature is valid
  ✅ Token is not expired
  ✅ Issuer (iss) is your auth server
  ✅ Audience (aud) is your API

Never:
  ❌ Accept tokens signed with a symmetric key when you expected asymmetric
  ❌ Trust claims without verifying the signature
  ❌ Cache tokens server-side and skip verification
```

---

## 4. Rate Limiting for Security

Rate limiting is covered as a building block (02-Building-Blocks/rate-limiting.md) from a throughput perspective. From a security perspective, it is a critical defence against:

**Brute force attacks:**

```
Without rate limiting:
  Attacker script: try 1 million passwords at 10,000/second
  → cracks average account in < 2 minutes

With rate limiting on /auth/login:
  5 failed attempts per 15 minutes per IP + per account
  → same 1 million passwords takes 208 days
  → economically infeasible
```

**Credential stuffing:**

```
Attack: take 1 billion leaked username/password pairs
        try each against your login endpoint

Rate limiting per IP: attacker uses 1 billion IPs (botnets)
Rate limiting per account: attacker slowed to 1 try per 15 min per account
Combined with: MFA, breach detection → stuffing becomes impractical
```

**Enumeration attacks:**

```
Attack: "does user alice@example.com have an account?"
        Try reset password for alice@example.com:
          "We sent a reset email" → account exists
          "Email not found" → no account

Rate limit AND:
  Return same response regardless ("If this email exists, we sent a link")
  → Prevents enumeration even without rate limiting
```

**Scraping and data harvesting:**

```
Rate limit per API key + per IP:
  Limits how fast an attacker can download your entire database via your own API
  Legitimate users don't notice; scrapers are throttled to infeasibility
```

**Security-specific rate limit configuration:**

```
Authentication endpoints:     5 requests / 15 minutes / IP + account
Password reset:               3 requests / hour / email
Account creation:             10 / hour / IP
Sensitive data endpoints:     100 / minute / user
Public search/listing:        1000 / minute / IP
```

---

## 5. CORS — Cross-Origin Resource Sharing

CORS controls which external origins (domains) are allowed to make requests to your API from a browser.

```
Browser security model (same-origin policy):
  JavaScript on https://app.company.com can only fetch from company.com by default
  To allow cross-origin requests, the server must explicitly opt in via CORS headers
```

**CORS headers:**

```
Server response:
  Access-Control-Allow-Origin: https://app.company.com   (specific allowed origin)
  Access-Control-Allow-Methods: GET, POST, PUT
  Access-Control-Allow-Headers: Content-Type, Authorization
  Access-Control-Allow-Credentials: true   (include cookies in cross-origin requests)
```

**The dangerous misconfiguration:**

```
❌ Access-Control-Allow-Origin: *
   With: Access-Control-Allow-Credentials: true
   → Any website can make authenticated requests to your API on behalf of users
   → Note: browsers actually reject this combination; but misconfigured CORS
     with overly broad origins still allows data theft

✅ Access-Control-Allow-Origin: https://app.company.com
   (only your specific frontend domain allowed)
```

**CORS allowlist — not an allowall:**

```
Allowed origins:
  https://app.company.com        (main web app)
  https://admin.company.com      (admin panel)
  http://localhost:3000           (local development ONLY - not in production)

Never:
  * (wildcard - too broad)
  https://*.company.com (wildcard subdomain - attacker creates sub.company.com)
```

---

## 6. API Versioning and Deprecation Security

Old API versions with known vulnerabilities cannot be left running indefinitely.

```
Security-aware versioning:
  Maintain security patches on all supported versions simultaneously
  Set explicit end-of-life dates for each version
  Force clients off deprecated versions — do not allow indefinite use of old versions

Deprecation process:
  1. Announce EOL date (6+ months in advance for major versions)
  2. Add deprecation headers to all requests to old version:
     Deprecation: true
     Sunset: Thu, 01 Jan 2026 00:00:00 GMT
  3. Block new client registrations from using deprecated version
  4. Hard sunset: return 410 Gone after EOL date
  5. Shut down old version infrastructure
```

---

## 7. GraphQL-Specific Security

GraphQL introduces unique security considerations compared to REST.

### Query Depth and Complexity Limiting

GraphQL allows clients to specify exactly what data they want, including nested relationships. Without limits, a client can craft a deeply nested query that overwhelms the server.

```
Malicious deep query:
  {
    users {               ← 1,000 users
      orders {            ← each with 100 orders
        products {        ← each with 50 products
          reviews {       ← each with 100 reviews
            ...           ← exponential data retrieval
          }
        }
      }
    }
  }
```

**Defences:**

```
Query depth limit:  reject queries with nesting > 5 levels
Query complexity:   assign cost to each field; reject if total cost > threshold
Timeout:            abort queries running longer than 10 seconds
Persisted queries:  only allow pre-registered query strings (eliminates ad-hoc)
```

### Introspection in Production

GraphQL's introspection feature lets clients query the full schema — every type, field, and relationship. In development, this is useful. In production, it hands attackers a complete map of your API.

```
Disable introspection in production:
  → Attackers cannot discover hidden fields, internal types, or schema structure
  → Legitimate clients should know the schema from your documentation
  → Enable only for specific trusted clients (via IP allowlist or special header)
```

---

## 8. Bot Protection and Abuse Prevention

Automated clients (bots, scrapers, fraud tools) can abuse APIs at a scale no human attacker could match.

### Signals for Bot Detection

```
Behavioural signals:
  Request rate far exceeding human capability
  Perfectly regular timing (no human variability)
  Access patterns inconsistent with normal usage
  No session cookies (headless browser)
  Rotating through multiple IP addresses

Technical signals:
  Missing or invalid User-Agent
  Missing Accept-Language, Accept-Encoding headers
  No browser fingerprint (no JavaScript execution)
  Data centre IP ranges (ASN lookup)
```

### Defences

```
CAPTCHA:
  Challenge-response test that humans pass and bots fail
  Invisible CAPTCHAs (Google reCAPTCHA v3) score invisibly
  Only present visible challenge when score is low

Device fingerprinting:
  Collect browser/device characteristics to identify repeated access
  Even with IP rotation, fingerprint may remain consistent

Progressive friction:
  Low suspicion: allow request
  Medium suspicion: rate limit + CAPTCHA
  High suspicion: block + flag for review

IP reputation services:
  Known bad IPs (Tor exits, data centres, proxy services) → extra scrutiny
  Allowlist legitimate crawler IPs (Googlebot, etc.)
```

---

## 9. API Gateway as Security Enforcement Point

The API Gateway (covered in Building Blocks #2) is the ideal place to enforce many API security controls centrally.

```
Security controls enforced at API Gateway:
  Authentication:     Validate JWT/API key before request reaches service
  Authorisation:      Basic scope checking before routing
  Rate limiting:      Per client, per endpoint, per IP
  Input validation:   Schema validation, size limits
  CORS:               Centrally configured for all services
  TLS termination:    All external traffic decrypted here
  WAF integration:    Web Application Firewall blocks known attack patterns
  Logging:            Every request logged with client identity
  DDoS protection:    Absorb volumetric attacks before they reach services
```

**What the gateway cannot replace:**
- Application-level authorisation (IDOR checks must happen in the service)
- Business logic validation
- Service-to-service security (needs mTLS)

---

## 10. How Large Companies Apply This

| Company | Practice | Source |
|---|---|---|
| **Stripe** | API keys with prefixes per environment (sk-test, sk-live); hashed storage; scoped permissions | Stripe public documentation |
| **GitHub** | Fine-grained personal access tokens with repository and permission scope | GitHub public documentation |
| **Cloudflare** | WAF + bot management at edge; protects millions of APIs | Cloudflare public docs |
| **Twitter/X** | OAuth 2.0 for developer API; strict rate limits per endpoint per app | Twitter public API docs |

---

## 11. Best Practices

- **Validate all input** at every layer — gateway, application, and database.
- **Rate limit all authentication endpoints** aggressively — brute force is always attempted.
- **Configure CORS explicitly** — specific origin allowlist, never wildcard with credentials.
- **Disable GraphQL introspection in production** — don't hand attackers your schema.
- **Enforce API key scoping** — keys should have minimum necessary permissions.
- **Use the API Gateway as a central security enforcement point** — don't duplicate security logic in every service.
- **Set explicit API version sunset dates** — old versions with vulnerabilities must be retired.

---

## 12. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| No rate limiting on auth endpoints | Brute force and stuffing succeed | 5 attempts / 15 min on all auth endpoints |
| CORS wildcard with credentials | Any site can make authenticated requests | Explicit origin allowlist |
| GraphQL introspection in production | Attacker gets complete API schema | Disable in production |
| No input validation on API | Injection, SSRF, and logic bypass | Schema validation at gateway + application |
| API keys stored in plaintext | Key store breach = all keys compromised | Hash API keys before storage |
| Indefinite old API version support | Known vulnerabilities remain exploitable | Sunset policy with hard removal date |

---

## 13. Interview Questions

1. What are the key differences between API security and traditional web application security?
2. How should API keys be generated, stored, and rotated?
3. What is CORS and what is the security risk of misconfiguring it?
4. How do you protect a GraphQL API from query-based DoS attacks?
5. What rate limiting rules would you apply to a login endpoint?
6. How does an API Gateway function as a security enforcement point?
7. What is the OWASP API Security Top 10 and why does it exist separately from the web Top 10?

---

## 14. Summary

| Concern | Key Control |
|---|---|
| **Input validation** | Allowlist approach at every layer |
| **Authentication** | API keys (hashed, scoped) or JWT (short TTL, algorithm verified) |
| **Rate limiting** | Aggressive on auth; moderate on data; by IP + identity |
| **CORS** | Explicit origin allowlist; never wildcard with credentials |
| **GraphQL** | Depth + complexity limits; introspection disabled in production |
| **Bot protection** | CAPTCHA + fingerprinting + IP reputation + behavioural analysis |
| **Gateway** | Central enforcement: auth, rate limit, CORS, WAF, logging |

---

## 15. Cross References

**Prerequisites:** 02-authentication.md · 03-authorisation.md · Rate Limiting (BB #5) · API Gateway (BB #2)

**Related Topics:** 05-application-security.md · 06-secrets-and-key-management.md

**What to Learn Next:** 10-System-Design-Patterns section

---

*System Design Engineering Handbook — 09-Security Series*

---

> **09-Security Series complete.**
> Covered: Fundamentals · Authentication · Authorisation · Transport Security ·
> Application Security · Secrets & Key Management · API Security