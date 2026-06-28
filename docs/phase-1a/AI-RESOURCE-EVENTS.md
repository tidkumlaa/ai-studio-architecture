# AI Resource Events — Complete Event Catalog

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

This document is the complete catalog of every event published by AI ROS subsystems. Events are the primary integration mechanism between subsystems and between AI ROS and external consumers (Desktop dashboards, notification systems, analytics pipelines, audit systems).

---

## 2. Event Architecture

### 2.1 Event Bus

AI ROS uses an internal event bus with two delivery modes:

| Mode | Mechanism | Consumers |
|------|-----------|----------|
| In-process | Python asyncio event emitter | Same-process subsystems |
| Persistent | Redis Streams (with consumer groups) | Cross-process consumers, Desktop dashboards |

External consumers subscribe via the WebSocket API (see AI-RESOURCE-API.md) or read directly from Redis Streams (internal services only).

### 2.2 Event Envelope

Every event is wrapped in a standard envelope:

```json
{
  "event_id": "uuid",
  "event_type": "provider.health.changed",
  "schema_version": "1.0",
  "source": "ai_ros.provider_health_monitor",
  "org_id": "uuid | null",
  "emitted_at": "2026-06-29T10:00:00.000Z",
  "data": { ... }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | UUID | Globally unique; idempotency key |
| `event_type` | string | Dot-namespaced type (see catalog below) |
| `schema_version` | string | Semver of the event data schema |
| `source` | string | Subsystem that emitted the event |
| `org_id` | UUID \| null | Organization scope (null for system-wide events) |
| `emitted_at` | ISO 8601 | Timestamp with milliseconds |
| `data` | object | Event-specific payload |

### 2.3 Naming Convention

```
{domain}.{entity}.{action}

Examples:
  provider.health.changed
  execution.completed
  quota.warning
  conversation.session.created
  budget.exhausted
```

---

## 3. Provider Events

### provider.registered

Emitted when a new provider is loaded.

```json
{
  "provider_id": "anthropic",
  "display_name": "Anthropic Claude",
  "adapter_version": "1.0.0",
  "protocol_version": "1.0",
  "models_count": 4
}
```

### provider.unregistered

Emitted when a provider is disabled or removed.

```json
{
  "provider_id": "string",
  "reason": "admin_disabled | adapter_load_error | protocol_incompatible"
}
```

### provider.health.changed

Emitted when a provider's health status changes.

```json
{
  "provider_id": "string",
  "previous_status": "healthy | degraded | unavailable",
  "current_status": "healthy | degraded | unavailable",
  "latency_ms": 423,
  "error_message": "string | null",
  "circuit_breaker_state": "closed | open | half_open"
}
```

### provider.model.available

Emitted when a model comes online.

```json
{
  "provider_id": "string",
  "canonical_model_id": "string",
  "provider_model_id": "string"
}
```

### provider.model.unavailable

Emitted when a model goes offline.

```json
{
  "provider_id": "string",
  "canonical_model_id": "string",
  "reason": "string | null"
}
```

### provider.circuit_breaker.opened

Emitted when a circuit breaker trips OPEN.

```json
{
  "provider_id": "string",
  "failure_count": 15,
  "failure_rate_pct": 87.3,
  "recovery_probe_seconds": 60
}
```

### provider.circuit_breaker.closed

Emitted when a circuit breaker recovers to CLOSED.

```json
{
  "provider_id": "string",
  "recovery_latency_ms": 412
}
```

---

## 4. Execution Events

### execution.submitted

Emitted when a request is accepted by the Scheduler.

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "project_id": "uuid | null",
  "user_id": "uuid | null",
  "product_id": "string",
  "task_category": "string | null",
  "mode": "immediate | queued | batch",
  "priority": 50,
  "submitted_at": "iso8601"
}
```

### execution.queued

Emitted when a request enters the priority queue.

```json
{
  "request_id": "uuid",
  "queue_position": 14,
  "estimated_wait_seconds": 45,
  "priority": 50
}
```

### execution.dispatched

Emitted when a request is sent to the Execution Engine.

```json
{
  "request_id": "uuid",
  "provider_id": "string",
  "canonical_model_id": "string",
  "queue_wait_ms": 1200,
  "routing_rationale": "string"
}
```

### execution.completed

Emitted when an execution succeeds.

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "canonical_model_id": "string",
  "stop_reason": "end_turn | max_tokens | tool_use | stop_sequence",
  "input_tokens": 245,
  "output_tokens": 1832,
  "cache_read_tokens": 0,
  "cache_write_tokens": 0,
  "total_cost_usd": "0.028850",
  "latency_ms": 3241,
  "ttft_ms": 412,
  "attempts": 1
}
```

### execution.failed

Emitted when an execution fails terminally (all retries exhausted).

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string | null",
  "canonical_model_id": "string | null",
  "error_code": "string",
  "error_message": "string",
  "attempts": 3,
  "total_latency_ms": 12000
}
```

