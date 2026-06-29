---
knowledge_id: KNW-PLAT-ARCH-030
title: "Platform API Architecture"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Specify all platform HTTP API endpoints, request/response schemas, and versioning rules"
canonical_source: "architecture/docs/platform-consolidation/30-API.md"
dependencies:
  - "09-SERVICE-MODEL.md"
  - "08-SDK-MODEL.md"
  - "20-DATA-STRUCTURES.md"
related_documents:
  - "29-CLI.md"
  - "28-DASHBOARD.md"
acceptance_criteria:
  - "Every endpoint has a method, path, request body, and response schema"
  - "All paths are versioned (/api/v1/)"
  - "Error response format is standardized"
verification_checklist:
  - "[ ] All endpoints return JSON"
  - "[ ] Error codes documented"
  - "[ ] Versioning strategy defined"
future_extensions:
  - "OpenAPI spec generation from route definitions"
  - "API gateway integration"
---

# Platform API Architecture

## API Versioning

All routes are prefixed with `/api/v{major}/`.

Current version: `v1`

Version bump policy:
- Breaking change (removed field, changed type) → new major version
- Additive change (new optional field, new endpoint) → same version

Old versions supported for 2 platform releases after deprecation.

---

## Error Response Format

All errors return:
```json
{
  "error": {
    "code": "QUOTA_EXCEEDED",
    "message": "Token quota exceeded for account acc-001",
    "details": {"account_id": "acc-001", "limit": 100000, "used": 100001},
    "request_id": "req-abc123"
  }
}
```

Standard error codes:

| Code | HTTP Status | Meaning |
|------|-------------|---------|
| `VALIDATION_ERROR` | 422 | Request body invalid |
| `NOT_FOUND` | 404 | Resource not found |
| `QUOTA_EXCEEDED` | 429 | Token or cost quota exceeded |
| `BUDGET_EXCEEDED` | 429 | API budget exceeded |
| `PROVIDER_UNAVAILABLE` | 503 | Provider circuit open |
| `INTERNAL_ERROR` | 500 | Unexpected server error |

---

## Resource Intelligence API

**Router prefix:** `/api/v1/intelligence/`

### POST /api/v1/intelligence/analyze
Full pipeline: profile → complexity → quality → tokens → cost → provider → plan

**Request:**
```json
{
  "prompt": "string",
  "context_tokens": 0,
  "max_cost_usd": "0.10",
  "preferred_provider": "anthropic",
  "constraints": {
    "max_latency_ms": 5000,
    "require_vision": false
  }
}
```

**Response:**
```json
{
  "workload": {"type": "coding", "confidence": 0.92},
  "complexity": {"level": "moderate", "score": 0.43, "parallelizable": false},
  "quality": {"required": "advanced"},
  "tokens": {"input": 450, "output": 1125, "confidence": 0.85},
  "cost": {"best_case": "0.002", "expected": "0.003", "worst_case": "0.005"},
  "plan": {
    "plan_id": "plan-uuid",
    "strategy": "single_agent",
    "model_id": "claude-sonnet-4-6",
    "provider_id": "anthropic",
    "estimated_cost_usd": "0.003",
    "estimated_latency_ms": 2100
  }
}
```

---

### POST /api/v1/intelligence/plan
Returns execution plan only (no cost/token predictions if already computed).

**Request:**
```json
{
  "workload_type": "coding",
  "complexity_level": "complex",
  "max_cost_usd": "0.50"
}
```

**Response:** `ExecutionPlan` schema (see 20-DATA-STRUCTURES.md)

---

### POST /api/v1/intelligence/predict
Token and cost prediction only.

**Request:**
```json
{
  "prompt": "string",
  "model_id": "claude-sonnet-4-6",
  "workload_type": "coding"
}
```

**Response:**
```json
{
  "tokens": {"input": 450, "output": 1125},
  "cost": {"best_case": "0.002", "expected": "0.003", "worst_case": "0.005", "currency": "USD"}
}
```

---

### POST /api/v1/intelligence/recommend
Recommendations for a given account state.

**Request:**
```json
{
  "account_id": "acc-001",
  "workload_type": "documentation",
  "context": {}
}
```

**Response:**
```json
{
  "recommendations": [
    {
      "action": "use_cheaper_model",
      "reason": "Documentation tasks work well with Gemini Flash",
      "confidence": 0.87,
      "potential_savings_usd": "0.001"
    }
  ]
}
```

---

### POST /api/v1/intelligence/history
Record a prediction outcome for learning.

**Request:**
```json
{
  "prediction_id": "pred-uuid",
  "prediction_type": "token",
  "predicted_value": 1125,
  "actual_value": 987,
  "model_id": "claude-sonnet-4-6",
  "workload_type": "coding"
}
```

**Response:** `{"recorded": true, "accuracy": 0.88}`

---

### GET /api/v1/intelligence/history
Learning engine statistics.

**Response:** `LearningStats` schema (see 20-DATA-STRUCTURES.md)

---

## Quota & Budget API

**Router prefix:** `/api/v1/resources/`

### GET /api/v1/resources/quota/{account_id}
Returns current quota status for an account.

**Response:**
```json
{
  "account_id": "acc-001",
  "daily_token_limit": 100000,
  "tokens_used_today": 72000,
  "monthly_cost_limit": "50.00",
  "cost_used_month": "32.50",
  "reset_at": "2026-06-30T00:00:00Z"
}
```

---

### POST /api/v1/resources/quota/check
Check if a quota would be exceeded.

**Request:**
```json
{"account_id": "acc-001", "tokens_requested": 1000, "cost_usd": "0.003"}
```

**Response:**
```json
{"allowed": true, "remaining_tokens": 28000, "remaining_budget_usd": "17.50"}
```

---

## Provider API

**Router prefix:** `/api/v1/providers/`

### GET /api/v1/providers/
List all registered providers and health.

**Response:**
```json
{
  "providers": [
    {"id": "anthropic", "state": "HEALTHY", "p99_latency_ms": 2100, "error_rate": 0.001},
    {"id": "openai", "state": "HEALTHY", "p99_latency_ms": 1800, "error_rate": 0.002}
  ]
}
```

---

### GET /api/v1/providers/{provider_id}/health

**Response:** `ProviderHealth` schema

---

## Platform Status API

**Router prefix:** `/api/v1/platform/`

### GET /api/v1/platform/health
Overall platform health.

**Response:**
```json
{
  "status": "healthy",
  "runtimes": [
    {"id": "runtime:ai:v2", "state": "READY", "uptime_seconds": 3600}
  ],
  "services": [
    {"id": "service.auth", "state": "up"}
  ]
}
```

### GET /api/v1/platform/manifest
Returns current platform manifest summary.

---

## API Authentication

All API calls require an `Authorization: Bearer {token}` header.
Token verified by `AuthService.verify_token()`.

Unauthenticated requests return `401 Unauthorized`.
