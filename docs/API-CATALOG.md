# AI Studio Platform — Enterprise API Catalog

**Document ID:** API-CATALOG  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** RATIFIED  
**Classification:** INTERNAL — PLATFORM GOVERNANCE  
**Authority:** Chief Software Architect  
**Predecessor:** PLATFORM-CONTRACTS.md, EVENT-CATALOG.md  
**Review Cycle:** Quarterly — new endpoints require architect approval; breaking changes require CTO approval

---

## Catalog Authority Notice

> This catalog is the authoritative registry of every REST API exposed by the AI Studio Platform.
>
> An endpoint not in this catalog is not a platform API. Any route not registered here must be treated as an implementation detail, not a public contract.
>
> Every endpoint in this catalog has a defined contract. Changing a response field, status code, or authentication requirement without a version bump is a breaking change.
>
> This document is the source of truth for API consumer documentation. It supersedes all inline docstrings, OpenAPI annotations, and comments in the codebase.

---

## How to Read This Catalog

Each endpoint is documented across 18 dimensions. Dimensions that are shared across all endpoints in a group (e.g., base path, authentication) are stated once at the group header and inherited by all endpoints unless overridden.

**Endpoint block structure:**

```
Method              HTTP verb
Path                Full path including {path_params}
Purpose             What this endpoint does and why it exists
Request Body        JSON schema for the request body (if applicable)
Path Params         URL path parameters
Query Params        URL query string parameters
Response (2xx)      Successful response schema
Validation          Input validation rules enforced before processing
Authentication      Authentication requirement
Authorization       Role required; permission checked
Rate Limit          Requests-per-minute limit for this endpoint
Idempotency         Whether repeated identical calls are safe
Error Codes         HTTP status codes and when each is returned
Example Request     Complete curl example
Example Response    Complete JSON response example
Latency Target      P99 response time target in milliseconds
Logging             What is logged for this request (never secrets)
Audit               Whether this call is written to the immutable audit log
Deprecation Policy  When this endpoint may be removed and how
Versioning          Which API version this endpoint belongs to
```

---

## Global Conventions

### Base URL

```
http://localhost:8000      (local)
https://api.aistudio.internal  (staging / production)
```

### API Versioning

All APIs are versioned at the path level:

```
/api/v1/...    Current stable version (v1)
/api/v2/...    Future version (not yet active)
```

The version prefix `/api/v1` is present on all endpoints **except**:
- `/health` — public health check, no version
- `/health/live` — Kubernetes liveness probe
- `/health/ready` — Kubernetes readiness probe
- `/metrics` — Prometheus scrape target

### Authentication Header

All authenticated endpoints require:

```
Authorization: Bearer <api_key>
```

API keys are created via `POST /api/v1/auth/keys`. The key value is shown only once at creation.

### Public Paths (No Authentication Required)

```
GET  /health
GET  /health/live
GET  /health/ready
GET  /metrics
GET  /api/v1/workspace/capabilities
```

All other paths require authentication.

### Request Content-Type

All request bodies must be `Content-Type: application/json`. Requests with a body and no Content-Type header receive `415 Unsupported Media Type`.

### Response Envelope

All API responses — success and error — use this envelope:

**Success:**
```json
{
  "data": { ... },
  "meta": {
    "request_id": "uuid",
    "timestamp": "ISO8601",
    "version": "1.0"
  }
}
```

**Error:**
```json
{
  "error": {
    "code": "WORKFLOW_NOT_FOUND",
    "message": "Workflow 550e... does not exist",
    "request_id": "uuid",
    "timestamp": "ISO8601",
    "details": { }
  }
}
```

`error.details` contains additional structured context. It is never empty — if no additional context exists, it is `{}` not null.

### Pagination

All list endpoints support cursor-based pagination:

| Query Param | Type | Default | Description |
|-------------|------|---------|-------------|
| `limit` | integer | 20 | Results per page. Max 100. |
| `cursor` | string | null | Opaque cursor from previous response's `meta.next_cursor`. |

Paginated responses include:
```json
{
  "data": { "items": [...], "total": 142 },
  "meta": { "next_cursor": "...", "has_more": true, ... }
}
```

### Error Code Vocabulary

| HTTP Status | When |
|-------------|------|
| 400 Bad Request | Request fails validation or contains invalid parameters |
| 401 Unauthorized | No `Authorization` header, or key not found, revoked, or expired |
| 403 Forbidden | Authenticated but lacks required permission |
| 404 Not Found | Resource does not exist |
| 409 Conflict | State conflict (e.g., workflow already cancelled) |
| 410 Gone | Resource existed but was permanently deleted |
| 422 Unprocessable Entity | Request is syntactically valid but semantically invalid |
| 429 Too Many Requests | Rate limit exceeded |
| 500 Internal Server Error | Platform internal error (never expose details) |
| 503 Service Unavailable | Upstream dependency unavailable (NATS, DB, provider) |

### Correlation ID

Every request receives a `X-Request-Id` response header. This value matches `meta.request_id` in the response and `correlation_id` in all events published as a result of the request.

Callers may supply `X-Correlation-Id: <uuid>` on the request to propagate an upstream correlation ID. If provided, it is used instead of a generated one.

### Rate Limiting Headers

All authenticated responses include:
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 47
X-RateLimit-Reset: 1719580800
```

---

## Roles and Permission Reference

| Role | Level | Can do |
|------|-------|--------|
| VIEWER | 1 | Read-only access to status and outputs |
| DEVELOPER | 2 | Submit and manage workflows; render prompts; use AI |
| REVIEWER | 3 | All DEVELOPER + approve gates; promote prompt templates |
| ADMIN | 4 | All REVIEWER + manage keys; configure platform; access audit log |

Permission tokens used in this catalog:

| Permission | Required For |
|-----------|-------------|
| `auth:read` | List / get API keys |
| `auth:write` | Create / revoke API keys |
| `workspace:read` | Read workspace state |
| `workflow:read` | Read workflow status and output |
| `workflow:write` | Submit, cancel, pause, resume workflows |
| `workflow:approve` | Approve or reject gates |
| `prompt:read` | Read templates |
| `prompt:render` | Render templates |
| `prompt:write` | Register, update templates |
| `prompt:govern` | Transition template governance state |
| `brain:read` | Query patterns and recommendations |
| `brain:write` | Record experiences |
| `provider:read` | Read provider catalog and status |
| `knowledge:read` | Query knowledge graph |
| `knowledge:write` | Add nodes and relationships |
| `plugin:read` | List plugins |
| `plugin:write` | Install, uninstall, enable, disable plugins |
| `admin:read` | Read admin-only data (audit log, config) |
| `admin:write` | Modify platform configuration |
| `monitoring:read` | Read metrics, alerts, SLA reports |

---

---

# Group 1: Authentication

**Base Path:** `/api/v1/auth`  
**NATS Stream:** SECURITY  
**All endpoints audit:** see per-endpoint  
**Default Rate Limit:** 30 RPM per key

Authentication endpoints manage API key lifecycle. There is no session-based authentication on this platform. Every request is authenticated by API key.

---

## AUTH-001: Create API Key

**Method:** `POST`  
**Path:** `/api/v1/auth/keys`

**Purpose:** Create a new API key with a specified role. The key value is returned exactly once in the response and is never stored in plaintext on the server. If the caller does not save the key, it cannot be recovered.

**Request Body:**

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `label` | string | * | 1–100 chars. Human-readable name for the key. |
| `role` | enum | * | `VIEWER` \| `DEVELOPER` \| `REVIEWER` \| `ADMIN` |
| `expires_at` | ISO8601 \| null | | Expiry date. Null = no expiry. Must be in the future. |
| `scope` | enum | * | `GLOBAL` \| `PRODUCT` |
| `product_id` | string \| null | | Required if scope=PRODUCT. Must be a registered product. |
| `allowed_ips` | [string] \| null | | CIDR ranges. Null = no IP restriction. |

**Response (201 Created):**

| Field | Type | Description |
|-------|------|-------------|
| `key_id` | UUID | Permanent identifier. Use for revocation. |
| `key_value` | string | The API key. Shown once. Never retrievable again. |
| `key_prefix` | string | First 8 chars, for display in subsequent list calls. |
| `role` | enum | Role assigned to the key. |
| `label` | string | |
| `scope` | enum | |
| `product_id` | string \| null | |
| `expires_at` | ISO8601 \| null | |
| `created_at` | ISO8601 | |
| `created_by` | string | Hashed identity of the caller. |

**Validation:**
- `role` ADMIN may only be created by an existing ADMIN key
- `role` may not exceed the caller's own role (a DEVELOPER key cannot create a REVIEWER key)
- `expires_at` must be >= 1 hour in the future if provided
- `product_id` must be a currently registered product if scope=PRODUCT

**Authentication:** Required  
**Authorization:** ADMIN role for ADMIN keys; REVIEWER for REVIEWER keys; any role for lower roles  
**Permission:** `auth:write`  
**Rate Limit:** 10 RPM  
**Idempotency:** Not idempotent — each call creates a new key. No idempotency key supported.

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `ROLE_ESCALATION_FORBIDDEN` | 403 | Caller trying to create a key with higher role than their own |
| `ADMIN_ONLY` | 403 | Non-ADMIN attempting to create an ADMIN key |
| `PRODUCT_NOT_FOUND` | 404 | `product_id` does not exist in workspace |
| `INVALID_EXPIRY` | 400 | `expires_at` is in the past or too soon |

**Example Request:**
```
POST /api/v1/auth/keys
Authorization: Bearer sk_live_admin_xxxx
Content-Type: application/json