### execution.timeout

Emitted when a request times out.

```json
{
  "request_id": "uuid",
  "timeout_type": "connect | request | stream_first_token | stream_idle | queue_wait",
  "elapsed_ms": 60000,
  "provider_id": "string | null"
}
```

### execution.cancelled

Emitted when a request is cancelled.

```json
{
  "request_id": "uuid",
  "cancelled_by": "user | system",
  "reason": "string",
  "previous_status": "queued | in_flight"
}
```

### execution.retry

Emitted on each retry attempt.

```json
{
  "request_id": "uuid",
  "attempt_number": 2,
  "max_attempts": 3,
  "previous_error_code": "rate_limited",
  "retry_delay_ms": 2000,
  "next_provider_id": "openai | null"
}
```

---

## 5. Quota Events

### quota.check.passed

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "rules_checked": 3
}
```

### quota.check.failed

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "blocking_rule_id": "uuid",
  "resource": "INPUT_TOKENS",
  "window": "PER_MINUTE",
  "limit": 50000,
  "current_usage": 49821,
  "requested": 1200,
  "action_taken": "reject | queue | fallback_provider"
}
```

### quota.deducted

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "rule_id": "uuid",
  "resource": "INPUT_TOKENS",
  "amount": 245,
  "new_total": 28341
}
```

### quota.refunded

```json
{
  "request_id": "uuid",
  "rule_id": "uuid",
  "resource": "string",
  "amount": 245,
  "reason": "execution_failed | cancelled | timeout"
}
```

### quota.warning

```json
{
  "rule_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "resource": "string",
  "window": "string",
  "used_pct": 82.4,
  "used": 8240000,
  "limit": 10000000,
  "reset_at": "iso8601"
}
```

### quota.critical

Same schema as `quota.warning`; `used_pct` will be ≥ `critical_threshold_pct`.

### quota.exhausted

```json
{
  "rule_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "resource": "string",
  "window": "string",
  "reset_at": "iso8601",
  "action": "reject | queue | fallback_provider | alert_only"
}
```

### quota.reset

```json
{
  "rule_id": "uuid",
  "org_id": "uuid",
  "window": "string",
  "previous_usage": 8240000,
  "limit": 10000000,
  "reset_at": "iso8601"
}
```

### quota.forecast.exhaustion_24h / quota.forecast.exhaustion_6h

```json
{
  "rule_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "resource": "string",
  "current_burn_rate_per_hour": 420000,
  "projected_exhaustion_at": "iso8601",
  "hours_remaining": 19.3
}
```

---

## 6. Budget Events

### budget.check.passed

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "estimated_cost_usd": "0.028",
  "available_usd": "42.71"
}
```

### budget.check.failed

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "rule_id": "uuid",
  "estimated_cost_usd": "0.028",
  "available_usd": "0.015",
  "action_taken": "reject | queue | approve_override"
}
```

### budget.cost_recorded

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "project_id": "uuid | null",
  "provider_id": "string",
  "canonical_model_id": "string",
  "total_cost_usd": "0.028850",
  "input_tokens": 245,
  "output_tokens": 1832,
  "subscription_covered": false
}
```

### budget.warning / budget.critical / budget.exhausted

```json
{
  "rule_id": "uuid",
  "org_id": "uuid",
  "project_id": "uuid | null",
  "period": "MONTHLY",
  "spent_usd": "850.00",
  "limit_usd": "1000.00",
  "spent_pct": 85.0,
  "reset_at": "iso8601 | null"
}
```

### budget.approval_required

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "estimated_cost_usd": "6.50",
  "threshold_usd": "5.00",
  "approver_roles": ["org_admin", "billing_admin"]
}
```

### budget.recommendation

```json
{
  "org_id": "uuid",
  "recommendation_type": "model_downgrade | caching | scheduling | subscription",
  "recommendation_text": "string",
  "estimated_savings_usd_per_month": "120.00",
  "confidence": 0.87
}
```

---

## 7. Policy Events

### policy.evaluation.passed

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "policies_evaluated": 5,
  "policies_matched": 2,
  "constrained_to_providers": ["anthropic"] 
}
```

