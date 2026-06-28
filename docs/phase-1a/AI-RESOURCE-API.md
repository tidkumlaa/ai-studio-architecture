# AI Resource API — REST, WebSocket, and Streaming Specification

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

This document specifies the complete API surface that AI ROS exposes to AI Studio products and administrative tools. The API is the integration boundary — products use it exclusively to access all AI capabilities. No product may import AI ROS internals or call provider adapters directly.

---

## 2. Responsibilities

- REST API: request/response operations (execute, session management, admin)
- WebSocket API: streaming execution, live events, real-time dashboards
- Streaming API: token-by-token streaming for interactive products
- Authentication: API key + JWT for internal services
- Authorization: RBAC with org/project/user scoping
- Rate limiting: per-client, per-org API call limits
- Versioning: `/v1/` prefix; breaking changes increment major version

---

## 3. Architecture

```
AI Studio Products
        │
        │ HTTP / WebSocket
        ▼
┌──────────────────────────────────────────────────────────────────────┐
│                         AI Resource API                                │
│                                                                        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  REST API        │  │  WebSocket API   │  │  Event Stream    │   │
│  │  (FastAPI)       │  │  (FastAPI WS)    │  │  (SSE)           │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │                      │              │
│  ┌────────▼─────────────────────▼──────────────────────▼───────────┐  │
│  │                     API Gateway                                    │  │
│  │   Auth │ Rate Limit │ Request Validation │ Response Serialization │  │
│  └─────────────────────────────┬────────────────────────────────────┘  │
│                                 │                                       │
│           ┌─────────────────────┼─────────────────────┐               │
│           ▼                     ▼                     ▼               │
│     Execution API         Session API           Admin API             │
│     Quota API             Budget API            Benchmark API         │
│     Policy API            Provider API          Account API           │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 4. Authentication

### 4.1 Client Authentication

All requests to the AI Resource API must include one of:

| Method | Header | Use Case |
|--------|--------|---------|
| API Key | `X-AISF-API-Key: <key>` | Product-to-AI-ROS server-side calls |
| Bearer JWT | `Authorization: Bearer <token>` | User-scoped desktop and web client calls |
| Service Token | `X-Service-Token: <token>` | Inter-service calls within AI Studio |

### 4.2 JWT Claims

Internal JWT tokens include:

```json
{
  "sub": "user_uuid",
  "org_id": "org_uuid",
  "project_ids": ["proj_uuid_1", "proj_uuid_2"],
  "roles": ["org_member", "project_admin"],
  "product_id": "ai-software-factory",
  "iat": 1735000000,
  "exp": 1735003600,
  "jti": "unique_token_id"
}
```

### 4.3 Authorization (RBAC)

| Role | Permissions |
|------|------------|
| `org_admin` | All permissions within org |
| `project_admin` | All permissions within assigned projects |
| `org_member` | Execute requests, manage own sessions |
| `policy_admin` | Create/modify policies |
| `billing_admin` | View/modify budgets; generate chargeback reports |
| `system` | Reserved for AI ROS internal calls |

---

## 5. Base URL and Versioning

```
Base URL: http://ai-ros.internal/api/v1/

Production: https://ai-ros.yourdomain.com/api/v1/
Development: http://localhost:8900/api/v1/
```

All responses include:
```
AI-ROS-Version: 1.0.0
AI-ROS-Request-ID: <uuid>
Content-Type: application/json
```

---

## 6. REST API — Execution

### POST /api/v1/execute

Submit an AI execution request.

**Request:**
```json
{
  "session_id": "uuid | null",
  "product_id": "ai-software-factory",
  "task_category": "code.generation",
  "messages": [
    {
      "role": "user",
      "content": [{"type": "text", "text": "Generate a FastAPI health check endpoint"}]
    }
  ],
  "capability_profile": {
    "required_capabilities": ["text_generation"],
    "quality_tier_minimum": "STANDARD",
    "maximum_latency_class": "LOW"
  },
  "config": {
    "max_tokens": 2048,
    "temperature": 0.7,
    "stream": false,
    "scheduling_mode": "IMMEDIATE",
    "priority": "STANDARD",
    "retry_policy": {
      "max_attempts": 3
    }
  },
  "metadata": {}
}
```

**Response (200 OK):**
```json
{
  "request_id": "uuid",
  "session_id": "uuid",
  "status": "COMPLETED",
  "provider_id": "anthropic",
  "canonical_model_id": "anthropic/claude/claude-sonnet-4-6",
  "content": [
    {"type": "text", "text": "@app.get('/health')\nasync def health_check(): ..."}
  ],
  "stop_reason": "END_TURN",
  "usage": {
    "input_tokens": 45,
    "output_tokens": 312,
    "cache_read_tokens": 0,
    "cache_write_tokens": 0
  },
  "cost_usd": "0.00531",
  "latency_ms": 1823,
  "routing_rationale": "Selected claude-sonnet-4-6: benchmark_score=0.87 for code.generation; cost-optimized within STANDARD+ quality tier"
}
```

**Response (202 Accepted — QUEUED mode):**
```json
{
  "request_id": "uuid",
  "status": "QUEUED",
  "queue_position": 14,
  "estimated_wait_seconds": 45,
  "poll_url": "/api/v1/requests/uuid/status"
}
```

**Error Responses:**

| Status | Code | Meaning |
|--------|------|---------|
| 400 | `INVALID_REQUEST` | Malformed request body |
| 401 | `UNAUTHORIZED` | Missing or invalid auth |
| 403 | `POLICY_DENIED` | Policy Engine denied request |
| 402 | `BUDGET_EXCEEDED` | Budget exhausted |
| 429 | `QUOTA_EXCEEDED` | Quota limit hit |
| 503 | `NO_PROVIDER_AVAILABLE` | All providers unavailable |
| 504 | `EXECUTION_TIMEOUT` | Request timed out |

### POST /api/v1/execute/stream

Same as /execute but returns a Server-Sent Events stream. Set `Accept: text/event-stream`.

**SSE Events:**
```
event: stream_start
data: {"request_id": "uuid", "provider_id": "anthropic", "model": "claude-sonnet-4-6"}

