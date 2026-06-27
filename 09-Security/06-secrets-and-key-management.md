# 06 — Secrets & Key Management

> **09-Security Series** — Engineering Handbook
> Language-agnostic · 8–10 min read

---

## 1. What Are Secrets?

Secrets are any credentials, keys, or sensitive configuration values that grant access to systems or protect data. They are the skeleton keys of your infrastructure — whoever possesses them possesses the access they confer.

```
Categories of secrets:
  Credentials:     Database passwords, API keys, service account passwords
  Cryptographic:   Encryption keys, signing keys, TLS private keys, JWT secrets
  Certificates:    TLS certificates with private keys
  Tokens:          OAuth tokens, API bearer tokens, session secrets
  Configuration:   Connection strings containing passwords, webhook secrets
```

> **Secrets are the highest-value target for attackers.** A single leaked database password gives an attacker everything — all user data, all historical records, the ability to modify anything. Every secret must be treated with the same care as the resource it protects.

---

## 2. Where Secrets Must Not Live

The most common cause of secret exposure is storing secrets in the wrong place. These mistakes have caused some of the largest data breaches in history.

### Never in Source Code

```
❌ const DB_PASSWORD = "super_secret_password"
❌ apiKey: "sk-prod-abc123def456"
❌ config.yml: password: "mypassword"
```

Source code is read by many people, committed to version control, and often shared externally (GitHub, contractors, consultants). A secret committed to git history remains there forever — even if "deleted" later. Automated bots scan public repositories for exposed secrets and exploit them within minutes.

### Never in Version Control — Even .env Files

```
❌ .env file committed to git (even if later .gitignored)
❌ config/production.json with real credentials
❌ terraform.tfvars with cloud provider keys

Git history is permanent. "I deleted the commit" does not remove the secret
from every clone, fork, PR comment, and CI/CD log that captured it.
```

### Never in Container Images

```
❌ Dockerfile: ENV DB_PASSWORD=mypassword
   → The image layer containing this is permanent
   → Anyone who pulls the image can extract the secret
   → docker history reveals every ENV instruction
```

### Never in Client-Side Code

```
❌ JavaScript: const API_KEY = "sk-prod-abc123"
   → Anyone who opens DevTools sees it
   → Anyone who downloads your app's JavaScript sees it
   → Secrets in client code are immediately public
```

### Never in Logs

```
❌ logger.info("Connecting to DB with password: " + dbPassword)
   → Logs are often aggregated and widely accessible
   → Log retention may outlast the secret's validity
   → Secret scanners don't catch secrets in logs
```

---

## 3. Secret Managers — The Right Place for Secrets

A secret manager is a dedicated, secure store for secrets. It provides:
- Encrypted storage with access control
- Audit logging of every access
- Automatic rotation capabilities
- Programmatic API for applications to retrieve secrets at runtime

```
Without secret manager:
  Secret in environment variable → readable by anyone with server access
  Secret in config file → readable by anyone with file system access
  Secret in code → readable by anyone with code access

With secret manager:
  Secret stored encrypted in Vault/AWS Secrets Manager
  Application requests secret at runtime: "give me the DB password for service X"
  Secret manager checks: "is service X authorised to read this secret?"
  If yes → returns secret over TLS → secret exists in memory only during use
  Every access logged: who, what, when
```

**Major secret managers:**

| Tool | Type | Key Features |
|---|---|---|
| **HashiCorp Vault** | Open-source / Enterprise | Dynamic secrets, fine-grained policies, multiple auth methods |
| **AWS Secrets Manager** | Managed (AWS) | Automatic rotation for RDS, tight AWS IAM integration |
| **AWS Parameter Store** | Managed (AWS) | Simpler and cheaper than Secrets Manager; good for config |
| **GCP Secret Manager** | Managed (GCP) | IAM-based access; versioning; audit logging |
| **Azure Key Vault** | Managed (Azure) | Keys, secrets, and certificates; HSM-backed option |

---

## 4. Dynamic Secrets — The Most Secure Pattern

Traditional secrets are static: a password created once and used indefinitely. Dynamic secrets are generated on demand and expire automatically.

