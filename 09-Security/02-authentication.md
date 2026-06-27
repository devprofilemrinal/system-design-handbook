# 02 — Authentication

> **09-Security Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Authentication?

Authentication (AuthN) is the process of verifying **who someone is**. It answers the question: "Are you really who you claim to be?"

Authentication is always the first step before any other security decision. You cannot decide what someone is allowed to do (authorisation) until you know who they are.

```
AUTHENTICATION:        "Who are you?"
  → Verified via credential (password, token, certificate, biometric)

AUTHORISATION:         "What are you allowed to do?"
  → Checked after authentication (covered in file 03)

401 Unauthorized:      Authentication failed or not provided
403 Forbidden:         Authenticated but not authorised
```

> **The most common security mistake:** confusing 401 and 403. A 401 means "I don't know who you are." A 403 means "I know who you are but you can't do this." Getting these right matters for security and usability.

---

## 2. Password-Based Authentication

Passwords remain the most common authentication mechanism despite their weaknesses. The security of password-based auth depends almost entirely on **how passwords are stored**.

### How Passwords Must Be Stored

**Never store plaintext passwords.** If your database is breached, every user's password is immediately compromised — and because many users reuse passwords, the attacker gains access to their accounts on other sites too.

**Never store encrypted passwords.** Encryption is reversible — with the key, an attacker can decrypt every password.

**Always store password hashes.** A cryptographic hash is one-way and irreversible. You store the hash; when a user logs in, you hash their input and compare.

```
Registration:
  User provides: "mypassword123"
  Generate random salt: "xK9mP2..."
  Hash: bcrypt("mypassword123" + "xK9mP2...") → "$2a$12$xK9mP2..."
  Store in DB: {hash: "$2a$12$xK9mP2...", salt: "xK9mP2..."}

Login:
  User provides: "mypassword123"
  Retrieve hash from DB
  bcrypt("mypassword123" + stored_salt)
  Compare result to stored hash
  Match → authenticated
```

### Why bcrypt/scrypt/Argon2 — Not SHA-256

General-purpose hash functions (SHA-256, MD5) are designed to be fast — millions of operations per second. This is catastrophic for passwords: an attacker with a breached database can try billions of password guesses per second.

Password hashing functions are deliberately **slow**:

```
SHA-256:  ~10 billion hashes/second on a GPU
bcrypt:   ~20,000 hashes/second (configurable cost factor)
Argon2:   ~hundreds/second (configurable memory + time cost)

Attack on 1 billion user accounts:
  SHA-256: cracks in ~100 seconds
  bcrypt:  cracks in ~50,000 seconds (14 hours)
  Argon2:  cracks in years
```

**Salt** is a random value added per-user before hashing. It ensures identical passwords produce different hashes, preventing precomputed "rainbow table" attacks.

> **Use Argon2id as your first choice (2024 standard). bcrypt is acceptable for existing systems. Never use MD5, SHA-1, or SHA-256 alone for passwords.**

---

## 3. Sessions — Stateful Authentication

After a user authenticates with a password, you need a way to remember that authentication across requests. HTTP is stateless — it has no built-in concept of a session.

### Session-Based Authentication

```
1. User submits username + password
2. Server verifies credentials
3. Server creates a session record in the session store:
   {session_id: "abc123", user_id: "u-789", expires: "2024-01-15T18:00:00Z"}
4. Server sends session_id to client in a cookie:
   Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict
5. Client includes cookie on all subsequent requests
6. Server looks up session_id in session store → finds user → authenticated
```

**Session store options:**
- In-memory (single server only — loses sessions on restart)
- Redis (distributed, fast, supports TTL — the standard)
- Database (durable but slower)

**Cookie security flags:**

| Flag | Effect |
|---|---|
| **HttpOnly** | Cookie not accessible to JavaScript — prevents XSS theft |
| **Secure** | Cookie only sent over HTTPS — prevents interception |
| **SameSite=Strict** | Cookie not sent on cross-site requests — prevents CSRF |
| **Max-Age / Expires** | Session expires after N seconds |

### Session Invalidation

Sessions must be invalidated on logout, password change, and suspicious activity. This is sessions' key advantage over tokens — server-side invalidation is immediate.

```
User logs out:
  Client: DELETE /session
  Server: DELETE from session store WHERE session_id = 'abc123'
  → Immediately invalid; no reuse possible
```

