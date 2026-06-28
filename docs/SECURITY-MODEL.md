# AI Studio Platform — Security Model

**Document ID:** SECURITY-MODEL  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** RATIFIED  
**Classification:** INTERNAL — RESTRICTED  
**Authority:** Chief Security Architect  
**Predecessor:** PLATFORM-STANDARDS.md, DATABASE-CATALOG.md, API-CATALOG.md  
**Review Cycle:** Quarterly mandatory; immediate review on any security incident  
**Distribution:** Platform engineering, security team, CTO, compliance officer

---

## Security Authority Notice

> This document defines the platform's security architecture. It is not aspirational — it is prescriptive. Every section is a requirement, not a suggestion.
>
> The platform's previous security score was **2/15** in the architecture review. The root cause: authentication was disabled by default. That default is reversed in this architecture. **Security is on by default. Disabling it requires an explicit, audited, time-bounded exception.**
>
> This document has legal and compliance implications. Changes require Chief Security Architect approval. Changes to the Threat Model section require CTO sign-off.

---

## How to Read This Document

Security requirements are stated with mandatory language: **must**, **must not**, **shall**, **shall not**. Recommendations use **should** and **should not**.

Every security control is traceable to a threat in Section 17 (Threat Model). If a control has no traceable threat, it should not exist. If a threat has no traceable control, it is an open risk.

---

---

# Section 1: Security Philosophy

## 1.1 Core Security Principles

**SP-1: Secure by Default**  
The platform ships with all security controls enabled. Authentication, authorization, rate limiting, and audit logging are active out of the box. There is no "development mode" that disables security in any environment except `local`. In `staging` and `production`, all controls are always active.

**SP-2: Zero Trust**  
No request is trusted because of where it came from. Every request — from a product, from a human operator, from the desktop client, from an AI agent — must authenticate, must be authorized against the specific resource it requests, and must be rate-limited. Network adjacency confers no trust.

**SP-3: Defense in Depth**  
No single control is relied upon to stop an attack. Authentication is backed by rate limiting. Rate limiting is backed by anomaly detection. Authorization is backed by audit logging. Prompt rendering is backed by HMAC verification. Plugin execution is backed by sandboxing. Controls overlap intentionally.

**SP-4: Least Privilege**  
Every identity receives the minimum permissions required. A key scoped to `product:ai-software-factory` cannot access `product:content-factory` resources. A DEVELOPER cannot approve a gate. A plugin declared as read-only cannot write. Privilege is granted explicitly; it is never assumed.

**SP-5: Audit Everything Significant**  
Every security decision that has consequences is logged. Authentication failures, permission denials, key creation, key revocation, gate approvals, template suspensions, config changes, and anomalous patterns — all go to the immutable audit log. Audit is not optional.

**SP-6: Fail Secure**  
When the platform cannot determine whether a request is authorized, it denies it. A degraded auth service returns 503, not 200. A circuit-breaker-tripped provider triggers a budget block, not an unlimited retry. The safe state is rejection.

**SP-7: No Security by Obscurity**  
Error messages never reveal system internals. "Authentication failed" not "Key sk_live_ab... was revoked at 14:23 by admin@example.com." Security cannot depend on attackers not knowing the platform's architecture.

**SP-8: Immutable Audit Trail**  
The audit log cannot be deleted, modified, or selectively accessed. It is append-only at the database level (PostgreSQL RLS). Accessing the audit log is itself audited. The chain of custody is unbroken.

## 1.2 Security Stance by Environment

| Control | local | staging | production |
|---------|-------|---------|-----------|
| Authentication required | Yes | Yes | Yes |
| Authorization enforced | Yes | Yes | Yes |
| Rate limiting active | Yes | Yes | Yes |
| Audit logging active | Yes | Yes | Yes |
| TLS required | No | Yes | Yes |
| HMAC verification active | Yes | Yes | Yes |
| Plugin signature required | No | Yes | Yes |
| Anomaly detection active | No | Yes | Yes |
| Strict CORS | No | Yes | Yes |

**There is no toggle to disable authentication.** The `AISF_AUTH_ENABLED=false` environment variable that existed in the legacy codebase is removed. Its removal is migration 0003 in the database catalog.

---

---

# Section 2: Identity Model

## 2.1 Identity Types

The platform recognizes three identity types. Every authenticated request carries exactly one.

| Identity Type | Who | Credential | Persistence |
|--------------|-----|-----------|-------------|
| **Machine Identity** | Products, agents, automated systems | API key | Persistent until revoked |
| **Operator Identity** | Human platform administrators | API key (current) → SSO (v3.0) | Persistent until revoked |
| **Service Identity** | Internal platform modules | Service-to-service trust (no credential in v1) | — |

**v1.0 limitation:** Human operators use API keys like machines. There is no distinction at the credential level between a human and a machine in v1.0. This is addressed in v3.0 with SSO integration.

## 2.2 Identity Lifecycle

```
Creation ──▶ Active ──▶ Dormant ──▶ Active
                  │
                  ├──▶ Expired (auto, on expires_at)
                  │
                  └──▶ Revoked (explicit, irreversible)
```

**Dormant:** A key unused for > 24 hours that re-authenticates triggers a selective audit event (`security.auth.succeeded` with reason `DORMANT_KEY`). This is a signal, not a block — dormant keys are not auto-expired.

**Expiry:** A key past `expires_at` is rejected with 401 `KEY_EXPIRED`. There is no grace period. Callers must rotate keys before expiry.

**Revocation propagation:** Revocation is synchronous in the database (the row is updated in the same transaction as the audit log entry). The in-process key cache has a 5-second TTL. In a multi-instance deployment, a `security.api_key.revoked` event propagates the revocation to all instances via NATS subscription. Maximum propagation latency: 5 seconds (cache TTL) + NATS delivery time (~10ms).

## 2.3 Identity Representation in Events and Logs

Raw identity values (API key strings, IP addresses, email addresses) **must never** appear in:
- Event payloads
- Log lines
- Error messages
- Audit log entries
- API responses (other than the creation response)

Every identity in events and logs is represented as `SHA-256(raw_value)`. The 256-bit hash is collision-resistant and pre-image resistant at this security level. The hash is sufficient for correlation (two log entries from the same identity have matching hashes) without revealing the identity.

**No salting:** Unlike password hashing, identity hashing is intentionally unsalted. The use case is identification (matching hashes across log entries), not password verification. A salted hash would prevent cross-entry correlation.

## 2.4 Service-to-Service Identity (v1.0)

In v1.0, platform modules communicate in-process (they run in the same FastAPI process). There is no network boundary between modules, so no service-to-service authentication is needed.

In v2.5 (multi-service deployment), service identity will be established via mutual TLS (mTLS). Each service receives a certificate from the platform CA. No shared secrets. No API keys for services.

---

---

# Section 3: Authentication

## 3.1 Authentication Flow

Every request to an authenticated endpoint passes through the Security Middleware before reaching any route handler.

```
Incoming HTTP Request
        │
        ▼
┌─────────────────────────────────────────────────────────┐
│  SecurityMiddleware                                     │
│                                                         │
│  1. Is path in PUBLIC_PATHS?                            │
│     → YES: bypass auth, proceed to handler             │
│     → NO: continue                                      │
│                                                         │
│  2. Extract Authorization header                        │
│     → MISSING: return 401 MISSING_AUTH_HEADER          │
│     → FORMAT: must be "Bearer <key>"                   │
│       → MALFORMED: return 401 MALFORMED_AUTH_HEADER    │
│                                                         │
│  3. Hash the extracted key: SHA-256(raw_key)           │
│     Discard raw_key immediately after hashing          │
│                                                         │
│  4. Look up key_hash in key cache (5s TTL)             │
│     → CACHE HIT: use cached AuthContext                │
│     → CACHE MISS: query security.api_keys              │
│                                                         │
│  5. Validate key record                                 │
│     → NOT FOUND: return 401 KEY_NOT_FOUND              │
│     → STATUS=REVOKED: return 401 KEY_REVOKED           │
│     → expires_at < now(): return 401 KEY_EXPIRED       │
│     → allowed_ips set AND client_ip NOT in set:        │
│       return 401 IP_NOT_ALLOWED                        │
│                                                         │
│  6. Rate limit check (see Section 20)                  │
│     → EXCEEDED: return 429 RATE_LIMIT_EXCEEDED         │
│                                                         │
│  7. Build AuthContext and attach to request            │
│  8. Update key.last_used_at (async, non-blocking)      │
│  9. Proceed to route handler                           │
└─────────────────────────────────────────────────────────┘
        │
        ▼
   Route Handler
```

## 3.2 PUBLIC_PATHS

The following paths bypass authentication. This list is hardcoded — it cannot be extended by configuration.

```python
PUBLIC_PATHS = frozenset({
    "/health",
    "/health/live",
    "/health/ready",
    "/metrics",
    "/api/v1/workspace/capabilities",
})
```

Any request not matching a PUBLIC_PATH must pass authentication. There is no wildcard, no regex, no configurable extension to this list. Adding a path to PUBLIC_PATHS requires a code change, code review, and architect approval.

## 3.3 AuthContext

After successful authentication, a typed `AuthContext` is attached to the request and available to all route handlers and downstream calls.

| Field | Type | Description |
|-------|------|-------------|
| `key_id` | UUID | Key identifier (not the key value). |
| `identity_hash` | string | SHA-256 of the API key. Used for logging and events. |
| `role` | enum | `VIEWER` \| `DEVELOPER` \| `REVIEWER` \| `ADMIN` |
| `scope` | enum | `GLOBAL` \| `PRODUCT` |
| `product_id` | string \| null | Non-null if scope=PRODUCT. |
| `authenticated_at` | datetime | When this auth was performed. |
| `ip_hash` | string | SHA-256 of the client IP. |
| `correlation_id` | UUID | Request correlation ID (from X-Request-Id or X-Correlation-Id). |

The `AuthContext` is immutable once constructed. No route handler may modify it.

## 3.4 Authentication Failure Handling

Authentication failures are rate-limited per IP independently of key-based rate limits. This prevents enumeration attacks where an attacker cycles through key values.

**Failure response format:** All 401 responses use the standard error envelope with `error.code` but never reveal whether a key exists, who owns it, or when it was revoked. The error message is always generic: "Authentication failed."

**IP-based rate limiting on failures:** After 10 consecutive auth failures from the same IP in a 60-second window, that IP is blocked from authentication attempts for 300 seconds. This is implemented in memory per-instance (not distributed in v1.0 — distributed blocking in v2.5 via shared rate limit state).

## 3.5 JWT — Design Decision

The platform does **not** use JWT for authentication. This is a deliberate architectural decision.

**Why not JWT:**
- JWTs are self-contained and cannot be revoked without a blocklist (which reintroduces the database lookup we were trying to avoid)
- The platform's primary use case is machine-to-machine authentication (products calling platform APIs), not user sessions
- JWT expiry requires clock synchronization; API keys with explicit expiry dates are simpler and more predictable
- Short-lived JWTs require token refresh machinery; long-lived JWTs cannot be revoked — both are worse than managed API keys
- The audit requirement for every significant action is better served by a database-backed key that logs `last_used_at`

**When JWT will be introduced (v3.0):** For SSO human login flows where the identity provider issues JWTs. These JWTs will be exchanged for platform-issued API keys (API Key Exchange flow), not used directly as bearer tokens. The platform will remain API-key-native internally.

## 3.6 OAuth 2.0 — Roadmap (v3.0)

OAuth is not implemented in v1.0. When introduced in v3.0:

**Supported flows:**
- Authorization Code + PKCE — for human operator login via SSO
- Client Credentials — for machine-to-machine with external OAuth providers

**OAuth does not replace API keys.** The OAuth flow terminates in an API key issued by the platform. Internal platform authentication always uses API keys. OAuth is an external-facing adapter.

**Supported providers (v3.0):** Google Workspace, Microsoft Entra ID, GitHub (for developer onboarding).

---

---

# Section 4: Authorization

## 4.1 Authorization Model

The platform uses **Role-Based Access Control (RBAC)** with **resource-scoped permissions**. Every mutating operation and every sensitive read is checked against two criteria:

1. **Role check:** Does the caller's role satisfy the required minimum role?
2. **Scope check:** Is the caller's key authorized for the target resource?

Both checks must pass. Passing one and failing the other is a 403 Forbidden.

## 4.2 RBAC Hierarchy

Roles are hierarchical — a higher role satisfies all lower-role requirements.

```
ADMIN (4) ─── can do everything
  │
  ├── REVIEWER (3) ─── can approve gates, govern prompt templates
  │     │
  │     ├── DEVELOPER (2) ─── can submit/manage workflows, render prompts
  │     │     │
  │     │     └── VIEWER (1) ─── read-only access
  │     │
  │     └── (inherits DEVELOPER)
  │
  └── (inherits REVIEWER, DEVELOPER, VIEWER)
```

A REVIEWER satisfies any endpoint that requires DEVELOPER. An ADMIN satisfies any endpoint that requires any role. There are no lateral permissions — a DEVELOPER cannot approve gates even if they own the workflow.

## 4.3 Permission Catalog

Every permission maps to a set of operations. Permissions are checked by name in every route handler, not by role number.

