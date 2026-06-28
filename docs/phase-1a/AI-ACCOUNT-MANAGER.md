# AI Account Manager — Credential and Subscription Management

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture — Contains Security-Sensitive Design

---

## 1. Purpose

The AI Account Manager owns all AI provider credentials, API keys, OAuth tokens, subscription states, and entitlement information across every organization using AI ROS. It is the exclusive source of credentials; no other subsystem may hold, cache, or log provider secrets.

---

## 2. Responsibilities

- Secure storage and retrieval of provider API keys and OAuth tokens
- OAuth 2.0 flows for providers that support token-based authentication
- Subscription state tracking (Claude Max, Claude Pro, OpenAI Plus, Gemini Advanced, enterprise plans)
- Per-organization and per-user credential isolation
- API key rotation scheduling and enforcement
- Expiry monitoring and pre-expiry alerts
- Audit logging of all credential access
- Credential health validation (test API call against provider)

---

## 3. Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      Account Manager                              │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────┐  ┌────────────────┐  │
│  │  Credential      │  │  Subscription    │  │  OAuth         │  │
│  │  Vault           │  │  Registry        │  │  Handler       │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬───────┘  │
│           │                     │                     │           │
│  ┌────────▼─────────────────────▼─────────────────────▼────────┐  │
│  │                  Account Manager Core                         │  │
│  │                                                               │  │
│  │  get_credential(org_id, provider_id) → ProviderCredential    │  │
│  │  get_subscription(org_id, provider_id) → SubscriptionState   │  │
│  │  register_api_key(org_id, provider_id, key) → void           │  │
│  │  revoke_credential(org_id, provider_id) → void               │  │
│  │  rotate_api_key(org_id, provider_id) → void                  │  │
│  │  validate_credential(org_id, provider_id) → ValidationResult │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────┐  ┌──────────────────────────────────────┐  │
│  │  Rotation        │  │  Audit Logger                         │  │
│  │  Scheduler       │  │  (append-only; no secret values)      │  │
│  └──────────────────┘  └──────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
         │ credentials (encrypted, masked in transit)
         ▼
  Provider Adapter
  (single-use per request; never stored by adapter)
```

---

## 4. Data Models

### 4.1 ProviderCredential

| Field | Type | Description |
|-------|------|-------------|
| `credential_id` | `UUID` | Primary key |
| `org_id` | `UUID` | Organization scope |
| `user_id` | `UUID \| None` | Optional user scope (personal vs org-wide) |
| `provider_id` | `str` | Provider identifier |
| `credential_type` | `CredentialType` | `API_KEY` / `OAUTH_TOKEN` / `SERVICE_ACCOUNT` |
| `encrypted_value` | `bytes` | AES-256-GCM encrypted credential |
| `key_id` | `str` | Encryption key identifier (for rotation) |
| `alias` | `str` | Human-readable label (e.g., "Production Claude Key") |
| `status` | `CredentialStatus` | `ACTIVE` / `EXPIRED` / `REVOKED` / `PENDING_ROTATION` |
| `expiry` | `datetime \| None` | When credential expires (from provider or policy) |
| `last_used_at` | `datetime \| None` | Last successful use |
| `last_validated_at` | `datetime \| None` | Last health check |
| `rotation_policy` | `RotationPolicy \| None` | Automatic rotation configuration |
| `created_at` | `datetime` | |
| `created_by` | `UUID` | User who registered this credential |

### 4.2 SubscriptionState

| Field | Type | Description |
|-------|------|-------------|
| `subscription_id` | `UUID` | Primary key |
| `org_id` | `UUID` | |
| `user_id` | `UUID \| None` | User-scoped subscription (e.g., personal Claude Max) |
| `provider_id` | `str` | |
| `plan_name` | `str` | `claude-max` / `claude-pro` / `openai-plus` / `openai-team` / `enterprise` / etc. |
| `status` | `SubscriptionStatus` | `ACTIVE` / `CANCELLED` / `PAST_DUE` / `TRIAL` |
| `entitlements` | `SubscriptionEntitlements` | What this plan includes |
| `billing_period_start` | `date` | |
| `billing_period_end` | `date` | |
| `renewal_date` | `date \| None` | |
| `linked_credential_id` | `UUID` | Which API key/OAuth token authenticates this subscription |

### 4.3 SubscriptionEntitlements

| Field | Type | Description |
|-------|------|-------------|
| `monthly_token_limit` | `int \| None` | Null = unlimited (true enterprise) |
| `requests_per_minute_limit` | `int \| None` | |
| `models_included` | `List[str]` | Canonical model IDs included at no per-token cost |
| `priority_queue` | `bool` | Whether this plan gets priority access during load |
| `extended_context` | `bool` | Whether this plan unlocks extended context windows |
| `usage_is_billable` | `bool` | False for flat-rate subscriptions (Claude Max, Claude Pro) |

### 4.4 RotationPolicy

| Field | Type | Description |
|-------|------|-------------|
| `rotation_interval_days` | `int` | Rotate key every N days |
| `notification_days_before` | `int` | Alert N days before rotation is due |
| `rotation_method` | `RotationMethod` | `MANUAL_CONFIRM` / `AUTO_GENERATE` / `PROVIDER_API` |
| `next_rotation_at` | `datetime` | |

---

## 5. Supported Provider Account Types

### 5.1 Anthropic (Claude)

| Account Type | Authentication | Notes |
|-------------|---------------|-------|
| API Key | `x-api-key` header | Standard API access |
| Claude Pro | API key (linked to Pro account) | Higher rate limits; per-token billing still applies |
| Claude Max | API key (linked to Max account) | Flat-rate subscription; selected models at null cost |
| Anthropic Console | API key from Console | Enterprise teams; org-level key management |
| Bedrock (AWS) | AWS IAM credentials (SigV4) | Claude models via Amazon Bedrock |
| Google Vertex | Google service account JSON | Claude models via Vertex AI |

### 5.2 OpenAI

| Account Type | Authentication | Notes |
|-------------|---------------|-------|
| API Key | `Authorization: Bearer` header | Standard API access |
| OpenAI Plus | API key (linked to Plus account) | GPT-4o access; per-token billing |
| OpenAI Team | Org-scoped API key | Team account with higher limits |
| Azure OpenAI | Azure API key + endpoint URL | Regional deployment |

### 5.3 Google Gemini

| Account Type | Authentication | Notes |
|-------------|---------------|-------|
| Google AI Studio Key | `x-goog-api-key` header | Developer access |
| Vertex AI | Google service account JSON or OAuth | Production enterprise |
| Gemini Advanced | Google account OAuth 2.0 | Consumer subscription |

### 5.4 Local Providers

| Provider | Authentication | Notes |
|---------|---------------|-------|
| Ollama | None (local only) | Endpoint URL only |
| LM Studio | None (local only) | OpenAI-compatible local endpoint |

### 5.5 OpenRouter

| Account Type | Authentication | Notes |
|-------------|---------------|-------|
| OpenRouter API | `Authorization: Bearer` header | Meta-provider; routes to many backends |

### 5.6 DeepSeek

| Account Type | Authentication | Notes |
|-------------|---------------|-------|
| DeepSeek API | `Authorization: Bearer` header | Direct API access |
| DeepSeek via OpenRouter | OpenRouter key | Via OpenRouter routing |

---

## 6. OAuth 2.0 Integration

### Supported OAuth Flows

| Flow | Use Case | Providers |
|------|---------|-----------|
| Authorization Code + PKCE | User personal accounts | Gemini Advanced, future providers |
| Client Credentials | Service-to-service | Vertex AI, Azure |
| Device Authorization | Desktop app flows | Future |

### OAuth Token Lifecycle

```
User initiates OAuth
        │
        ▼