---

## 4. JWT — Stateless Tokens

JSON Web Tokens (JWTs) are a stateless alternative to sessions. Instead of looking up a session in a store, the token contains all necessary information, cryptographically signed by the server.

### JWT Structure

A JWT consists of three base64-encoded parts separated by dots:

```
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1LTc4OSIsInJvbGUiOiJ1c2VyIiwiZXhwIjoxNzA1MzM2MDAwfQ.signature

Header:    {"alg": "RS256", "typ": "JWT"}
           (algorithm used to sign)

Payload:   {"sub": "u-789", "role": "user", "exp": 1705336000}
           (claims: who this is, what they can do, when it expires)

Signature: HMAC-SHA256(base64(header) + "." + base64(payload), secret_key)
           (proves the token was issued by the server and hasn't been tampered with)
```

### How JWT Authentication Works

```
1. User authenticates with credentials
2. Server generates JWT signed with its private key
3. Server returns JWT to client
4. Client stores JWT (memory or localStorage; NOT a cookie for API clients)
5. Client includes JWT in Authorization header on every request:
   Authorization: Bearer eyJhbGci...
6. Server receives request, validates JWT signature with its public key
7. If valid and not expired → user is authenticated (no DB lookup needed)
```

**JWT advantages:**
- Stateless — server holds no session state; scales horizontally without session sharing
- Portable — works across domains and services; ideal for microservices

**JWT weaknesses:**
- **Cannot be invalidated before expiry.** If a token is stolen, the attacker can use it until it expires. Solution: short expiry + refresh tokens.
- Sensitive data in payload is base64-encoded, not encrypted — visible to anyone who decodes it. Don't put sensitive data in JWT payload.
- Algorithm confusion attacks — always explicitly specify and validate the algorithm.

### Access Token + Refresh Token Pattern

```
Access token:  Short-lived (15 minutes). Used on every request.
Refresh token: Long-lived (7 days). Stored securely. Used only to get new access tokens.

Flow:
  Login → server issues access_token (15min) + refresh_token (7d)
  Client uses access_token on every API request
  access_token expires → client sends refresh_token to /auth/refresh
  Server validates refresh_token → issues new access_token
  refresh_token can be stored in server-side DB for revocation
```

This gives you stateless API calls (using short-lived access tokens) with the ability to revoke sessions (by invalidating refresh tokens in the DB).

---

## 5. OAuth 2.0 — Delegated Authentication

OAuth 2.0 is not an authentication protocol — it is an **authorisation framework** for delegating access. "Sign in with Google" uses OAuth 2.0 to let users grant your application access to their Google account without giving you their Google password.

### Core Concept

```
Without OAuth:
  User wants to use your app with their Google account
  → User gives you their Google password
  → You have their Google password — this is terrible for both parties

With OAuth 2.0:
  User: "I want to sign in with Google"
  Your app → redirects to Google
  User: authenticates directly with Google
  Google: "Do you grant this app access to your profile?"
  User: "Yes"
  Google → issues access token to your app
  Your app → uses token to read user profile from Google API
  → You never see the user's Google password
```

### The Authorization Code Flow (Recommended)

```
1. Your app redirects user to Google:
   https://accounts.google.com/oauth/authorize?
     client_id=YOUR_APP_ID&
     redirect_uri=https://yourapp.com/callback&
     scope=profile email&
     response_type=code&
     state=RANDOM_STATE_VALUE

2. User logs in to Google, approves permissions

3. Google redirects back to your app with a code:
   https://yourapp.com/callback?code=AUTH_CODE&state=RANDOM_STATE_VALUE

4. Your app exchanges the code for tokens (server-to-server, not visible to browser):
   POST https://accounts.google.com/oauth/token
   {code: AUTH_CODE, client_id: ..., client_secret: ..., grant_type: authorization_code}

5. Google returns: {access_token, refresh_token, id_token}

6. Your app uses access_token to fetch user profile from Google API
```

**Why the code exchange step?** Tokens are never passed through the browser URL (visible in history, referrer headers). The code is short-lived and useless without the client secret.

### OpenID Connect (OIDC)

OIDC is a thin authentication layer on top of OAuth 2.0. OAuth 2.0 handles "can this app access your data?" — OIDC adds "who is this user?" via the `id_token`.