| Permission | Minimum Role | Operations |
|-----------|-------------|-----------|
| `auth:read` | VIEWER | List/get API keys (own only for non-ADMIN) |
| `auth:write` | ADMIN | Create/revoke API keys |
| `workspace:read` | VIEWER | Get workspace, list products, health |
| `workflow:read` | VIEWER | Get/list workflows and task outputs |
| `workflow:write` | DEVELOPER | Submit, cancel, pause, resume workflows |
| `workflow:approve` | REVIEWER | Approve or reject gates |
| `prompt:read` | VIEWER | List/get templates |
| `prompt:render` | DEVELOPER | Render templates, record A/B results |
| `prompt:write` | DEVELOPER | Register, update templates |
| `prompt:govern` | DEVELOPER (DRAFT→REVIEW), REVIEWER (REVIEW→ACTIVE), ADMIN (SUSPEND) | Transition template state |
| `brain:read` | VIEWER | Get patterns, recommendations |
| `brain:write` | DEVELOPER | Record experiences |
| `provider:read` | VIEWER | List providers, get capabilities |
| `knowledge:read` | VIEWER | Query graph, get memory |
| `knowledge:write` | DEVELOPER | Add nodes, set memory |
| `plugin:read` | VIEWER | List plugins |
| `plugin:write` | ADMIN | Install, uninstall, enable, disable plugins |
| `monitoring:read` | VIEWER | Alerts, SLA, metrics |
| `admin:read` | ADMIN | Config, audit log, migrations |
| `admin:write` | ADMIN | Update config, toggle flags, reset circuits |

## 4.4 Authorization Enforcement Points

Authorization is checked in **two places** by design (defense in depth):

**Layer 1 — FastAPI Dependency:** Every route handler declares a `Depends(security_client.require_permission("permission:name"))` dependency. FastAPI resolves this before the handler body executes. If the check fails, the handler body never runs.

**Layer 2 — Service Layer:** The platform service layer (WorkflowService, PromptService, etc.) re-checks permissions before any mutating operation. This is the defense against a hypothetical bug in the routing layer that calls a service method directly.

These two checks are redundant by design. Both must pass.

## 4.5 Scope-Based Authorization

A key with `scope=PRODUCT` and `product_id=ai-software-factory` is additionally restricted:
- Can only submit workflows with `product_id=ai-software-factory`
- Can only read workflows belonging to `product_id=ai-software-factory`
- Cannot list all workflows across products (even with `workflow:read`)
- Cannot access any resource belonging to a different product

This is enforced in the service layer by appending `AND product_id = $authenticated_product_id` to all queries when `scope=PRODUCT`.

A `scope=GLOBAL` key can access all products' resources (within its role's permission set). Only ADMIN keys should be GLOBAL-scoped.

## 4.6 Authorization Decision Logging

Every **denied** authorization decision is:
1. Returned to the caller as 403 Forbidden
2. Published as `security.permission.denied` event
3. Written to the audit log (at MEDIUM severity)
4. Counted against the identity's anomaly baseline

A spike in permission denials from one identity triggers `security.anomaly.detected` (see Section 18).

---

---

# Section 5: API Key Security

## 5.1 Key Generation

API keys are generated using a cryptographically secure random number generator (Python `secrets.token_bytes(32)` — 256 bits of entropy). The key is then encoded as a URL-safe base64 string with a deterministic prefix for visual identification.

**Key format:**
```
sk_<environment>_<role_prefix>_<32_bytes_base64url>

Examples:
  sk_live_dev_a1b2c3d4e5f6789012345678901234  (DEVELOPER, production)
  sk_live_admin_xxxxxxxxxxxxxxxxxxxxxxxxxxxx  (ADMIN, production)
  sk_local_dev_yyyyyyyyyyyyyyyyyyyyyyyyyyyy   (DEVELOPER, local)
```

**Prefix components:**
- `sk_` — Identifies this as a secret key (not a publishable key)
- `<environment>` — `live` (production/staging) or `local` (development)
- `<role_prefix>` — `admin`, `rev`, `dev`, `view` — visual hint for the holder
- `<random>` — 32 bytes (256 bits) of cryptographic randomness

**Why a prefix:** The prefix enables key scanning in codebases (grep for `sk_live_`) and in logs (alert on any log line containing `sk_live_`). Secret scanning tools can be configured to detect platform keys with a regex pattern.

## 5.2 Key Storage

**Never stored:** The raw key value is computed once, returned to the caller in the creation response, and immediately discarded. It is never stored in the database, in memory beyond the creation response, or in any log.

**What is stored:** `SHA-256(raw_key_value)` as `api_keys.key_hash`.

**Authentication lookup:** `SELECT * FROM security.api_keys WHERE key_hash = SHA256(incoming_key) AND status = 'ACTIVE'`

**Why SHA-256 (not bcrypt/argon2):** API key lookup is on the hot path of every authenticated request. bcrypt and argon2 are intentionally slow (for password hashing). At 60 RPM per key × N concurrent keys, bcrypt would add unacceptable latency. SHA-256 is acceptable here because:
- API keys have 256 bits of entropy (vs. user passwords with ~40 bits) — brute force is infeasible
- There is no dictionary of API keys to attack against
- Pre-image attacks on SHA-256 are computationally infeasible at current compute

## 5.3 Key Lifecycle Controls

**Expiry:** Keys may be created with `expires_at`. Expiry is checked on every auth attempt — there is no grace period. Callers must rotate keys before expiry. The platform does not automatically extend or renew keys.

**IP Allowlisting:** Keys may restrict authentication to specific CIDR ranges via `allowed_ips`. This is checked after hash validation. A key with IP allowlisting that receives a request from an unlisted IP returns 401 — the response does not reveal whether the IP allowlisting rejected it (same generic error as all other 401s).

**Rotation policy:**
- ADMIN keys: Must be rotated every 90 days
- Service keys (DEVELOPER/REVIEWER): Must be rotated every 180 days
- Keys with `expires_at` set: Must be rotated before expiry

**Rotation enforcement:** The platform does not force-revoke expired-by-policy keys in v1.0. Monitoring alerts at 30 days and 7 days before `expires_at`. In v2.5, keys without an `expires_at` that are older than 180 days will trigger a mandatory rotation alert.

## 5.4 Key Transmission Security

- Keys are only transmitted over TLS in staging and production
- Keys must never appear in URL query strings (logged by proxies/CDNs)
- Keys must never appear in HTTP response bodies except the creation response
- Keys must never appear in log lines — log scanning alerts on `sk_live_` patterns
- The creation response must be consumed immediately — it is the only time the key is readable

## 5.5 Emergency Key Procedures

**Lost key:** A lost key cannot be recovered. The caller must create a new key and revoke the old one. If the old key cannot be found to revoke by key_id, an ADMIN may search by `key_prefix` (first 8 chars visible in list responses) to find the `key_id`.

**Compromised key:** Treat any suspected compromise as confirmed. Steps:
1. Revoke the key immediately via `DELETE /api/v1/auth/keys/{key_id}` with reason `KEY_COMPROMISED`
2. Review the audit log for all actions taken by the compromised key
3. Assess whether any unauthorized actions occurred
4. If unauthorized actions occurred: initiate Incident Response (Section 25)
5. Create a replacement key with a different role scope and IP allowlist if appropriate

---

---

# Section 6: Secrets Management

## 6.1 Secret Classification

| Secret Type | Classification | Storage | Access |
|-------------|---------------|---------|--------|
| AI Provider API keys (Anthropic, OpenAI, etc.) | CRITICAL | Environment variable or vault | Platform AI ROS only |
| HMAC signing key (prompt integrity) | CRITICAL | Environment variable | Prompt OS only |
| Database connection strings | CRITICAL | Environment variable | Platform startup only |
| NATS credentials | CRITICAL | Environment variable | Platform startup only |
| Platform API keys (user-created) | SENSITIVE | SHA-256 hash in DB | Auth Service only |
| Plugin bundle URLs | SENSITIVE | SHA-256 hash in DB | Plugin Manager only |
| Client IP addresses | SENSITIVE | SHA-256 hash in DB / discarded | Auth Service only |

## 6.2 Secret Storage Rules

**Rule S-1:** Raw secret values are never stored in the database. Only cryptographic hashes or metadata about secrets are stored.

**Rule S-2:** Raw secret values are never logged. Log lines must not contain connection strings, API keys, tokens, or credentials. Structured loggers must sanitize fields named `password`, `secret`, `key`, `token`, `credential`, `auth`.

**Rule S-3:** Secrets are never hardcoded in source code. No secrets in `.env` files committed to version control. Secret detection must be part of the CI pipeline.

**Rule S-4:** Secrets are injected at runtime. The platform reads secrets from environment variables at startup. Secrets do not flow through any configuration API.

**Rule S-5:** Secrets are scoped. The AI ROS module has access to provider API keys. The Prompt OS module has access to the HMAC key. No module has access to another module's secrets.

## 6.3 Secret Naming Convention

All platform secrets use the `AISF_SECRET_` prefix in environment variables:

```
AISF_SECRET_ANTHROPIC_API_KEY
AISF_SECRET_OPENAI_API_KEY
AISF_SECRET_GEMINI_API_KEY
AISF_SECRET_OLLAMA_API_KEY
AISF_SECRET_PROMPT_HMAC_KEY
AISF_SECRET_DB_PASSWORD
AISF_SECRET_NATS_PASSWORD
AISF_SECRET_PLUGIN_VERIFY_KEY
```

Non-secret configuration uses `AISF_` prefix without `_SECRET_`:
```
AISF_DAILY_BUDGET_USD
AISF_MAX_CONCURRENT_EXECUTIONS
AISF_ENVIRONMENT
```

This naming distinction allows automated secret-scanning tools to identify and protect `AISF_SECRET_*` variables specifically.

## 6.4 Vault Integration (v2.0)

In v1.0, secrets are environment variables. In v2.0, a vault integration is added:
- HashiCorp Vault (preferred) or AWS Secrets Manager
- Dynamic secrets: provider API keys are rotated automatically by the vault
- Secret leases: secrets expire and are renewed without platform restart
- Audit trail: vault logs every secret access (separate from platform audit log)

The platform's secret abstraction (`SecretProvider` interface) is designed to swap from environment variables to vault without changing the consuming code.

## 6.5 HMAC Key Management

The HMAC key (`AISF_SECRET_PROMPT_HMAC_KEY`) is the most sensitive non-provider secret:
- It protects the integrity of all prompt templates
- Compromise means an attacker could tamper with any prompt without detection
- It is used by Prompt OS only — no other module accesses it

**Key rotation:** Rotating the HMAC key requires re-signing all ACTIVE prompt template versions. The rotation procedure:
1. Generate new HMAC key
2. Re-compute HMAC signatures for all ACTIVE template versions using the new key
3. UPDATE `prompt.versions.hmac_signature` for all ACTIVE versions (migration)
4. Set the new HMAC key in the environment
5. Restart the platform

This is a maintenance window operation in v1.0. Dual-key support (accept old and new key during rotation window) is planned for v1.1.

---

---

# Section 7: Encryption

## 7.1 Encryption at Rest

**Database:** PostgreSQL data at rest is encrypted at the storage layer using AES-256. This is infrastructure-level encryption — the database process sees plaintext, but the storage medium is encrypted. Column-level encryption is not used in v1.0 (the security boundary is the database server itself, not individual columns).

**Object Storage:** Audit log exports and billing record archives are stored in object storage with AES-256 server-side encryption (SSE).

**Event Bus:** NATS JetStream message storage is encrypted at rest using the same infrastructure-level disk encryption.

**No client-side encryption:** The platform does not encrypt data before writing it to the database. Encryption is the responsibility of the storage layer, not the application layer. This simplifies queries and backups while maintaining the same security guarantee (encrypted storage medium).

## 7.2 Encryption in Transit

**TLS requirement:** All inter-service communication and all client-to-platform communication in staging and production must use TLS 1.2 minimum, TLS 1.3 preferred.

**TLS configuration:**
- Minimum version: TLS 1.2
- Preferred version: TLS 1.3
- Cipher suites: AEAD only (AES-GCM, ChaCha20-Poly1305). No RC4, 3DES, or CBC-mode ciphers.
- Certificate: Valid, not self-signed (in production). ECDSA P-256 or RSA 4096 minimum.
- HSTS: `Strict-Transport-Security: max-age=31536000; includeSubDomains` on all production responses.

**Local environment exception:** TLS is not required in `local` environment. HTTP is permitted locally. This exception does not apply to any other environment.

**NATS TLS:** In staging and production, NATS connections require TLS. The NATS client configuration must include `tls_required: true`.

## 7.3 Data Minimization

The best encryption for sensitive data is not storing it. The platform minimizes what it stores:

- Raw API key values: never stored (hash only)
- Raw IP addresses: never stored (hash only)
- Rendered prompt content: never stored (only the template and variables)
- AI model responses: never stored in the platform database (task output stores a summary, not the raw response)
- User-supplied tool call parameters: stored only in `task_outputs.tool_calls` as a summary (parameter values are not retained)

---

---

# Section 8: Signing and Certificates

## 8.1 Prompt Template Signing (HMAC-SHA256)

Every prompt template version has an HMAC-SHA256 signature computed over its content. This signature is verified on every render.

**Signing:**
```
signature = HMAC-SHA256(
    key = AISF_SECRET_PROMPT_HMAC_KEY,
    message = template_content + "\x00" + str(version_number)
)
```

The version number is included in the message to prevent version rollback attacks (an attacker cannot swap a higher-version content with a lower-version content by reusing the signature).