```
Static DB credentials (traditional):
  DB user: app_service
  DB password: static123  ← same password, forever
  → If leaked: attacker has persistent access until manual rotation
  → Rotation is manual, infrequent, risky (requires coordinated update)

Dynamic DB credentials (Vault):
  Application requests credentials from Vault
  Vault creates a new DB user: vault_app_1705336000
  Vault grants minimal permissions to that user
  Vault issues credentials with TTL of 1 hour
  After 1 hour: Vault revokes the DB user automatically

  If leaked: credentials expire in < 1 hour automatically
  No rotation needed: each credential set is already temporary
```

**HashiCorp Vault** supports dynamic secrets for databases (PostgreSQL, MySQL, MongoDB), cloud providers (AWS, GCP), SSH, and more.

---

## 5. Secret Rotation

Even static secrets must be rotated periodically — and immediately on suspected compromise.

```
Rotation frequency guidance:
  High-sensitivity (payment, production DB): every 30-90 days
  API keys to external services: every 90 days
  Service-to-service tokens: daily or on each deployment (ideally short-lived)
  TLS certificates: automated renewal (Let's Encrypt rotates every 90 days)
  Encryption keys: annually, or on compromise

Rotation process:
  1. Generate new secret
  2. Deploy new secret alongside old (dual acceptance window)
  3. Verify all consumers are using new secret
  4. Revoke old secret
  5. Document rotation in audit log
```

**The dual acceptance window** prevents downtime: during rotation, both old and new secrets are valid for a brief period while all service instances update to the new value.

**Automated rotation** is the only reliable approach at scale. AWS Secrets Manager and Vault can rotate RDS passwords automatically and update the secret store — applications retrieve fresh credentials on their next fetch.

---

## 6. Encryption Key Management

Encryption keys are themselves a type of secret that require additional care. A compromised encryption key doesn't just expose credentials — it exposes every piece of data encrypted with it.

### Envelope Encryption

Rather than encrypting data directly with a master key (which would require rotating the encryption of all data when the key rotates), use envelope encryption:

```
Envelope encryption:
  1. Generate a Data Encryption Key (DEK) — unique per record or per batch
  2. Encrypt the actual data with the DEK (fast, symmetric AES-256)
  3. Encrypt the DEK with the Key Encryption Key (KEK) — stored in HSM/KMS
  4. Store the encrypted DEK alongside the encrypted data

To decrypt:
  1. Retrieve encrypted DEK from storage
  2. Send encrypted DEK to KMS: "decrypt this DEK for me"
  3. KMS checks authorisation → decrypts DEK → returns plaintext DEK
  4. Use plaintext DEK to decrypt data
  5. Discard DEK from memory immediately after use

To rotate the KEK:
  Only need to re-encrypt the DEKs (which are small) — not all the data
```

**Why this matters:** If you encrypt 1 billion records directly with one key and that key is compromised, you must re-encrypt 1 billion records. With envelope encryption, you only re-wrap the DEKs — a much smaller operation.

### Key Management Services (KMS)

Cloud KMS services (AWS KMS, GCP Cloud KMS, Azure Key Vault) provide:

```
Hardware Security Module (HSM) backing:
  Keys never leave the HSM in plaintext — even the cloud provider can't see them
  FIPS 140-2 Level 3 compliance (required for many regulated industries)

Audit logging:
  Every encrypt/decrypt operation logged with caller identity
  Immutable audit trail for compliance

Access control:
  IAM policies control which services can use which keys
  Key policies separate from IAM for fine-grained control

Automatic rotation:
  Scheduled annual key rotation
  Old key versions retained for decryption of existing data
```

---

## 7. Secret Detection — Finding Leaks Before Attackers Do

Automated secret scanning should run at multiple layers:

```
Pre-commit hooks (developer's machine):
  Block commit if a secret pattern is detected in staged files
  Tools: git-secrets, gitleaks, detect-secrets
  Immediate feedback; lowest cost to fix

CI/CD pipeline:
  Scan every commit and pull request for secrets
  Block merge/deploy if secrets detected
  Tools: GitHub Secret Scanning, GitLab SAST, Snyk

Repository scanning:
  Continuous scan of entire repository history
  Catches secrets that pre-existed the hook
  Tools: GitHub Secret Scanning (built-in), TruffleHog

Dependency scanning:
  Scan for secrets accidentally included in third-party packages
```

**What happens when a secret is detected in git history:**