```
OAuth 2.0: "Your app can access this user's Google Drive"
OIDC:      "This user is alice@gmail.com (id_token proves their identity)"

OIDC id_token is a JWT containing:
  {sub: "google-user-id-123", email: "alice@gmail.com", name: "Alice", ...}
```

---

## 6. Multi-Factor Authentication (MFA)

MFA requires a user to provide two or more factors from different categories:

| Factor | Description | Examples |
|---|---|---|
| **Something you know** | Knowledge factor | Password, PIN, security question |
| **Something you have** | Possession factor | TOTP app (Google Authenticator), hardware key (YubiKey), SMS code |
| **Something you are** | Inherence factor | Fingerprint, face recognition |

MFA dramatically reduces account takeover risk. Even if an attacker obtains a user's password (via phishing, breach, guessing), they still cannot authenticate without the second factor.

**TOTP (Time-based One-Time Password):**

```
User sets up MFA:
  Server generates a secret key, shares with user's authenticator app (via QR code)

Login flow:
  User enters password → correct
  Server asks for MFA code
  User opens authenticator app → sees 6-digit code (changes every 30 seconds)
  Code = TOTP(secret_key, floor(current_time / 30))
  User enters code → server computes same formula → codes match → authenticated
```

The code changes every 30 seconds and requires knowledge of the secret key — an attacker cannot reuse a captured code or guess the next one.

---

## 7. Common Authentication Vulnerabilities

| Vulnerability | How It Works | Defence |
|---|---|---|
| **Credential stuffing** | Attacker uses breached username/password lists from other sites | MFA; breach detection; rate limiting |
| **Brute force** | Attacker tries millions of password combinations | Rate limiting; account lockout; CAPTCHA |
| **Session hijacking** | Attacker steals session cookie | HttpOnly + Secure cookies; short session TTL |
| **JWT algorithm confusion** | Attacker changes JWT header to `alg: none` to bypass signature | Always explicitly verify algorithm |
| **Phishing** | Attacker tricks user into entering credentials on a fake site | FIDO2/WebAuthn (phishing-resistant MFA) |
| **Replay attack** | Attacker reuses a captured valid token | Token expiry; nonce; timestamp binding |

---

## 8. Best Practices

- **Hash passwords with Argon2id or bcrypt** — never encrypt, never plain hash.
- **Short-lived access tokens (15 min) + revocable refresh tokens** for JWT-based systems.
- **Enforce MFA** for all privileged accounts and offer it to all users.
- **Set all cookie security flags:** HttpOnly, Secure, SameSite=Strict.
- **Rate limit authentication endpoints aggressively** — no more than 5 failed attempts per 15 minutes.
- **Use OIDC/OAuth 2.0** for third-party login — never handle third-party credentials directly.
- **Never put sensitive data in JWT payload** — it is encoded, not encrypted; anyone can read it.

---

## 9. Interview Questions

1. What is the difference between authentication and authorisation?
2. Why must passwords be hashed with bcrypt/Argon2 rather than SHA-256?
3. What is a salt and why is it needed?
4. What are the three parts of a JWT? What does the signature prove?
5. What is the key weakness of JWTs and how do you mitigate it?
6. Explain the OAuth 2.0 authorisation code flow.
7. What is TOTP and how does MFA protect against stolen passwords?

---

## 10. Summary

| Concept | Key Takeaway |
|---|---|
| **Password hashing** | Argon2id/bcrypt + salt. Never plaintext, encrypt, or fast-hash. |
| **Sessions** | Stateful. Server-side store. Immediately revocable. HttpOnly + Secure cookies. |
| **JWT** | Stateless. Signed. Cannot revoke before expiry. Short TTL + refresh token. |
| **OAuth 2.0** | Delegated access framework. Authorization code flow is the standard. |
| **OIDC** | Authentication layer on OAuth 2.0. `id_token` proves user identity. |
| **MFA** | Second factor defeats stolen passwords. TOTP is the standard. |

---

## 11. Cross References

**Prerequisites:** 01-security-fundamentals.md · Security (NFR #7)

**Related Topics:** 03-authorisation.md · 07-api-security.md · 04-transport-security.md

**What to Learn Next:** 03-authorisation.md

---

*System Design Engineering Handbook — 09-Security Series*