# 04 — Transport Security

> **09-Security Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Is Transport Security?

Transport security protects data **while it is moving** across a network — between a user's browser and your server, between two microservices, between your application and a database. Without it, anyone with access to the network path (an ISP, a coffee shop router, a compromised internal switch) can read or modify the traffic.

The primary protocol for transport security is **TLS (Transport Layer Security)**. HTTPS is HTTP running over TLS. Every modern secure communication channel uses TLS or is built on its principles.

> **Rule without exceptions:** All network traffic in production must use TLS. Internal service-to-service traffic, database connections, message broker connections — everything. "Internal network" does not mean "trusted network."

---

## 2. What TLS Provides

TLS provides three distinct guarantees, each addressing a different threat:

| Guarantee | What It Means | Threat It Prevents |
|---|---|---|
| **Confidentiality** | Data is encrypted — only the intended recipient can read it | Eavesdropping, packet capture |
| **Integrity** | Data cannot be modified in transit without detection | Man-in-the-middle tampering |
| **Authentication** | The server's identity is verified via certificate | Impersonation, DNS hijacking |

> **All three together are required for secure communication.** Encryption without authentication means you're encrypting traffic to an attacker's server. Authentication without encryption reveals your identity but not your data. You need all three.

---

## 3. How TLS Works — The Handshake

The TLS handshake establishes a secure connection before any application data is sent. Understanding it conceptually is essential for system design.

```
CLIENT                                    SERVER

1. ClientHello
   → TLS version supported
   → Cipher suites supported
   → Random value (client_random)
                                  2. ServerHello
                                     ← Chosen TLS version
                                     ← Chosen cipher suite
                                     ← Random value (server_random)

                                  3. Certificate
                                     ← Server's TLS certificate
                                        (contains public key + identity)

4. Certificate Verification
   → Client checks certificate is signed by a trusted CA
   → Client checks hostname matches certificate
   → Client checks certificate is not expired or revoked

5. Key Exchange
   → Client and server use their random values + certificate public key
      to derive a shared symmetric session key
   → This key is never sent over the network

6. Finished
   → Both sides confirm: "I'm ready to communicate securely"
   → All subsequent communication encrypted with session key
```

**Why a session key instead of just using the public key?**

Asymmetric encryption (public/private key) is computationally expensive. It's used only for the handshake to establish a shared secret. The shared secret is used to derive a fast symmetric key for the actual data transfer.

---

## 4. TLS Certificates — The Identity System

A TLS certificate is a digital document that:
- Proves the identity of a server (this server really is `api.company.com`)
- Contains the server's public key
- Is signed by a Certificate Authority (CA) that clients trust

### Certificate Chain

Certificates form a chain of trust:

```
Root CA (trusted by all browsers/OS by default)
  └── Intermediate CA (signed by Root CA)
       └── Your server certificate (signed by Intermediate CA)
            └── Proves: this server is api.company.com
```

When a client connects to your server:
1. Server presents its certificate
2. Client verifies the certificate is signed by an Intermediate CA it knows
3. Client verifies that Intermediate CA is signed by a Root CA it trusts
4. Client checks the hostname matches the certificate's Subject Alternative Names
5. Client checks the certificate is not expired or in the Certificate Revocation List (CRL)

### Certificate Types

| Type | Validates | Use Case |
|---|---|---|
| **DV (Domain Validated)** | Domain ownership only | Most websites; automated via Let's Encrypt |
| **OV (Organisation Validated)** | Domain + organisation identity | Company websites requiring trust signals |
| **EV (Extended Validation)** | Rigorous legal identity check | Financial institutions, high-value transactions |
| **Wildcard** | Domain and all subdomains (`*.company.com`) | Multiple subdomains under one certificate |
| **SAN (Subject Alt Name)** | Multiple specific domains in one certificate | APIs serving multiple domains |

### Certificate Management

```
Key lifecycle:
  Generate private key (kept secret — NEVER shared)
  Generate Certificate Signing Request (CSR) containing public key
  Submit CSR to CA → CA verifies ownership → issues certificate
  Install certificate on server

Renewal:
  Certificates have expiry dates (usually 90 days for Let's Encrypt, 1 year for commercial)
  Automate renewal (certbot, cloud-provider managed certs)
  Expired certificate = users see browser warning = trust destroyed

Private key protection:
  Never commit to version control
  Store in hardware security module (HSM) for high-value certs
  Rotate if potentially compromised (key compromise = cert invalid)
```