{
  "label": "AISF Product Worker",
  "role": "DEVELOPER",
  "expires_at": null,
  "scope": "PRODUCT",
  "product_id": "ai-software-factory",
  "allowed_ips": null
}
```

**Example Response:**
```json
{
  "data": {
    "key_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "key_value": "sk_live_dev_a1b2c3d4e5f6789012345678",
    "key_prefix": "sk_live_d",
    "role": "DEVELOPER",
    "label": "AISF Product Worker",
    "scope": "PRODUCT",
    "product_id": "ai-software-factory",
    "expires_at": null,
    "created_at": "2026-06-28T10:00:00Z",
    "created_by": "key_prefix:sk_live_ad"
  },
  "meta": { "request_id": "uuid", "timestamp": "2026-06-28T10:00:00Z", "version": "1.0" }
}
```

**Latency Target:** P99 < 200ms  
**Logging:** Log key_id, role, scope, created_by. Never log key_value.  
**Audit:** Yes — every key creation is written to audit log  
**Deprecation Policy:** Stable. No planned removal.  
**Versioning:** v1

---

## AUTH-002: List API Keys

**Method:** `GET`  
**Path:** `/api/v1/auth/keys`

**Purpose:** List all API keys visible to the caller. ADMIN sees all keys. Non-ADMIN sees only keys they created.

**Query Params:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `role` | enum | null | Filter by role. |
| `scope` | enum | null | Filter by scope. |
| `product_id` | string | null | Filter to keys scoped to this product. |
| `active_only` | boolean | true | If true, exclude expired and revoked keys. |
| `limit` | integer | 20 | Max 100. |
| `cursor` | string | null | |

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `items` | [KeySummary] | |
| `total` | integer | Total matching keys (across all pages). |

**KeySummary fields:** `key_id`, `key_prefix`, `role`, `label`, `scope`, `product_id`, `expires_at`, `created_at`, `last_used_at` (null if never used), `status` (ACTIVE | EXPIRED | REVOKED)

**Authentication:** Required | **Authorization:** Any role | **Permission:** `auth:read`  
**Rate Limit:** 60 RPM | **Idempotency:** Idempotent (read-only)

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `INVALID_CURSOR` | 400 | Cursor is expired or malformed |

**Example Request:**
```
GET /api/v1/auth/keys?role=DEVELOPER&active_only=true&limit=10
Authorization: Bearer sk_live_admin_xxxx
```

**Example Response:**
```json
{
  "data": {
    "items": [
      {
        "key_id": "a1b2c3d4-...",
        "key_prefix": "sk_live_d",
        "role": "DEVELOPER",
        "label": "AISF Product Worker",
        "scope": "PRODUCT",
        "product_id": "ai-software-factory",
        "expires_at": null,
        "created_at": "2026-06-28T10:00:00Z",
        "last_used_at": "2026-06-28T14:22:00Z",
        "status": "ACTIVE"
      }
    ],
    "total": 1
  },
  "meta": { "request_id": "uuid", "timestamp": "...", "version": "1.0", "next_cursor": null, "has_more": false }
}
```

**Latency Target:** P99 < 150ms | **Logging:** Log caller, filter params (no key values) | **Audit:** No  
**Versioning:** v1

---

## AUTH-003: Get API Key

**Method:** `GET`  
**Path:** `/api/v1/auth/keys/{key_id}`

**Purpose:** Get details for a specific API key. Never returns key_value.

**Path Params:** `key_id` (UUID)

**Response (200 OK):** Full KeySummary (same fields as list items) plus `allowed_ips`.

**Authentication:** Required | **Authorization:** ADMIN sees any key; others only see own keys  
**Permission:** `auth:read` | **Rate Limit:** 60 RPM | **Idempotency:** Idempotent

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `KEY_NOT_FOUND` | 404 | Key doesn't exist or caller can't see it |

**Latency Target:** P99 < 100ms | **Audit:** No | **Versioning:** v1

---

## AUTH-004: Revoke API Key

**Method:** `DELETE`  
**Path:** `/api/v1/auth/keys/{key_id}`

**Purpose:** Permanently revoke an API key. The key is invalidated immediately — all subsequent requests using the revoked key return 401. Revocation is irreversible.

**Path Params:** `key_id` (UUID)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | * | Required justification. 1–500 chars. |

**Response (200 OK):** `{ "key_id": "...", "revoked_at": "ISO8601", "was_active": true }`

**Validation:** Cannot revoke your own active key if it is the only ADMIN key in the system.

**Authentication:** Required  
**Authorization:** ADMIN may revoke any key; non-ADMIN may only revoke own keys  
**Permission:** `auth:write` | **Rate Limit:** 10 RPM | **Idempotency:** Idempotent — revoking an already-revoked key returns 200

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `KEY_NOT_FOUND` | 404 | |
| `LAST_ADMIN_KEY` | 409 | Cannot revoke the last ADMIN key |
| `PERMISSION_DENIED` | 403 | Non-ADMIN trying to revoke another user's key |
| `REASON_REQUIRED` | 400 | `reason` is empty or missing |

**Latency Target:** P99 < 200ms | **Audit:** Yes | **Versioning:** v1

---

## AUTH-005: Get Rate Limit Status

**Method:** `GET`  
**Path:** `/api/v1/auth/rate-limits`

**Purpose:** Return current rate limit consumption for the calling key across all limit types.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `global_rpm` | object | `{"limit": 60, "used": 14, "reset_at": "ISO8601"}` |
| `per_path_limits` | [object] | Per-path current usage. |
| `ai_executions_rpm` | object | AI-specific limit (tighter than global). |

**Authentication:** Required | **Authorization:** Any role | **Permission:** `auth:read`  
**Rate Limit:** 120 RPM (this endpoint itself is not rate-limited against the global RPM counter)  
**Idempotency:** Idempotent | **Latency Target:** P99 < 50ms | **Audit:** No | **Versioning:** v1

---

## AUTH-006: Get Audit Log

**Method:** `GET`  
**Path:** `/api/v1/auth/audit-log`

**Purpose:** Retrieve the immutable audit log. Only ADMIN role can access this. Results are never editable or deletable.

**Query Params:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `start` | ISO8601 | 24h ago | Start of time range. |
| `end` | ISO8601 | now | End of time range. Max range: 90 days. |
| `event_type` | string | null | Filter to a specific event type. |
| `identity` | string | null | Filter to a specific identity (hashed). |
| `limit` | integer | 50 | Max 200. |
| `cursor` | string | null | |

**Response (200 OK):** Paginated list of audit entries. Each entry contains: `entry_id`, `event_type`, `timestamp`, `identity` (hashed), `ip_hash`, `details` (varies by event type — never contains secrets).

**Authentication:** Required | **Authorization:** ADMIN only | **Permission:** `admin:read`  
**Rate Limit:** 30 RPM | **Idempotency:** Idempotent

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `RANGE_TOO_LARGE` | 400 | Requested time range exceeds 90 days |
| `INVALID_DATE` | 400 | `start` or `end` is not valid ISO8601 |
| `ADMIN_REQUIRED` | 403 | Non-ADMIN caller |

**Latency Target:** P99 < 500ms (database scan) | **Audit:** Yes (accessing the audit log is itself audited) | **Versioning:** v1

---

---

# Group 2: Workspace

**Base Path:** `/api/v1/workspace`  
**NATS Stream:** WORKSPACE  
**Default Rate Limit:** 60 RPM

Workspace endpoints expose the registry of active platform resources: products, providers, and platform health.

---

## WS-001: Get Workspace

**Method:** `GET`  
**Path:** `/api/v1/workspace`

**Purpose:** Return a complete snapshot of the current workspace state — all registered products, their health, and platform-wide health summary. This is the primary endpoint for the Desktop WorkspacePanel.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `platform_version` | string | |
| `environment` | string | |
| `uptime_seconds` | integer | |
| `overall_health` | enum | `HEALTHY` \| `DEGRADED` \| `OFFLINE` |
| `products` | [ProductSummary] | All registered products. |
| `providers` | [ProviderSummary] | All registered AI providers. |
| `module_health` | object | Per-module health status map. |

**ProductSummary fields:** `product_id`, `product_name`, `version`, `health`, `registered_since`, `capabilities`  
**ProviderSummary fields:** `provider_id`, `display_name`, `health`, `circuit_state`, `models`

**Authentication:** Required | **Authorization:** Any role | **Permission:** `workspace:read`  
**Rate Limit:** 120 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 100ms (cached; max staleness 30s)  
**Audit:** No | **Versioning:** v1

---

## WS-002: Get Platform Capabilities

**Method:** `GET`  
**Path:** `/api/v1/workspace/capabilities`

**Purpose:** Return the set of capabilities the platform supports. This is a **public endpoint** — no authentication required. Used by clients to determine what features are enabled before authenticating.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `api_version` | string | Current API version. |
| `features` | [string] | Enabled platform features (e.g., `["workflows", "brain", "prompt-os"]`). |
| `feature_flags` | object | Non-sensitive feature flags: `{"marketplace": false, "multi_tenant": false}`. |
| `ai_providers_available` | integer | Count of healthy AI providers. |
| `auth_required` | boolean | Always true on this platform. |

**Authentication:** **NOT REQUIRED** (public endpoint)  
**Rate Limit:** 600 RPM (unauthenticated IP-based limit) | **Idempotency:** Idempotent  
**Latency Target:** P99 < 50ms | **Audit:** No | **Versioning:** v1

---

## WS-003: List Products

**Method:** `GET`  
**Path:** `/api/v1/workspace/products`

**Purpose:** List all currently registered products in the workspace.

**Query Params:** `health_status` (filter), `limit`, `cursor`

**Response (200 OK):** Paginated list of ProductSummary objects.

**Authentication:** Required | **Authorization:** Any role | **Permission:** `workspace:read`  
**Rate Limit:** 60 RPM | **Idempotency:** Idempotent | **Latency Target:** P99 < 100ms  
**Audit:** No | **Versioning:** v1

---

## WS-004: Get Product

**Method:** `GET`  
**Path:** `/api/v1/workspace/products/{product_id}`

**Purpose:** Get detailed information for a specific registered product, including its declared capabilities and current health.

**Path Params:** `product_id` (string)

**Response (200 OK):** Full ProductDetail — all ProductSummary fields plus: `api_prefix`, `required_platform_modules`, `capabilities` (full list), `health_history` (last 24h).

**Authentication:** Required | **Permission:** `workspace:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 100ms | **Audit:** No | **Versioning:** v1

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `PRODUCT_NOT_FOUND` | 404 | Product not registered |

---

## WS-005: Get Platform Health

**Method:** `GET`  
**Path:** `/health`

**Purpose:** Returns the overall platform health status. Used by load balancers, monitoring systems, and the Desktop health panel. **Public endpoint** — no authentication required.

**Response (200 OK — HEALTHY or DEGRADED):**

| Field | Type | Description |
|-------|------|-------------|
| `status` | enum | `HEALTHY` \| `DEGRADED` \| `OFFLINE` |
| `modules` | object | Per-module status map. |
| `timestamp` | ISO8601 | |

**Response (503 Service Unavailable — OFFLINE):** Same schema. HTTP 503 allows load balancers to remove the instance.

**Authentication:** NOT REQUIRED | **Rate Limit:** 600 RPM (IP-based)  
**Idempotency:** Idempotent | **Latency Target:** P99 < 30ms | **Audit:** No | **Versioning:** Unversioned

---

## WS-006: Kubernetes Liveness Probe

**Method:** `GET`  
**Path:** `/health/live`

**Purpose:** Returns 200 if the process is alive. Returns 503 if the process should be restarted.

**Response (200 OK):** `{"alive": true}`  
**Response (503):** `{"alive": false, "reason": "..."}`

**Authentication:** NOT REQUIRED | **Rate Limit:** Unlimited | **Latency Target:** P99 < 10ms  
**Audit:** No | **Versioning:** Unversioned

---

## WS-007: Kubernetes Readiness Probe

**Method:** `GET`  
**Path:** `/health/ready`

**Purpose:** Returns 200 if the platform is ready to receive traffic (all required dependencies connected). Returns 503 during startup, shutdown, or when a critical dependency is unavailable.

**Response (200 OK):** `{"ready": true}`  
**Response (503):** `{"ready": false, "waiting_for": ["nats", "postgres"]}`

**Authentication:** NOT REQUIRED | **Rate Limit:** Unlimited | **Latency Target:** P99 < 20ms  
**Audit:** No | **Versioning:** Unversioned

---

## WS-008: Register Product (Internal)

**Method:** `POST`  
**Path:** `/api/v1/workspace/products/{product_id}/register`

**Purpose:** Register a product's manifest with the workspace. This endpoint is called by products on startup, not by human operators. Requires ADMIN-scoped key.

**Path Params:** `product_id` (string)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_name` | string | * | |
| `version` | string | * | SemVer. |
| `api_prefix` | string | * | |
| `capabilities` | [string] | * | |
| `required_platform_modules` | [string] | * | |

**Response (201 Created):** `{ "product_id": "...", "registered_at": "..." }`  
**Response (200 OK):** If already registered — performs an update (idempotent re-registration).

**Validation:** `version` must be valid SemVer. `product_id` must match `^[a-z][a-z0-9-]*$`.

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:write`  
**Rate Limit:** 10 RPM | **Idempotency:** Idempotent — re-registering with updated version returns 200  
**Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

---

## WS-009: Deregister Product (Internal)

**Method:** `DELETE`  
**Path:** `/api/v1/workspace/products/{product_id}`

**Purpose:** Remove a product from the workspace registry. Called by products during graceful shutdown. Any in-flight workflows belonging to this product are checkpointed.

**Response (200 OK):** `{ "product_id": "...", "deregistered_at": "...", "workflows_checkpointed": 3 }`

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:write`  
**Rate Limit:** 10 RPM | **Idempotency:** Idempotent — deregistering an already-deregistered product returns 200  
**Latency Target:** P99 < 500ms (checkpoint overhead) | **Audit:** No | **Versioning:** v1

---

---

# Group 3: Workflow

**Base Path:** `/api/v1/workflows`  
**NATS Stream:** WORKFLOWS  
**Default Rate Limit:** 60 RPM

Workflow endpoints cover the full lifecycle of AI workflow plans from submission to completion. The WorkflowRuntime executes these plans. The `_simulate_task_execution()` bypass is removed — all tasks run through WorkflowRuntime.

---

## WF-001: Submit Workflow

**Method:** `POST`  
**Path:** `/api/v1/workflows`

**Purpose:** Submit a workflow plan for execution. The plan defines tasks, dependencies, agent assignments, and optional human approval gates. Execution begins immediately after submission.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | The product submitting this workflow. |
| `plan_name` | string | * | Human-readable name. 1–200 chars. |
| `priority` | enum | | `URGENT` \| `STANDARD` \| `BATCH`. Default: `STANDARD`. |
| `idempotency_key` | string | | If provided, a duplicate submission with the same key returns the existing workflow (200). |
| `tasks` | [TaskDefinition] | * | At least 1 task required. |
| `dag_edges` | [DagEdge] | | If omitted, tasks execute sequentially in array order. |
| `gates` | [GateDefinition] | | Human approval gates. |
| `metadata` | object | | Caller-defined key-value metadata (max 20 keys, values max 500 chars each). |

**TaskDefinition:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `task_id` | string | * | Unique within the plan. `^[a-z0-9_-]{1,64}$` |
| `task_type` | string | * | Registered task type. |
| `agent_type` | string | * | Registered agent type. |
| `input` | object | * | Task-specific input. |
| `priority` | integer | | 1–10. Default: 5. |
| `timeout_seconds` | integer | | Default: 300. Max: 3600. |
| `max_retries` | integer | | 0–5. Default: 3. |
| `tools` | [string] | | Tool names the agent may use. |

**GateDefinition:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `gate_id` | string | * | Unique within the plan. |
| `after_task_id` | string | * | Task that must complete before this gate blocks. |
| `required_role` | enum | * | `REVIEWER` \| `ADMIN` |
| `description` | string | * | What the reviewer is being asked to approve. |
| `auto_approve_after_seconds` | integer | | **FORBIDDEN.** If present, request is rejected with 400 `AUTO_APPROVE_FORBIDDEN`. |

**DagEdge:** `{ "from": "task_id_a", "to": "task_id_b" }`

**Response (201 Created):**

| Field | Type | Description |
|-------|------|-------------|
| `workflow_id` | UUID | |
| `product_id` | string | |
| `plan_name` | string | |
| `status` | enum | Initial status: `QUEUED` |
| `task_count` | integer | |
| `gate_count` | integer | |
| `submitted_at` | ISO8601 | |
| `priority` | enum | |

**Response (200 OK):** Returned when `idempotency_key` matches an existing workflow. Body contains the existing workflow status — not a new submission.

**Validation:**
- `task_count` >= 1 and <= 100
- All `dag_edges` reference valid `task_id` values
- No cycles in the DAG
- `auto_approve_after_seconds` on any gate → 400
- `product_id` must be a currently registered product
- `task_type` and `agent_type` must be registered types
- Tools in `tasks[].tools` must be permitted for the product

**Authentication:** Required | **Authorization:** DEVELOPER+ | **Permission:** `workflow:write`  
**Rate Limit:** 30 RPM | **Idempotency:** `idempotency_key` field

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `PRODUCT_NOT_FOUND` | 404 | `product_id` not registered |
| `AUTO_APPROVE_FORBIDDEN` | 400 | `auto_approve_after_seconds` present on any gate |
| `DAG_CYCLE_DETECTED` | 422 | Dependency graph contains a cycle |
| `UNKNOWN_TASK_TYPE` | 422 | `task_type` not registered |
| `UNKNOWN_AGENT_TYPE` | 422 | `agent_type` not registered |
| `TASK_LIMIT_EXCEEDED` | 422 | More than 100 tasks in a single plan |
| `TOOL_NOT_PERMITTED` | 403 | Product not authorized to use a requested tool |
| `BUDGET_EXCEEDED` | 402 | Product's daily budget is fully consumed |

**Example Request:**
```
POST /api/v1/workflows
Authorization: Bearer sk_live_dev_xxxx
Content-Type: application/json