**Verification:** Before every render, the stored HMAC signature is recomputed and compared using `hmac.compare_digest()` — a constant-time comparison that prevents timing side-channel attacks. A mismatch aborts the render and publishes `prompts.template.render_failed` with `failure_reason=HMAC_MISMATCH`.

**When does this protect against tampering?** The HMAC signature protects against:
- Direct database modification of template content
- SQL injection that modifies template content
- An attacker with database write access who changes a template to inject malicious instructions

The HMAC key is not in the database. An attacker who can write to the database but cannot read the environment variable cannot forge a valid signature.

## 8.2 Plugin Bundle Signing

Plugin bundles must be signed with a trusted private key before they can be installed in staging or production.

**Signing model:**
- The platform maintains a set of trusted public keys (`AISF_SECRET_PLUGIN_VERIFY_KEY`)
- Plugin publishers sign their bundle with a private key the platform trusts
- Installation verifies the signature before allowing sandbox initialization

**Why signed plugins:** An unsigned plugin bundle could contain malicious code that bypasses the sandbox, exfiltrates secrets from the environment, or makes unauthorized network calls. Signature verification ensures only authorized plugin publishers can deploy code.

**Signature algorithm:** Ed25519 (compact, fast, no parameter confusion attacks).

## 8.3 Certificate Management