event: text_delta
data: {"index": 0, "delta": "Here is a FastAPI"}

event: text_delta
data: {"index": 0, "delta": " health check endpoint:"}

event: stream_end
data: {"stop_reason": "END_TURN", "usage": {...}, "cost_usd": "0.00531", "latency_ms": 823}
```

### GET /api/v1/requests/{request_id}/status

Poll request status for QUEUED requests.

**Response:**
```json
{
  "request_id": "uuid",
  "status": "IN_FLIGHT | QUEUED | COMPLETED | FAILED",
  "queue_position": 3,
  "estimated_wait_seconds": 12,
  "submitted_at": "2026-06-29T10:00:00Z",
  "dispatched_at": "2026-06-29T10:00:45Z | null",
  "completed_at": null
}
```

### DELETE /api/v1/requests/{request_id}

Cancel a pending or in-flight request.

---

## 7. REST API — Sessions

### POST /api/v1/sessions

Create a new conversation session.

**Request:**
```json
{
  "product_id": "ai-software-factory",
  "project_id": "uuid | null",
  "system_prompt": "You are an expert software architect...",
  "config": {
    "compression_strategy": "ROLLING_SUMMARY",
    "memory_injection_enabled": true,
    "idle_expiry_hours": 24
  },
  "metadata": {}
}
```

**Response (201 Created):**
```json
{
  "session_id": "uuid",
  "created_at": "2026-06-29T10:00:00Z",
  "expires_at": "2026-07-02T10:00:00Z"
}
```

### GET /api/v1/sessions

List sessions for the authenticated user/org.

**Query params:** `status`, `product_id`, `project_id`, `page`, `page_size`

### GET /api/v1/sessions/{session_id}

Get session details including message count and cumulative usage.

### GET /api/v1/sessions/{session_id}/messages

Get message history.

**Query params:** `after_sequence` (integer), `limit` (default 50, max 200)

### POST /api/v1/sessions/{session_id}/branch

Create a branched session.

**Request:** `{"from_message_id": "uuid"}`

### GET /api/v1/sessions/{session_id}/export

Export session as JSON, Markdown, or PDF.

**Query params:** `format=json|markdown|pdf`

### DELETE /api/v1/sessions/{session_id}

Archive session.

---

## 8. REST API — Quota

### GET /api/v1/quota

Get current quota positions for the authenticated org.

**Response:**
```json
{
  "positions": [
    {
      "rule_id": "uuid",
      "provider_id": "anthropic",
      "resource": "INPUT_TOKENS",
      "window": "PER_DAY",
      "limit": 10000000,
      "used": 3241567,
      "remaining": 6758433,
      "used_pct": 32.4,
      "reset_at": "2026-06-30T00:00:00Z",
      "alert_level": "OK"
    }
  ],
  "forecasts": [...]
}
```

### GET /api/v1/quota/forecast

Get depletion forecasts for all active quota rules.

---

## 9. REST API — Budget

### GET /api/v1/budget

Get budget positions for the authenticated org/project.

### GET /api/v1/budget/forecast

Get burn rate and spend forecasts.

### GET /api/v1/budget/report

Generate chargeback report.

**Query params:** `period_start` (ISO date), `period_end` (ISO date), `group_by` (`project|department|user|model|provider`)

**Response:** CSV or JSON (controlled by `Accept` header)

---

## 10. REST API — Models and Providers (Read-Only)

### GET /api/v1/models

List all available models.

**Query params:** `capability`, `quality_tier`, `provider_id`, `status`

**Response:** Paginated list of ModelRecord (see AI-MODEL-CATALOG.md)

### GET /api/v1/models/{canonical_model_id}

Get single model record with benchmark scores.

### GET /api/v1/providers

List all registered providers with current health status.

### GET /api/v1/providers/{provider_id}/health

Get health status and recent error rates for a provider.

---

## 11. REST API — Admin

All admin endpoints require `org_admin` or specific admin roles.

### POST /api/v1/admin/providers

Register a new provider.

### PUT /api/v1/admin/providers/{provider_id}

Update provider configuration.

### POST /api/v1/admin/credentials

Register a credential for a provider.

### DELETE /api/v1/admin/credentials/{credential_id}

Revoke a credential.

### POST /api/v1/admin/policies

Create a new policy.

### PUT /api/v1/admin/policies/{policy_id}

Update a policy.

### PUT /api/v1/admin/policies/{policy_id}/status

Activate, suspend, or archive a policy.

### POST /api/v1/admin/budget-rules

Create a budget rule.

### POST /api/v1/admin/quota-rules

Create a quota rule.

### GET /api/v1/admin/audit-log

Query audit log.

**Query params:** `from`, `to`, `event_type`, `org_id`, `user_id`, `page`, `page_size`

---

## 12. REST API — Benchmark

### GET /api/v1/benchmarks/scores

Get benchmark scores by model and task category.

**Query params:** `model_id`, `task_category`, `window_days` (default 30)

### GET /api/v1/benchmarks/recommendations

Get model recommendations for a task profile.

**Request body:** TaskProfile (see AI-BENCHMARK-CENTER.md)

### POST /api/v1/benchmarks/suites/{suite_id}/run

Trigger a synthetic benchmark run (admin only).

---

## 13. WebSocket API

### ws://ai-ros/api/v1/ws

Persistent WebSocket connection for:
- Live streaming execution results
- Real-time quota and budget alerts
- Provider health change notifications
- Policy evaluation notifications

**Connection authentication:** Pass API key or JWT as query parameter `?token=<key>` (HTTPS only; TLS required in production).

**Message format:**
```json
{
  "type": "subscribe | unsubscribe | ping",
  "topics": ["execution.streaming.*", "quota.warning.*", "provider.health.*"]
}
```

**Topic patterns:**

| Topic | Description |
|-------|-------------|
| `execution.streaming.{request_id}` | Token stream for specific request |
| `execution.status.{request_id}` | Status changes for queued request |
| `quota.warning.{org_id}` | Quota warning/critical/exhausted events |
| `budget.warning.{org_id}` | Budget warning events |
| `provider.health.{provider_id}` | Provider health changes |
| `session.{session_id}` | All events for a session |

---

## 14. Common Response Envelope

All REST responses follow a common envelope:

**Success:**
```json
{
  "data": { ... },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2026-06-29T10:00:00Z",
    "api_version": "1.0.0"
  }
}
```

**Error:**
```json
{
  "error": {
    "code": "POLICY_DENIED",
    "message": "Request denied by policy 'HIPAA: No PHI to non-EU providers'",
    "policy_id": "uuid",
    "request_id": "uuid",
    "retryable": false,
    "retry_after_seconds": null
  },
  "meta": {
    "request_id": "uuid",
    "timestamp": "2026-06-29T10:00:00Z",
    "api_version": "1.0.0"
  }
}
```

---

## 15. Pagination

All list endpoints use cursor-based pagination:

**Request query params:** `page_size` (default 50, max 500), `cursor` (opaque string from previous response)

**Response:**
```json
{
  "data": [...],
  "pagination": {
    "has_next": true,
    "next_cursor": "opaque_cursor_string",
    "total_count": 1247
  }
}
```

---

## 16. API Rate Limits

| Endpoint Group | Limit |
|---------------|-------|
| POST /execute | 100 req/min per org |
| GET (read) | 1,000 req/min per org |
| Admin endpoints | 60 req/min per user |
| WebSocket connections | 10 concurrent per org |

Rate limit headers on all responses:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1735000060
```