Account Manager builds authorization URL
        │
        ▼
User completes provider consent screen
        │
        ▼
Callback received → exchange code for tokens
        │
        ├── Access Token (short-lived)  → encrypted in vault
        └── Refresh Token (long-lived)  → encrypted in vault
                │
                ▼
        Token expiry monitor (scheduled check every 15 min)
                │
                ▼
        Auto-refresh 5 minutes before expiry
```

### Token Storage

Access tokens and refresh tokens are encrypted with AES-256-GCM before storage. The plaintext value is never written to disk, logs, or events. Only the encrypted bytes and the key ID are persisted.

---

## 7. API Key Rotation

### Manual Rotation Flow

1. Admin initiates rotation for `(org_id, provider_id)` pair
2. Account Manager sets credential `status: PENDING_ROTATION`
3. Admin supplies new key value
4. Account Manager validates new key against provider (test API call)
5. On success: old key `status: REVOKED`, new key `status: ACTIVE`
6. Audit event `credential.rotated` published
7. Routing Engine picks up new credential on next request (no restart required)

### Automatic Rotation Flow (PROVIDER_API method)

Available for providers with key management APIs (future: Anthropic API key rotation API).

1. Rotation Scheduler triggers at `next_rotation_at`
2. Account Manager calls provider API to generate new key
3. Validates new key
4. Atomically swaps credentials
5. Schedules revocation of old key (grace period: 15 minutes for in-flight requests)

### Rotation Failure

If rotation fails:
- Old key remains `ACTIVE`
- Alert sent to org admin
- Event `credential.rotation_failed` published with no secret data
- Retry after 1 hour, 4 hours, 24 hours

---

## 8. Credential Validation

`validate_credential(org_id, provider_id) → ValidationResult`

Performs a lightweight probe against the provider (e.g., list models or single-token generation). Returns:

| Field | Type | Description |
|-------|------|-------------|
| `valid` | `bool` | Whether credential works |
| `provider_id` | `str` | |
| `error_code` | `str \| None` | `AUTH_FAILED` / `QUOTA_EXCEEDED` / `NETWORK_ERROR` |
| `account_info` | `dict \| None` | Provider-returned account details (no secrets) |
| `validated_at` | `datetime` | |

Validation is run:
- At registration time
- Daily by the health monitor
- On first use after a rotation
- On-demand by admin request

---

## 9. Multi-Tenancy Isolation

- Every credential is scoped to `(org_id, provider_id)` at minimum
- User-scoped credentials are additionally scoped to `(user_id)`
- No cross-organization credential sharing is possible at the data layer (row-level constraint)
- The Account Manager API requires a valid org_id in every request; it is enforced at the service boundary, not assumed
- Encrypted credential values are scoped with an AEAD additional data tag containing `org_id` — decryption fails if the wrong org_id is supplied

---

## 10. Interfaces

```
Account Manager Service (internal gRPC or in-process call)