**API certificates (production):**
- Issued by a trusted CA (Let's Encrypt or enterprise CA)
- Minimum: RSA 2048 or ECDSA P-256
- Renewal: Automated (certbot or ACME client), renewed 30 days before expiry
- Alert: Certificate expiry monitoring with alerts at 30 days and 7 days

**Internal certificates (v2.5 — mTLS):**
- Issued by a platform-internal CA
- Short-lived: 24-hour validity
- Automatically rotated by the certificate management service
- Pinned to the internal CA — no public CA can issue for internal service identities

## 8.4 Cryptographic Standards

| Algorithm | Usage | Status |
|-----------|-------|--------|
| AES-256-GCM | Storage encryption, TLS | APPROVED |
| ChaCha20-Poly1305 | TLS (mobile/constrained) | APPROVED |
| SHA-256 | Identity hashing, HMAC message | APPROVED |
| HMAC-SHA256 | Prompt signing | APPROVED |
| Ed25519 | Plugin signing | APPROVED |
| ECDSA P-256 | TLS certificates | APPROVED |
| RSA 4096 | TLS certificates (legacy compat) | APPROVED (prefer ECDSA) |
| RSA 2048 | TLS certificates (minimum) | APPROVED (prefer 4096) |
| MD5 | — | PROHIBITED |
| SHA-1 | — | PROHIBITED |
| RC4 | — | PROHIBITED |
| DES / 3DES | — | PROHIBITED |
| RSA < 2048 | — | PROHIBITED |
| ECB mode (any cipher) | — | PROHIBITED |
| CBC mode with HMAC | — | PROHIBITED (use AEAD) |

---

---

# Section 9: Zero Trust Architecture

## 9.1 Zero Trust Principles Applied

**"Never trust, always verify"** is applied to every interaction layer:

| Layer | Zero Trust Application |
|-------|----------------------|
| Network | No trusted network segments. Internal traffic gets no elevated trust. |
| Identity | Every request authenticated regardless of source. Products authenticate to the platform even if on the same host. |
| Device | Desktop client authenticates via API key — no implicit trust from being on a developer's machine. |
| Application | Every module authenticates every operation. Prompt OS does not trust that the caller "already checked" authorization. |
| Data | Data access validated per-request against current permissions. Past-authenticated sessions do not expand to new resources. |

## 9.2 Micro-Segmentation Model

In v1.0 (single process), micro-segmentation is enforced at the schema ownership level (see DATABASE-CATALOG Section 2). No module accesses another module's schema directly.

In v2.5 (multi-service), micro-segmentation is enforced by:
- Network policy: Services may only communicate on declared ports
- mTLS: Service-to-service communication requires mutual certificate authentication
- Service mesh (optional): Envoy sidecar enforces policy without application code changes

## 9.3 Lateral Movement Prevention

**Platform modules cannot impersonate each other.** There is no elevated internal API that bypasses authentication. When the WorkflowRuntime calls PromptOS to render a template, it uses the same API that external callers use, with a service-scoped API key.

**No ambient authority.** A compromised workflow task cannot escalate to AI ROS budget controls. A compromised prompt renderer cannot access billing data. Each module has a key scoped to its declared permissions.

## 9.4 Continuous Verification

Authorization is not a one-time decision at login. It is re-checked:
- On every request (permission check in FastAPI dependency)
- On every service-layer mutation (second permission check)
- On every data access (scope check — product_id filtering)

A key that had permission at T=0 and is revoked at T=5 will fail at T=5+ε. There is no "already authenticated" bypass.

---

---

# Section 10: Session Management

## 10.1 Session Model: Stateless

The platform is **stateless**. There are no server-side sessions. Every request carries its own authentication (API key in Authorization header). The server does not maintain session state between requests.

**Implications:**
- No session fixation attacks
- No session hijacking attacks (no session cookie)
- No CSRF attacks (no cookies)
- No session expiry to manage (key expiry is the equivalent)

## 10.2 WebSocket Session (Desktop)

The desktop WebSocket connection (`GET /api/v1/desktop/ws`) is the one exception to the fully stateless model. The WebSocket is a long-lived connection.

**WebSocket authentication:**
1. Client obtains a `session_id` via `GET /api/v1/desktop/session` (authenticated with API key)
2. Client upgrades to WebSocket, presenting both the API key and the session_id
3. The platform validates the API key at upgrade time
4. The WebSocket connection is held in memory on the server (not in the database)
5. If the API key is revoked during an active WebSocket session, the connection is closed within 5 seconds (key cache TTL + periodic revalidation)

**WebSocket session revalidation:** The platform re-validates the WebSocket's API key every 60 seconds. If the key is revoked, expired, or changed, the connection is closed with code 4001 (unauthorized).

**WebSocket session limits:**
- Maximum 5 concurrent WebSocket connections per API key
- Maximum message rate: 100 messages/second from server (controlled by platform)
- No inbound messages accepted after handshake (server-push only model)

## 10.3 Correlation ID as Request Context

While there are no sessions, a `correlation_id` flows through every request as a lightweight request context:

- Generated by the platform if not provided by the caller
- Caller may supply `X-Correlation-Id: <uuid>` to chain with an upstream request
- Included in every log line, event payload, and error response
- Used to trace a request through the system without storing server-side state

This is not a session — it is a tracing primitive. Correlation IDs do not confer any authorization.

---

---

# Section 11: Permission Model (Detailed)

## 11.1 Permission Evaluation Algorithm

```
function can(identity: AuthContext, permission: str, resource: Resource) -> bool:

    # Step 1: Role check
    required_role = PERMISSION_ROLE_MAP[permission]
    if identity.role < required_role:
        log("permission_denied", permission, "role_insufficient")
        return False

    # Step 2: Scope check
    if identity.scope == "PRODUCT":
        if resource.product_id != identity.product_id:
            log("permission_denied", permission, "scope_mismatch")
            return False

    # Step 3: Permission-specific overrides
    # (e.g., prompt:govern has role-dependent target states)
    if permission == "prompt:govern":
        if not governance_role_check(identity.role, target_state):
            log("permission_denied", permission, "governance_role")
            return False

    # Step 4: Resource state check
    # (e.g., cannot cancel a COMPLETED workflow)
    if not resource_allows_operation(resource, permission):
        return False  # 409 Conflict, not 403

    return True
```

## 11.2 Permission Inheritance and Delegation

**Inheritance:** RBAC roles are strictly hierarchical. ADMIN inherits all REVIEWER permissions. REVIEWER inherits all DEVELOPER permissions. There is no mechanism to grant individual permissions outside the role hierarchy.

**No delegation:** An ADMIN cannot delegate ADMIN permissions to a DEVELOPER key. A key's role is fixed at creation. Escalating a key's role requires revoking the old key and creating a new one with the higher role — and the creator must hold at least the target role.

**No permission subsets:** You cannot create a key that has `workflow:write` but not `workflow:read`. Permissions come with their role's full set. Future work: attribute-based access control (ABAC) for fine-grained permission subsets.

## 11.3 Sensitive Permission Pairs

Some permissions are more sensitive when combined. These combinations require ADMIN role or additional review:

| Permission Combination | Risk | Mitigation |
|----------------------|------|-----------|
| `auth:write` + any | Can create privileged keys | ADMIN only. All key creations are audited. |
| `prompt:govern` + `prompt:write` | Can write and self-approve a malicious template | REVIEWER can govern only to ACTIVE; content was written by DEVELOPER first. The two-step process (write + separate governance approval) prevents self-signing. |
| `admin:write` + `plugin:write` | Can install plugins and configure them | ADMIN only. Both actions are audited. |
| `workflow:approve` alone | Can approve gates without submitting workflows | Acceptable — this is the REVIEWER's purpose. Gate context required for approval. |

## 11.4 Time-Bounded Permissions (Future: v2.0)

In v2.0, permissions may have a `valid_until` timestamp — a key may be granted REVIEWER permissions for a specific code review window, then revert to DEVELOPER. This is not implemented in v1.0. In v1.0, time-bounded access requires creating a key with `expires_at`.

---

---

# Section 12: Workspace Isolation

## 12.1 Isolation Model

Products registered in the workspace are isolated from each other by the `product_id` scope enforcement. No product can access another product's:
- Workflows
- Task outputs
- Budget data
- Knowledge graph entries (workspace-scoped)

**Isolation boundary:** A PRODUCT-scoped API key enforces isolation at the authorization layer (scope check in the permission evaluation algorithm). The database layer additionally filters all queries with `WHERE product_id = $authenticated_product_id`.

**GLOBAL-scoped keys:** Only ADMIN keys should be GLOBAL-scoped. A GLOBAL DEVELOPER key can access all products' workflows — this is appropriate for platform tooling and monitoring, not for individual products.

## 12.2 Shared Platform Resources

Some platform resources are shared across products by design. These shared resources have their own isolation mechanisms:

| Shared Resource | Isolation Mechanism |
|----------------|-------------------|
| Prompt templates | `owner` field scopes templates; ACTIVE templates are readable by all DEVELOPER+ keys |
| Brain patterns | Shared — all products contribute to and benefit from the shared pattern library |
| Knowledge graph (global scope) | Shared global nodes; workspace-scoped nodes isolated by workspace_id |
| Providers | Shared — all products use the same provider pool; budget isolation per product |
| Plugin registry | Shared — installed plugins available to all products |
| Audit log | ADMIN only — not a product resource |

## 12.3 Workspace Health Visibility

Every authenticated identity can read workspace health (`GET /api/v1/workspace`). This includes health of all registered products. This is intentional: platform health is not a secret, and products need to know if a dependency is degraded.

**What is not visible:** One product cannot see another product's workflows, costs, or outputs — only their own.

---

---

# Section 13: Tenant Isolation (v2.5)

## 13.1 Current State: Single-Tenant

v1.0 is a single-tenant platform. All products run in a shared environment. There is no tenant boundary — workspace isolation (Section 12) is the only separation layer.

## 13.2 Multi-Tenant Architecture (v2.5 Design)

When multi-tenancy is introduced in v2.5:

**Tenant boundary:** A tenant is a logical group of products with its own:
- API key namespace (keys scoped to a tenant)
- Budget limit (per-tenant daily spend cap)
- Knowledge graph (tenant-scoped knowledge is not shared across tenants)
- Audit log (tenants see only their own audit entries)

**Tenant isolation enforcement:**
1. API keys carry a `tenant_id` claim
2. All database queries are filtered by `tenant_id` (same pattern as `product_id` scope today)
3. NATS subjects are tenant-namespaced: `tenant.{tenant_id}.workflows.>`
4. Shared resources (Brain patterns, providers, plugins) remain shared

**Cross-tenant access:** No identity may access another tenant's data. There is no cross-tenant API. ADMIN keys are tenant-scoped — a platform super-admin role (distinct from ADMIN) has cross-tenant visibility for platform operations.

## 13.3 Tenant Isolation Test Requirements

Before v2.5 ships, tenant isolation must pass these tests:
1. A tenant-A API key cannot read tenant-B workflows (scope check)
2. A tenant-A API key cannot see tenant-B audit log entries
3. A tenant-A product cannot exhaust tenant-B's AI budget
4. A compromised tenant-A key cannot pivot to tenant-B resources via any API endpoint

---

---

# Section 14: Rate Limiting

## 14.1 Rate Limit Architecture

Rate limiting is enforced in the Security Middleware, after authentication and before route handling. An unauthenticated request that passes PUBLIC_PATHS is subject to IP-based rate limiting.

**Rate limit hierarchy:**

```
┌───────────────────────────────────────────────────────┐
│  Tier 1: Global platform RPM                          │
│  Limit: 1,000 RPM total across all callers            │
│  Purpose: Protect platform from aggregate overload    │
└───────────────────────────────────────────────────────┘
                         │ if under global limit
                         ▼
┌───────────────────────────────────────────────────────┐
│  Tier 2: Per-identity RPM                             │
│  Limit: per-endpoint (see table below)                │
│  Purpose: Fair share per consumer                     │
└───────────────────────────────────────────────────────┘
                         │ if under identity limit
                         ▼
┌───────────────────────────────────────────────────────┐
│  Tier 3: Per-path RPM (for sensitive endpoints)       │
│  Limit: stricter for auth:write, admin:write          │
│  Purpose: Prevent targeted abuse                      │
└───────────────────────────────────────────────────────┘
                         │ if all pass
                         ▼
                    Route Handler
```

## 14.2 Rate Limit Table by Endpoint Group

| Group | Limit | Burst | Window |
|-------|-------|-------|--------|
| Prompt render (PR-001) | 600 RPM | 60 | 1 minute |
| Workflow status reads | 120 RPM | 20 | 1 minute |
| Workflow writes (submit, cancel) | 30 RPM | 5 | 1 minute |
| Brain experience record | 300 RPM | 30 | 1 minute |
| Authentication endpoints | 30 RPM | 5 | 1 minute |
| Admin endpoints | 10 RPM | 2 | 1 minute |
| Plugin management | 10 RPM | 2 | 1 minute |
| Key creation / revocation | 10 RPM | 2 | 1 minute |
| Health endpoints (public) | 600 RPM | 100 | 1 minute |
| Metrics (public) | 300 RPM | 50 | 1 minute |
| WebSocket connections | 5 concurrent | — | — |

## 14.3 Rate Limit Algorithm

**Token bucket** per identity per endpoint group:
- Bucket capacity = burst limit
- Refill rate = RPM / 60 tokens per second
- On each request: if bucket has >= 1 token, consume 1 and proceed; else 429

**Storage:** In-process per-instance in v1.0 (memory dict, rate limit state is per-instance). In v2.5, rate limit state is stored in shared Redis to enforce limits across instances.

**v1.0 limitation:** In a multi-instance deployment with v1.0 rate limiting, a caller could exceed the intended limit by distributing requests across instances. This is acceptable for v1.0 (single instance). Fix is in v2.5 backlog.

## 14.4 Rate Limit Responses

All rate-limited responses return 429 with:
- `Retry-After: <seconds>` header
- `X-RateLimit-Limit: <limit>`
- `X-RateLimit-Remaining: 0`
- `X-RateLimit-Reset: <unix_timestamp>`
- Event `security.rate_limit.exceeded` published
- Written to audit log

## 14.5 Unauthenticated Rate Limiting

Requests to PUBLIC_PATHS that fail or arrive without authentication are rate-limited by IP:

| Scenario | Limit | Window |
|----------|-------|--------|
| Public endpoints (normal) | 600 RPM per IP | 1 minute |
| Authentication failures per IP | 10 per IP | 60 seconds |
| After 10 auth failures: backoff | Block for 300 seconds | — |

The auth failure rate limit is specifically designed to slow brute-force key enumeration.

---

---

---

# Section 15: Provider Sandboxing

## 15.1 Provider Trust Model

Providers are external AI model services (Anthropic Claude, OpenAI, Google Gemini, Ollama, etc.). They are trusted for their declared function (returning AI model completions) and untrusted for everything else.

**Provider capabilities:** A provider may return any text content in its response. This content:
- Is never executed as code by the platform
- Is never rendered as HTML without sanitization
- Is stored only as structured task output (not raw response body)
- Is passed to the next workflow step only as an input parameter — never as a code path

**Provider attack surface:**
1. **Prompt injection:** A malicious document in the AI's context instructs it to ignore its system prompt and perform unauthorized actions
2. **Exfiltration via tool calls:** A prompt-injected model might try to use tool calls to exfiltrate data
3. **Resource exhaustion:** A provider that never returns (or always returns maximum tokens) can exhaust the budget or timeout pool
4. **Credential abuse:** A compromised provider key could be used externally

## 15.2 Provider Sandbox Controls

**Budget hard cap:** AI ROS enforces a per-request token budget. Requests exceeding the budget are aborted before the response is processed. Daily budget enforcement prevents a runaway provider key from exhausting spend.

**Timeout enforcement:** Every provider call has a timeout:
- URGENT queue: 30 seconds
- STANDARD queue: 300 seconds
- BATCH queue: 3,600 seconds

A response not received within the timeout causes the task to fail (not the workflow). Budgets are not charged for timed-out requests.

**Tool call allowlist:** When AI models make tool calls (function calls), the AI ROS validates each tool call against the workflow's declared tool allowlist. A tool call to a tool not in the allowlist is rejected — the model's response is discarded, the task fails, and `security.tool_call.unauthorized` event is published.

**Provider key isolation:** Each provider has a dedicated API key stored as `AISF_SECRET_<PROVIDER>_API_KEY`. Provider keys are used only by the AI ROS provider adapter for that provider — no other module or product has access to provider keys.

**Response content validation:** Provider responses are validated against a schema (expected JSON structure for structured outputs, or a max-length cap for text). Responses exceeding the max length are truncated and flagged.

## 15.3 Prompt Injection Mitigation

Prompt injection is when untrusted content in the AI's context successfully overrides its instructions.

**Detection (detective control):** The platform monitors for signatures of prompt injection attempts:
- Phrases like "ignore previous instructions", "you are now a", "disregard your system prompt" in AI responses trigger `security.prompt_injection.detected` (SC-001)
- The event is RESTRICTED classification, triggers anomaly review

**Prevention (preventive controls):**
1. System prompts are rendered from HMAC-verified templates — the system prompt cannot be tampered by an attacker who can modify the database
2. User content (documents, code) is injected into the prompt only as a delimited, labeled section
3. Untrusted content is wrapped in structured markers (```user-input\n...\n```) and the system prompt instructs the model to treat content in those markers as untrusted data

**Limitation:** Prompt injection is an unsolved problem at the AI model level. The platform's controls reduce risk but cannot eliminate it. This is a known limitation (see Threat Model, T-AI-01).

## 15.4 Provider Circuit Breaker

The AI ROS CircuitBreaker protects against a provider that is degraded or unreliable:

```
CLOSED ──(5 failures in 60s)──▶ OPEN ──(30s)──▶ HALF_OPEN ──(success)──▶ CLOSED
                                                      │
                                                (failure)──▶ OPEN (again)
```

A CLOSED-to-OPEN transition:
- Publishes `providers.provider.circuit_opened`
- Blocks all new requests to that provider
- Logs to audit log
- Triggers monitoring alert

A provider in OPEN state is not silently skipped — the requesting workflow task fails with `PROVIDER_CIRCUIT_OPEN`. This is a deliberate choice to surface the problem rather than silently fall back to a different provider (which would confuse cost attribution and debugging).

---

---

# Section 16: Plugin Sandboxing

## 16.1 Plugin Security Model

Plugins extend the platform with custom functionality. They are the highest-risk extension point because they execute arbitrary code. The plugin security model assumes plugins are untrusted until verified.

**Threat model for plugins:**
- A malicious plugin could exfiltrate platform secrets from the environment
- A malicious plugin could access other products' data via internal API calls
- A compromised legitimate plugin (supply chain attack) could be used to pivot
- A buggy plugin could exhaust CPU, memory, or disk

## 16.2 Plugin Permission Declaration

Every plugin declares its permissions in a manifest at installation time. The platform uses this manifest to enforce capability restrictions in the sandbox.

**Declared permission categories:**

| Category | What it controls |
|----------|----------------|
| `filesystem:read` | Can read files from declared paths |
| `filesystem:write` | Can write files to declared paths |
| `network:http` | Can make outbound HTTP calls to declared domains |
| `network:none` | No outbound network (default) |
| `platform:workflow_read` | Can read workflow status (own workflows only) |
| `platform:brain_write` | Can record experiences |
| `env:read:names` | Can read declared environment variable names |
| `process:none` | Cannot spawn subprocesses (default) |

**Principle of least privilege:** A plugin that says it needs no network should have `network:none` — the sandbox enforces it. A plugin that needs to call one specific API declares `network:http:["https://api.example.com"]`.

Permissions are immutable once a plugin is installed. Changing permissions requires uninstalling and reinstalling with a new manifest.

## 16.3 Sandbox Implementation Model

The plugin sandbox in v1.0 is a Python restricted execution environment. In v1.1, this upgrades to a WebAssembly (WASM) sandbox for stronger isolation.

**v1.0 sandbox controls:**
- Plugin code runs in a separate Python subprocess with reduced privileges
- `sys.modules` is filtered — stdlib modules not matching declared permissions are unavailable
- `subprocess` is blocked entirely (no `process:spawn` permission in v1.0)
- File system access is enforced by path prefix checking before any open() call
- Network calls are proxied through the platform's HTTP client, which checks the domain against the declared allowlist

**v1.1 sandbox controls (WASM):**
- Plugin compiled to WASM binary at installation time
- WASM runtime enforces memory isolation
- System calls (network, filesystem) are mediated by the WASM host
- Stronger isolation — a WASM plugin cannot access Python memory space

## 16.4 Plugin Signature Verification

In staging and production, a plugin bundle must be signed:

**Verification steps at installation:**
1. Verify bundle signature against trusted public key (Ed25519)
2. Verify bundle manifest hash matches the embedded hash
3. Verify all declared permissions are in the platform's permitted permission set
4. Verify the plugin has not been revoked in the plugin revocation list

**Revocation list:** The platform maintains a list of revoked plugin bundles (by bundle hash). Before installing, the bundle hash is checked against this list. The revocation list is updated via the admin API.

**Bypass (local only):** In `local` environment, plugin signature verification can be bypassed with `AISF_PLUGIN_SKIP_SIGNATURE_VERIFY=true`. This variable has no effect in `staging` or `production`.

## 16.5 Plugin Isolation Boundaries

| Boundary | v1.0 enforcement | v1.1 enforcement |
|----------|-----------------|-----------------|
| Memory | Python subprocess (weak) | WASM linear memory (strong) |
| Filesystem | Path prefix check | WASM host mediation |
| Network | HTTP proxy with allowlist | WASM host mediation |
| CPU | Signal-based timeout (30s max) | WASM fuel metering |
| Platform API | Service key with plugin permissions | Same |
| Environment | Subprocess env is filtered | Same |

## 16.6 Plugin Violation Response

When the sandbox detects a violation (permission not declared, timeout exceeded, invalid signature during load):
1. Plugin execution is aborted
2. `security.plugin.sandbox_violated` event published (SC-002)
3. The plugin is automatically disabled
4. Audit log entry written (CRITICAL severity)
5. Alert triggered (immediate notification)
6. The workflow task that invoked the plugin fails with `PLUGIN_SANDBOX_VIOLATION`

A plugin that triggers a sandbox violation is disabled immediately and automatically. Re-enabling requires ADMIN action, which is audited. Two sandbox violations from the same plugin → permanent ban (must reinstall with a new manifest and explanation).

---

---

# Section 17: Desktop Security

## 17.1 Desktop Client Threat Model

The PySide6 desktop application runs on a developer's local machine. Its specific threats differ from server-side API threats:

| Threat | Description |
|--------|-------------|
| DT-1: Local key exposure | API key stored in OS keychain or config file — accessible to other processes |
| DT-2: WebSocket interception | Local WebSocket connection intercepted by malicious process |
| DT-3: XSS via content display | AI-generated content rendered in a WebView and executes JS |
| DT-4: Malicious update | Auto-update mechanism delivers a malicious binary |
| DT-5: Debug endpoint exposure | Desktop app exposes a debug port that is accessible network-wide |

## 17.2 API Key Storage in Desktop

The desktop client must not store the API key in plaintext on disk. Storage options by OS:

| OS | Storage Mechanism | Security Level |
|----|------------------|---------------|
| macOS | macOS Keychain via `keyring` | System-level encryption |
| Windows | Windows Credential Manager via `keyring` | System-level encryption |
| Linux | Secret Service API (D-Bus) via `keyring` | Varies by implementation |

**First run:** If no credential store is available, the desktop client prompts for the API key on every launch and holds it only in process memory. It never writes to a plaintext config file.

**Config file fallback:** If the user explicitly configures a config file path, the key is stored as `SHA-256(key)` in the config (for display/identification only) — never the raw key. The raw key must come from the credential store or user input.

## 17.3 WebSocket Security

The desktop WebSocket connection (`wss://` in staging/production; `ws://localhost:8000/api/v1/desktop/ws` in local):

**Authentication:** The WebSocket connection carries the API key in an initial authentication frame (not in the URL, not in a cookie).

**No remote WebSocket:** The desktop WebSocket endpoint binds to `localhost` only. It is not accessible from the network. This prevents DT-2 (local WebSocket interception from a network attacker).

**Binding enforcement:** The platform configuration `AISF_DESKTOP_BIND_HOST=127.0.0.1` is enforced. Setting this to `0.0.0.0` is rejected at startup with a fatal error in staging and production.

## 17.4 Content Rendering Security

AI-generated content displayed in the desktop must not be executed as code.

**WebView sandboxing:** Any content rendered in a WebView (embedded browser) must have JavaScript disabled. AI-generated HTML output is rendered with `QWebEngineSettings.JavascriptEnabled = False`.

**Plain text for AI output:** The primary display for AI model output is a plain text / markdown renderer, not a full HTML/JS renderer. Markdown is parsed to a sanitized subset (no `<script>`, no `<iframe>`, no inline event handlers, no raw HTML passthrough).

**Hyperlink validation:** Links in AI output are not automatically followed. Clicking a link opens the system browser after displaying the URL to the user for confirmation. Links with `javascript:` scheme are blocked.

## 17.5 Debug Interface Security

The desktop client may expose a local debug interface (e.g., PyCharm debugger port, uvicorn reload port) during development.

**Controls:**
- Debug ports must bind to `127.0.0.1` only — never `0.0.0.0`
- Debug ports must not be open in production builds
- Production builds are compiled with debug symbols stripped

**Auto-update (v1.1):**
- Updates are delivered as signed bundles (Ed25519 signature)
- The desktop verifies the signature against the platform's update CA before installing
- Auto-update is disabled in v1.0 — manual updates only

## 17.6 Desktop Key Lifetime

The API key held by the desktop client:
- Is validated at every app launch (a startup call to `GET /health`)
- Is re-validated every 15 minutes during active use (background health check)
- Is cleared from memory on app exit
- Is never written to a temp file, log file, or crash dump

---

---

# Section 18: Audit Log Architecture

## 18.1 Audit Log Design Principles

The audit log is the platform's security record of truth. Its design requirements:

1. **Immutable:** Records cannot be updated or deleted after insertion
2. **Append-only:** Only INSERT is permitted — no UPDATE, no DELETE
3. **Self-auditing:** Accessing the audit log is itself audited
4. **Complete:** All 41 audited event types generate an audit record
5. **Non-repudiable:** Audit records include the identity_hash — the identity cannot deny the action
6. **Correlated:** Audit records include correlation_id — every audit record can be linked to the request that caused it

## 18.2 Audit Schema

```
security.audit_log
├── audit_id         UUID (PK, immutable)
├── occurred_at      TIMESTAMPTZ (immutable, set by trigger)
├── event_category   ENUM (immutable)
├── event_type       TEXT (immutable, format: "<domain>.<entity>.<verb>")
├── severity         ENUM (LOW/MEDIUM/HIGH/CRITICAL)
├── identity_hash    TEXT (SHA-256 of API key, immutable)
├── identity_role    ENUM (VIEWER/DEVELOPER/REVIEWER/ADMIN)
├── product_id       TEXT NULLABLE (non-null for PRODUCT-scoped keys)
├── resource_type    TEXT NULLABLE (what was acted on)
├── resource_id      UUID NULLABLE (which specific resource)
├── action           TEXT (what was attempted)
├── outcome          ENUM (SUCCESS/DENIED/FAILED/ANOMALY)
├── ip_hash          TEXT (SHA-256 of client IP)
├── correlation_id   UUID (request correlation ID)
├── details          JSONB (event-specific metadata)
└── [no updated_at, no deleted_at — immutability enforced by RLS]
```

**PostgreSQL RLS policy for audit log:**
```sql
ALTER TABLE security.audit_log ENABLE ROW LEVEL SECURITY;

-- Allow only INSERT (no UPDATE, no DELETE)
CREATE POLICY audit_log_insert_only
    ON security.audit_log
    FOR INSERT
    TO platform_role
    WITH CHECK (true);

-- Enforce no UPDATE via separate policy (default deny is implicit)
-- Enforce no DELETE via separate policy (default deny is implicit)
```

## 18.3 Audited Event Types (41)

**Authentication (7):**
- `security.auth.succeeded` — Successful authentication
- `security.auth.failed` — Authentication failure (with failure reason)
- `security.api_key.created` — API key created
- `security.api_key.revoked` — API key revoked
- `security.api_key.expired` — API key rejected as expired
- `security.api_key.rotated` — Old key deactivated, new key created (linked by metadata)
- `security.auth.dormant_key` — Successful auth from a dormant key

**Authorization (2):**
- `security.permission.denied` — Permission check failed
- `security.scope.denied` — Scope check failed (wrong product_id)

**Security Events (4):**
- `security.rate_limit.exceeded` — Rate limit triggered
- `security.prompt_injection.detected` — Prompt injection signature detected in AI response
- `security.plugin.sandbox_violated` — Plugin violated sandbox permissions
- `security.anomaly.detected` — Anomaly baseline breached

**Workflow (5):**
- `workflows.workflow.submitted` — Workflow submitted
- `workflows.workflow.cancelled` — Workflow cancelled
- `workflows.workflow.gate_approved` — Human gate approved
- `workflows.workflow.gate_rejected` — Human gate rejected
- `workflows.workflow.paused` — Workflow paused

**Prompt (5):**
- `prompts.template.created` — Template registered
- `prompts.template.updated` — Template updated
- `prompts.template.state_changed` — Governance state transition
- `prompts.template.suspended` — Template suspended
- `prompts.template.render_failed` — Render failed (including HMAC mismatch)

**Provider (3):**
- `providers.provider.circuit_opened` — Circuit breaker opened
- `providers.provider.budget_hard_cap_hit` — Daily budget exhausted
- `providers.provider.provider_disabled` — Provider administratively disabled

**Plugin (4):**
- `plugins.plugin.installed` — Plugin installed
- `plugins.plugin.uninstalled` — Plugin uninstalled
- `plugins.plugin.enabled` — Plugin enabled
- `plugins.plugin.disabled` — Plugin disabled (manual or automatic)

**Admin / Config (5):**
- `admin.config.updated` — Configuration value changed
- `admin.feature_flag.toggled` — Feature flag toggled
- `admin.migration.applied` — Database migration applied
- `admin.circuit_breaker.reset` — Circuit breaker manually reset
- `admin.audit_log.accessed` — Audit log read (self-audit)

**Billing (4):**
- `billing.invoice.created` — Invoice generated
- `billing.budget.hard_cap_enforced` — Budget hard cap blocked a request
- `billing.budget.daily_budget_updated` — Daily budget changed
- `billing.cost_record.disputed` — Cost record flagged for dispute

**Knowledge / Brain (2):**
- `knowledge.graph.node_created` — Knowledge graph node added
- `brain.experience.recorded` — Experience recorded (audit only for CRITICAL experiences)

## 18.4 Audit Log Retention

The audit log is retained for **7 years** (compliant with SOC 2 and most financial audit requirements).

- Partition strategy: Monthly partitions on `occurred_at`
- Archival: Partitions > 90 days old are compressed and moved to cold storage (object storage)
- Deletion: Partitions are never dropped (unlike other time-series tables)
- Access to partitions > 90 days old: Requires querying cold storage (slower, appropriate for forensics)

## 18.5 Audit Log Access

**Who can read:** Only ADMIN role with `admin:read` permission.

**How to access:** `GET /api/v1/admin/audit` with time range and optional filters. Pagination required (max 1,000 records per page).

**Access is audited:** Every access to `GET /api/v1/admin/audit` generates an `admin.audit_log.accessed` event that itself writes to the audit log. This creates a chain of custody — if someone tries to read and selectively edit the audit log, their access is itself recorded.

**Export format:** JSONL (one JSON object per line) for bulk forensic export. The export operation generates an `admin.audit_log.accessed` record with `details.export=true`.

---

---

# Section 19: Compliance

## 19.1 Compliance Posture

| Framework | Target Status | Current Gap |
|-----------|-------------|------------|
| SOC 2 Type II | Target (v2.0) | Authentication gaps (fixing in this architecture) |
| GDPR | Applicable | Data minimization ✓; DPA needed; Right to erasure requires audit log carve-out |
| HIPAA | Not applicable (no PHI) | — |
| PCI-DSS | Not applicable (no cardholder data) | — |
| ISO 27001 | Long-term target (v3.0) | Large gap; formal ISMS needed |
| NIST CSF | Framework guidance | Used as design reference |

## 19.2 SOC 2 Trust Service Criteria Mapping

**CC6 (Logical and Physical Access):**
- CC6.1: API key authentication on all endpoints ✓
- CC6.2: Role-based access control, 4-role hierarchy ✓
- CC6.3: Key revocation and expiry ✓
- CC6.6: Anomaly detection on auth failures ✓
- CC6.7: Transmission encryption (TLS 1.2+) ✓
- CC6.8: Plugin sandboxing ✓

**CC7 (System Operations):**
- CC7.1: Infrastructure monitoring, SLA tracking ✓
- CC7.2: Anomaly detection → `security.anomaly.detected` ✓
- CC7.3: Incident response plan ✓ (Section 25)
- CC7.4: Incident identification and notification ✓
- CC7.5: Disaster recovery plan ✓ (Section 26)

**CC8 (Change Management):**
- CC8.1: Migration ledger (46 migrations), audit of config changes ✓

**CC9 (Risk Management):**
- CC9.1: Threat model documented ✓ (Section 20)
- CC9.2: Vendor risk (providers) ✓ (Section 15)

## 19.3 GDPR Compliance Controls

**Applicable to the platform if it processes EU personal data (e.g., if a product uploads EU user documents).**

**Data minimization (Article 5(1)(c)):** The platform stores only what is necessary — no raw IP addresses, no email addresses in audit logs, no raw API key values. Only hashes. ✓

**Right to erasure (Article 17):** The audit log is append-only and cannot be erased. This is a conflict with the right to erasure. The legal basis for retaining audit log records (legitimate interest, legal obligation) must be documented in the Data Processing Agreement. Audit records are pseudonymized (identity_hash only) — this reduces the erasure obligation. ✓ (with legal carve-out documented)

**Data Processing Agreement (DPA):** If the platform processes data on behalf of a customer (controller), a DPA must be signed. The platform itself is a processor in that scenario.

**Data Subject Access Request (DSAR):** A product operator can be identified only by identity_hash in the audit log. The process for linking identity_hash to a real identity requires the platform ADMIN to have a key registry that maps key_id → owner. This mapping should be maintained outside the platform in a separate identity management system.

## 19.4 Data Residency

v1.0: Single-region deployment. Data residency is determined by the deployment region. EU customers must deploy the platform in an EU region.

v2.5 (multi-tenant): Tenant-level data residency configuration. Each tenant declares a preferred region. Data does not leave the declared region (enforced by cloud infrastructure, not by platform code).

## 19.5 Compliance Evidence Generation

For SOC 2 audits, the following evidence is available from the platform:

| Evidence | Source |
|---------|--------|
| User access list with roles | `GET /api/v1/admin/keys` (ADMIN) |
| Authentication log | `security.audit_log` (event_category=AUTHENTICATION) |
| Permission change log | `security.audit_log` (event_type=security.api_key.*) |
| Failed authentication attempts | `security.audit_log` (event_type=security.auth.failed) |
| Rate limit events | `security.audit_log` (event_type=security.rate_limit.exceeded) |
| Admin action log | `security.audit_log` (event_category=ADMIN) |
| Incident log | Incident response records (external) |
| Change management log | `security.audit_log` (event_type=admin.migration.applied) |

---

---

# Section 20: Threat Model

## 20.1 Threat Modeling Methodology

The platform threat model uses STRIDE (Spoofing, Tampering, Repudiation, Information Disclosure, Denial of Service, Elevation of Privilege) applied to the system's data flow diagrams.

**Scope:** API boundary, event bus, database, desktop client, plugins, and AI provider integrations.

**Out of scope:** Infrastructure-level threats (cloud provider compromise, physical datacenter attacks), supply chain attacks on Python/OS packages (mitigated by dependency pinning and scanning, not by the platform).

## 20.2 Threat Catalog

### T-AUTH: Authentication Threats

**T-AUTH-01: API Key Brute Force**  
Attacker enumerates API key values by making authentication requests.  
- Likelihood: LOW (256-bit key space is infeasible to enumerate)
- Impact: HIGH (if a key is guessed, full access under that key's role)
- Control: IP-based rate limiting on auth failures; 10-failure lockout; key format designed for scanner detection

**T-AUTH-02: API Key Theft via Log Exfiltration**  
An attacker reads application logs and finds a raw API key logged by mistake.  
- Likelihood: MEDIUM (developers frequently log request objects without sanitization)
- Impact: HIGH
- Control: Log sanitization rules; log scanning alerts for `sk_live_` pattern; structured logging with field-level sanitization

**T-AUTH-03: Replay Attack**  
An attacker captures a valid API key in transit and replays it.  
- Likelihood: LOW (TLS in staging/production prevents capture in transit)
- Impact: HIGH (replayed key has full access until revoked)
- Control: TLS mandatory in staging/production; short-lived keys with `expires_at`; IP allowlisting for sensitive keys

**T-AUTH-04: Key Persistence in Desktop**  
A desktop application stores the API key in plaintext on disk, which another process reads.  
- Likelihood: MEDIUM (developers often take shortcuts with credential storage)
- Impact: HIGH
- Control: OS credential store (keychain/credential manager) mandatory; no plaintext config file fallback

### T-AUTHZ: Authorization Threats

**T-AUTHZ-01: Horizontal Privilege Escalation (Product Crossing)**  
A PRODUCT-scoped key for product-A accesses product-B's resources by manipulating resource IDs.  
- Likelihood: MEDIUM (IDOR — Insecure Direct Object Reference is a common API bug)
- Impact: HIGH (data leakage between products)
- Control: Scope check in permission evaluation (Step 2 of algorithm); all queries appended with `AND product_id = $authenticated_product_id`

**T-AUTHZ-02: Vertical Privilege Escalation**  
A DEVELOPER-scoped key performs ADMIN or REVIEWER operations.  
- Likelihood: LOW (role check in FastAPI dependency)
- Impact: HIGH
- Control: Two-layer authorization check (FastAPI dependency + service layer); RBAC hierarchy strictly enforced

**T-AUTHZ-03: Auto-Approve Gate Bypass**  
A caller sets `auto_approve_after_seconds` on a gate to bypass human review.  
- Likelihood: HIGH (this is an obvious attack path if the field is accepted)
- Impact: CRITICAL (bypasses human oversight on critical workflow gates)
- Control: `auto_approve_after_seconds` on any gate input → HTTP 400 AUTO_APPROVE_FORBIDDEN (see API-CATALOG WF-005)

### T-INJ: Injection Threats

**T-INJ-01: Prompt Injection**  
Malicious content in a document or tool result hijacks the AI model's behavior.  
- Likelihood: HIGH (prompt injection is a common and active attack)
- Impact: MEDIUM-HIGH (model may perform unauthorized actions, leak context)
- Control: HMAC-protected system prompts; structured input delimiters; injection signature detection; tool call allowlist

**T-INJ-02: Template Content Injection**  
An attacker with database write access modifies a prompt template to inject malicious instructions.  
- Likelihood: LOW (requires database access)
- Impact: CRITICAL (every workflow using the template is affected)
- Control: HMAC signature verification on every render; HMAC key not in database

**T-INJ-03: SQL Injection via API Parameters**  
An attacker sends SQL syntax in API parameters to manipulate database queries.  
- Likelihood: LOW (parameterized queries via SQLAlchemy ORM)
- Impact: HIGH
- Control: All queries use parameterized SQL; no string concatenation in queries

### T-DOS: Denial of Service Threats

**T-DOS-01: API Rate Exhaustion**  
An attacker with a valid key makes requests at the maximum rate to degrade service for other callers.  
- Likelihood: MEDIUM
- Impact: MEDIUM (affects other callers' experience; platform keeps running)
- Control: Per-identity rate limiting; global rate limiting; 429 responses with Retry-After

**T-DOS-02: Budget Exhaustion**  
An attacker with a valid key submits workflows that consume all of the daily AI budget.  
- Likelihood: MEDIUM
- Impact: HIGH (no AI capability for legitimate workflows until budget resets)
- Control: Per-product budget limits; daily hard cap; circuit breaker on unusual spend patterns

**T-DOS-03: Plugin Resource Exhaustion**  
A malicious plugin consumes all CPU or memory.  
- Likelihood: MEDIUM
- Impact: HIGH (platform instability)
- Control: Plugin sandbox CPU timeout (30s); subprocess isolation; memory limit via OS ulimit

**T-DOS-04: Event Bus Flooding**  
A compromised internal component floods the NATS event bus with high-volume messages.  
- Likelihood: LOW (internal components only, not accessible externally)
- Impact: MEDIUM
- Control: NATS JetStream per-stream subject quotas; monitoring on event volume anomalies

### T-EXP: Exfiltration Threats

**T-EXP-01: Audit Log Exfiltration**  
An attacker with ADMIN credentials exports the entire audit log.  
- Likelihood: LOW (requires ADMIN key compromise)
- Impact: HIGH (reveals all security-relevant actions, identity hashes)
- Control: Audit log access is itself audited; anomaly detection on bulk audit log reads

**T-EXP-02: Plugin Data Exfiltration**  
A malicious plugin reads data via platform API calls and sends it to an external endpoint.  
- Likelihood: MEDIUM (plugin with `network:http` permission could do this if given broad access)
- Impact: HIGH
- Control: Network allowlist in plugin manifest; platform API calls by plugins checked against plugin's declared permissions; outbound call logging

**T-EXP-03: AI Response Exfiltration**  
A prompt-injected model response instructs the caller to send data to an external URL.  
- Likelihood: MEDIUM (prompt injection is common)
- Impact: MEDIUM (the platform itself cannot prevent the human from following the instruction)
- Control: Prompt injection detection; link validation in desktop client; awareness guidance for operators

### T-SUP: Supply Chain Threats

**T-SUP-01: Malicious Plugin Package**  
A legitimate-looking plugin is published with malicious code.  
- Likelihood: MEDIUM (open plugin ecosystems have had this attack)
- Impact: HIGH
- Control: Plugin signature verification; restricted plugin permission set; sandboxing

**T-SUP-02: Compromised AI Provider**  
An AI provider is compromised and returns malicious tool call instructions.  
- Likelihood: LOW (major providers have strong security programs)
- Impact: HIGH
- Control: Tool call allowlist; provider response validation; circuit breaker

### T-PRIV: Privilege Escalation Threats

**T-PRIV-01: Self-Approval of Prompt Templates**  
A DEVELOPER writes a malicious template and promotes it to ACTIVE without REVIEWER approval.  
- Likelihood: LOW (governance FSM prevents self-approval)
- Impact: HIGH (affects all workflows using the template)
- Control: REVIEWER minimum role for REVIEW→ACTIVE transition; these are different permissions (`prompt:write` vs `prompt:govern` at REVIEWER level)

**T-PRIV-02: ADMIN Key Compromise**  
An ADMIN API key is leaked, allowing full platform control.  
- Likelihood: LOW (admin keys are most carefully protected)
- Impact: CRITICAL
- Control: Admin keys with IP allowlisting and short `expires_at`; anomaly detection on admin key usage; immediate alert on any admin:write action; separate ADMIN key for operations vs configuration

## 20.3 Threat Risk Matrix

| ID | Threat | Likelihood | Impact | Risk | Status |
|----|--------|-----------|--------|------|--------|
| T-AUTH-01 | Key brute force | LOW | HIGH | LOW | Controlled |
| T-AUTH-02 | Key in logs | MEDIUM | HIGH | MEDIUM | Controlled |
| T-AUTH-03 | Replay attack | LOW | HIGH | LOW | Controlled |
| T-AUTH-04 | Desktop key persistence | MEDIUM | HIGH | MEDIUM | Controlled |
| T-AUTHZ-01 | Product crossing (IDOR) | MEDIUM | HIGH | MEDIUM | Controlled |
| T-AUTHZ-02 | Vertical escalation | LOW | HIGH | LOW | Controlled |
| T-AUTHZ-03 | Auto-approve bypass | HIGH | CRITICAL | HIGH | Controlled |
| T-INJ-01 | Prompt injection | HIGH | MEDIUM | HIGH | **Residual Risk** |
| T-INJ-02 | Template tampering | LOW | CRITICAL | MEDIUM | Controlled |
| T-INJ-03 | SQL injection | LOW | HIGH | LOW | Controlled |
| T-DOS-01 | API rate exhaustion | MEDIUM | MEDIUM | MEDIUM | Controlled |
| T-DOS-02 | Budget exhaustion | MEDIUM | HIGH | MEDIUM | Controlled |
| T-DOS-03 | Plugin resource exhaust | MEDIUM | HIGH | MEDIUM | Controlled |
| T-EXP-01 | Audit log exfiltration | LOW | HIGH | LOW | Controlled |
| T-EXP-02 | Plugin data exfiltration | MEDIUM | HIGH | MEDIUM | Controlled |
| T-SUP-01 | Malicious plugin | MEDIUM | HIGH | MEDIUM | Controlled |
| T-PRIV-01 | Template self-approval | LOW | HIGH | LOW | Controlled |
| T-PRIV-02 | ADMIN key compromise | LOW | CRITICAL | MEDIUM | Controlled |

**Residual Risk — T-INJ-01 (Prompt Injection):** This is an unsolved problem in the AI industry. The platform's controls reduce but do not eliminate the risk. This risk is accepted as inherent to the AI model capability. Operators are advised to treat AI model outputs as untrusted and never act on them without human review for high-consequence actions.

---

---

# Section 21: Attack Surface

## 21.1 External Attack Surface

The external attack surface is any entry point reachable from outside the deployment.

| Entry Point | Type | Authentication | Rate Limited | Notes |
|-------------|------|---------------|-------------|-------|
| `GET /health` | HTTP | None (public) | Yes (IP-based) | Returns 200 OK or 503 |
| `GET /health/live` | HTTP | None (public) | Yes | Kubernetes liveness |
| `GET /health/ready` | HTTP | None (public) | Yes | Kubernetes readiness |
| `GET /metrics` | HTTP | None (public) | Yes | Prometheus scrape endpoint |
| `GET /api/v1/workspace/capabilities` | HTTP | None (public) | Yes | Capability discovery |
| All other `/api/v1/*` | HTTP | Required (API key) | Yes | See API-CATALOG |
| `GET /api/v1/desktop/ws` | WebSocket | Required (API key) | Yes (5 concurrent) | Desktop only |

**Total external surface:** 5 public endpoints + authenticated API + 1 WebSocket.

## 21.2 Internal Attack Surface (v1.0)

In v1.0 (single process), there is no internal network surface — modules communicate in-process. The only internal surface is:

| Component | Surface | Controls |
|-----------|---------|---------|
| PostgreSQL | Port 5432 (localhost binding recommended) | Platform DB user has limited privileges; RLS on audit_log |
| NATS | Port 4222 (localhost or VPC) | TLS + credentials required in staging/production |
| Platform process | OS process boundary | OS-level isolation |

## 21.3 Trust Boundaries

```
[Internet / External Callers]
         │
         │ TLS (staging/production)
         ▼
[API Gateway / Load Balancer]  ◄── trust boundary 1
         │
         ▼
[Platform API (FastAPI)]
         │
         ├──[in-process]──► [Security Middleware]  ◄── trust boundary 2
         │                       │
         │                  [AuthContext]
         │                       │
         ├──[in-process]──► [Route Handlers]
         │                       │
         ├──[in-process]──► [Service Layer]
         │                       │
         ├──[in-process]──► [AI ROS]  ◄── trust boundary 3
         │                       │
         │                  [Provider Adapters]
         │                       │
         │               [HTTP to External AI APIs]  ◄── trust boundary 4
         │
         ├──[TCP/TLS]──► [PostgreSQL]  ◄── trust boundary 5
         │
         ├──[TCP/TLS]──► [NATS JetStream]  ◄── trust boundary 6
         │
         └──[subprocess]──► [Plugin Sandbox]  ◄── trust boundary 7
```

**Trust boundary 1** (Internet → API Gateway): TLS termination; DDoS protection (cloud provider).  
**Trust boundary 2** (API request → Auth): API key authentication; rate limiting.  
**Trust boundary 3** (Platform → AI ROS): Budget enforcement; queue priority; circuit breaker.  
**Trust boundary 4** (AI ROS → Providers): API key scoped to one provider; timeout enforcement; tool call allowlist.  
**Trust boundary 5** (Platform → PostgreSQL): DB credentials; TLS; schema-level access control.  
**Trust boundary 6** (Platform → NATS): TLS + credentials; subject-level authorization.  
**Trust boundary 7** (Platform → Plugin): Subprocess isolation; declared permission sandbox; signature verification.

## 21.4 Attack Surface Reduction Roadmap

| Version | Reduction |
|---------|-----------|
| v1.0 | Remove `AISF_AUTH_ENABLED=false` toggle (done); close debug endpoints in production |
| v1.1 | WASM plugin sandbox (reduces T-DOS-03, T-EXP-02); dual-key HMAC rotation |
| v2.0 | Vault secrets (reduces T-AUTH-02, T-PRIV-02); Redis rate limit state (reduces T-DOS-01 in multi-instance) |
| v2.5 | mTLS internal (closes trust boundary 5 and 6); distributed rate limiting |
| v3.0 | SSO/OAuth (human identities separated from machine identities); ABAC |

---

---

---

# Section 22: Anomaly Detection

## 22.1 Anomaly Detection Model

The platform uses a lightweight statistical anomaly detection model. It maintains a baseline of expected behavior per identity and alerts when behavior deviates significantly.

**Baseline metrics tracked per identity (rolling 1-hour window):**

| Metric | Description |
|--------|-------------|
| `auth_failure_rate` | Authentication failures per minute |
| `permission_denial_rate` | Permission denied responses per minute |
| `request_rate` | Total requests per minute |
| `unique_resources_accessed` | Count of distinct resource IDs accessed |
| `product_id_violations` | Scope check failures per minute |
| `ai_cost_per_minute` | USD spent on AI calls per minute |
| `error_rate` | 4xx/5xx response rate |

**Baseline window:** The baseline is computed from the rolling 24-hour average for each identity, excluding the most recent 1-hour window. The 1-hour window is the detection window.

## 22.2 Anomaly Rules

| Rule | Trigger | Event Published | Severity |
|------|---------|----------------|---------|
| AN-1: Auth failure spike | > 10 auth failures in 60s from one IP | `security.anomaly.detected` | HIGH |
| AN-2: Permission denial spike | > 5 denials in 60s from one identity | `security.anomaly.detected` | MEDIUM |
| AN-3: Rate × 10 | Request rate > 10x the 24h baseline | `security.anomaly.detected` | MEDIUM |
| AN-4: Cost spike | AI spend > 5x the hourly baseline | `security.anomaly.detected` | HIGH |
| AN-5: Dormant key activity | Key unused > 24h makes a request | `security.auth.dormant_key` | LOW |
| AN-6: Product scope violation | Any product_id scope check failure | `security.anomaly.detected` | HIGH |
| AN-7: ADMIN key unusual time | ADMIN key used outside 08:00–20:00 local | `security.anomaly.detected` | MEDIUM |
| AN-8: Bulk audit log read | Audit log export > 10,000 records | `security.anomaly.detected` | HIGH |
| AN-9: Plugin sandbox violation | Any sandbox violation | `security.plugin.sandbox_violated` | CRITICAL |
| AN-10: HMAC mismatch | Any prompt HMAC verification failure | `security.anomaly.detected` | CRITICAL |

**Note:** Anomaly rules trigger events and alerts, not automatic blocking (except AN-1 which triggers the IP auth lockout after 10 failures). Human review is required for all MEDIUM and HIGH anomalies. CRITICAL anomalies trigger automated platform alerts and should be treated as active security incidents.

## 22.3 Anomaly Response Flow

```
Anomaly rule triggers
        │
        ▼
security.anomaly.detected event published → SECURITY NATS stream
        │
        ├──▶ Audit log entry (CRITICAL or HIGH severity)
        │
        ├──▶ Monitoring alert (PagerDuty/Slack/email)
        │
        └──▶ (for CRITICAL): automatic key suspension pending review
                    │
                    ▼
              Human review within 4 hours
                    │
                    ├──▶ False positive: unsuspend key, document reason
                    │
                    └──▶ Confirmed: initiate Incident Response (Section 25)
```

---

---

# Section 23: Incident Response

## 23.1 Incident Classification

| Severity | Definition | Response Time | Examples |
|----------|-----------|--------------|---------|
| P1 — Critical | Active data breach or ongoing attack | 15 minutes | ADMIN key compromised, mass data exfiltration, budget exhaustion under attack |
| P2 — High | Confirmed security violation, contained | 1 hour | Plugin sandbox escape, prompt injection producing data leakage, key revoked post-theft |
| P3 — Medium | Suspected security issue, unconfirmed | 4 hours | Anomaly detected but unclear, dormant key unusual activity, permission denial spike |
| P4 — Low | Security hygiene issue, no active threat | 24 hours | Expired certificate approaching, key rotation overdue, unused ADMIN key |

## 23.2 Incident Response Team

| Role | Responsibility |
|------|---------------|
| **Incident Commander** | Coordinates the response; makes escalation decisions; single point of authority |
| **Security Lead** | Technical investigation; determines scope; implements containment |
| **Platform Engineer** | Platform-level response actions (key revocation, service isolation); coordinates with Security Lead |
| **Communications Lead** | Internal communication; customer notification if needed |
| **Legal / Compliance** | Regulatory notification (GDPR 72-hour requirement if applicable) |

## 23.3 Incident Response Phases

### Phase 1: Detection and Triage (0–15 minutes for P1; 0–1 hour for P2)

1. **Alert received** — from anomaly detection, external report, or monitoring
2. **Initial triage** — Security Lead reviews the alert and classifies severity
3. **Incident Commander declared** — for P1 and P2
4. **War room activated** — dedicated communication channel opened
5. **Initial evidence preserved** — audit log export started immediately (evidence may be needed for forensics; do not allow rollover)

**P1 immediate actions (within 15 minutes):**
- Revoke all potentially compromised keys
- If source is identified: block IP at network level
- If budget is being exhausted: hard-disable AI provider integration
- Notify CTO

### Phase 2: Containment (15 minutes – 2 hours)

**Containment objectives:**
1. Stop the active threat from progressing
2. Preserve evidence for forensics
3. Prevent lateral movement

**Containment actions by threat type:**

| Threat | Containment Action |
|--------|------------------|
| Compromised API key | Revoke key; rotate ADMIN keys if any admin actions occurred; review all actions taken by the key |
| Prompt injection producing data leakage | Suspend the affected template; disable the affected workflow type; review task outputs |
| Plugin sandbox violation | Auto-disabled by platform; review all actions taken by the plugin in its execution window |
| Budget exhaustion attack | Temporarily reduce daily budget to $0 until attack stops; re-enable specific products as confirmed safe |
| Database breach | Isolate the database instance from network; contact cloud provider security team |

### Phase 3: Investigation (2–24 hours)

1. **Full audit log review** — Export and analyze all audit log entries from the incident window
2. **Correlation** — Correlate identity_hashes with known API keys to determine which identities were involved
3. **Scope determination** — Determine what data was accessed, modified, or exfiltrated
4. **Root cause analysis** — Identify the vulnerability or failure that allowed the incident
5. **Evidence packaging** — Document all findings for legal/compliance use

**Key forensic queries:**
```
Audit log: all actions by identity_hash in window
Audit log: all ADMIN actions in prior 48 hours
Audit log: all API key creation events in prior 7 days
Event bus: all SECURITY stream events in window
Event bus: all WORKFLOWS events for affected workflows
```

### Phase 4: Eradication (24–72 hours)

1. Remove the root cause (patch the vulnerability, remove the malicious plugin, fix the misconfiguration)
2. Verify the fix in a test environment
3. Rotate all credentials that could have been compromised (conservative approach)
4. Update threat model with the new threat vector

### Phase 5: Recovery (24–72 hours)

1. Restore affected capabilities with enhanced monitoring
2. Verify the restored system with security tests
3. Re-enable any features or keys suspended during containment (with review)
4. Brief all customers affected (if applicable)

### Phase 6: Post-Incident Review (within 1 week)

1. **Timeline reconstruction** — Write the complete incident timeline
2. **Root cause report** — Technical and process root causes
3. **Control gap analysis** — Which security controls failed or were absent?
4. **Remediation plan** — Specific changes to close the gaps
5. **Lessons learned** — What should change in the threat model, controls, or processes?

**Required outputs:**
- Incident Report (internal, available to auditors)
- Customer notification (if applicable, within 72 hours for GDPR)
- Security Review Checklist update (add the new control that would have prevented this)
- Threat model update

## 23.4 GDPR Breach Notification

If the incident involves EU personal data:
- **72-hour rule:** Notify the relevant Supervisory Authority within 72 hours of becoming aware of the breach (Article 33)
- **Affected individuals:** Notify affected individuals without undue delay if the breach is likely to result in high risk (Article 34)
- **Documentation:** Even if notification is not required, document the breach and the reasoning for not notifying (Article 33(5))

## 23.5 Communication Templates

**P1 — Internal Alert (within 15 minutes):**
```
[SECURITY INCIDENT P1] Active security incident declared.
Incident ID: INC-YYYY-MMDD-NNN
Type: [type]
Detected: [timestamp]
Affected: [component/product]
Commander: [name]
War room: [channel]
Immediate action required: [action]
```

**Customer Notification (within 72 hours, if applicable):**
```
Subject: AI Studio Platform Security Incident Notice

On [date], we detected [brief description]. We took immediate action
to [containment action]. The incident was contained as of [date/time].

Impact: [what data/functionality was affected]
Your data: [was/was not potentially affected; specific details if known]

Actions you should take: [specific steps, e.g., rotate your API key]

We have made the following changes to prevent recurrence: [changes]

For questions: [security contact email]
```

---

---

# Section 24: Disaster Recovery

## 24.1 Disaster Scenarios

| Scenario | Description | Target RTO | Target RPO |
|----------|-------------|-----------|-----------|
| Database server failure | PostgreSQL instance down | 30 minutes | 5 minutes |
| Database data corruption | Data corrupted by bug or attack | 4 hours | 1 hour |
| Complete platform failure | All platform services down | 1 hour | 5 minutes |
| Security incident recovery | Platform taken offline post-breach | 8 hours | Point of breach |
| NATS cluster failure | Event bus unavailable | 15 minutes | 30 seconds |
| Provider outage | AI provider API unavailable | N/A (handled by circuit breaker) | N/A |

## 24.2 Recovery Time Objective (RTO)

RTO is the maximum acceptable time for recovery after a disaster is declared.

| Component | RTO |
|-----------|-----|
| Platform API (no data loss) | 15 minutes (restart/redeploy) |
| Platform API (data recovery needed) | 1 hour |
| PostgreSQL (replica failover) | 30 minutes |
| PostgreSQL (restore from backup) | 4 hours (depends on data volume) |
| NATS (restart) | 15 minutes |
| Full platform (worst case) | 8 hours |

## 24.3 Recovery Point Objective (RPO)

RPO is the maximum acceptable data loss (expressed as how far back recovery goes).

| Data Type | RPO | Backup Mechanism |
|-----------|-----|-----------------|
| Security/audit data | 5 minutes | Streaming WAL replication |
| Workflow state | 5 minutes | Streaming WAL replication |
| Billing records | 5 minutes | Streaming WAL replication |
| Prompt templates | 1 hour | Continuous WAL + hourly snapshot |
| Knowledge graph | 1 hour | Hourly snapshot |
| Configuration | 24 hours | Daily snapshot + git (config is in code) |
| NATS messages (unprocessed) | 30 seconds (JetStream acknowledgment window) | NATS JetStream replication |

## 24.4 Backup Architecture

**PostgreSQL backup tiers:**

| Backup Type | Frequency | Retention | Storage |
|-------------|-----------|-----------|---------|
| WAL archiving (PITR) | Continuous (streaming) | 7 days | Object storage |
| Base snapshot (full) | Daily at 02:00 UTC | 30 days | Object storage |
| Weekly snapshot | Sunday 03:00 UTC | 90 days | Cold storage |
| Monthly snapshot | 1st of month | 1 year | Cold storage |
| Quarterly snapshot | Jan/Apr/Jul/Oct | 7 years | Archival storage |
| Pre-migration snapshot | Before each migration | Indefinite | Tagged object storage |

**Backup integrity:** Every backup is verified by a restore test to a separate environment. Schedule: daily snapshot → weekly restore test.

**Backup encryption:** All backups are AES-256 encrypted. The backup encryption key is stored separately from the data (key escrow in a separate cloud account).

## 24.5 Disaster Recovery Procedure

### Step 1: Declaration (0–5 minutes)

**Criteria for declaring DR:**
- Platform API health endpoint returning 503 for > 5 minutes
- Database connection failing for all platform instances
- NATS cluster down for > 5 minutes
- Security incident requiring platform shutdown

**Declaration:** The Incident Commander (or on-call engineer if IC unavailable) declares DR. All stakeholders are notified via the P1 incident communication channel.

### Step 2: Triage and Recovery Path Selection (5–15 minutes)

Determine the appropriate recovery path:
- **Path A (restart):** Platform process died; database and NATS are healthy → restart process, done
- **Path B (failover):** Database replica available → promote replica, update connection string, restart platform
- **Path C (restore):** No replica; restore from backup → select restore point (PITR for minimum data loss)
- **Path D (full rebuild):** All infrastructure lost → provision new infrastructure, restore database, reconfigure

### Step 3: Recovery Execution

**Path A — Process restart:**
1. Identify why the process died (check logs)
2. Ensure database and NATS are healthy
3. Restart platform (rolling restart for zero-downtime in multi-instance)
4. Verify health endpoint returns 200
5. Verify audit log integrity (most recent entry matches expected)

**Path B — Database failover:**
1. Confirm primary is unreachable and replica is healthy
2. Promote replica to primary (PostgreSQL `pg_promote()`)
3. Update `AISF_DB_URL` to point to the new primary
4. Restart platform with updated config
5. Verify health and run smoke tests
6. Re-establish replication to a new replica

**Path C — Restore from backup:**
1. Identify the target restore point (PITR timestamp)
2. Provision a new database instance
3. Restore base snapshot
4. Apply WAL logs up to the target timestamp
5. Verify database integrity (row counts, audit log continuity)
6. Update platform config, restart
7. If data loss occurred: notify affected parties of the data loss window

**Path D — Full rebuild:**
1. Provision infrastructure from infrastructure-as-code
2. Restore database (Path C procedure)
3. Restore NATS configuration
4. Restore platform configuration (from git)
5. Verify all components healthy
6. Gradual traffic restoration (start with 10% of traffic, verify, then full)

### Step 4: Validation (post-recovery)

Before declaring recovery complete:
- [ ] Health endpoint returns 200 (all components healthy)
- [ ] Audit log entries from pre-disaster are present
- [ ] Authentication succeeds for a test key
- [ ] A test workflow can be submitted and completes
- [ ] All providers return a healthy circuit breaker status
- [ ] NATS consumer lag is zero (caught up on any backlog)
- [ ] Monitoring alerts are green

### Step 5: Post-Recovery Review

- Document the timeline (what happened, when, what was done)
- Calculate actual RTO and RPO achieved
- Identify gaps vs. targets
- Update DR procedure based on lessons learned

## 24.6 NATS Recovery

NATS JetStream is configured with replication factor 3 in production. A single NATS node failure is handled automatically by NATS cluster consensus — no manual intervention.

**If all NATS nodes fail:**
1. Platform API continues serving (events are enqueued in memory with a 1,000-message buffer per stream)
2. If NATS is not restored within 5 minutes: events beyond the buffer are dropped (logged, not silently discarded)
3. Restore NATS from JetStream snapshot
4. Platform reconnects automatically (NATS client reconnect policy: exponential backoff, max 5 minutes)

## 24.7 Security-Specific Recovery

Recovery from a security incident differs from a technical disaster:

1. **Root cause must be identified before restore** — do not restore a compromised system to the same state
2. **Audit trail must be preserved** — export the audit log before any recovery operation that touches the database
3. **All credentials must be rotated** — after recovery from a breach, rotate every API key and provider key
4. **Evidence must be preserved** — take a forensic snapshot of the compromised state before recovery

The security incident recovery is coordinated by the Security Lead and may take longer than the technical RTO (8 hours target) due to the forensic requirements.

---

---

# Section 25: Security Review Checklist

## 25.1 Pre-Development Security Review

Completed before any new feature or module is implemented.

**Architecture review:**
- [ ] Does this feature introduce a new authentication pattern? (If yes: review Section 3)
- [ ] Does this feature introduce a new authorization check? (If yes: add permission to catalog in Section 4.3)
- [ ] Does this feature store new sensitive data? (If yes: classify in Section 6.1)
- [ ] Does this feature introduce a new external dependency? (If yes: assess trust boundary in Section 21.3)
- [ ] Does this feature add a new audit event type? (If yes: add to Section 18.3 and DATABASE-CATALOG Section 9)
- [ ] Is this feature exposed via a new API endpoint? (If yes: add to API-CATALOG; assign rate limit in Section 14.2)
- [ ] Does this feature publish new events? (If yes: add to EVENT-CATALOG)
- [ ] Does this feature touch the plugin system? (If yes: review Section 16)
- [ ] Does this feature affect the desktop client? (If yes: review Section 17)
- [ ] Have all applicable threats in Section 20.2 been considered?

## 25.2 Implementation Security Review

Completed during or immediately after implementation (before merge).

**Authentication and authorization:**
- [ ] Every new endpoint has the `Depends(security_client.require_permission(...))` dependency
- [ ] Service layer re-checks permission for every mutating operation
- [ ] No endpoint in PUBLIC_PATHS list without explicit architect approval
- [ ] Scope check (product_id filtering) applied to all queries that return product-specific data
- [ ] No `auto_approve_after_seconds` accepted in any gate-related request body

**Secret and credential handling:**
- [ ] No raw API key values in any log line
- [ ] No raw IP addresses in any log line or event payload
- [ ] All new secrets use the `AISF_SECRET_` environment variable prefix
- [ ] No new credentials hardcoded or committed to version control
- [ ] New provider integrations use only `AISF_SECRET_<PROVIDER>_API_KEY`

**Cryptography:**
- [ ] No prohibited algorithms used (see Section 8.4)
- [ ] All comparisons of secret values use `hmac.compare_digest()` (constant-time)
- [ ] No new hash functions introduced without security review
- [ ] HMAC verification applied to any new template-like construct

**Input validation:**
- [ ] All user-supplied string inputs have a maximum length validation
- [ ] All user-supplied enum values are validated against an allowlist (not a denylist)
- [ ] All database queries use parameterized SQL (no string concatenation)
- [ ] All file paths (if any) are validated against an allowlist of permitted directories

**Error handling:**
- [ ] Error messages do not reveal system internals (stack traces, internal IDs, SQL errors)
- [ ] Error responses use the standard error envelope format
- [ ] Authentication failures return generic "Authentication failed" — no detail about why

**Rate limiting:**
- [ ] New endpoint added to rate limit table in Section 14.2
- [ ] New endpoint's rate limit reviewed for DoS risk

**Audit logging:**
- [ ] New security-relevant actions generate an audit log entry
- [ ] All new audit event types added to Section 18.3
- [ ] Audit entries include: occurred_at, event_type, identity_hash, resource_type, resource_id, outcome

**Plugin and provider integration:**
- [ ] New provider integrations go through the AI ROS budget and timeout enforcer
- [ ] New provider adapters validate the response schema before passing to the platform
- [ ] New plugin permissions reviewed against declared permission catalog

## 25.3 Pre-Deployment Security Checklist (Staging)

Before deploying to staging:

**Configuration review:**
- [ ] `AISF_ENVIRONMENT=staging` set
- [ ] `AISF_AUTH_ENABLED` variable does not exist (there is no such variable — confirm)
- [ ] All `AISF_SECRET_*` variables set from vault or secrets manager (not from `.env` file)
- [ ] TLS enabled (`AISF_TLS_REQUIRED=true`)
- [ ] Plugin signature verification active (`AISF_PLUGIN_SKIP_SIGNATURE_VERIFY` not set)
- [ ] `AISF_DESKTOP_BIND_HOST=127.0.0.1` (not 0.0.0.0)
- [ ] CORS origins list restricted (not `*`)
- [ ] Debug endpoints not exposed

**Secret scanning:**
- [ ] No `sk_live_` patterns in any committed file
- [ ] No `AISF_SECRET_` values in any committed file
- [ ] No database connection strings in any committed file
- [ ] Secret scanning CI check passes

**Dependency review:**
- [ ] All Python dependencies pinned to exact versions
- [ ] No dependency with known HIGH or CRITICAL CVEs (scan via `pip-audit` or `safety`)
- [ ] No new transitive dependencies without review

**Certificate check:**
- [ ] TLS certificate valid and not expiring within 30 days
- [ ] Certificate from trusted CA (not self-signed)
- [ ] HSTS header configured in production

## 25.4 Pre-Deployment Security Checklist (Production)

Before deploying to production (in addition to staging checklist):

**Architecture:**
- [ ] Architecture review completed (signed off by Platform Architect)
- [ ] Security model updated for any new features (this document)
- [ ] Threat model updated with any new threat vectors
- [ ] API-CATALOG updated for any new endpoints
- [ ] EVENT-CATALOG updated for any new events

**Key management:**
- [ ] All ADMIN keys have `expires_at` set within 90 days
- [ ] All ADMIN keys have IP allowlisting set
- [ ] No ADMIN keys with `scope=GLOBAL` that are not actively needed
- [ ] Provider API keys rotated within the past 90 days

**Backup and recovery:**
- [ ] Database backup verified (most recent restore test passed)
- [ ] DR procedure is current and documented
- [ ] Monitoring alerts for backup failure are configured

**Compliance:**
- [ ] Any new data types classified per Section 6.1
- [ ] GDPR applicability assessed for any new EU-personal-data handling
- [ ] SOC 2 control matrix updated (quarterly)

**Rollback plan:**
- [ ] Pre-migration database snapshot taken
- [ ] Rollback procedure documented (which migration to revert, how)
- [ ] Rollback tested in staging for any schema-changing migrations

## 25.5 Periodic Security Review Schedule

| Review | Frequency | Owner | Output |
|--------|-----------|-------|--------|
| API key audit | Monthly | Platform Engineer | Revoke unused keys; rotate ADMIN keys |
| Dependency vulnerability scan | Weekly (automated) | CI pipeline | CVE report; update dependencies |
| Audit log anomaly review | Weekly | Security Lead | Investigate any unexplained anomalies |
| TLS certificate expiry review | Monthly | Platform Engineer | Renew approaching-expiry certificates |
| Plugin permission review | Quarterly | Security Lead | Review installed plugins' declared permissions |
| Threat model review | Quarterly | Security Lead + CTO | Update threats for new capabilities |
| Full security document review | Quarterly | Chief Security Architect | Update this document |
| SOC 2 evidence collection | Quarterly | Compliance Officer | Package evidence for SOC 2 audit |
| DR drill | Bi-annually | Platform Engineer + Security Lead | Validate RTO/RPO targets |
| Penetration test | Annually | External vendor | Independent security assessment |

## 25.6 Security Anti-Patterns Checklist

These patterns have been observed in the legacy codebase. Never introduce them.

**Forbidden:**
- [ ] `AISF_AUTH_ENABLED=false` or any similar authentication disable toggle
- [ ] `auto_approve_after_seconds` accepted in any API request body
- [ ] Raw API key values logged (even at DEBUG level)
- [ ] Raw IP addresses logged (even at DEBUG level)
- [ ] Cross-schema foreign keys in the database
- [ ] A module accessing another module's schema directly (bypass the API)
- [ ] Prompt HMAC verification skipped "for performance"
- [ ] Plugin sandboxing disabled "for local testing" in staging or production
- [ ] `*` (wildcard) in CORS allowed origins in staging or production
- [ ] `0.0.0.0` binding for any debug or internal port in staging or production
- [ ] Hardcoded credentials of any kind in source code
- [ ] `except Exception: pass` swallowing security-relevant errors silently
- [ ] Gates without a human approval requirement ("trivial gates")
- [ ] Token-level response budget set to unlimited
- [ ] Database connection with superuser/admin credentials (use least-privilege DB user)
- [ ] Audit log writes wrapped in try/except that silently discards failures (audit write failure should propagate)
- [ ] A provider response trusted and executed as code
- [ ] Admin API endpoints with no rate limiting
- [ ] Password/secret comparison with `==` (use `hmac.compare_digest`)

---

---

# Appendix A: Security Scores and Improvement Targets

## A.1 Architecture Review Score (Security)

The platform's initial architecture review scored security at **2/15** — the lowest component score.

| Control Area | Before Recovery | After Recovery (Architecture) | After Recovery (Implemented) |
|-------------|----------------|-------------------------------|------------------------------|
| Authentication | 0/3 (disabled by default) | 3/3 (mandatory, designed) | TBD |
| Authorization (RBAC) | 1/3 (partial) | 3/3 (full 4-role hierarchy) | TBD |
| Secret Handling | 0/2 (raw keys in DB) | 2/2 (hash-only, vault roadmap) | TBD |
| Audit Trail | 1/2 (partial logging) | 2/2 (full audit + self-audit) | TBD |
| Cryptography | 0/2 (no signing) | 2/2 (HMAC + plugin signing) | TBD |
| Sandboxing | 0/2 (no sandboxing) | 2/2 (plugin + provider) | TBD |
| Threat Model | 0/1 (none existed) | 1/1 (complete) | TBD |
| **Total** | **2/15** | **15/15 (design)** | TBD |

## A.2 Security Event Quick Reference

| Event | NATS Subject | Classification | Stream |
|-------|-------------|---------------|--------|
| Auth succeeded | `security.auth.succeeded` | CONFIDENTIAL | SECURITY |
| Auth failed | `security.auth.failed` | CONFIDENTIAL | SECURITY |
| API key created | `security.api_key.created` | RESTRICTED | SECURITY |
| API key revoked | `security.api_key.revoked` | RESTRICTED | SECURITY |
| Permission denied | `security.permission.denied` | CONFIDENTIAL | SECURITY |
| Rate limit exceeded | `security.rate_limit.exceeded` | INTERNAL | SECURITY |
| Prompt injection detected | `security.prompt_injection.detected` | RESTRICTED | SECURITY |
| Plugin sandbox violated | `security.plugin.sandbox_violated` | RESTRICTED | SECURITY |
| Anomaly detected | `security.anomaly.detected` | RESTRICTED | SECURITY |

---

---

# Appendix B: Cryptographic Key Inventory

| Key | Algorithm | Length | Storage | Rotation | Scope |
|-----|-----------|--------|---------|----------|-------|
| Prompt HMAC key | HMAC-SHA256 | 256-bit | `AISF_SECRET_PROMPT_HMAC_KEY` | Maintenance window (re-sign all templates) | Platform-wide |
| Plugin verify key | Ed25519 (public) | 256-bit | `AISF_SECRET_PLUGIN_VERIFY_KEY` | With plugin publisher rotation | Platform-wide |
| Provider API keys | Per-provider | Varies | `AISF_SECRET_<PROVIDER>_API_KEY` | Every 90 days | Per-provider |
| Database password | n/a (credential) | — | `AISF_SECRET_DB_PASSWORD` | Every 90 days | Platform startup |
| NATS password | n/a (credential) | — | `AISF_SECRET_NATS_PASSWORD` | Every 90 days | Platform startup |
| TLS private key | ECDSA P-256 or RSA 4096 | 256/4096-bit | Certificate store | Auto-renew (before expiry) | API endpoint |

---

---

# Appendix C: Regulatory Reference

| Requirement | Standard | Section | Platform Control |
|-------------|---------|---------|-----------------|
| Access control | SOC 2 CC6.1 | 3, 4 | API key auth; RBAC |
| Encryption in transit | SOC 2 CC6.7 | 7.2 | TLS 1.2+ mandatory |
| Encryption at rest | SOC 2 CC6.1 | 7.1 | AES-256 storage |
| Audit logging | SOC 2 CC7.2 | 18 | Immutable audit log |
| Incident response | SOC 2 CC7.3–7.5 | 23 | IR plan |
| Availability monitoring | SOC 2 CC7.1 | Monitoring module | SLA tracking |
| Change management | SOC 2 CC8.1 | 18.3 (admin.migration.applied) | Migration ledger |
| Personal data protection | GDPR Art. 5 | 6, 7.1, 19.3 | Data minimization; encryption |
| Breach notification | GDPR Art. 33–34 | 23.4 | 72-hour notification procedure |
| Right to erasure | GDPR Art. 17 | 19.3 | Audit log carve-out documented |

---

---

# Appendix D: Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0.0 | 2026-06-28 | Chief Security Architect | Initial ratified version — complete security architecture for AI Studio Platform v1.0 |

---

---

# Appendix E: Document Index

| Document | Content | Relationship to This Document |
|----------|---------|------------------------------|
| `PLATFORM-STANDARDS.md` | Engineering principles, coding standards, error hierarchy | Defines `hmac.compare_digest` requirement; `SHA-256` storage requirement; `AISF_` prefix convention |
| `PLATFORM-CONTRACTS.md` | Interface contracts for all platform modules | SecurityClient interface (Sections 3, 4); ProviderClient interface (Section 15) |
| `API-CATALOG.md` | Complete REST API documentation | Permission×Role matrix (Appendix A.8); Rate limit tiers (Appendix A.7); AUTO_APPROVE_FORBIDDEN (WF-005) |
| `EVENT-CATALOG.md` | Complete event catalog | Security event definitions (SC-001 prompt injection; SC-002 sandbox violation); SECURITY NATS stream configuration |
| `DATABASE-CATALOG.md` | Logical data architecture | `security.api_keys` schema (Section 5.2); `security.audit_log` schema (Section 18.2); RLS policy design |
| `AI-STUDIO-PLATFORM-REFACTORING-BLUEPRINT.md` | Phase 0 refactoring blueprint | Security score 2/15 baseline; Recovery Program motivation; ADR-002 (auth) |

---

*End of SECURITY-MODEL.md*  
*Document: SECURITY-MODEL v1.0.0 | Status: RATIFIED | Classification: INTERNAL — RESTRICTED*  
*28 sections | 27 required topics covered | 4 appendices | Complete enterprise security architecture*