### policy.evaluation.denied

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "policy_id": "uuid",
  "policy_name": "string",
  "denial_reason": "string"
}
```

### policy.evaluation.approval_required

```json
{
  "request_id": "uuid",
  "org_id": "uuid",
  "policy_id": "uuid",
  "reason": "string",
  "approver_roles": ["string"]
}
```

### policy.created / policy.modified / policy.activated / policy.suspended

```json
{
  "policy_id": "uuid",
  "org_id": "uuid",
  "policy_name": "string",
  "policy_type": "string",
  "changed_by": "uuid",
  "version_number": 2
}
```

---

## 8. Conversation Events

### conversation.session.created

```json
{
  "session_id": "uuid",
  "org_id": "uuid",
  "product_id": "string",
  "user_id": "uuid | null"
}
```

### conversation.session.archived / conversation.session.expired

```json
{
  "session_id": "uuid",
  "org_id": "uuid",
  "message_count": 42,
  "total_cost_usd": "1.24"
}
```

### conversation.session.branched

```json
{
  "new_session_id": "uuid",
  "parent_session_id": "uuid",
  "branch_point_message_id": "uuid",
  "org_id": "uuid"
}
```

### conversation.history.compressed

```json
{
  "session_id": "uuid",
  "messages_compressed": 28,
  "summary_tokens": 412,
  "context_fill_pct_before": 87.3,
  "context_fill_pct_after": 42.1,
  "compression_strategy": "rolling_summary"
}
```

### conversation.context.overflow

```json
{
  "session_id": "uuid",
  "model_id": "string",
  "context_window_tokens": 200000,
  "messages_trimmed": 5,
  "memory_tokens_reduced": 800
}
```

---

## 9. Credential Events

All credential events contain no secret values — only identifiers and status.

### credential.registered

```json
{
  "credential_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "alias": "string",
  "credential_type": "api_key | oauth_token",
  "registered_by": "uuid"
}
```

### credential.validated / credential.validation_failed

```json
{
  "credential_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "valid": true,
  "error_code": "null | auth_failed | network_error | quota_exceeded"
}
```

### credential.expiry_warning

```json
{
  "credential_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "days_until_expiry": 14,
  "expires_at": "iso8601"
}
```

### credential.rotated

```json
{
  "old_credential_id": "uuid",
  "new_credential_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "rotation_method": "manual_confirm | auto_generate | provider_api"
}
```

### credential.revoked

```json
{
  "credential_id": "uuid",
  "org_id": "uuid",
  "provider_id": "string",
  "revoked_by": "uuid",
  "reason": "string"
}
```

---

## 10. Benchmark Events

### benchmark.outcome_recorded

```json
{
  "outcome_id": "uuid",
  "request_id": "uuid",
  "canonical_model_id": "string",
  "task_category": "string",
  "latency_ms": 1823,
  "total_cost_usd": "0.028",
  "stop_reason": "end_turn",
  "automated_quality_score": 0.91
}
```

### benchmark.score_updated

```json
{
  "canonical_model_id": "string",
  "task_category": "string",
  "composite_score": 0.87,
  "previous_composite_score": 0.84,
  "sample_size": 2841
}
```

### benchmark.regression.detected

```json
{
  "canonical_model_id": "string",
  "task_category": "string",
  "metric": "latency_p95 | success_rate | quality_score",
  "baseline_value": 1200,
  "current_value": 1800,
  "degradation_pct": 50.0,
  "severity": "warning | critical"
}
```

### benchmark.regression.cleared

```json
{
  "canonical_model_id": "string",
  "task_category": "string",
  "metric": "string",
  "current_value": 1250,
  "baseline_value": 1200
}
```

### benchmark.recommendation.updated

```json
{
  "task_category": "string",
  "top_model_id": "string",
  "previous_top_model_id": "string | null",
  "composite_score": 0.91,
  "reason": "regression_cleared | new_data | model_added"
}
```

---

## 11. Model Catalog Events

### model.registered / model.capability.updated / model.pricing.updated

```json
{
  "canonical_model_id": "string",
  "provider_id": "string",
  "change_type": "registered | capability | pricing",
  "updated_by": "uuid | null"
}
```

### model.deprecated / model.retired

```json
{
  "canonical_model_id": "string",
  "deprecation_date": "iso8601 | null",
  "successor_canonical_id": "string | null"
}
```

---

## 12. Event Retention and Delivery Guarantees

| Event Group | Retention in Redis Streams | Guaranteed Delivery |
|-------------|--------------------------|-------------------|
| Execution events | 7 days | At-least-once (consumer groups) |
| Quota events | 7 days | At-least-once |
| Budget events | 30 days | At-least-once |
| Provider health | 24 hours | Best-effort (high frequency) |
| Credential events | 90 days | At-least-once + audit log |
| Conversation events | 7 days | At-least-once |
| Benchmark events | 30 days | At-least-once |

---

## 13. Consumer Registration

Internal consumers (subsystems within AI ROS) register consumer groups on Redis Streams:

```python
# Pattern:
stream_key = f"ai_ros:events:{event_domain}"
consumer_group = f"ai_ros:{subsystem_name}"
```

External consumers (Desktop, external analytics) subscribe via WebSocket API topics.

---

## 14. Event Schema Versioning

- Each event type has a `schema_version` field
- When event data schema changes in a breaking way, `schema_version` is incremented
- Consumers must handle unknown fields gracefully (ignore additional fields)
- Schema registry maintained in `ai_ros/events/schemas/`

---

## 15. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-RESOURCE-API.md | WebSocket topics that surface these events |
| AI-RESOURCE-DESKTOP.md | Dashboard subscriptions to these events |
| AI-RESOURCE-DATABASE.md | Audit log persists selected events |
| All subsystem docs | Each references which events it publishes |