{
  "product_id": "ai-software-factory",
  "plan_name": "Build REST API — User Service",
  "priority": "STANDARD",
  "idempotency_key": "aisf-build-user-service-v1",
  "tasks": [
    {
      "task_id": "design",
      "task_type": "design_architecture",
      "agent_type": "architect_agent",
      "input": { "requirements": "REST API for user management" },
      "timeout_seconds": 300,
      "max_retries": 2
    },
    {
      "task_id": "implement",
      "task_type": "write_code",
      "agent_type": "developer_agent",
      "input": { "language": "python" },
      "timeout_seconds": 600
    }
  ],
  "dag_edges": [
    { "from": "design", "to": "implement" }
  ],
  "gates": [
    {
      "gate_id": "design_review",
      "after_task_id": "design",
      "required_role": "REVIEWER",
      "description": "Review architecture before implementation begins"
    }
  ]
}
```

**Example Response:**
```json
{
  "data": {
    "workflow_id": "550e8400-e29b-41d4-a716-446655440000",
    "product_id": "ai-software-factory",
    "plan_name": "Build REST API — User Service",
    "status": "QUEUED",
    "task_count": 2,
    "gate_count": 1,
    "submitted_at": "2026-06-28T10:00:00Z",
    "priority": "STANDARD"
  },
  "meta": { "request_id": "uuid", "timestamp": "2026-06-28T10:00:00Z", "version": "1.0" }
}
```

**Latency Target:** P99 < 500ms (plan validation + DAG analysis + NATS publish)  
**Logging:** Log workflow_id, product_id, task_count, gate_count, priority. Never log task input.  
**Audit:** No (workflow.created event covers it) | **Versioning:** v1

---

## WF-002: List Workflows

**Method:** `GET`  
**Path:** `/api/v1/workflows`

**Purpose:** List workflows, optionally filtered by product, status, or time range.

**Query Params:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `product_id` | string | null | Filter to a specific product. |
| `status` | enum | null | `QUEUED` \| `ACTIVE` \| `PAUSED` \| `COMPLETED` \| `FAILED` \| `CANCELLED` |
| `submitted_after` | ISO8601 | null | |
| `submitted_before` | ISO8601 | null | |
| `limit` | integer | 20 | Max 100. |
| `cursor` | string | null | |

**Response (200 OK):** Paginated list of WorkflowSummary objects.

**WorkflowSummary:** `workflow_id`, `product_id`, `plan_name`, `status`, `priority`, `task_count`, `tasks_completed`, `tasks_failed`, `submitted_at`, `completed_at` (null if not done), `total_cost_usd` (null if not done)

**Authentication:** Required | **Authorization:** DEVELOPER+ sees own product's workflows; ADMIN sees all  
**Permission:** `workflow:read` | **Rate Limit:** 60 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

---

## WF-003: Get Workflow Status

**Method:** `GET`  
**Path:** `/api/v1/workflows/{workflow_id}`

**Purpose:** Get the full status of a workflow, including per-task status and gate state.

**Path Params:** `workflow_id` (UUID)

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `workflow_id` | UUID | |
| `product_id` | string | |
| `plan_name` | string | |
| `status` | enum | |
| `priority` | enum | |
| `submitted_at` | ISO8601 | |
| `started_at` | ISO8601 \| null | |
| `completed_at` | ISO8601 \| null | |
| `duration_ms` | integer \| null | |
| `tasks` | [TaskStatus] | Per-task current state. |
| `gates` | [GateStatus] | Per-gate current state. |
| `total_cost_usd` | float \| null | Null if not yet complete. |
| `total_tokens` | integer \| null | |
| `checkpoint_available` | boolean | Whether a checkpoint exists. |

**TaskStatus fields:** `task_id`, `task_type`, `agent_type`, `status` (PENDING | READY | IN_PROGRESS | COMPLETED | FAILED | CANCELLED | SKIPPED), `started_at`, `completed_at`, `retry_count`, `error_code` (if failed)

**GateStatus fields:** `gate_id`, `status` (PENDING | APPROVED | REJECTED | SKIPPED), `after_task_id`, `resolved_at`, `resolved_by` (hashed)

**Authentication:** Required | **Permission:** `workflow:read`  
**Rate Limit:** 120 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 150ms | **Audit:** No | **Versioning:** v1

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `WORKFLOW_NOT_FOUND` | 404 | |

---

## WF-004: Get Task Output

**Method:** `GET`  
**Path:** `/api/v1/workflows/{workflow_id}/tasks/{task_id}/output`

**Purpose:** Retrieve the structured output produced by a completed task. Only available if the task is in COMPLETED state.

**Path Params:** `workflow_id` (UUID), `task_id` (string)

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `task_id` | string | |
| `task_type` | string | |
| `status` | enum | |
| `output` | object | Task-type-specific output. |
| `artifacts` | [string] | Artifact keys (reference by path). |
| `tool_calls` | [ToolCallSummary] | Summary of tool calls made (not full content). |
| `ai_execution_ids` | [UUID] | Execution IDs for cost attribution. |

**Authentication:** Required | **Permission:** `workflow:read`  
**Rate Limit:** 120 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `WORKFLOW_NOT_FOUND` | 404 | |
| `TASK_NOT_FOUND` | 404 | |
| `TASK_NOT_COMPLETED` | 409 | Task is not yet in COMPLETED state |

---

## WF-005: Cancel Workflow

**Method:** `POST`  
**Path:** `/api/v1/workflows/{workflow_id}/cancel`

**Purpose:** Cancel a running or paused workflow. In-progress tasks are allowed to complete. Pending tasks are cancelled. The workflow transitions to CANCELLED.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | * | Cancellation reason. 1–500 chars. |

**Response (200 OK):** `{ "workflow_id": "...", "status": "CANCELLED", "cancelled_at": "..." }`

**Validation:** Cannot cancel a workflow already in COMPLETED, FAILED, or CANCELLED state.

**Authentication:** Required | **Authorization:** DEVELOPER+ (own product); ADMIN (any)  
**Permission:** `workflow:write` | **Rate Limit:** 30 RPM | **Idempotency:** Idempotent — cancelling an already-cancelled workflow returns 200

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `WORKFLOW_NOT_FOUND` | 404 | |
| `WORKFLOW_TERMINAL` | 409 | Already COMPLETED, FAILED, or CANCELLED |
| `REASON_REQUIRED` | 400 | |

**Latency Target:** P99 < 500ms | **Audit:** Yes | **Versioning:** v1

---

## WF-006: Pause Workflow

**Method:** `POST`  
**Path:** `/api/v1/workflows/{workflow_id}/pause`

**Purpose:** Pause a running workflow. No new tasks are dispatched. In-progress tasks complete. The workflow can be resumed with WF-007.

**Request Body:** `{ "reason": "string (required)" }`

**Response (200 OK):** `{ "workflow_id": "...", "status": "PAUSED", "next_task_id": "implement", "paused_at": "..." }`

**Authentication:** Required | **Permission:** `workflow:write` | **Rate Limit:** 30 RPM  
**Idempotency:** Idempotent — pausing a paused workflow returns 200 with current state  
**Latency Target:** P99 < 300ms | **Audit:** No | **Versioning:** v1

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `WORKFLOW_NOT_FOUND` | 404 | |
| `WORKFLOW_NOT_PAUSABLE` | 409 | Workflow is QUEUED, COMPLETED, FAILED, or CANCELLED |

---

## WF-007: Resume Workflow

**Method:** `POST`  
**Path:** `/api/v1/workflows/{workflow_id}/resume`

**Purpose:** Resume a paused workflow from where it stopped.

**Response (200 OK):** `{ "workflow_id": "...", "status": "ACTIVE", "resumed_at": "...", "next_task_id": "..." }`

**Authentication:** Required | **Permission:** `workflow:write` | **Rate Limit:** 30 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 300ms | **Audit:** No | **Versioning:** v1

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `WORKFLOW_NOT_PAUSED` | 409 | Workflow is not in PAUSED state |

---

## WF-008: Get Workflow Checkpoint

**Method:** `GET`  
**Path:** `/api/v1/workflows/{workflow_id}/checkpoint`

**Purpose:** Retrieve the latest checkpoint for a workflow. Checkpoints capture the state of all completed tasks and are used for resume after platform restart.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `checkpoint_id` | UUID | |
| `workflow_id` | UUID | |
| `created_at` | ISO8601 | |
| `completed_tasks` | [string] | Task IDs of completed tasks at checkpoint time. |
| `completed_task_outputs` | object | Map of task_id → output summary. |
| `can_resume` | boolean | Whether the checkpoint is sufficient to resume. |

**Authentication:** Required | **Permission:** `workflow:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `NO_CHECKPOINT` | 404 | No checkpoint exists for this workflow |

---

## WF-009: Approve Gate

**Method:** `POST`  
**Path:** `/api/v1/workflows/{workflow_id}/gates/{gate_id}/approve`

**Purpose:** Approve a pending human approval gate. The workflow resumes from the next task after the gate.

**Path Params:** `workflow_id` (UUID), `gate_id` (string)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `comment` | string | | Optional reviewer comment. Max 2000 chars. |

**Response (200 OK):** `{ "gate_id": "...", "status": "APPROVED", "approved_at": "...", "next_task_id": "..." }`

**Validation:** Caller must hold at least `gate.required_role`.

**Authentication:** Required | **Authorization:** REVIEWER+ (or ADMIN) | **Permission:** `workflow:approve`  
**Rate Limit:** 30 RPM | **Idempotency:** Idempotent — approving an already-approved gate returns 200

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `GATE_NOT_PENDING` | 409 | Gate is already APPROVED, REJECTED, or SKIPPED |
| `INSUFFICIENT_ROLE` | 403 | Caller's role is below the gate's `required_role` |
| `WORKFLOW_NOT_PAUSED` | 409 | Workflow is not in PAUSED state at this gate |

**Latency Target:** P99 < 300ms | **Audit:** Yes | **Versioning:** v1

---

## WF-010: Reject Gate

**Method:** `POST`  
**Path:** `/api/v1/workflows/{workflow_id}/gates/{gate_id}/reject`

**Purpose:** Reject a pending approval gate. The workflow transitions to FAILED.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `rejection_reason` | string | * | Required. 1–2000 chars. |

**Response (200 OK):** `{ "gate_id": "...", "status": "REJECTED", "rejected_at": "...", "workflow_status": "FAILED" }`

**Authentication:** Required | **Authorization:** REVIEWER+ | **Permission:** `workflow:approve`  
**Rate Limit:** 30 RPM | **Idempotency:** Idempotent

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `GATE_NOT_PENDING` | 409 | |
| `REASON_REQUIRED` | 400 | `rejection_reason` empty or missing |

**Latency Target:** P99 < 300ms | **Audit:** Yes | **Versioning:** v1

---

---

# Group 4: Prompt

**Base Path:** `/api/v1/prompts`  
**NATS Stream:** PROMPTS  
**Default Rate Limit:** 120 RPM

Prompt endpoints expose Prompt OS — the canonical prompt management system. All prompt renders go through Prompt OS with HMAC signature verification. No product may bypass Prompt OS.

---

## PR-001: Render Template

**Method:** `POST`  
**Path:** `/api/v1/prompts/render`

**Purpose:** Render a prompt template with variable substitution and HMAC signature verification. Returns the rendered prompt string ready for AI execution. This is the most called Prompt endpoint.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `version` | integer \| null | | If null, renders the current ACTIVE version. |
| `variables` | object | * | Key-value map matching the template's declared variables. |
| `context` | object | | Optional context for conditional rendering. |
| `ab_test_id` | string \| null | | If provided, assigns this render to the specified A/B test. |

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `template_id` | UUID | |
| `version` | integer | The version that was rendered. |
| `rendered_text` | string | The fully rendered prompt. |
| `system_prompt` | string \| null | Rendered system prompt if the template has one. |
| `hmac_verified` | boolean | Always true. If HMAC fails, request returns 422, not 200. |
| `ab_variant` | enum \| null | `A` or `B` if render was part of an A/B test. |
| `token_estimate` | integer | Rough token count of the rendered output. |
| `render_id` | UUID | Unique ID for this render (used in A/B result recording). |

**Validation:**
- All variables declared in the template must be provided
- No extra variables may be provided (strict mode)
- Template must be in ACTIVE state (not DRAFT, REVIEW, DEPRECATED, ARCHIVED, or SUSPENDED)
- HMAC signature of the ACTIVE version content must match server record

**Authentication:** Required | **Authorization:** DEVELOPER+ | **Permission:** `prompt:render`  
**Rate Limit:** 600 RPM (this is the hot path) | **Idempotency:** Not idempotent (render_id changes each call; A/B assignment may differ)

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `TEMPLATE_NOT_FOUND` | 404 | |
| `TEMPLATE_NOT_ACTIVE` | 409 | Template is not in ACTIVE state |
| `TEMPLATE_SUSPENDED` | 403 | Template is SUSPENDED — all renders blocked |
| `MISSING_VARIABLES` | 422 | Required variables not provided |
| `UNKNOWN_VARIABLES` | 422 | Variables provided that are not declared in the template |
| `HMAC_VERIFICATION_FAILED` | 422 | Template content has been tampered with |

**Latency Target:** P99 < 50ms (cached template + in-memory HMAC verify)  
**Logging:** Log template_id, version, render_id, variable keys (not values). Never log rendered_text.  
**Audit:** No (PM-007 event only on HMAC failure) | **Versioning:** v1

---

## PR-002: Compose Templates

**Method:** `POST`  
**Path:** `/api/v1/prompts/compose`

**Purpose:** Render and compose multiple templates into a single combined prompt. Useful for building multi-section prompts from reusable components.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `components` | [RenderRequest] | * | Ordered list of templates to render and compose. Min 2, max 10. |
| `separator` | string | | String inserted between rendered components. Default: `"\n\n"`. |

**RenderRequest:** Same fields as PR-001 request body.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `composed_text` | string | All rendered templates joined by separator. |
| `system_prompt` | string \| null | Combined system prompt if any component had one. |
| `components` | [ComponentResult] | Per-template render result (template_id, version, render_id). |
| `total_token_estimate` | integer | |

**Authentication:** Required | **Permission:** `prompt:render` | **Rate Limit:** 120 RPM  
**Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

---

## PR-003: Register Template

**Method:** `POST`  
**Path:** `/api/v1/prompts/templates`

**Purpose:** Register a new prompt template in Prompt OS. New templates are always created in DRAFT state.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `display_name` | string | * | 1–200 chars. |
| `owner` | string | * | Team or product identifier. |
| `content` | string | * | Template content with `{{variable}}` placeholders. |
| `system_prompt` | string \| null | | System prompt content. |
| `variables` | [VariableDefinition] | * | Declared variables. |
| `tags` | [string] | | Classification tags. |
| `description` | string | | Human description of the template's purpose. |

**VariableDefinition:** `{ "name": "user_input", "type": "string", "required": true, "description": "..." }`

**Response (201 Created):**

| Field | Type | Description |
|-------|------|-------------|
| `template_id` | UUID | |
| `version` | integer | Always 1. |
| `state` | enum | Always `DRAFT`. |
| `hmac_signature` | string | HMAC of the stored content. |
| `created_at` | ISO8601 | |

**Validation:**
- `display_name` must be unique within the `owner`
- All `{{variable}}` placeholders in `content` must be declared in `variables`
- No declared `variables` may be absent from `content`

**Authentication:** Required | **Authorization:** DEVELOPER+ | **Permission:** `prompt:write`  
**Rate Limit:** 30 RPM | **Idempotency:** Not idempotent  
**Latency Target:** P99 < 300ms | **Audit:** No | **Versioning:** v1

---

## PR-004: Update Template (New Version)

**Method:** `PUT`  
**Path:** `/api/v1/prompts/templates/{template_id}/versions`

**Purpose:** Create a new version of an existing template. New versions are always DRAFT. The previous version's state is unaffected.

**Path Params:** `template_id` (UUID)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `content` | string | * | New template content. |
| `system_prompt` | string \| null | | |
| `variables` | [VariableDefinition] | * | |
| `change_notes` | string | * | What changed and why. 1–1000 chars. |

**Response (201 Created):**

| Field | Type | Description |
|-------|------|-------------|
| `template_id` | UUID | |
| `version` | integer | New version number (previous + 1). |
| `parent_version` | integer | |
| `state` | enum | Always `DRAFT`. |
| `hmac_signature` | string | HMAC of the new version. |