---

## 5. TLS Configuration — Getting It Right

Having TLS is not enough — you must configure it correctly. Weak configuration negates the security.

### TLS Versions

| Version | Status | Notes |
|---|---|---|
| SSL 3.0 | Deprecated | Critically broken (POODLE attack); never use |
| TLS 1.0 | Deprecated | Broken; disabled by PCI-DSS compliance |
| TLS 1.1 | Deprecated | Disabled in all modern browsers |
| TLS 1.2 | Acceptable | Still widely used; requires careful cipher selection |
| TLS 1.3 | Recommended | Significantly improved; faster handshake; stronger by design |

> **Support TLS 1.2 and 1.3. Disable all older versions explicitly.** TLS 1.3 removes the weak cipher suites that made 1.2 misconfiguration dangerous, making 1.3 safer by default.

### Cipher Suites

A cipher suite is the combination of algorithms used in a TLS session: key exchange + authentication + encryption + message authentication.

```
Good (TLS 1.3 examples):
  TLS_AES_256_GCM_SHA384
  TLS_CHACHA20_POLY1305_SHA256

Avoid (weak algorithms):
  Anything with RC4 (stream cipher, broken)
  Anything with MD5 or SHA-1 (weak hash functions)
  Anything with NULL (no encryption!)
  Anything with EXPORT (intentionally weakened for export compliance — historical)
```

### HSTS — HTTP Strict Transport Security

HSTS instructs browsers to always use HTTPS for your domain, even if the user types `http://` or clicks an `http://` link.

```
Response header:
  Strict-Transport-Security: max-age=31536000; includeSubDomains; preload

Effect:
  Browser remembers: always use HTTPS for this domain for 31536000 seconds (1 year)
  Even if user types http://company.com → browser automatically uses https://
  Prevents SSL stripping attacks (downgrade from HTTPS to HTTP)
```

**HSTS preload:** Submit your domain to the browser preload list — browsers ship with your domain already marked as HTTPS-only, even on first visit.

---

## 6. Mutual TLS (mTLS) — Both Sides Authenticate

Standard TLS authenticates the **server** to the **client**. The client knows it's talking to the real server — but the server doesn't know who the client is (from a transport perspective; application auth handles user identity).

**Mutual TLS (mTLS)** requires both sides to present certificates. Both the server and the client prove their identity.

```
Standard TLS:
  Client verifies server certificate ✅
  Server trusts any client ❌ (anyone can connect)

mTLS:
  Client verifies server certificate ✅
  Server verifies client certificate ✅ (only authorised clients can connect)
```

**Where mTLS is used:**

```
Service-to-service communication in microservices:
  Payment Service calls Order Service
  → Payment Service presents its certificate: "I am Payment Service"
  → Order Service verifies: "Is this certificate from a service I trust?"
  → If yes → connection established
  → An attacker who gains network access cannot impersonate Payment Service
     without its private key

API clients with high-privilege access:
  Bank-to-bank connections, payment processors, government APIs
  → Client certificate proves the calling party's identity at the transport layer
```

**mTLS in practice:** Managing certificates for hundreds of microservices is operationally complex. Service meshes (Istio, Linkerd) automate mTLS by issuing short-lived certificates to each service instance and rotating them automatically.

---

## 7. Certificate Pinning

Certificate pinning is the practice of hardcoding an expected certificate (or its public key hash) into a client application. The client refuses to connect unless the server presents exactly that certificate.

```
Normal TLS: Client trusts any certificate signed by any trusted CA
  → Attacker who compromises a CA can issue a valid-looking certificate
  → Man-in-the-middle attack possible

Certificate pinning: Client only trusts one specific certificate or public key
  → Even a CA-signed certificate from a different key is rejected
  → MITM attack fails even if a CA is compromised
```

**Where it's used:** Mobile apps for high-security applications (banking, payments) pin the server's certificate to prevent interception via corporate proxies or malicious CAs.

**Risks:**
- If the pinned certificate expires or is rotated, all existing app versions stop working — requires forced app updates
- Must pin to a public key hash (not the certificate) and include backup pins to avoid complete lockout