```
1. Rotate the secret immediately (assume it's compromised)
2. Remove from git history (git filter-branch or BFG Repo Cleaner)
   → This rewrites history; all collaborators must re-clone
   → GitHub/GitLab can "block" the secret to prevent future pushes
3. Audit: was the secret accessed? Who accessed it? What did they do?
4. Post-mortem: how did this happen? What process failed?
```

> **Rotating is more important than removing.** A secret removed from git history but not rotated is still compromised — anyone who saw it before removal still has it.

---

## 8. Secrets in CI/CD

CI/CD pipelines need secrets to deploy — database migrations need DB credentials, deployment scripts need cloud API keys. Managing these safely requires specific practices.

```
❌ Wrong:
  hardcoded in .github/workflows/deploy.yml
  passed as environment variable literals in Jenkinsfile

✅ Right:
  Store secrets in CI/CD platform's secret store:
    GitHub Actions Secrets
    GitLab CI/CD Variables (marked as protected + masked)
    CircleCI Environment Variables
  
  Inject at runtime:
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
  
  Mask in logs:
    Secrets marked as "masked" are replaced with *** in all logs
  
  Scope appropriately:
    Production secrets only available to production deployment jobs
    Never available in PR pipelines from forked repositories
```

---

## 9. How Large Companies Apply This

| Company | Practice | Source |
|---|---|---|
| **HashiCorp** | Created Vault; widely used for dynamic secrets and key management | HashiCorp public documentation |
| **Netflix** | Lemur for certificate management; Vault for secrets | Netflix Tech Blog (public) |
| **GitHub** | Secret scanning automatically detects and alerts on pushed secrets | GitHub public documentation |
| **Stripe** | Separate key pairs per environment; automated rotation; HSM-backed key material | Stripe security documentation (public) |

---

## 10. Best Practices

- **Zero secrets in code** — use a secret manager; inject at runtime.
- **Rotate all secrets on a schedule** — and immediately on suspected compromise.
- **Use dynamic secrets** wherever possible — they expire automatically, limiting blast radius.
- **Use envelope encryption** for application-level data encryption — enables key rotation without re-encrypting all data.
- **Run secret detection in pre-commit hooks AND CI/CD** — catch leaks at the earliest possible point.
- **Audit every secret access** — log who retrieved what and when.
- **Scope secrets by environment** — production secrets never accessible in dev or CI.

---

## 11. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Secrets in source code or git | Permanent exposure; automated exploitation within minutes | Secret manager; pre-commit detection |
| No rotation policy | Compromised secret provides indefinite access | Scheduled rotation; automated where possible |
| Same secrets across environments | Dev breach exposes production | Separate secrets per environment |
| No audit logging on secret access | Cannot detect or investigate unauthorised access | Every secret access must be logged |
| Logging secrets | Secret visible in log aggregation tools | Never log secrets; mask in CI/CD outputs |
| Encrypting data directly with master key | Re-encryption of all data required on key rotation | Envelope encryption pattern |

---

## 12. Interview Questions

1. Where must secrets never be stored?
2. What is a secret manager and what does it provide beyond storing the secret?
3. What are dynamic secrets and why are they more secure than static ones?
4. What is envelope encryption and why is it used?
5. What is the first thing you should do when a secret is leaked in git history?
6. How should secrets be managed in a CI/CD pipeline?
7. What is a Hardware Security Module (HSM) and when do you need one?

---

## 13. Summary

| Concept | Key Takeaway |
|---|---|
| **Never in code/git** | Permanent exposure. Automated exploitation. |
| **Secret manager** | Encrypted storage + access control + audit + rotation. |
| **Dynamic secrets** | Generated on demand, expire automatically. Minimal blast radius. |
| **Rotation** | Scheduled + immediate on compromise. Automate. |
| **Envelope encryption** | DEK encrypts data; KEK encrypts DEK. Key rotation without re-encrypting all data. |
| **Secret detection** | Pre-commit hooks + CI/CD scanning. Catch early. |
| **Rotate before remove** | Removing from git history doesn't help if it hasn't been rotated. |

---

## 14. Cross References

**Prerequisites:** 01-security-fundamentals.md · 04-transport-security.md

**Related Topics:** 07-api-security.md · Microservices Operational (MS #5) · Observability (02-logging.md)

**What to Learn Next:** 07-api-security.md

---

*System Design Engineering Handbook — 09-Security Series*