**Authentication:** Required | **Permission:** `prompt:write` | **Rate Limit:** 30 RPM  
**Idempotency:** Not idempotent (creates new version each call)  
**Latency Target:** P99 < 300ms | **Audit:** No | **Versioning:** v1

---

## PR-005: Transition Template State

**Method:** `POST`  
**Path:** `/api/v1/prompts/templates/{template_id}/transition`

**Purpose:** Transition a template version through the governance state machine.

**Allowed transitions:**
- `DRAFT` → `REVIEW` (submit for review)
- `REVIEW` → `ACTIVE` (promote — requires REVIEWER role)
- `ACTIVE` → `DEPRECATED`
- `DEPRECATED` → `ARCHIVED`
- `ACTIVE` → `SUSPENDED` (emergency block — ADMIN only)
- `SUSPENDED` → `ACTIVE` (lift suspension — ADMIN only)

**Path Params:** `template_id` (UUID)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `version` | integer | * | The version being transitioned. |
| `target_state` | enum | * | Target governance state. |
| `notes` | string | | Required for DEPRECATED, SUSPENDED. |
| `removal_after` | ISO8601 | | Required for DEPRECATED. Must be future. |
| `replacement_template_id` | UUID | | Recommended for DEPRECATED. |

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `template_id` | UUID | |
| `version` | integer | |
| `previous_state` | enum | |
| `current_state` | enum | |
| `transitioned_at` | ISO8601 | |

**Validation:**
- Transition must follow the governance FSM (invalid transitions → 409)
- ACTIVE transition clears the template cache (cache TTL 60s)
- SUSPENDED transition immediately blocks renders

**Authentication:** Required  
**Authorization:** `DRAFT→REVIEW`: DEVELOPER+; `REVIEW→ACTIVE`: REVIEWER+; `ACTIVE→SUSPENDED`: ADMIN  
**Permission:** `prompt:govern` | **Rate Limit:** 30 RPM

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `INVALID_TRANSITION` | 409 | Transition not allowed by governance FSM |
| `VERSION_NOT_FOUND` | 404 | Specified version doesn't exist |
| `NOTES_REQUIRED` | 400 | Notes required for DEPRECATED or SUSPENDED |
| `REMOVAL_DATE_REQUIRED` | 400 | `removal_after` required for DEPRECATED |

**Latency Target:** P99 < 300ms | **Audit:** Yes | **Versioning:** v1

---

## PR-006: List Templates

**Method:** `GET`  
**Path:** `/api/v1/prompts/templates`

**Purpose:** List prompt templates with optional filters.

**Query Params:** `owner`, `state`, `tag`, `limit`, `cursor`

**Response (200 OK):** Paginated list. Each item: `template_id`, `display_name`, `owner`, `current_state`, `active_version` (null if no active version), `total_versions`, `tags`, `created_at`.

**Authentication:** Required | **Permission:** `prompt:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 150ms | **Audit:** No | **Versioning:** v1

---

## PR-007: Get Template

**Method:** `GET`  
**Path:** `/api/v1/prompts/templates/{template_id}`

**Purpose:** Get full detail for a specific template, including all versions.

**Query Params:** `version` (integer, default: latest)

**Response (200 OK):** Full template detail including content, variables, state, all versions list.

**Authentication:** Required | **Permission:** `prompt:read` | **Rate Limit:** 120 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 100ms | **Audit:** No | **Versioning:** v1

---

## PR-008: Get Version Diff

**Method:** `GET`  
**Path:** `/api/v1/prompts/templates/{template_id}/diff`

**Purpose:** Return a unified diff between two template versions.

**Query Params:** `from_version` (integer, required), `to_version` (integer, required)

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `template_id` | UUID | |
| `from_version` | integer | |
| `to_version` | integer | |
| `diff` | string | Unified diff format. |
| `lines_added` | integer | |
| `lines_removed` | integer | |
| `variables_changed` | boolean | |

**Authentication:** Required | **Permission:** `prompt:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

---

## PR-009: Record A/B Test Result

**Method:** `POST`  
**Path:** `/api/v1/prompts/ab-tests/{test_id}/results`

**Purpose:** Record the outcome of an A/B test render. The caller reports whether the rendered output was successful for their use case.

**Path Params:** `test_id` (string)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `render_id` | UUID | * | The render_id from a prior PR-001 call with this test_id. |
| `variant` | enum | * | `A` \| `B` |
| `success` | boolean | * | Whether the render produced a satisfactory outcome. |
| `feedback` | string \| null | | Optional qualitative feedback. Max 500 chars. |

**Response (200 OK):** `{ "recorded": true, "test_id": "...", "variant_counts": {"A": 142, "B": 138} }`

**Authentication:** Required | **Permission:** `prompt:render` | **Rate Limit:** 300 RPM  
**Idempotency:** `render_id` is unique per render — duplicate submissions for the same render_id are ignored  
**Latency Target:** P99 < 50ms | **Audit:** No | **Versioning:** v1

---


---

# Group 5: Brain

**Base Path:** `/api/v1/brain`  
**NATS Stream:** BRAIN  
**Stability Tier:** BETA — contracts may evolve between minor versions  
**Default Rate Limit:** 60 RPM

Brain endpoints expose the Central Brain's recommendation and pattern-learning capabilities. The Brain is platform-owned — no product owns it. All products contribute experiences and receive recommendations through these endpoints.

---

## BR-001: Get Recommendations

**Method:** `POST`  
**Path:** `/api/v1/brain/recommend`

**Purpose:** Request recommendations from the Central Brain based on the current task context. The Brain applies its learned pattern library to suggest optimal approaches (agent type, tools, prompt template, execution strategy).

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `task_type` | string | * | The type of task being planned. |
| `context` | object | * | Task-specific context (language, domain, complexity, etc.). |
| `history_window` | integer | | Look back this many prior executions for similarity. Default: 50, max: 500. |
| `max_recommendations` | integer | | Max recommendations to return. Default: 5, max: 20. |

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `recommendations` | [Recommendation] | Ranked list, highest confidence first. |
| `pattern_basis` | string | Which pattern cluster these recommendations are based on. |
| `confidence_overall` | float | Overall confidence in these recommendations [0.0, 1.0]. |
| `similar_executions` | integer | Number of past executions the Brain matched against. |
| `computed_at` | ISO8601 | |

**Recommendation fields:**
- `rank` (integer)
- `type` (enum: AGENT_SELECTION, TOOL_SELECTION, PROMPT_TEMPLATE, STRATEGY, PARAMETER_TUNING)
- `value` (string — the recommendation itself, e.g., `"use developer_agent"`)
- `confidence` (float)
- `reasoning` (string — human-readable explanation)
- `based_on_n_executions` (integer)

**Authentication:** Required | **Authorization:** DEVELOPER+ | **Permission:** `brain:read`  
**Rate Limit:** 60 RPM | **Idempotency:** Not idempotent (context changes recommendations at each call)

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `INSUFFICIENT_DATA` | 422 | Brain has fewer than 10 experiences for this task type — recommendations unavailable |
| `CONTEXT_TOO_LARGE` | 400 | `context` object exceeds 50KB |

**Example Request:**
```
POST /api/v1/brain/recommend
Authorization: Bearer sk_live_dev_xxxx
Content-Type: application/json

{
  "product_id": "ai-software-factory",
  "task_type": "write_code",
  "context": {
    "language": "python",
    "domain": "REST API",
    "complexity": "medium",
    "test_coverage_required": true
  },
  "max_recommendations": 3
}
```

**Example Response:**
```json
{
  "data": {
    "recommendations": [
      {
        "rank": 1,
        "type": "AGENT_SELECTION",
        "value": "developer_agent",
        "confidence": 0.87,
        "reasoning": "developer_agent succeeds on python REST API tasks 91% of the time vs 72% for generic_agent",
        "based_on_n_executions": 234
      },
      {
        "rank": 2,
        "type": "PROMPT_TEMPLATE",
        "value": "template-id:a1b2c3d4",
        "confidence": 0.81,
        "reasoning": "python_developer_prompt has 23% higher first-pass success rate for this domain",
        "based_on_n_executions": 156
      }
    ],
    "pattern_basis": "python-rest-api-medium-complexity",
    "confidence_overall": 0.84,
    "similar_executions": 234,
    "computed_at": "2026-06-28T10:00:00Z"
  },
  "meta": { "request_id": "uuid", "timestamp": "2026-06-28T10:00:00Z", "version": "1.0" }
}
```

**Latency Target:** P99 < 300ms (Jaccard similarity on local index; Qdrant in v1.1)  
**Logging:** Log product_id, task_type, recommendations count. Never log context values.  
**Audit:** No | **Versioning:** v1 (BETA)

---

## BR-002: Record Experience

**Method:** `POST`  
**Path:** `/api/v1/brain/experiences`

**Purpose:** Record a task execution outcome as an experience for the Brain to learn from. Products call this after each completed task.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `task_type` | string | * | |
| `agent_type` | string | * | |
| `context` | object | * | Same context object passed to recommend, for correlation. |
| `outcome` | enum | * | `SUCCESS` \| `FAILURE` \| `PARTIAL` |
| `duration_ms` | integer | * | |
| `total_tokens` | integer | * | |
| `total_cost_usd` | float | * | |
| `retry_count` | integer | * | |
| `tools_used` | [string] | * | |
| `prompt_template_id` | UUID \| null | | If a Prompt OS template was used. |
| `workflow_id` | UUID \| null | | If part of a workflow. |
| `failure_reason` | string \| null | | Required if outcome=FAILURE or PARTIAL. |
| `notes` | string \| null | | Optional qualitative notes. Max 500 chars. |

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `experience_id` | UUID | |
| `recorded_at` | ISO8601 | |
| `pattern_update` | enum | `REINFORCED` \| `NEW_PATTERN` \| `ANOMALY_FLAGGED` \| `INSUFFICIENT_DATA` |

**Authentication:** Required | **Authorization:** DEVELOPER+ | **Permission:** `brain:write`  
**Rate Limit:** 300 RPM (products call this frequently) | **Idempotency:** Not idempotent

**Latency Target:** P99 < 100ms (async write — Brain processing is asynchronous) | **Audit:** No | **Versioning:** v1

---

## BR-003: Find Similar Executions

**Method:** `POST`  
**Path:** `/api/v1/brain/search`

**Purpose:** Find past task executions similar to the provided context. Useful for diagnosis and for products that want raw similarity results rather than processed recommendations.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `task_type` | string | * | |
| `context` | object | * | |
| `outcome_filter` | enum \| null | | Filter to `SUCCESS`, `FAILURE`, or `PARTIAL` only. |
| `limit` | integer | | Max 50. Default 10. |

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `results` | [SimilarExecution] | Ranked by similarity score. |
| `algorithm` | enum | `JACCARD` \| `QDRANT_COSINE` — indicates which backend was used. |

**SimilarExecution fields:** `experience_id`, `task_type`, `agent_type`, `outcome`, `similarity_score` (float), `duration_ms`, `tokens`, `cost_usd`, `recorded_at`

**Authentication:** Required | **Permission:** `brain:read` | **Rate Limit:** 60 RPM  
**Latency Target:** P99 < 500ms (Jaccard is O(n²) — will improve with Qdrant in v1.1)  
**Audit:** No | **Versioning:** v1 (BETA)

---

## BR-004: Get Patterns

**Method:** `GET`  
**Path:** `/api/v1/brain/patterns`

**Purpose:** Return the Brain's discovered pattern clusters — groups of similar execution contexts and their aggregate statistics.

**Query Params:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `task_type` | string | null | Filter to patterns for a specific task type. |
| `min_experiences` | integer | 10 | Only return patterns backed by at least this many experiences. |
| `limit` | integer | 20 | Max 100. |

**Response (200 OK):** Paginated list of Pattern objects.

**Pattern fields:** `pattern_id`, `name`, `task_types` (covered), `experience_count`, `avg_success_rate`, `avg_duration_ms`, `avg_cost_usd`, `top_agents` (ranked list), `top_tools` (ranked list), `discovered_at`, `last_updated_at`

**Authentication:** Required | **Permission:** `brain:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

---

## BR-005: Get Improvement Proposals

**Method:** `GET`  
**Path:** `/api/v1/brain/improvements`

**Purpose:** Return the Brain's active improvement proposals — specific, actionable suggestions derived from pattern analysis.

**Query Params:** `product_id` (filter), `status` (`PROPOSED` | `ACCEPTED` | `REJECTED`), `limit`, `cursor`

**Response (200 OK):** Paginated list of Improvement objects.

**Improvement fields:** `proposal_id`, `proposal_type` (enum: AGENT_SWAP, PROMPT_UPDATE, TOOL_ADDITION, TIMEOUT_ADJUSTMENT, RETRY_REDUCTION), `description`, `affected_task_type`, `estimated_improvement_pct`, `confidence`, `evidence_count`, `proposed_at`, `status`

**Authentication:** Required | **Permission:** `brain:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

---

---

# Group 6: Provider

**Base Path:** `/api/v1/providers`  
**NATS Stream:** AI  
**Default Rate Limit:** 60 RPM

Provider endpoints expose the AI provider registry and allow inspection of provider health and capabilities. Direct provider invocation is not available through these endpoints — all AI execution goes through AI ROS via WorkflowClient or AIClient.

---

## PV-001: List Providers

**Method:** `GET`  
**Path:** `/api/v1/providers`

**Purpose:** List all registered AI provider adapters and their current health.

**Query Params:** `health` (filter: `HEALTHY` | `DEGRADED` | `OFFLINE`), `model` (filter to providers that support a specific model ID)

**Response (200 OK):** List of ProviderSummary objects.

**ProviderSummary fields:** `provider_id`, `display_name`, `health`, `circuit_state` (CLOSED | OPEN | HALF_OPEN), `models` (list of model IDs), `latency_p50_ms`, `latency_p99_ms`, `error_rate_5m`, `supports_streaming`, `supports_function_calling`, `supports_vision`

**Authentication:** Required | **Authorization:** Any role | **Permission:** `provider:read`  
**Rate Limit:** 60 RPM | **Idempotency:** Idempotent | **Latency Target:** P99 < 100ms | **Audit:** No  
**Versioning:** v1

---

## PV-002: Get Provider

**Method:** `GET`  
**Path:** `/api/v1/providers/{provider_id}`

**Purpose:** Get full detail for a specific provider including model catalog and current performance metrics.

**Path Params:** `provider_id` (string, e.g., `"anthropic"`)

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | string | |
| `display_name` | string | |
| `health` | enum | |
| `circuit_state` | enum | |
| `circuit_open_until` | ISO8601 \| null | Non-null when circuit is OPEN. |
| `models` | [ModelDetail] | |
| `latency_p50_ms` | float | Last 5 minutes. |
| `latency_p99_ms` | float | |
| `error_rate_5m` | float | |
| `requests_today` | integer | |
| `cost_today_usd` | float | Platform-wide cost for this provider today. |
| `rate_limited_until` | ISO8601 \| null | Set if the provider is currently rate-limiting us. |