---

## 8. Encrypting Data at Rest

Transport security protects data in motion. **Encryption at rest** protects stored data — in databases, object stores, backups, and logs.

```
Data at rest encryption layers:
  Disk/volume encryption:    Encrypts the entire disk
    → If a physical disk is stolen, data is unreadable
    → Cloud providers offer this (EBS encryption, S3 server-side encryption)

  Database-level encryption: Encrypts the database files
    → Transparent to the application
    → AES-256 standard

  Application-level encryption: Application encrypts specific fields before storing
    → Database sees only ciphertext
    → Even a compromised DB admin can't read sensitive fields
    → Required for highest-sensitivity data (PAN, SSN, health records)
    → Uses envelope encryption (see Secrets file 06)
```

**What to always encrypt at rest:**
- Payment card data (PAN, CVV) — required by PCI-DSS
- Government IDs (SSN, passport numbers)
- Health records — required by HIPAA
- Passwords (hashed, which is a form of one-way encryption)
- Encryption keys themselves (key wrapping)

---

## 9. How Large Companies Apply This

| Company | Application | Source |
|---|---|---|
| **Google** | HTTPS everywhere; internal services use mTLS via BeyondCorp | Google BeyondCorp paper (public) |
| **Cloudflare** | TLS termination at edge; TLS 1.3 deployed globally; automated cert management | Cloudflare Blog (public) |
| **Netflix** | Full mTLS between microservices via Envoy proxy sidecar | Netflix Tech Blog (public) |
| **Let's Encrypt** | Automated free DV certificates; enabled HTTPS adoption across the web | Let's Encrypt public documentation |

---

## 10. Best Practices

- **TLS everywhere** — external and internal traffic, service-to-service, database connections.
- **Support TLS 1.2 and 1.3 only** — disable all older versions explicitly.
- **Automate certificate renewal** — expired certificates cause outages; never manage manually.
- **Enable HSTS with preloading** — prevents protocol downgrade attacks.
- **Use mTLS for service-to-service communication** — use a service mesh to automate it.
- **Encrypt sensitive fields at the application level** — don't rely solely on disk encryption.
- **Never store private keys in code or configuration files** — use a secrets manager.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| HTTP for internal services | Internal traffic interceptable on compromised node | TLS everywhere, including internal |
| Expired certificates | Browser warnings; user trust destroyed; service outage | Automate renewal; alert 30 days before expiry |
| TLS 1.0/1.1 still enabled | Vulnerable to known attacks (BEAST, POODLE) | Explicitly disable; TLS 1.2 minimum |
| No HSTS | SSL stripping attacks possible | HSTS header with long max-age |
| Self-signed certs in production | Browser warnings; no CA validation | Use CA-signed certificates everywhere |
| Private key in source code | Credential exposure; cert compromise | Secrets manager; never in version control |

---

## 12. Interview Questions

1. What three properties does TLS provide and what threat does each prevent?
2. Walk through the TLS handshake at a conceptual level.
3. What is a certificate chain and what is a Certificate Authority?
4. What is the difference between standard TLS and mTLS?
5. When would you use mTLS instead of application-level authentication?
6. What is HSTS and what attack does it prevent?
7. What is the difference between encryption in transit and encryption at rest?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **TLS** | Confidentiality + Integrity + Authentication for data in transit. |
| **Handshake** | Asymmetric crypto establishes shared secret → fast symmetric encryption for data. |
| **Certificate** | Proves server identity. Signed by CA. Automate renewal. |
| **TLS 1.3** | Recommended. Safer by design. Faster handshake. |
| **HSTS** | Always use HTTPS. Prevents downgrade attacks. |
| **mTLS** | Both sides authenticate. Standard for service-to-service in zero trust. |
| **Encryption at rest** | Disk-level + DB-level + field-level for highest sensitivity. |

---

## 14. Cross References

**Prerequisites:** 01-security-fundamentals.md · 02-authentication.md

**Related Topics:** 06-secrets-and-key-management.md · Microservices (03-inter-service-communication.md) · Zero Trust (file 01)

**What to Learn Next:** 05-application-security.md

---

*System Design Engineering Handbook — 09-Security Series*