---

## 17. Versioning Policy

- `/api/v1/` — current stable version
- Breaking changes: new major version `/api/v2/`; `/v1/` supported for 12 months after `/v2/` release
- Non-breaking additions: added to existing version; clients must be tolerant of new fields
- Deprecations: announced in `Deprecation` header 90 days before removal

---

## 18. Security

- All API endpoints require authentication; no public anonymous endpoints
- HTTPS required in all non-local environments
- CORS: restricted to known AI Studio origins; no wildcard
- Input validation via Pydantic schemas on all request bodies
- SQL injection: not applicable (ORM parameterized queries)
- Prompt injection: handled by Policy Engine before execution

---

## 19. Implementation Roadmap

### Phase 1A.0 (Week 1–2)
- POST /execute (non-streaming, IMMEDIATE mode)
- Session CRUD
- Basic auth (API key)

### Phase 1A.1 (Week 3–4)
- POST /execute/stream (SSE streaming)
- GET /quota, /budget
- WebSocket for streaming

### Phase 1A.2 (Week 5–6)
- Full admin endpoints
- RBAC enforcement
- GET /models, /providers

### Phase 1A.3 (Week 7–8)
- Benchmark endpoints
- WebSocket topics: quota, provider health
- Chargeback report generation

---

## 20. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-RESOURCE-SECURITY.md | Auth implementation detail |
| AI-RESOURCE-EVENTS.md | Events surfaced via WebSocket |
| AI-PROVIDER-CONTRACTS.md | Request/response types |
| AI-CONVERSATION-ENGINE.md | Session management implementation |
| AI-QUOTA-ENGINE.md | Quota endpoint backing |
| AI-BUDGET-ENGINE.md | Budget endpoint backing |