**ModelDetail fields:** `model_id`, `display_name`, `context_window_tokens`, `max_output_tokens`, `supports_streaming`, `supports_function_calling`, `supports_vision`, `input_cost_per_1k_tokens`, `output_cost_per_1k_tokens`

**Authentication:** Required | **Permission:** `provider:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 100ms | **Audit:** No | **Versioning:** v1

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `PROVIDER_NOT_FOUND` | 404 | |

---

## PV-003: Get Provider Capabilities

**Method:** `GET`  
**Path:** `/api/v1/providers/{provider_id}/capabilities`

**Purpose:** Return a structured capability matrix for the provider — what it can and cannot do.

**Response (200 OK):** `CapabilityMatrix` object with boolean flags for each capability dimension (streaming, function_calling, vision, json_mode, system_prompt, multi_turn, etc.) per model.

**Authentication:** Required | **Permission:** `provider:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 50ms | **Audit:** No | **Versioning:** v1

---

## PV-004: Estimate Execution Cost

**Method:** `POST`  
**Path:** `/api/v1/providers/{provider_id}/estimate`

**Purpose:** Estimate the cost of an AI execution before submitting it. Useful for budget planning and gate decisions.

**Path Params:** `provider_id` (string)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model_id` | string | * | |
| `estimated_prompt_tokens` | integer | * | |
| `estimated_completion_tokens` | integer | * | |
| `include_cache_savings` | boolean | | If true, returns estimate with and without prompt caching. Default false. |

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | string | |
| `model_id` | string | |
| `estimated_cost_usd` | float | |
| `cached_cost_usd` | float \| null | Cost with prompt caching (if `include_cache_savings=true`). |
| `cache_savings_usd` | float \| null | |
| `input_cost_per_1k` | float | Current rate. |
| `output_cost_per_1k` | float | |

**Authentication:** Required | **Permission:** `provider:read` | **Rate Limit:** 120 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 50ms | **Audit:** No | **Versioning:** v1

---

## PV-005: Trigger Provider Health Check

**Method:** `POST`  
**Path:** `/api/v1/providers/{provider_id}/health-check`

**Purpose:** Manually trigger a health check for a specific provider. Used by ADMIN operators when diagnosing provider issues.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `provider_id` | string | |
| `health` | enum | Current health after the check. |
| `latency_ms` | integer | Time the health check took. |
| `checked_at` | ISO8601 | |
| `error_detail` | string \| null | Populated if health is DEGRADED or OFFLINE. |

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:write`  
**Rate Limit:** 10 RPM | **Idempotency:** Not idempotent (triggers a real health check)  
**Latency Target:** P99 < 5000ms (bounded by provider response) | **Audit:** No | **Versioning:** v1

---

---

# Group 7: Knowledge

**Base Path:** `/api/v1/knowledge`  
**NATS Stream:** KNOWLEDGE  
**Stability Tier:** BETA  
**Default Rate Limit:** 60 RPM

Knowledge endpoints expose the Central Knowledge Graph. The current backend is SQL; migration to Kuzu and then Neo4j is planned. These endpoints are backend-agnostic.

---

## KN-001: Add Knowledge Node

**Method:** `POST`  
**Path:** `/api/v1/knowledge/nodes`

**Purpose:** Add a node to the knowledge graph.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `node_type` | string | * | e.g., `"File"`, `"Module"`, `"Concept"`. |
| `label` | string | * | Human-readable label. 1–500 chars. |
| `properties` | object | * | Key-value properties. Max 50 keys. Value max 1KB each. |
| `workspace_id` | string \| null | | If workspace-scoped. |

**Response (201 Created):** `{ "node_id": "...", "node_type": "...", "label": "...", "created_at": "..." }`

**Validation:** `node_type` must be a registered node type.

**Authentication:** Required | **Permission:** `knowledge:write` | **Rate Limit:** 60 RPM  
**Idempotency:** Not idempotent (creates a new node each call)  
**Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

---

## KN-002: Update Knowledge Node

**Method:** `PATCH`  
**Path:** `/api/v1/knowledge/nodes/{node_id}`

**Purpose:** Update properties on an existing knowledge graph node.

**Path Params:** `node_id` (string)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `properties` | object | * | Properties to set or update. Existing properties not mentioned are unchanged. |
| `remove_properties` | [string] | | Property keys to remove. |

**Response (200 OK):** `{ "node_id": "...", "updated_at": "...", "properties_updated": 3 }`

**Authentication:** Required | **Permission:** `knowledge:write` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `NODE_NOT_FOUND` | 404 | |

---

## KN-003: Add Relationship

**Method:** `POST`  
**Path:** `/api/v1/knowledge/relationships`

**Purpose:** Add a directional relationship between two knowledge graph nodes.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `relationship_type` | string | * | e.g., `"DEPENDS_ON"`, `"AUTHORED_BY"`. |
| `from_node_id` | string | * | |
| `to_node_id` | string | * | |
| `properties` | object | | Properties on the relationship. |

**Response (201 Created):** `{ "relationship_id": "...", "type": "...", "from": "...", "to": "...", "created_at": "..." }`

**Authentication:** Required | **Permission:** `knowledge:write` | **Rate Limit:** 60 RPM  
**Idempotency:** Not idempotent — multiple calls create multiple relationships  
**Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

---

## KN-004: Query Graph

**Method:** `POST`  
**Path:** `/api/v1/knowledge/query`

**Purpose:** Execute a structured graph query against the knowledge graph. This endpoint accepts a platform-defined query DSL — not raw graph query language.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `start_node_id` | string \| null | | Start traversal from this node. |
| `node_types` | [string] \| null | | Filter to these node types. |
| `relationship_types` | [string] \| null | | Traverse only these relationship types. |
| `direction` | enum | | `OUTBOUND` \| `INBOUND` \| `BOTH`. Default: `OUTBOUND`. |
| `max_depth` | integer | | Max traversal depth. Default: 3, max: 10. |
| `property_filters` | [PropertyFilter] | | Filter nodes by property values. |
| `limit` | integer | | Max nodes to return. Default: 50, max: 500. |

**PropertyFilter:** `{ "key": "language", "operator": "eq", "value": "python" }`

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `nodes` | [GraphNode] | Matching nodes. |
| `relationships` | [GraphRelationship] | Relationships between returned nodes. |
| `paths` | [GraphPath] | If `start_node_id` provided, the traversal paths. |
| `query_time_ms` | integer | |

**Authentication:** Required | **Permission:** `knowledge:read` | **Rate Limit:** 30 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 500ms | **Audit:** No | **Versioning:** v1

---

## KN-005: Search Nodes

**Method:** `POST`  
**Path:** `/api/v1/knowledge/search`

**Purpose:** Full-text search across knowledge graph node labels and properties.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `query` | string | * | Search terms. 1–500 chars. |
| `node_types` | [string] \| null | | Restrict to these node types. |
| `workspace_id` | string \| null | | Restrict to workspace-scoped nodes. |
| `limit` | integer | | Default: 20, max: 100. |

**Response (200 OK):** Ranked list of matching nodes with relevance scores.

**Authentication:** Required | **Permission:** `knowledge:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 300ms | **Audit:** No | **Versioning:** v1

---

## KN-006: Get Memory

**Method:** `GET`  
**Path:** `/api/v1/knowledge/memory/{key}`

**Purpose:** Retrieve a stored memory value by key.

**Path Params:** `key` (string — URL-encoded)

**Query Params:** `scope` (enum: `GLOBAL` | `WORKSPACE` | `PRODUCT` | `SESSION`), `scope_id` (required if scope != GLOBAL)

**Response (200 OK):** `{ "key": "...", "value": <any>, "scope": "...", "expires_at": "...", "set_at": "..." }`  
**Response (404):** `{ "error": { "code": "MEMORY_NOT_FOUND", ... } }`

**Authentication:** Required | **Permission:** `knowledge:read` | **Rate Limit:** 300 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 50ms | **Audit:** No | **Versioning:** v1

---

## KN-007: Set Memory

**Method:** `PUT`  
**Path:** `/api/v1/knowledge/memory/{key}`

**Purpose:** Store or update a memory value. If the key exists, it is overwritten.

**Path Params:** `key` (string)

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `value` | any | * | JSON-serializable value. Max 1MB. |
| `scope` | enum | * | `GLOBAL` \| `WORKSPACE` \| `PRODUCT` \| `SESSION` |
| `scope_id` | string | | Required if scope != GLOBAL. |
| `ttl_seconds` | integer \| null | | Expiry. Null = no expiry. |

**Response (200 OK):** `{ "key": "...", "scope": "...", "set_at": "...", "expires_at": "..." }`

**Validation:** Value must be <= 1MB when serialized. Key must match `^[a-zA-Z0-9:._-]{1,256}$`.

**Authentication:** Required | **Permission:** `knowledge:write` | **Rate Limit:** 300 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 100ms | **Audit:** No | **Versioning:** v1

---

---

# Group 8: Marketplace

**Base Path:** `/api/v1/marketplace`  
**Status:** PROPOSED — endpoints not yet active. Available in v2.5 (multi-tenant release).  
**Feature Flag:** `marketplace = false` — all endpoints return 404 when flag is off.

---

## MK-001: List Marketplace Listings (PROPOSED)

**Method:** `GET`  
**Path:** `/api/v1/marketplace/listings`

**Purpose:** Browse available marketplace listings (product templates, prompt packs, plugins).

**Query Params:** `category`, `price_type` (`FREE` | `PAID` | `SUBSCRIPTION`), `limit`, `cursor`

**Response (200 OK):** Paginated list of ListingSummary objects.

**Authentication:** Required | **Permission:** `workspace:read` | **Versioning:** v1 (PROPOSED)

---

## MK-002: Get Marketplace Listing (PROPOSED)

**Method:** `GET`  
**Path:** `/api/v1/marketplace/listings/{listing_id}`

**Response (200 OK):** Full listing detail including description, pricing, compatibility, reviews summary.

**Authentication:** Required | **Versioning:** v1 (PROPOSED)

---

## MK-003: Install Marketplace Listing (PROPOSED)

**Method:** `POST`  
**Path:** `/api/v1/marketplace/listings/{listing_id}/install`

**Request Body:** `{ "license_key": "string (for PAID)" }`

**Response (201 Created):** `{ "installation_id": "...", "listing_id": "...", "installed_at": "..." }`

**Authentication:** Required | **Authorization:** ADMIN | **Versioning:** v1 (PROPOSED)

---

## MK-004: List Installed Listings (PROPOSED)

**Method:** `GET`  
**Path:** `/api/v1/marketplace/installed`

**Response (200 OK):** List of installed marketplace items with installation metadata.

**Authentication:** Required | **Versioning:** v1 (PROPOSED)

---

---

# Group 9: Plugin

**Base Path:** `/api/v1/plugins`  
**NATS Stream:** PLUGINS  
**Default Rate Limit:** 30 RPM

Plugin endpoints manage the lifecycle of platform plugins. All plugins run in a sandboxed runtime with declared permissions. Unsigned plugins are rejected in staging and production.

---

## PL-001: Install Plugin

**Method:** `POST`  
**Path:** `/api/v1/plugins`

**Purpose:** Install a new plugin into the platform sandbox.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `plugin_id` | string | * | Unique ID. Must match `^[a-z][a-z0-9-]{0,63}$`. |
| `plugin_name` | string | * | Human-readable name. |
| `version` | string | * | SemVer. |
| `plugin_type` | enum | * | `WORKER` \| `TOOL` \| `PROMPT` \| `KNOWLEDGE` |
| `bundle_url` | string | * | URL to the plugin bundle (must be on the allowlist). |
| `bundle_hash` | string | * | SHA-256 of the plugin bundle for integrity verification. |
| `signature` | string | * | Plugin digital signature. Required for staging/production. |
| `declared_permissions` | [string] | * | Capabilities the plugin requests. |
| `configuration` | object | | Plugin-specific configuration (no secrets). |

**Response (201 Created):**

| Field | Type | Description |
|-------|------|-------------|
| `plugin_id` | string | |
| `status` | enum | `INSTALLING` — sandbox initialization is asynchronous. |
| `granted_permissions` | [string] | Admin may have restricted declared permissions. |
| `installation_id` | UUID | Track installation progress. |
| `integrity_verified` | boolean | Whether `bundle_hash` matched the downloaded bundle. |
| `signature_verified` | boolean | |

**Validation:**
- `bundle_hash` must match SHA-256 of the downloaded bundle
- `signature` must verify against the platform's trusted key store
- `plugin_id` must not already be installed
- `bundle_url` must be on the configured allowlist

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `plugin:write`  
**Rate Limit:** 10 RPM | **Idempotency:** Not idempotent

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `PLUGIN_ALREADY_INSTALLED` | 409 | `plugin_id` already installed |
| `SIGNATURE_VERIFICATION_FAILED` | 422 | Plugin signature invalid |
| `INTEGRITY_VERIFICATION_FAILED` | 422 | Bundle hash mismatch |
| `URL_NOT_ALLOWLISTED` | 403 | `bundle_url` not on the allowlist |
| `PERMISSION_NOT_AVAILABLE` | 422 | Requested permission doesn't exist on this platform |

**Latency Target:** P99 < 5000ms (download + verify + sandbox init) | **Audit:** Yes | **Versioning:** v1

---

## PL-002: List Plugins

**Method:** `GET`  
**Path:** `/api/v1/plugins`

**Purpose:** List all installed plugins and their current state.

**Query Params:** `plugin_type`, `status` (`ACTIVE` | `DISABLED` | `INSTALLING` | `FAILED`), `limit`, `cursor`

**Response (200 OK):** Paginated list of PluginSummary objects.

**PluginSummary fields:** `plugin_id`, `plugin_name`, `version`, `plugin_type`, `status`, `granted_permissions`, `installed_at`, `last_invoked_at`, `invocation_count`, `consecutive_health_failures`

**Authentication:** Required | **Permission:** `plugin:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 150ms | **Audit:** No | **Versioning:** v1

---

## PL-003: Get Plugin

**Method:** `GET`  
**Path:** `/api/v1/plugins/{plugin_id}`

**Purpose:** Get full detail for a specific installed plugin.

**Response (200 OK):** Full plugin detail including permissions, sandbox config (sanitized), invocation history (last 24h summary), health check history.

**Authentication:** Required | **Permission:** `plugin:read` | **Rate Limit:** 60 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 100ms | **Audit:** No | **Versioning:** v1

---

## PL-004: Uninstall Plugin