get_credential(
  org_id: UUID,
  provider_id: str,
  user_id: UUID | None = None
) → ProviderCredential | None

get_subscription(
  org_id: UUID,
  provider_id: str,
  user_id: UUID | None = None
) → SubscriptionState | None

register_api_key(
  org_id: UUID,
  provider_id: str,
  api_key: str,
  alias: str,
  rotation_policy: RotationPolicy | None = None,
  registered_by: UUID
) → credential_id: UUID

register_oauth_token(
  org_id: UUID,
  provider_id: str,
  access_token: str,
  refresh_token: str | None,
  expiry: datetime | None,
  registered_by: UUID
) → credential_id: UUID

revoke_credential(
  credential_id: UUID,
  revoked_by: UUID,
  reason: str
) → void

validate_credential(
  org_id: UUID,
  provider_id: str
) → ValidationResult

list_credentials(
  org_id: UUID
) → List[CredentialSummary]  # never returns encrypted values

register_subscription(
  org_id: UUID,
  provider_id: str,
  plan_name: str,
  entitlements: SubscriptionEntitlements,
  linked_credential_id: UUID
) → subscription_id: UUID
```

---

## 11. Events Published

| Event | Payload (no secrets) |
|-------|---------------------|
| `credential.registered` | `org_id`, `provider_id`, `credential_id`, `alias` |
| `credential.validated` | `org_id`, `provider_id`, `valid`, `error_code` |
| `credential.expiry_warning` | `org_id`, `provider_id`, `days_until_expiry` |
| `credential.expired` | `org_id`, `provider_id`, `credential_id` |
| `credential.rotated` | `org_id`, `provider_id`, `credential_id` |
| `credential.rotation_failed` | `org_id`, `provider_id`, `error_code` |
| `credential.revoked` | `org_id`, `provider_id`, `credential_id`, `reason` |
| `subscription.registered` | `org_id`, `provider_id`, `plan_name` |
| `subscription.status_changed` | `org_id`, `provider_id`, `old_status`, `new_status` |

---

## 12. Security

Full security model in `AI-RESOURCE-SECURITY.md`. Summary:

- Credentials encrypted with AES-256-GCM; key management via envelope encryption
- Encryption keys stored in OS keyring / Hardware Security Module (HSM) in production
- Zero plaintext in logs: credential values masked with `[REDACTED]` before any log write
- Audit log records who accessed a credential (no value, only metadata)
- Credential values are never included in API responses, events, or error messages
- Service-to-service calls require mutual TLS
- Minimum key length enforced: API keys < 20 characters are rejected

---

## 13. Scalability

- Account Manager is read-heavy (credential lookup per request); read replicas for vault queries
- Credential cache (in-memory, TTL 60 seconds) within each AI ROS instance to reduce vault round-trips
- Cache invalidated immediately on rotation or revocation (publish invalidation event)
- Vault database is a separate, dedicated PostgreSQL instance (not shared with main AI ROS database)

---

## 14. Implementation Roadmap

### Phase 1A.0 (Week 1–2)
- Credential Vault (PostgreSQL, encrypted columns)
- `register_api_key`, `get_credential`, `revoke_credential`
- AES-256-GCM encryption implementation
- Audit logger

### Phase 1A.1 (Week 3–4)
- Subscription Registry
- Entitlements model
- `validate_credential`
- Expiry monitor

### Phase 1A.2 (Week 5–6)
- OAuth 2.0 Authorization Code + Client Credentials flows
- Token refresh scheduler
- Key rotation scheduler (MANUAL_CONFIRM mode)

### Phase 1A.4 (Week 9–10)
- AUTO_GENERATE and PROVIDER_API rotation modes
- HSM integration for production key storage

---

## 15. Future Evolution

| Feature | Notes |
|---------|-------|
| FIDO2 / Passkey for admin access | Replaces password-based admin UI |
| Provider-side key management API | Anthropic, OpenAI rolling out programmatic key creation |
| Secret Manager integrations | AWS Secrets Manager, Azure Key Vault, HashiCorp Vault as backends |
| Short-lived dynamic credentials | Zero-standing-privilege access patterns |

---

## 16. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-PROVIDER-CONTRACTS.md | Adapters receive credentials from here |
| AI-RESOURCE-SECURITY.md | Encryption and key management detail |
| AI-RESOURCE-EVENTS.md | Event schemas published by Account Manager |
| AI-RESOURCE-DATABASE.md | Vault table schemas |
| AI-RESOURCE-DESKTOP.md | Desktop credential management UI |