**Method:** `DELETE`  
**Path:** `/api/v1/plugins/{plugin_id}`

**Purpose:** Permanently remove a plugin from the platform. Any in-flight invocations complete first.

**Request Body:** `{ "reason": "string (required)" }`

**Response (200 OK):** `{ "plugin_id": "...", "uninstalled_at": "...", "final_invocation_count": 1420 }`

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `plugin:write`  
**Rate Limit:** 10 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 2000ms (drain in-flight) | **Audit:** Yes | **Versioning:** v1

---

## PL-005: Enable Plugin

**Method:** `POST`  
**Path:** `/api/v1/plugins/{plugin_id}/enable`

**Purpose:** Re-enable a disabled plugin.

**Response (200 OK):** `{ "plugin_id": "...", "status": "ACTIVE", "enabled_at": "..." }`

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `plugin:write`  
**Rate Limit:** 10 RPM | **Idempotency:** Idempotent | **Latency Target:** P99 < 300ms  
**Audit:** No | **Versioning:** v1

---

## PL-006: Disable Plugin

**Method:** `POST`  
**Path:** `/api/v1/plugins/{plugin_id}/disable`

**Purpose:** Disable a plugin without uninstalling it. No new invocations will be dispatched. In-flight invocations complete.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reason` | string | * | 1–500 chars. |
| `permanent` | boolean | | If true, signals intent not to re-enable. Default false. |

**Response (200 OK):** `{ "plugin_id": "...", "status": "DISABLED", "disabled_at": "..." }`

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `plugin:write`  
**Rate Limit:** 10 RPM | **Idempotency:** Idempotent | **Latency Target:** P99 < 300ms  
**Audit:** Yes | **Versioning:** v1

---

---

# Group 10: Admin

**Base Path:** `/api/v1/admin`  
**Default Rate Limit:** 30 RPM  
**All endpoints require:** ADMIN role

Admin endpoints provide platform operational controls — configuration management, circuit breaker resets, migration status, and platform-wide overrides.

---

## AD-001: Get Platform Configuration

**Method:** `GET`  
**Path:** `/api/v1/admin/config`

**Purpose:** Return the current platform configuration. Sensitive fields (secrets, keys) are never returned — only their metadata (presence, source, last changed).

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `config_version` | string | |
| `environment` | string | |
| `settings` | [ConfigEntry] | All non-secret config entries. |
| `secret_keys` | [SecretMetadata] | Metadata about secrets (presence, source). No values. |

**ConfigEntry:** `{ "key": "AISF_DAILY_BUDGET_USD", "value": "50.00", "source": "env", "last_changed": "..." }`

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:read`  
**Rate Limit:** 30 RPM | **Idempotency:** Idempotent | **Latency Target:** P99 < 100ms  
**Audit:** No | **Versioning:** v1

---

## AD-002: Update Platform Configuration

**Method:** `PATCH`  
**Path:** `/api/v1/admin/config`

**Purpose:** Update one or more non-secret platform configuration values at runtime. Changes take effect immediately without restart (for hot-reloadable config) or at next restart (for cold config).

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `updates` | [ConfigUpdate] | * | List of key-value pairs to update. Max 10 per call. |
| `change_reason` | string | * | Required. 1–500 chars. |
| `dry_run` | boolean | | If true, validates changes without applying. Default false. |

**ConfigUpdate:** `{ "key": "AISF_DAILY_BUDGET_USD", "value": "100.00" }`

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `applied` | [string] | Keys that were applied. |
| `deferred` | [string] | Keys that require restart (applied at next restart). |
| `dry_run` | boolean | |
| `change_id` | UUID | Reference ID for this change (in audit log). |

**Validation:**
- Key must be a known config key (unknown keys → 400)
- Value must pass the key's type validation
- Security-sensitive keys require a second ADMIN to confirm (returns 202 with `confirmation_required: true`)

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:write`  
**Rate Limit:** 10 RPM | **Idempotency:** Not idempotent

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `UNKNOWN_CONFIG_KEY` | 400 | |
| `INVALID_CONFIG_VALUE` | 400 | Value fails type or range validation |
| `CONFIRMATION_REQUIRED` | 202 | Security-sensitive change requires second ADMIN approval |

**Latency Target:** P99 < 500ms | **Audit:** Yes | **Versioning:** v1

---

## AD-003: List Feature Flags

**Method:** `GET`  
**Path:** `/api/v1/admin/feature-flags`

**Response (200 OK):** List of all feature flags with current state, owner, expiry date, and last changed info.

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:read`  
**Rate Limit:** 30 RPM | **Latency Target:** P99 < 100ms | **Audit:** No | **Versioning:** v1

---

## AD-004: Toggle Feature Flag

**Method:** `POST`  
**Path:** `/api/v1/admin/feature-flags/{flag_name}`

**Purpose:** Enable or disable a feature flag.

**Request Body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `enabled` | boolean | * | New state. |
| `reason` | string | * | 1–500 chars. |

**Response (200 OK):** `{ "flag_name": "...", "enabled": true, "previous": false, "changed_at": "..." }`

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:write`  
**Rate Limit:** 10 RPM | **Audit:** Yes | **Versioning:** v1

**Error Codes:**

| Code | HTTP | Condition |
|------|------|-----------|
| `FLAG_NOT_FOUND` | 404 | |

---

## AD-005: Reset Circuit Breaker

**Method:** `POST`  
**Path:** `/api/v1/admin/providers/{provider_id}/circuit-reset`

**Purpose:** Manually reset a provider's circuit breaker to CLOSED state. Useful when automatic recovery is too slow during an incident.

**Request Body:** `{ "reason": "string (required)" }`

**Response (200 OK):** `{ "provider_id": "...", "circuit_state": "CLOSED", "reset_at": "...", "was_state": "OPEN" }`

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:write`  
**Rate Limit:** 10 RPM | **Idempotency:** Idempotent — resetting an already-CLOSED circuit returns 200  
**Latency Target:** P99 < 300ms | **Audit:** Yes | **Versioning:** v1

---

## AD-006: Get Migration Status

**Method:** `GET`  
**Path:** `/api/v1/admin/migrations`

**Purpose:** Return the history and current state of database schema migrations.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `current_schema_version` | string | |
| `migrations` | [MigrationEntry] | All migrations, latest first. |
| `pending_count` | integer | Migrations that would run on next startup. |
| `last_applied_at` | ISO8601 \| null | |

**MigrationEntry fields:** `migration_id`, `direction`, `applied_at`, `duration_ms`, `status` (SUCCESS | FAILED), `schema_version_before`, `schema_version_after`

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:read`  
**Rate Limit:** 10 RPM | **Latency Target:** P99 < 200ms | **Audit:** No | **Versioning:** v1

---

## AD-007: Get Audit Log (Admin)

**Method:** `GET`  
**Path:** `/api/v1/admin/audit-log`

**Purpose:** Full audit log access with advanced filtering. This is the ADMIN-level version of AUTH-006 with extended filter options.

**Query Params:** `start`, `end`, `event_type`, `identity`, `severity` (`LOW` | `MEDIUM` | `HIGH` | `CRITICAL`), `product_id`, `limit` (max 500), `cursor`

**Response (200 OK):** Paginated list of audit entries (same schema as AUTH-006 but with additional fields: `severity`, `product_id`, `changes` for config-change events).

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:read`  
**Rate Limit:** 10 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 1000ms (large range scans) | **Audit:** Yes (audit log access is itself audited) | **Versioning:** v1

---


---

# Group 11: Monitoring

**Base Path:** `/api/v1/monitoring` (and unversioned `/metrics`)  
**Default Rate Limit:** 60 RPM

Monitoring endpoints provide operational visibility into platform health, SLA performance, active alerts, and usage statistics.

---

## MO-001: Get Prometheus Metrics

**Method:** `GET`  
**Path:** `/metrics`

**Purpose:** Expose platform metrics in Prometheus text exposition format for scraping. This is a **public endpoint** — no authentication required. It intentionally exposes no sensitive data (no user data, no cost details, no identity information).

**Response (200 OK):** Prometheus text format. Content-Type: `text/plain; version=0.0.4`.

**Metrics exposed:**
- `aistudio_ai_executions_total{product,provider,model,outcome}` — Counter
- `aistudio_ai_execution_duration_ms{product,provider,model,quantile}` — Summary
- `aistudio_ai_queue_depth{priority}` — Gauge
- `aistudio_workflow_active_total{product}` — Gauge
- `aistudio_workflow_completed_total{product,outcome}` — Counter
- `aistudio_prompt_renders_total{template_id,outcome}` — Counter
- `aistudio_provider_circuit_state{provider}` — Gauge (0=CLOSED, 1=HALF_OPEN, 2=OPEN)
- `aistudio_platform_uptime_seconds` — Counter
- `aistudio_nats_pending_ack_total` — Gauge
- `aistudio_db_pool_active` — Gauge
- `aistudio_websocket_sessions_active` — Gauge
- `aistudio_http_requests_total{method,path,status}` — Counter
- `aistudio_http_request_duration_ms{method,path,quantile}` — Summary

**Authentication:** NOT REQUIRED | **Rate Limit:** 300 RPM (Prometheus scrape)  
**Idempotency:** Idempotent | **Latency Target:** P99 < 50ms | **Audit:** No | **Versioning:** Unversioned

---

## MO-002: List Active Alerts

**Method:** `GET`  
**Path:** `/api/v1/monitoring/alerts`

**Purpose:** List currently firing alerts on the platform.

**Query Params:** `severity` (`INFO` | `WARNING` | `HIGH` | `CRITICAL`), `module_id`, `limit`, `cursor`

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `items` | [AlertSummary] | Currently firing alerts. |
| `total` | integer | |
| `critical_count` | integer | Alerts at CRITICAL severity. |
| `high_count` | integer | |

**AlertSummary fields:** `alert_id`, `alert_name`, `severity`, `summary`, `source_module`, `fired_at`, `duration_seconds`, `runbook_url`

**Authentication:** Required | **Authorization:** Any role | **Permission:** `monitoring:read`  
**Rate Limit:** 60 RPM | **Idempotency:** Idempotent | **Latency Target:** P99 < 100ms  
**Audit:** No | **Versioning:** v1

---

## MO-003: Get Alert

**Method:** `GET`  
**Path:** `/api/v1/monitoring/alerts/{alert_id}`

**Purpose:** Get full detail for a specific alert including its history and the triggering event.

**Response (200 OK):** Full AlertDetail including `alert_id`, `alert_name`, `severity`, `summary`, `labels`, `fired_at`, `source_event_type`, `source_event_id`, `runbook_url`, `resolution_notes` (if resolved), `history` (previous firings of the same alert in last 24h).

**Authentication:** Required | **Permission:** `monitoring:read` | **Rate Limit:** 60 RPM  
**Latency Target:** P99 < 100ms | **Audit:** No | **Versioning:** v1

---

## MO-004: Get SLA Report

**Method:** `GET`  
**Path:** `/api/v1/monitoring/sla`

**Purpose:** Return SLA compliance report for a specified time period.

**Query Params:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `start` | ISO8601 | 24h ago | |
| `end` | ISO8601 | now | Max range: 90 days. |
| `product_id` | string | null | Filter to a specific product. |
| `sla_type` | enum | null | `WORKFLOW_COMPLETION` \| `TASK_COMPLETION` \| `AI_EXECUTION` \| `GATE_RESPONSE` |

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `period_start` | ISO8601 | |
| `period_end` | ISO8601 | |
| `overall_sla_met_pct` | float | Percentage of items that met SLA. |
| `by_type` | object | SLA metrics broken down by SLA type. |
| `breaches` | [SLABreach] | All individual SLA breaches in the period. |
| `total_measured` | integer | Total items measured against SLA. |
| `total_breaches` | integer | |

**SLABreach fields:** `entity_id`, `sla_type`, `sla_target_seconds`, `actual_seconds`, `breach_by_seconds`, `product_id`, `occurred_at`

**Authentication:** Required | **Permission:** `monitoring:read` | **Rate Limit:** 30 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 1000ms (range scan) | **Audit:** No | **Versioning:** v1

---

## MO-005: Get Usage Statistics

**Method:** `GET`  
**Path:** `/api/v1/monitoring/usage`

**Purpose:** Return aggregated usage statistics for platform operators.

**Query Params:**

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `period` | enum | `day` | `hour` \| `day` \| `week` \| `month` |
| `start` | ISO8601 | null | Default: 1 period ago. |
| `product_id` | string | null | Filter to a product. |

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `period` | string | |
| `total_ai_executions` | integer | |
| `total_tokens` | integer | |
| `total_cost_usd` | float | |
| `total_workflows` | integer | |
| `success_rate` | float | Execution success rate. |
| `avg_execution_latency_ms` | float | |
| `by_product` | object | Per-product breakdown. |
| `by_provider` | object | Per-provider breakdown. |
| `by_model` | object | Per-model breakdown. |

**Authentication:** Required | **Permission:** `monitoring:read` | **Rate Limit:** 30 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 500ms | **Audit:** No | **Versioning:** v1

---

## MO-006: Get Platform Metrics Snapshot

**Method:** `GET`  
**Path:** `/api/v1/monitoring/snapshot`

**Purpose:** Return the latest real-time platform metrics snapshot. This is the same data published as MO-005 events but available via pull.

**Response (200 OK):** Same schema as `monitoring.platform.metrics_snapshot` event payload (see EVENT-CATALOG). Includes `snapshot_at`, `ai_executions_in_flight`, `ai_queue_depth`, `workflow_active_count`, etc.

**Authentication:** Required | **Permission:** `monitoring:read` | **Rate Limit:** 300 RPM  
**Idempotency:** Idempotent | **Latency Target:** P99 < 30ms (in-memory) | **Audit:** No | **Versioning:** v1

---

---

# Group 12: Desktop

**Base Path:** `/api/v1/desktop`  
**Default Rate Limit:** 60 RPM

Desktop endpoints support the PySide6 desktop application. These endpoints are consumed exclusively by the desktop client — not by products or external integrations.

---

## DK-001: Get Desktop Session Info

**Method:** `GET`  
**Path:** `/api/v1/desktop/session`

**Purpose:** Return the initial session context the desktop app needs on startup — workspace state, active workflows, alerts summary, and current cost.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `session_id` | UUID | Assigned session ID for WebSocket authentication. |
| `identity` | string | Hashed identity (for display). |
| `role` | enum | Caller's role. |
| `workspace` | object | Same as WS-001 workspace summary. |
| `active_workflows` | [WorkflowSummary] | Active workflows visible to this identity. |
| `active_alerts` | [AlertSummary] | Currently firing alerts. |
| `daily_cost_usd` | float | Today's total spend. |
| `daily_budget_usd` | float | Today's budget limit. |
| `platform_version` | string | |

**Authentication:** Required | **Permission:** `workspace:read` + `workflow:read` + `monitoring:read`  
**Rate Limit:** 10 RPM (startup call, not frequently polled) | **Idempotency:** Idempotent  
**Latency Target:** P99 < 300ms (aggregates multiple data sources) | **Audit:** No | **Versioning:** v1

---

## DK-002: WebSocket Connection

**Method:** `GET` (WebSocket Upgrade)  
**Path:** `/api/v1/desktop/ws`

**Purpose:** Establish a persistent WebSocket connection for real-time desktop UI updates. The desktop app connects once and receives push events as platform events occur — no polling required.

**Upgrade Headers Required:**
```
Authorization: Bearer <api_key>
X-Session-Id: <session_id from DK-001>
Connection: Upgrade
Upgrade: websocket
```

**Message Protocol:** All messages are JSON. Server-to-client only (the client does not send messages after handshake).

**Message Format:**
```json
{
  "type": "event",
  "subject": "desktop.ui.workflow_panel_update",
  "payload": { ... }
}
```

**Subscribed Events (server pushes these):**
- All `desktop.ui.*` events
- All `desktop.session.*` events
- `monitoring.alert.fired` (CRITICAL and HIGH only)
- `workspace.health.*`
- `ai.budget.limit_exceeded` (for the authenticated identity's products)

**Reconnection:** Client must reconnect with exponential backoff on disconnect. The session remains valid for 1 hour after disconnect; reconnecting with the same `X-Session-Id` restores subscriptions.

**Authentication:** Required (upgrade is rejected if key is invalid)  
**Rate Limit:** 5 concurrent connections per identity | **Latency Target:** < 100ms for event delivery  
**Audit:** No | **Versioning:** v1

---

## DK-003: Get Workspace Snapshot

**Method:** `GET`  
**Path:** `/api/v1/desktop/workspace-snapshot`

**Purpose:** Return a complete denormalized snapshot of all state needed to render the full desktop UI. Called once on startup or on reconnect to rebuild UI state.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `workspace` | object | Full workspace state (WS-001). |
| `active_workflows` | [WorkflowStatus] | Full status for each active workflow. |
| `recent_workflows` | [WorkflowSummary] | Last 20 completed/failed/cancelled workflows. |
| `active_alerts` | [AlertSummary] | |
| `daily_cost_usd` | float | |
| `daily_budget_usd` | float | |
| `plugins` | [PluginSummary] | |
| `prompt_templates_active` | integer | Count of ACTIVE templates. |
| `brain_patterns_count` | integer | |
| `snapshot_at` | ISO8601 | |

**Authentication:** Required | **Permission:** All read permissions  
**Rate Limit:** 10 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 500ms | **Audit:** No | **Versioning:** v1

---

---

# Group 13: Configuration

**Base Path:** `/api/v1/config`  
**Default Rate Limit:** 60 RPM

Configuration endpoints expose non-sensitive platform configuration to authenticated callers. These complement the Admin configuration endpoints — Config endpoints are read-mostly and accessible to all roles; Admin endpoints allow modification and require ADMIN role.

---

## CF-001: Get Public Configuration

**Method:** `GET`  
**Path:** `/api/v1/config`

**Purpose:** Return platform configuration values that are safe for any authenticated caller to read. Sensitive values (budget limits, security thresholds, secrets) are excluded.

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `platform_version` | string | |
| `environment` | string | |
| `ai_ros` | object | AI ROS public config: queue priorities, default timeouts. |
| `workflow` | object | Workflow defaults: max tasks, default timeout, retry defaults. |
| `prompt_os` | object | Prompt OS settings: template cache TTL, max variable count. |
| `rate_limits` | object | Public rate limit values (what callers are subject to). |
| `feature_flags` | object | Non-security feature flags (e.g., `marketplace`, `brain`). |

**Authentication:** Required | **Authorization:** Any role | **Permission:** `workspace:read`  
**Rate Limit:** 60 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 50ms (cached; TTL 60s) | **Audit:** No | **Versioning:** v1

---

## CF-002: Get Configuration Schema

**Method:** `GET`  
**Path:** `/api/v1/config/schema`

**Purpose:** Return the JSON Schema for the platform configuration. Used by tooling and the Admin UI to validate configuration changes before submitting.

**Response (200 OK):** JSON Schema object describing all configurable fields, their types, defaults, and constraints.

**Authentication:** Required | **Authorization:** ADMIN | **Permission:** `admin:read`  
**Rate Limit:** 10 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 50ms | **Audit:** No | **Versioning:** v1

---

## CF-003: Get Feature Flags (Non-Admin)

**Method:** `GET`  
**Path:** `/api/v1/config/feature-flags`

**Purpose:** Return the current state of all non-security feature flags. Security-related flags are excluded (use admin endpoint for those).

**Response (200 OK):**

| Field | Type | Description |
|-------|------|-------------|
| `flags` | object | Map of flag name → `{ "enabled": bool, "description": "string" }` |
| `as_of` | ISO8601 | When these values were last evaluated. |

**Authentication:** Required | **Authorization:** Any role | **Permission:** `workspace:read`  
**Rate Limit:** 60 RPM | **Idempotency:** Idempotent  
**Latency Target:** P99 < 50ms | **Audit:** No | **Versioning:** v1

---

---

# Appendix A: Endpoint Quick Reference

## A.1 All Endpoints by Group

| ID | Method | Path | Auth | Permission | Rate Limit | Audit |
|----|--------|------|------|------------|------------|-------|
| AUTH-001 | POST | `/api/v1/auth/keys` | Required | `auth:write` | 10 RPM | Yes |
| AUTH-002 | GET | `/api/v1/auth/keys` | Required | `auth:read` | 60 RPM | No |
| AUTH-003 | GET | `/api/v1/auth/keys/{key_id}` | Required | `auth:read` | 60 RPM | No |
| AUTH-004 | DELETE | `/api/v1/auth/keys/{key_id}` | Required | `auth:write` | 10 RPM | Yes |
| AUTH-005 | GET | `/api/v1/auth/rate-limits` | Required | `auth:read` | 120 RPM | No |
| AUTH-006 | GET | `/api/v1/auth/audit-log` | Required | `admin:read` | 30 RPM | Yes |
| WS-001 | GET | `/api/v1/workspace` | Required | `workspace:read` | 120 RPM | No |
| WS-002 | GET | `/api/v1/workspace/capabilities` | **Public** | — | 600 RPM | No |
| WS-003 | GET | `/api/v1/workspace/products` | Required | `workspace:read` | 60 RPM | No |
| WS-004 | GET | `/api/v1/workspace/products/{id}` | Required | `workspace:read` | 60 RPM | No |
| WS-005 | GET | `/health` | **Public** | — | 600 RPM | No |
| WS-006 | GET | `/health/live` | **Public** | — | Unlimited | No |
| WS-007 | GET | `/health/ready` | **Public** | — | Unlimited | No |
| WS-008 | POST | `/api/v1/workspace/products/{id}/register` | Required | `admin:write` | 10 RPM | No |
| WS-009 | DELETE | `/api/v1/workspace/products/{id}` | Required | `admin:write` | 10 RPM | No |
| WF-001 | POST | `/api/v1/workflows` | Required | `workflow:write` | 30 RPM | No |
| WF-002 | GET | `/api/v1/workflows` | Required | `workflow:read` | 60 RPM | No |
| WF-003 | GET | `/api/v1/workflows/{id}` | Required | `workflow:read` | 120 RPM | No |
| WF-004 | GET | `/api/v1/workflows/{id}/tasks/{tid}/output` | Required | `workflow:read` | 120 RPM | No |
| WF-005 | POST | `/api/v1/workflows/{id}/cancel` | Required | `workflow:write` | 30 RPM | Yes |
| WF-006 | POST | `/api/v1/workflows/{id}/pause` | Required | `workflow:write` | 30 RPM | No |
| WF-007 | POST | `/api/v1/workflows/{id}/resume` | Required | `workflow:write` | 30 RPM | No |
| WF-008 | GET | `/api/v1/workflows/{id}/checkpoint` | Required | `workflow:read` | 60 RPM | No |
| WF-009 | POST | `/api/v1/workflows/{id}/gates/{gid}/approve` | Required | `workflow:approve` | 30 RPM | Yes |
| WF-010 | POST | `/api/v1/workflows/{id}/gates/{gid}/reject` | Required | `workflow:approve` | 30 RPM | Yes |
| PR-001 | POST | `/api/v1/prompts/render` | Required | `prompt:render` | 600 RPM | No |
| PR-002 | POST | `/api/v1/prompts/compose` | Required | `prompt:render` | 120 RPM | No |
| PR-003 | POST | `/api/v1/prompts/templates` | Required | `prompt:write` | 30 RPM | No |
| PR-004 | PUT | `/api/v1/prompts/templates/{id}/versions` | Required | `prompt:write` | 30 RPM | No |
| PR-005 | POST | `/api/v1/prompts/templates/{id}/transition` | Required | `prompt:govern` | 30 RPM | Yes |
| PR-006 | GET | `/api/v1/prompts/templates` | Required | `prompt:read` | 60 RPM | No |
| PR-007 | GET | `/api/v1/prompts/templates/{id}` | Required | `prompt:read` | 120 RPM | No |
| PR-008 | GET | `/api/v1/prompts/templates/{id}/diff` | Required | `prompt:read` | 60 RPM | No |
| PR-009 | POST | `/api/v1/prompts/ab-tests/{id}/results` | Required | `prompt:render` | 300 RPM | No |
| BR-001 | POST | `/api/v1/brain/recommend` | Required | `brain:read` | 60 RPM | No |
| BR-002 | POST | `/api/v1/brain/experiences` | Required | `brain:write` | 300 RPM | No |
| BR-003 | POST | `/api/v1/brain/search` | Required | `brain:read` | 60 RPM | No |
| BR-004 | GET | `/api/v1/brain/patterns` | Required | `brain:read` | 60 RPM | No |
| BR-005 | GET | `/api/v1/brain/improvements` | Required | `brain:read` | 60 RPM | No |
| PV-001 | GET | `/api/v1/providers` | Required | `provider:read` | 60 RPM | No |
| PV-002 | GET | `/api/v1/providers/{id}` | Required | `provider:read` | 60 RPM | No |
| PV-003 | GET | `/api/v1/providers/{id}/capabilities` | Required | `provider:read` | 60 RPM | No |
| PV-004 | POST | `/api/v1/providers/{id}/estimate` | Required | `provider:read` | 120 RPM | No |
| PV-005 | POST | `/api/v1/providers/{id}/health-check` | Required | `admin:write` | 10 RPM | No |
| KN-001 | POST | `/api/v1/knowledge/nodes` | Required | `knowledge:write` | 60 RPM | No |
| KN-002 | PATCH | `/api/v1/knowledge/nodes/{id}` | Required | `knowledge:write` | 60 RPM | No |
| KN-003 | POST | `/api/v1/knowledge/relationships` | Required | `knowledge:write` | 60 RPM | No |
| KN-004 | POST | `/api/v1/knowledge/query` | Required | `knowledge:read` | 30 RPM | No |
| KN-005 | POST | `/api/v1/knowledge/search` | Required | `knowledge:read` | 60 RPM | No |
| KN-006 | GET | `/api/v1/knowledge/memory/{key}` | Required | `knowledge:read` | 300 RPM | No |
| KN-007 | PUT | `/api/v1/knowledge/memory/{key}` | Required | `knowledge:write` | 300 RPM | No |
| MK-001 | GET | `/api/v1/marketplace/listings` | Required | `workspace:read` | 60 RPM | No |
| MK-002 | GET | `/api/v1/marketplace/listings/{id}` | Required | `workspace:read` | 60 RPM | No |
| MK-003 | POST | `/api/v1/marketplace/listings/{id}/install` | Required | `admin:write` | 10 RPM | No |
| MK-004 | GET | `/api/v1/marketplace/installed` | Required | `workspace:read` | 60 RPM | No |
| PL-001 | POST | `/api/v1/plugins` | Required | `plugin:write` | 10 RPM | Yes |
| PL-002 | GET | `/api/v1/plugins` | Required | `plugin:read` | 60 RPM | No |
| PL-003 | GET | `/api/v1/plugins/{id}` | Required | `plugin:read` | 60 RPM | No |
| PL-004 | DELETE | `/api/v1/plugins/{id}` | Required | `plugin:write` | 10 RPM | Yes |
| PL-005 | POST | `/api/v1/plugins/{id}/enable` | Required | `plugin:write` | 10 RPM | No |
| PL-006 | POST | `/api/v1/plugins/{id}/disable` | Required | `plugin:write` | 10 RPM | Yes |
| AD-001 | GET | `/api/v1/admin/config` | Required | `admin:read` | 30 RPM | No |
| AD-002 | PATCH | `/api/v1/admin/config` | Required | `admin:write` | 10 RPM | Yes |
| AD-003 | GET | `/api/v1/admin/feature-flags` | Required | `admin:read` | 30 RPM | No |
| AD-004 | POST | `/api/v1/admin/feature-flags/{flag}` | Required | `admin:write` | 10 RPM | Yes |
| AD-005 | POST | `/api/v1/admin/providers/{id}/circuit-reset` | Required | `admin:write` | 10 RPM | Yes |
| AD-006 | GET | `/api/v1/admin/migrations` | Required | `admin:read` | 10 RPM | No |
| AD-007 | GET | `/api/v1/admin/audit-log` | Required | `admin:read` | 10 RPM | Yes |
| MO-001 | GET | `/metrics` | **Public** | — | 300 RPM | No |
| MO-002 | GET | `/api/v1/monitoring/alerts` | Required | `monitoring:read` | 60 RPM | No |
| MO-003 | GET | `/api/v1/monitoring/alerts/{id}` | Required | `monitoring:read` | 60 RPM | No |
| MO-004 | GET | `/api/v1/monitoring/sla` | Required | `monitoring:read` | 30 RPM | No |
| MO-005 | GET | `/api/v1/monitoring/usage` | Required | `monitoring:read` | 30 RPM | No |
| MO-006 | GET | `/api/v1/monitoring/snapshot` | Required | `monitoring:read` | 300 RPM | No |
| DK-001 | GET | `/api/v1/desktop/session` | Required | `workspace:read` | 10 RPM | No |
| DK-002 | WS | `/api/v1/desktop/ws` | Required | `workspace:read` | 5 conns | No |
| DK-003 | GET | `/api/v1/desktop/workspace-snapshot` | Required | All read | 10 RPM | No |
| CF-001 | GET | `/api/v1/config` | Required | `workspace:read` | 60 RPM | No |
| CF-002 | GET | `/api/v1/config/schema` | Required | `admin:read` | 10 RPM | No |
| CF-003 | GET | `/api/v1/config/feature-flags` | Required | `workspace:read` | 60 RPM | No |

**Total endpoints: 76 (72 ACTIVE + 4 PROPOSED)**

---

## A.2 Public Endpoints (No Authentication Required)

```
GET  /health
GET  /health/live
GET  /health/ready
GET  /metrics
GET  /api/v1/workspace/capabilities
```

---

## A.3 Endpoints That Are Always Audited

```
POST   /api/v1/auth/keys              (AUTH-001) — key creation
DELETE /api/v1/auth/keys/{id}         (AUTH-004) — key revocation
GET    /api/v1/auth/audit-log         (AUTH-006) — audit log access
POST   /api/v1/workflows/{id}/cancel  (WF-005)
POST   /api/v1/workflows/{id}/gates/{id}/approve  (WF-009)
POST   /api/v1/workflows/{id}/gates/{id}/reject   (WF-010)
POST   /api/v1/prompts/templates/{id}/transition  (PR-005)
POST   /api/v1/plugins                (PL-001)   — plugin install
DELETE /api/v1/plugins/{id}           (PL-004)   — plugin uninstall
POST   /api/v1/plugins/{id}/disable   (PL-006)
PATCH  /api/v1/admin/config           (AD-002)
POST   /api/v1/admin/feature-flags/{flag} (AD-004)
POST   /api/v1/admin/providers/{id}/circuit-reset (AD-005)
GET    /api/v1/admin/audit-log        (AD-007)   — audit log access
```

---

## A.4 Latency Targets Summary

| Category | Endpoint | P99 Target |
|----------|----------|------------|
| Health probes | `/health/live`, `/health/ready` | < 10ms |
| Metrics | `/metrics` | < 50ms |
| Public capabilities | `/api/v1/workspace/capabilities` | < 50ms |
| Prompt render (HOT PATH) | `PR-001` | < 50ms |
| Rate limit status | `AUTH-005` | < 50ms |
| Config reads | `CF-001`, `CF-003` | < 50ms |
| Memory get | `KN-006` | < 50ms |
| Cost estimate | `PV-004` | < 50ms |
| Workspace health | `/health` | < 30ms |
| Platform snapshot | `MO-006` | < 30ms |
| Auth operations | `AUTH-001–004` | < 200ms |
| Workflow status | `WF-003` | < 150ms |
| List operations | Most GET list endpoints | < 150ms |
| Brain recommendations | `BR-001` | < 300ms |
| Workflow submit | `WF-001` | < 500ms |
| Desktop session | `DK-001`, `DK-003` | < 300–500ms |
| SLA report | `MO-004` | < 1000ms |
| Admin audit log | `AD-007` | < 1000ms |
| Plugin install | `PL-001` | < 5000ms |
| Provider health check | `PV-005` | < 5000ms |

---

## A.5 Idempotency Reference

**Safe (Idempotent) — call as many times as needed:**
All `GET` endpoints; `WF-005 cancel`; `WF-006 pause`; `WF-007 resume`; `WF-009 approve`; `WF-010 reject`; `AUTH-004 revoke`; `PL-005 enable`; `PL-006 disable`; `KN-007 set memory`; `AD-005 circuit-reset`; `WS-008 register product`; `WS-009 deregister product`

**Not idempotent — repeated calls have different effects:**
`WF-001 submit` (without idempotency_key); `PR-001 render` (new render_id each call); `PR-002 compose`; `PR-003 register template`; `PR-004 update template` (creates new version); `BR-002 record experience`; `BR-001 recommend`; `KN-001 add node`; `KN-003 add relationship`; `PL-001 install`; `PL-004 uninstall`; `AUTH-001 create key`; `AD-002 update config`

**Idempotent via idempotency_key:**
`WF-001 submit` — supply `idempotency_key` in request body; duplicate returns existing workflow with 200

---

## A.6 Error Code Master List

| Code | HTTP | Meaning |
|------|------|---------|
| `ROLE_ESCALATION_FORBIDDEN` | 403 | Cannot create a key with higher role than your own |
| `ADMIN_ONLY` | 403 | Operation requires ADMIN role |
| `LAST_ADMIN_KEY` | 409 | Cannot revoke the last ADMIN key |
| `REASON_REQUIRED` | 400 | `reason` or `rejection_reason` is empty or missing |
| `RANGE_TOO_LARGE` | 400 | Date range exceeds maximum allowed window |
| `INVALID_DATE` | 400 | Date field is not valid ISO8601 |
| `PRODUCT_NOT_FOUND` | 404 | Product not registered in workspace |
| `WORKFLOW_NOT_FOUND` | 404 | Workflow does not exist |
| `TASK_NOT_FOUND` | 404 | Task does not exist in the specified workflow |
| `TASK_NOT_COMPLETED` | 409 | Task is not yet in COMPLETED state |
| `WORKFLOW_TERMINAL` | 409 | Workflow is already COMPLETED, FAILED, or CANCELLED |
| `WORKFLOW_NOT_PAUSED` | 409 | Workflow must be PAUSED for this operation |
| `WORKFLOW_NOT_PAUSABLE` | 409 | Workflow state does not allow pausing |
| `AUTO_APPROVE_FORBIDDEN` | 400 | `auto_approve_after_seconds` is not permitted |
| `DAG_CYCLE_DETECTED` | 422 | Task dependency graph contains a cycle |
| `UNKNOWN_TASK_TYPE` | 422 | Task type not registered |
| `UNKNOWN_AGENT_TYPE` | 422 | Agent type not registered |
| `TASK_LIMIT_EXCEEDED` | 422 | More than 100 tasks in a single plan |
| `TOOL_NOT_PERMITTED` | 403 | Product not authorized for a requested tool |
| `BUDGET_EXCEEDED` | 402 | Product's daily AI budget is fully consumed |
| `GATE_NOT_PENDING` | 409 | Gate is not in PENDING state |
| `INSUFFICIENT_ROLE` | 403 | Caller's role is below the gate's required_role |
| `TEMPLATE_NOT_FOUND` | 404 | Prompt template does not exist |
| `TEMPLATE_NOT_ACTIVE` | 409 | Template is not in ACTIVE governance state |
| `TEMPLATE_SUSPENDED` | 403 | Template is SUSPENDED — renders blocked |
| `MISSING_VARIABLES` | 422 | Required template variables not provided |
| `UNKNOWN_VARIABLES` | 422 | Extra variables provided not declared in template |
| `HMAC_VERIFICATION_FAILED` | 422 | Template content HMAC mismatch |
| `INVALID_TRANSITION` | 409 | Governance state transition not allowed |
| `VERSION_NOT_FOUND` | 404 | Specified template version does not exist |
| `NOTES_REQUIRED` | 400 | Notes required for DEPRECATED or SUSPENDED transitions |
| `REMOVAL_DATE_REQUIRED` | 400 | `removal_after` required for DEPRECATED |
| `INSUFFICIENT_DATA` | 422 | Brain has insufficient experience data for recommendations |
| `CONTEXT_TOO_LARGE` | 400 | Context object exceeds size limit |
| `PROVIDER_NOT_FOUND` | 404 | Provider ID not in registry |
| `NODE_NOT_FOUND` | 404 | Knowledge graph node does not exist |
| `PLUGIN_ALREADY_INSTALLED` | 409 | Plugin ID already installed |
| `SIGNATURE_VERIFICATION_FAILED` | 422 | Plugin signature invalid |
| `INTEGRITY_VERIFICATION_FAILED` | 422 | Plugin bundle hash mismatch |
| `URL_NOT_ALLOWLISTED` | 403 | Plugin bundle URL not on allowlist |
| `PERMISSION_NOT_AVAILABLE` | 422 | Requested plugin permission doesn't exist |
| `UNKNOWN_CONFIG_KEY` | 400 | Configuration key not recognized |
| `INVALID_CONFIG_VALUE` | 400 | Config value fails validation |
| `CONFIRMATION_REQUIRED` | 202 | Security-sensitive config change needs second ADMIN |
| `FLAG_NOT_FOUND` | 404 | Feature flag name not recognized |
| `INVALID_CURSOR` | 400 | Pagination cursor is malformed or expired |
| `NO_CHECKPOINT` | 404 | No checkpoint exists for workflow |
| `MEMORY_NOT_FOUND` | 404 | Memory key does not exist |
| `ADMIN_REQUIRED` | 403 | Endpoint requires ADMIN role |

---

## A.7 Rate Limit Tiers

| Tier | Limit | Endpoints |
|------|-------|-----------|
| CRITICAL HOT PATH | 600 RPM | `PR-001` render, `/api/v1/workspace/capabilities` |
| HIGH VOLUME | 300 RPM | `PR-009` AB result, `BR-002` experience, `MO-006` snapshot, `KN-006/7` memory, `/metrics` |
| STANDARD | 120 RPM | `AUTH-005`, `WF-003/4`, `PR-002`, `PV-004`, most GETs |
| REGULAR | 60 RPM | Most list and get endpoints |
| WRITE | 30 RPM | Most write endpoints |
| SENSITIVE | 10 RPM | Admin, plugin, key management |
| MANAGEMENT | 5 | WebSocket connections (concurrent, not per-minute) |
| UNLIMITED | — | `/health/live`, `/health/ready` |

---

## A.8 Permission × Role Matrix

The following table shows which roles satisfy each permission. A role satisfies a permission at its level or below — ADMIN satisfies all permissions.

| Permission | VIEWER | DEVELOPER | REVIEWER | ADMIN |
|-----------|--------|-----------|----------|-------|
| `auth:read` | ✓ | ✓ | ✓ | ✓ |
| `auth:write` | — | — | — | ✓ |
| `workspace:read` | ✓ | ✓ | ✓ | ✓ |
| `workflow:read` | ✓ | ✓ | ✓ | ✓ |
| `workflow:write` | — | ✓ | ✓ | ✓ |
| `workflow:approve` | — | — | ✓ | ✓ |
| `prompt:read` | ✓ | ✓ | ✓ | ✓ |
| `prompt:render` | — | ✓ | ✓ | ✓ |
| `prompt:write` | — | ✓ | ✓ | ✓ |
| `prompt:govern` | — | ✓ (DRAFT→REVIEW) | ✓ | ✓ |
| `brain:read` | ✓ | ✓ | ✓ | ✓ |
| `brain:write` | — | ✓ | ✓ | ✓ |
| `provider:read` | ✓ | ✓ | ✓ | ✓ |
| `knowledge:read` | ✓ | ✓ | ✓ | ✓ |
| `knowledge:write` | — | ✓ | ✓ | ✓ |
| `plugin:read` | ✓ | ✓ | ✓ | ✓ |
| `plugin:write` | — | — | — | ✓ |
| `monitoring:read` | ✓ | ✓ | ✓ | ✓ |
| `admin:read` | — | — | — | ✓ |
| `admin:write` | — | — | — | ✓ |

---

## A.9 Versioning and Deprecation Policy

### Versioning

The API is versioned at the path level (`/api/v1/`). A new version is created only when breaking changes are unavoidable. Breaking changes are:
- Removing an endpoint
- Removing or renaming a required request or response field
- Changing a response field's type
- Changing HTTP method or path structure
- Changing authentication requirements

Non-breaking changes do not require a version bump:
- Adding optional request fields
- Adding response fields
- Adding new endpoints
- Relaxing validation constraints (stricter → more permissive)
- Adding new error codes for new conditions

### Deprecation Timeline

When an endpoint is deprecated:
1. The endpoint continues to function normally
2. All responses include a `Deprecation` header: `Deprecation: true` and `Sunset: <date>`
3. The entry in this catalog is updated with `[DEPRECATED]` and the sunset date
4. Minimum 6-month notice for endpoints used by products; minimum 12 months for STABLE-tier endpoints
5. After sunset date: endpoint returns `410 Gone`

### Current Stability Tiers

| Tier | Endpoints | Guarantee |
|------|-----------|-----------|
| STABLE | AUTH, Workspace, Workflow, Prompt, Provider, Plugin, Admin, Monitoring, Configuration | No breaking changes without 6-month notice |
| BETA | Brain, Knowledge | Breaking changes with 1-month notice |
| PROPOSED | Marketplace | No guarantee; may change before activation |
| EXPERIMENTAL | Desktop WS (`DK-002`) | May change at any time |

---

## A.10 API Design Principles

The following principles govern all API decisions on this platform. New endpoints must satisfy all of them.

**P-API-1: One canonical path per operation.** There is no alternate URL, shortcut, or alias for any operation. `/api/v1/workflows` is the one URL for workflow submission. There is no `/api/v1/submit`, `/api/v1/run`, or product-specific override.

**P-API-2: Plural nouns, no verbs in paths.** `/workflows` not `/workflow`. `/prompts/templates` not `/prompt-template`. Path verbs are expressed as HTTP methods: `POST /workflows` = submit, `DELETE /workflows/{id}` = cancel-pending is wrong — cancel is `POST /workflows/{id}/cancel`.

**P-API-3: Actions on resources use sub-paths.** Non-CRUD actions on a resource use a sub-path with a verb: `POST /workflows/{id}/cancel`, `POST /workflows/{id}/pause`. Not `PATCH /workflows/{id}` with a `status` field.

**P-API-4: Never expose internal IDs in error messages.** Error messages are for the caller. Stack traces, database constraint names, internal module paths, and machine hostnames are never in API responses.

**P-API-5: 4xx errors are always the caller's fault.** If the platform can't process a valid request due to a transient condition, it returns 503, not 400. 4xx means the caller must change the request.

**P-API-6: All list endpoints are paginated.** No list endpoint returns an unbounded result set. Every list endpoint supports `limit` and `cursor` parameters and includes `next_cursor` and `has_more` in responses.

**P-API-7: Write endpoints validate before acting.** A `POST /api/v1/workflows` that fails DAG validation must not create any partial workflow state. Validation precedes all side effects.

**P-API-8: Idempotent where possible.** GET, DELETE on resource, cancel, pause, resume, approve, revoke — all idempotent. If a repeat call produces the same outcome as the first, it's idempotent.

**P-API-9: Security by default.** Every endpoint is authenticated by default. Public endpoints are explicitly listed in this catalog. There are exactly 5 public endpoints. Adding a 6th requires CTO approval.

**P-API-10: Rate limits are per API key, not per IP.** Products use dedicated API keys. Rate limiting by key allows per-product quotas and prevents one product from starving another.

---

*End of API-CATALOG v1.0.0*

*Document authority: Chief Software Architect*  
*Next review: 2026-09-28 (quarterly)*  
*Breaking changes require: CTO approval + 6-month migration notice for STABLE endpoints*  
*New endpoints require: Architect approval + entry in this catalog before implementation*

