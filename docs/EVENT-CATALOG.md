# AI Studio Platform — Enterprise Event Catalog

**Document ID:** EVENT-CATALOG  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** RATIFIED  
**Classification:** INTERNAL — PLATFORM GOVERNANCE  
**Authority:** Chief Software Architect  
**Predecessor:** PLATFORM-CONTRACTS.md  
**Review Cycle:** Quarterly — events may not be added or removed outside this cycle without CTO approval

---

## Catalog Authority Notice

> This catalog is the authoritative registry of every event on the AI Studio platform event bus.
>
> An event that is not in this catalog does not exist. Code that publishes an unregistered event subject is a platform violation — it will route to no stream, be consumed by nothing, and produce a silent data loss.
>
> An event that is in this catalog but has no publisher and no consumer is dead weight. It is reviewed for removal every quarter.
>
> This document supersedes all event references in PLATFORM-CONTRACTS.md. Where the two conflict, this catalog wins.

---

## How to Read This Catalog

Each event is specified in a consistent block format. Events within a category share a NATS stream. The `Subject` field is the NATS subject — the address events are published to and consumed from.

**Event block structure:**

```
Subject            Full NATS subject (stream routing key)
Description        What happened that caused this event
Producer           Platform module or product that publishes this event
Consumers          Platform modules or products that subscribe
Payload Schema     All fields in the event payload (envelope fields are universal)
Validation Rules   Constraints enforced before publish
Ordering           Whether ordering is guaranteed within this stream
Delivery           At-most-once | At-least-once | Exactly-once
Retry              How NATS re-delivers on consumer NAK or timeout
DLQ                Dead-letter queue subject for exhausted deliveries
TTL                How long the event is retained in JetStream
Priority           Delivery priority relative to other events in the stream
Version            Payload schema version
Security Class     Who may read this event (PUBLIC | INTERNAL | RESTRICTED | CONFIDENTIAL)
Audit              Whether this event is written to the immutable audit log
Example            Complete example payload
Lifecycle          PROPOSED | ACTIVE | DEPRECATED | RETIRED
```

---

## Universal Event Envelope

Every event on the platform, regardless of category, carries these envelope fields. They are always present, always populated, never optional. Payload schema tables below describe only the event-specific fields — the envelope is implicit.

| Field | Type | Description |
|-------|------|-------------|
| `event_id` | UUID | Unique per event. Used for deduplication. |
| `event_type` | string | Full NATS subject (e.g., `workflows.task.completed`). |
| `schema_version` | string | Payload schema version (e.g., `"1.0"`). |
| `source` | string | Publishing module path (e.g., `"platform/workflow-runtime"`). |
| `correlation_id` | UUID | Propagated from the triggering HTTP request or parent event. |
| `causation_id` | UUID \| null | `event_id` of the event that caused this one. |
| `timestamp` | ISO8601 | UTC timestamp of the moment the event occurred. |
| `environment` | string | `"local"` \| `"staging"` \| `"production"`. |
| `platform_version` | string | Running platform version (e.g., `"2.0.1"`). |

---

## Naming Convention

### Subject Structure

```
<domain>.<entity>.<verb>
```

| Segment | Rule | Examples |
|---------|------|---------|
| `<domain>` | Plural noun — the platform subsystem | `workflows`, `prompts`, `ai`, `security`, `brain`, `plugins`, `billing`, `system` |
| `<entity>` | Singular noun — the thing that changed | `task`, `workflow`, `template`, `execution`, `provider`, `pattern`, `key`, `invoice` |
| `<verb>` | Past-tense action | `created`, `started`, `completed`, `failed`, `activated`, `exceeded`, `expired` |

### Naming Rules

**R-NM-1: Domain is always plural.** `workflows.*` not `workflow.*`. This is a hard rule. The current codebase bug (`workflow.*` singular) produces silent data loss.

**R-NM-2: Verb is always past tense.** Events describe things that happened. `task.completed` not `task.complete`. `provider.circuit_opened` not `provider.open_circuit`.

**R-NM-3: No verbs in the domain segment.** `ai.execution.completed` not `ai.execute.completed`. The domain is a namespace, not an action.

**R-NM-4: No product names in platform events.** `workflows.task.completed` not `aisf.workflow.task.completed`. Platform events have no product prefix. Products emit events under their own namespace: `aisf.product.created`.

**R-NM-5: Compound entity names use underscores.** `security.api_key.created` not `security.apikey.created` or `security.api-key.created`.

**R-NM-6: No version in the subject.** Subject is stable; the `schema_version` field in the payload handles versioning. Never `workflows.task.completed.v2`.

**R-NM-7: Subject depth is exactly three segments.** No two-segment subjects (`workflows.completed`). No four-segment subjects (`workflows.task.phase.completed`). Three segments only.

### Reserved Domains

| Domain | NATS Stream | What it covers |
|--------|-------------|---------------|
| `workflows` | WORKFLOWS | Workflow and task lifecycle |
| `prompts` | PROMPTS | Prompt template lifecycle |
| `ai` | AI | AI executions, providers, budget, models |
| `brain` | BRAIN | Knowledge, patterns, experiences |
| `security` | SECURITY | Auth, authz, rate limiting |
| `workspace` | WORKSPACE | Platform health, products, providers |
| `knowledge` | KNOWLEDGE | Knowledge graph changes |
| `plugins` | PLUGINS | Plugin lifecycle |
| `billing` | BILLING | Cost, invoices, budgets |
| `usage` | USAGE | Token and request statistics |
| `monitoring` | MONITORING | Health, SLA, alerts |
| `system` | SYSTEM | Platform lifecycle, config |
| `desktop` | DESKTOP | Desktop UI events |

Product namespaces (not platform streams):
| Domain | Product | Example |
|--------|---------|---------|
| `aisf` | ai-software-factory | `aisf.product.created` |
| `content` | content-factory | `content.job.completed` |
| `mythic` | mythic-realms | `mythic.quest.started` |

---

## NATS Stream Configuration

| Stream | Subjects Filter | Retention | Max Age | Max Bytes | Replicas |
|--------|----------------|-----------|---------|-----------|---------|
| WORKFLOWS | `workflows.>` | limits | 7 days | 10 GB | 1 (3 in prod) |
| PROMPTS | `prompts.>` | limits | 30 days | 2 GB | 1 (3 in prod) |
| AI | `ai.>` | limits | 3 days | 20 GB | 1 (3 in prod) |
| BRAIN | `brain.>` | limits | 30 days | 5 GB | 1 (3 in prod) |
| SECURITY | `security.>` | limits | 90 days | 5 GB | 1 (3 in prod) |
| WORKSPACE | `workspace.>` | limits | 1 day | 500 MB | 1 (3 in prod) |
| KNOWLEDGE | `knowledge.>` | limits | 30 days | 5 GB | 1 (3 in prod) |
| PLUGINS | `plugins.>` | limits | 30 days | 1 GB | 1 (3 in prod) |
| BILLING | `billing.>` | limits | 365 days | 2 GB | 1 (3 in prod) |
| USAGE | `usage.>` | limits | 30 days | 10 GB | 1 (3 in prod) |
| MONITORING | `monitoring.>` | limits | 7 days | 2 GB | 1 (3 in prod) |
| SYSTEM | `system.>` | limits | 90 days | 1 GB | 1 (3 in prod) |
| DESKTOP | `desktop.>` | interest | 1 hour | 100 MB | 1 |

**Retention policy:** `limits` — events retained until max age OR max bytes reached (whichever first, oldest events evicted).  
**DESKTOP stream:** `interest` retention — events retained only while active consumers exist. Desktop events are ephemeral UI signals.

---

## Event Governance

### Adding a New Event

1. Add the event to this catalog with all required fields
2. Set `Lifecycle: PROPOSED`
3. Submit for review (Chief Architect approval required)
4. Upon approval: `Lifecycle: ACTIVE`
5. Publisher implementation: reference this catalog, not the other way around

### Deprecating an Event

1. Set `Lifecycle: DEPRECATED` in this catalog
2. Add `deprecated_in_version` and `removal_in_version` to the event block
3. Notify all consumers with a migration guide
4. Maintain the publisher for the duration of the compatibility period
5. After removal: `Lifecycle: RETIRED` (entry kept in catalog permanently for audit history)

### Changing a Payload

**Adding a field (backward-compatible):** Update `schema_version` MINOR (e.g., `"1.0"` → `"1.1"`). Consumers must handle unknown fields gracefully.

**Removing or renaming a field (breaking change):** New event version → increment `schema_version` MAJOR (e.g., `"1.1"` → `"2.0"`). Publish both `schema_version: "1.x"` and `schema_version: "2.0"` for the compatibility period.

**Changing a field type (breaking change):** Same process as removing. Never silently change a type.

### Consumer Registration

Every consumer of a platform event must be registered in this catalog under the event's `Consumers` field. An unregistered consumer is a shadow consumer — it may exist but is not governed and not notified of changes.

---

---

# Category 1: Workspace Events

**Stream:** WORKSPACE  
**TTL (default):** 24 hours  
**Priority (default):** LOW  
**Security Class (default):** INTERNAL

Workspace events describe changes to the registry of active platform resources: products, providers, platform modules, and their health states.

---

## WS-001: `workspace.product.registered`

**Description:** A product has successfully registered its manifest with the workspace. The product is now discoverable and its capabilities are available.

**Producer:** `platform/workspace`  
**Consumers:** Desktop (WorkspacePanel), monitoring/workspace-health-monitor

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `product_name` | string | * | |
| `version` | string | * | SemVer version of the product. |
| `api_prefix` | string | * | e.g., `/api/v1` |
| `capabilities` | [string] | * | Declared product capabilities. |
| `required_platform_modules` | [string] | * | |
| `registered_by` | string | * | Hostname/instance that registered. |

**Validation Rules:**
- `product_id` must be a non-empty string matching `^[a-z][a-z0-9-]*$`
- `version` must be valid SemVer
- `capabilities` must be non-empty

**Ordering:** Best-effort within stream. Products register at startup; ordering not critical.  
**Delivery:** At-least-once  
**Retry:** 3 attempts, 5s/10s/30s backoff  
**DLQ:** `workspace.product.registered.dlq`  
**TTL:** 24 hours  
**Priority:** LOW  
**Version:** 1.0  
**Security Classification:** INTERNAL  
**Audit Required:** No (workspace snapshots are not audit-grade events)

**Example Payload:**
```
event_type:    workspace.product.registered
schema_version: "1.0"
source:        platform/workspace
payload:
  product_id:   "ai-software-factory"
  product_name: "AI Software Factory"
  version:      "2.0.0"
  api_prefix:   "/api/v1"
  capabilities: ["product_creation", "blueprint_generation", "agent_execution"]
  required_platform_modules: ["ai-ros", "workflow-runtime", "prompt-os", "knowledge"]
  registered_by: "aisf-host-01"
```

**Lifecycle:** ACTIVE

---

## WS-002: `workspace.product.deregistered`

**Description:** A product has removed itself from the workspace registry, typically during graceful shutdown.

**Producer:** `platform/workspace`  
**Consumers:** Desktop (WorkspacePanel), monitoring/workspace-health-monitor

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `reason` | enum(SHUTDOWN, CRASH, EVICTION) | * | `CRASH` if deregistered by platform after missed heartbeat. |
| `uptime_seconds` | integer | * | How long the product was registered. |

**Validation Rules:** `product_id` must have been previously registered.  
**Delivery:** At-least-once | **TTL:** 24 hours | **Priority:** LOW | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## WS-003: `workspace.health.degraded`

**Description:** A platform module has transitioned from HEALTHY to DEGRADED state. The platform continues to function but with reduced capability.

**Producer:** `platform/workspace/health-monitor`  
**Consumers:** Desktop (WorkspacePanel), monitoring/alert-manager, system/on-call-notifier

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `module_id` | string | * | e.g., `"ai-ros"`, `"nats"`, `"postgres"` |
| `previous_status` | enum(HEALTHY) | * | Always HEALTHY when transitioning to DEGRADED. |
| `current_status` | enum(DEGRADED) | * | |
| `degraded_since` | ISO8601 | * | When degradation began. |
| `affected_capabilities` | [string] | * | What the platform can no longer do at full capacity. |
| `error_summary` | string | * | Human-readable description of the degradation. No internal details. |
| `error_code` | string | * | Machine-readable code (e.g., `"NATS_CONNECTION_SLOW"`). |

**Validation Rules:** `module_id` must be a registered platform module.  
**Ordering:** Not guaranteed. Health events may arrive out of order; consumers use timestamps.  
**Delivery:** At-least-once | **Retry:** 5 attempts, aggressive backoff | **DLQ:** `workspace.health.degraded.dlq`  
**TTL:** 24 hours | **Priority:** HIGH  
**Version:** 1.0 | **Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## WS-004: `workspace.health.offline`

**Description:** A platform module has become completely unavailable. This is a platform incident.

**Producer:** `platform/workspace/health-monitor`  
**Consumers:** Desktop (WorkspacePanel), monitoring/alert-manager, system/on-call-notifier, billing/cost-tracker (pauses billing if AI offline)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `module_id` | string | * | |
| `previous_status` | enum(HEALTHY, DEGRADED) | * | |
| `current_status` | enum(OFFLINE) | * | |
| `offline_since` | ISO8601 | * | |
| `impact_severity` | enum(LOW, MEDIUM, HIGH, CRITICAL) | * | Impact on overall platform function. |
| `affected_products` | [string] | * | Products that cannot function without this module. |
| `error_summary` | string | * | |
| `error_code` | string | * | |

**Delivery:** At-least-once | **Priority:** CRITICAL | **TTL:** 24 hours  
**Version:** 1.0 | **Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## WS-005: `workspace.health.recovered`

**Description:** A previously degraded or offline platform module has returned to HEALTHY state.

**Producer:** `platform/workspace/health-monitor`  
**Consumers:** Desktop (WorkspacePanel), monitoring/alert-manager, system/on-call-notifier

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `module_id` | string | * | |
| `previous_status` | enum(DEGRADED, OFFLINE) | * | |
| `current_status` | enum(HEALTHY) | * | |
| `downtime_seconds` | integer | * | Total seconds unavailable or degraded. |
| `recovered_at` | ISO8601 | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 24 hours  
**Version:** 1.0 | **Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

---

# Category 2: Workflow Events

**Stream:** WORKFLOWS  
**TTL (default):** 7 days  
**Priority (default):** STANDARD  
**Security Classification (default):** INTERNAL

Workflow events describe the full lifecycle of workflow plans and their constituent tasks. These are the highest-volume events on the platform during active usage.

---

## WF-001: `workflows.workflow.created`

**Description:** A new workflow has been submitted and accepted by the WorkflowRuntime. Execution has not yet begun.

**Producer:** `platform/workflow-runtime`  
**Consumers:** brain/experience-recorder (begin tracking), billing/cost-tracker (begin cost accumulation context), monitoring/sla-monitor (start SLA clock), desktop/workflow-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `product_id` | string | * | |
| `plan_name` | string | * | |
| `task_count` | integer | * | Total tasks in the submitted plan. |
| `gate_count` | integer | * | Number of human approval gates. |
| `priority` | enum(URGENT, STANDARD, BATCH) | * | |
| `has_dag` | boolean | * | True if dag_edges were provided. |
| `idempotency_key` | string \| null | ? | Echoed if provided in submit. |
| `submitted_by` | string | * | Identity from AuthContext. |

**Validation Rules:**
- `task_count` >= 1
- `workflow_id` must not already exist

**Ordering:** Not guaranteed across workflows. Guaranteed within a single workflow_id.  
**Delivery:** At-least-once  
**Retry:** 3 attempts, 1s/5s/15s backoff  
**DLQ:** `workflows.dlq`  
**TTL:** 7 days | **Priority:** STANDARD | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No

**Example Payload:**
```
event_type:    workflows.workflow.created
schema_version: "1.0"
source:        platform/workflow-runtime
payload:
  workflow_id:   "550e8400-e29b-41d4-a716-446655440000"
  product_id:    "ai-software-factory"
  plan_name:     "E-commerce Platform — Build Phase"
  task_count:    8
  gate_count:    2
  priority:      "STANDARD"
  has_dag:       true
  idempotency_key: "aisf-build-ecommerce-v3"
  submitted_by:  "key-123456"
```

**Lifecycle:** ACTIVE

---

## WF-002: `workflows.workflow.completed`

**Description:** All tasks in a workflow have completed successfully. This is the terminal success event for a workflow.

**Producer:** `platform/workflow-runtime`  
**Consumers:** brain/experience-recorder (record success pattern), billing/cost-tracker (finalize cost), monitoring/sla-monitor (record SLA outcome), desktop/workflow-panel, product consumers (via product-specific subscription)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `product_id` | string | * | |
| `plan_name` | string | * | |
| `started_at` | ISO8601 | * | |
| `completed_at` | ISO8601 | * | |
| `duration_ms` | integer | * | Wall-clock time from first task dispatch to last task completion. |
| `total_tasks` | integer | * | |
| `total_ai_executions` | integer | * | Number of AIClient.execute() calls across all tasks. |
| `total_cost_usd` | float | * | Sum of AI execution costs. |
| `total_tokens` | integer | * | Sum of tokens consumed. |
| `gates_approved` | integer | * | Number of human gates that were approved. |
| `sla_met` | boolean | * | Whether the workflow completed within its SLA. |

**Ordering:** Within a workflow_id: WF-001 always precedes WF-002.  
**Delivery:** At-least-once | **TTL:** 7 days | **Priority:** STANDARD | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## WF-003: `workflows.workflow.failed`

**Description:** A workflow has entered a terminal failure state. One or more tasks failed after all retries were exhausted.

**Producer:** `platform/workflow-runtime`  
**Consumers:** brain/experience-recorder (record failure), monitoring/alert-manager, desktop/workflow-panel, product consumers

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `product_id` | string | * | |
| `plan_name` | string | * | |
| `failed_task_id` | string | * | The task whose failure caused the workflow to fail. |
| `failure_reason` | string | * | Human-readable. No internal stack traces. |
| `failure_error_code` | string | * | Machine-readable. |
| `tasks_completed` | integer | * | Tasks that completed before failure. |
| `tasks_failed` | integer | * | Tasks that failed (may be > 1 if parallel tasks failed simultaneously). |
| `total_cost_usd` | float | * | Cost incurred before failure. |
| `duration_ms` | integer | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## WF-004: `workflows.workflow.cancelled`

**Description:** A workflow was cancelled by an explicit `WorkflowClient.cancel()` call. Not the same as failure.

**Producer:** `platform/workflow-runtime`  
**Consumers:** brain/experience-recorder, billing/cost-tracker (finalize partial cost), monitoring/sla-monitor, desktop/workflow-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `product_id` | string | * | |
| `cancelled_by` | string | * | Identity from AuthContext. |
| `reason` | string | * | Cancellation reason provided by caller. |
| `tasks_completed` | integer | * | |
| `tasks_cancelled` | integer | * | Tasks that were READY or IN_PROGRESS when cancellation was received. |
| `total_cost_usd` | float | * | Cost incurred before cancellation. |

**Delivery:** At-least-once | **Priority:** STANDARD | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## WF-005: `workflows.workflow.paused`

**Description:** A workflow has been paused. No new tasks will be dispatched until resumed or cancelled.

**Producer:** `platform/workflow-runtime`  
**Consumers:** Desktop/workflow-panel, monitoring/sla-monitor (pause SLA clock)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `product_id` | string | * | |
| `paused_by` | string | * | Identity (human or gate). |
| `pause_reason` | enum(USER_REQUEST, GATE_PENDING, RESOURCE_CONSTRAINT) | * | |
| `next_task_id` | string | * | Task that will execute when resumed. |
| `in_progress_tasks` | [string] | * | Tasks that were IN_PROGRESS when pause was called (these complete). |

**Delivery:** At-least-once | **Priority:** STANDARD | **TTL:** 7 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## WF-006: `workflows.workflow.resumed`

**Description:** A paused workflow has been resumed.

**Producer:** `platform/workflow-runtime`  
**Consumers:** Desktop/workflow-panel, monitoring/sla-monitor (resume SLA clock)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `product_id` | string | * | |
| `resumed_by` | string | * | |
| `next_task_id` | string | * | |
| `pause_duration_ms` | integer | * | How long the workflow was paused. |

**Delivery:** At-least-once | **Priority:** STANDARD | **TTL:** 7 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## WF-007: `workflows.task.started`

**Description:** A task has been dispatched to an agent and execution has begun. The highest-frequency workflow event.

**Producer:** `platform/workflow-runtime`  
**Consumers:** monitoring/sla-monitor (start per-task SLA), desktop/workflow-panel, brain/pattern-tracker

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `task_id` | string | * | |
| `task_type` | string | * | |
| `agent_type` | string | * | |
| `product_id` | string | * | |
| `priority` | integer | * | Task priority [1,10]. |
| `retry_attempt` | integer | * | 0 for first attempt; >0 for retry. |
| `timeout_seconds` | integer | * | Task-level timeout. |
| `depends_on` | [string] | * | Task IDs this task depended on (empty if no dependencies). |

**Validation Rules:** `workflow_id` must exist and be in ACTIVE state.  
**Ordering:** Guaranteed within a workflow_id. WF-001 precedes any WF-007 for the same workflow.  
**Delivery:** At-least-once | **Priority:** STANDARD | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No (too high volume)  
**Lifecycle:** ACTIVE

---

## WF-008: `workflows.task.completed`

**Description:** A task has successfully completed. Its output is available via `WorkflowClient.get_task_output()`.

**Producer:** `platform/workflow-runtime`  
**Consumers:** brain/experience-recorder, billing/cost-tracker (record task cost), monitoring/sla-monitor, desktop/workflow-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `task_id` | string | * | |
| `task_type` | string | * | |
| `agent_type` | string | * | |
| `product_id` | string | * | |
| `duration_ms` | integer | * | |
| `retry_count` | integer | * | Number of retries before success. 0 = first attempt succeeded. |
| `ai_execution_ids` | [UUID] | * | All AIClient execution IDs used to complete this task. |
| `total_tokens` | integer | * | Tokens consumed across all executions. |
| `total_cost_usd` | float | * | Cost of all executions. |
| `output_keys` | [string] | * | Keys present in TaskOutput.output (not the values). |
| `tool_calls_count` | integer | * | Number of tool calls made during execution. |

**Delivery:** At-least-once | **Priority:** STANDARD | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## WF-009: `workflows.task.failed`

**Description:** A task has failed. If `final_failure=true`, all retries are exhausted and the workflow will transition to FAILED.

**Producer:** `platform/workflow-runtime`  
**Consumers:** brain/failure-analyzer, monitoring/alert-manager, desktop/workflow-panel, monitoring/root-cause-engine

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `task_id` | string | * | |
| `task_type` | string | * | |
| `product_id` | string | * | |
| `retry_count` | integer | * | Current retry number. |
| `max_retries` | integer | * | |
| `final_failure` | boolean | * | True if this is the last retry. |
| `error_code` | string | * | |
| `error_message` | string | * | Sanitized — no stack traces. |
| `error_category` | enum(AI_EXECUTION, TOOL_FAILURE, TIMEOUT, AGENT_CRASH, DEPENDENCY_FAILED) | * | |
| `duration_ms` | integer | * | Time spent on this attempt. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes (final_failure=true only)  
**Lifecycle:** ACTIVE

---

## WF-010: `workflows.task.stalled`

**Description:** A task has been IN_PROGRESS for longer than the stall threshold without producing progress signals. The task is not yet failed — it may still complete.

**Producer:** `platform/workflow-runtime/sla-monitor`  
**Consumers:** monitoring/alert-manager, brain/root-cause-engine, desktop/workflow-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `task_id` | string | * | |
| `task_type` | string | * | |
| `product_id` | string | * | |
| `in_progress_since` | ISO8601 | * | When the task entered IN_PROGRESS state. |
| `stall_threshold_seconds` | integer | * | Configured threshold. |
| `elapsed_seconds` | integer | * | Time since task started (> stall_threshold_seconds). |
| `ai_execution_in_progress` | boolean | * | True if the task is currently waiting on an AI execution. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 7 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## WF-011: `workflows.gate.approved`

**Description:** A human approval gate has been approved. The workflow will resume from the next task.

**Producer:** `platform/workflow-runtime/approval-service`  
**Consumers:** platform/workflow-runtime (to resume), brain/experience-recorder, monitoring/audit-log, desktop/workflow-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `gate_id` | string | * | |
| `after_task_id` | string | * | Task the gate followed. |
| `before_task_id` | string | * | Task that will now execute. |
| `product_id` | string | * | |
| `approved_by` | string | * | Identity of the approver. |
| `approver_role` | enum(ADMIN, DEVELOPER, REVIEWER) | * | Role at time of approval. |
| `comment` | string \| null | ? | Reviewer comment. |
| `wait_duration_ms` | integer | * | How long the gate was pending before approval. |

**Ordering:** Guaranteed: WF-005 (paused at gate) precedes WF-011 (approved) for the same gate.  
**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** RESTRICTED (approver identity is sensitive) | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## WF-012: `workflows.gate.rejected`

**Description:** A human approval gate has been rejected. The workflow transitions to FAILED.

**Producer:** `platform/workflow-runtime/approval-service`  
**Consumers:** platform/workflow-runtime (to fail), brain/experience-recorder, monitoring/audit-log, desktop/workflow-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `gate_id` | string | * | |
| `after_task_id` | string | * | |
| `product_id` | string | * | |
| `rejected_by` | string | * | |
| `rejection_reason` | string | * | Required on rejection. |
| `wait_duration_ms` | integer | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** RESTRICTED | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

---

# Category 3: Prompt Events

**Stream:** PROMPTS  
**TTL (default):** 30 days  
**Priority (default):** LOW  
**Security Classification (default):** INTERNAL

Prompt events describe changes to the template lifecycle in Prompt OS. These are low-frequency, high-importance events — a template transition to ACTIVE can affect all products that use that template.

---

## PM-001: `prompts.template.registered`

**Description:** A new prompt template has been registered in Prompt OS. New templates are always created in DRAFT state.

**Producer:** `platform/prompt-os`  
**Consumers:** Desktop/prompt-panel, monitoring/prompt-registry-monitor

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `display_name` | string | * | |
| `owner` | string | * | Team or product. |
| `initial_version` | integer | * | Always 1. |
| `initial_state` | enum(DRAFT) | * | Always DRAFT. |
| `variable_count` | integer | * | Number of declared variables. |
| `has_system_prompt` | boolean | * | |
| `tags` | [string] | * | |
| `registered_by` | string | * | Identity. |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## PM-002: `prompts.template.updated`

**Description:** A new version of an existing template has been created. New versions are always DRAFT.

**Producer:** `platform/prompt-os`  
**Consumers:** Desktop/prompt-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `new_version` | integer | * | |
| `parent_version` | integer | * | |
| `change_notes` | string | * | |
| `content_changed` | boolean | * | False if only description/tags changed. |
| `variables_changed` | boolean | * | True if variables were added, removed, or renamed. |
| `hmac_signature` | string | * | HMAC of the new version content. |
| `updated_by` | string | * | |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## PM-003: `prompts.template.submitted_for_review`

**Description:** A template version has been submitted for governance review (DRAFT → REVIEW transition).

**Producer:** `platform/prompt-os`  
**Consumers:** Desktop/prompt-panel, monitoring/governance-tracker

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `version` | integer | * | |
| `display_name` | string | * | |
| `submitted_by` | string | * | |
| `reviewer_required_role` | enum(REVIEWER, ADMIN) | * | Minimum role to approve. |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Audit Required:** Yes (governance state changes are audited) | **Lifecycle:** ACTIVE

---

## PM-004: `prompts.template.activated`

**Description:** A template version has been promoted to ACTIVE state and is now the canonical version for renders. This is the most critical prompt event — it immediately affects all subsequent renders.

**Producer:** `platform/prompt-os`  
**Consumers:** platform/prompt-os (cache invalidation), brain/pattern-updater (record new template baseline), desktop/prompt-panel, monitoring/prompt-registry-monitor, all products using this template

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `version` | integer | * | The version now active. |
| `previous_version` | integer \| null | * | Previously active version. Null if first activation. |
| `display_name` | string | * | |
| `owner` | string | * | |
| `activated_by` | string | * | |
| `estimated_render_impact` | integer | * | Approximate renders per day using this template (from analytics). |
| `hmac_signature` | string | * | HMAC of the now-active content. |
| `ab_test_active` | boolean | * | True if an A/B test is currently running on this template. |

**Ordering:** Guaranteed within `template_id`. PM-003 precedes PM-004 for the same template version.  
**Delivery:** At-least-once | **Retry:** 5 attempts (critical — must reach cache invalidation consumers)  
**DLQ:** `prompts.template.activated.dlq`  
**TTL:** 30 days | **Priority:** HIGH | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## PM-005: `prompts.template.deprecated`

**Description:** A template has been marked as deprecated. Existing renders continue to work; no new integrations should use this template.

**Producer:** `platform/prompt-os`  
**Consumers:** Desktop/prompt-panel, monitoring/deprecation-tracker, all registered consumers of this template

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `version` | integer | * | The deprecated version. |
| `display_name` | string | * | |
| `deprecated_by` | string | * | |
| `deprecation_reason` | string | * | |
| `removal_after` | ISO8601 | * | Date after which the template may be archived. |
| `replacement_template_id` | UUID \| null | ? | Recommended replacement. |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Audit Required:** Yes | **Lifecycle:** ACTIVE

---

## PM-006: `prompts.template.suspended`

**Description:** A template has been emergency-suspended. All renders using this template will fail immediately. This is used when a security issue is discovered in a template.

**Producer:** `platform/prompt-os`  
**Consumers:** platform/prompt-os (immediate cache invalidation + block renders), monitoring/alert-manager (CRITICAL alert), desktop/prompt-panel, security/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `version` | integer | * | |
| `suspended_by` | string | * | |
| `suspension_reason` | string | * | |
| `suspension_category` | enum(SECURITY_ISSUE, CONTENT_VIOLATION, DATA_ERROR, ADMIN_ACTION) | * | |
| `renders_blocked` | boolean | * | Always true. Included for consumer clarity. |
| `incident_id` | string \| null | ? | Incident tracking ID if suspension is part of an incident response. |

**Delivery:** At-least-once | **Retry:** 10 attempts (must reach cache invalidation) | **Priority:** CRITICAL  
**TTL:** 30 days | **Version:** 1.0 | **Security Classification:** RESTRICTED | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## PM-007: `prompts.template.render_failed`

**Description:** A template render failed due to HMAC signature mismatch (tamper detection). Published for every failed render due to security violation — not for other render failures.

**Producer:** `platform/prompt-os`  
**Consumers:** security/audit-log, monitoring/alert-manager (potential tampering alert)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `version` | integer | * | |
| `failure_reason` | enum(HMAC_MISMATCH, SIGNATURE_MISSING, KEY_NOT_FOUND) | * | |
| `render_caller` | string | * | Identity that attempted the render. |
| `detected_at` | ISO8601 | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 30 days | **Version:** 1.0  
**Security Classification:** RESTRICTED | **Audit Required:** Yes (all HMAC failures are security events)  
**Lifecycle:** ACTIVE

---

## PM-008: `prompts.ab_test.winner_declared`

**Description:** A Prompt OS A/B test has concluded with a statistically significant winner.

**Producer:** `platform/prompt-os`  
**Consumers:** brain/pattern-updater, desktop/prompt-panel, template owner notification

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `template_id` | UUID | * | |
| `test_id` | UUID | * | |
| `winner_variant` | enum(A, B) | * | |
| `winner_version` | integer | * | The winning template version. |
| `loser_version` | integer | * | |
| `sample_size` | integer | * | Total renders in the test. |
| `winner_success_rate` | float | * | Success rate of the winning variant. |
| `loser_success_rate` | float | * | |
| `confidence_pct` | float | * | Statistical confidence (e.g., 95.0). |
| `test_duration_hours` | float | * | |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

---

# Category 4: Provider Events

**Stream:** AI  
**TTL (default):** 3 days  
**Priority (default):** HIGH  
**Security Classification (default):** INTERNAL

Provider events describe the health and operational state of AI provider adapters. These events drive circuit breaker behavior and provider failover decisions.

---

## PV-001: `ai.provider.circuit_opened`

**Description:** A provider's circuit breaker has tripped to OPEN state due to consecutive failures. Requests will not be routed to this provider until the circuit closes.

**Producer:** `platform/ai-ros/circuit-breaker`  
**Consumers:** platform/ai-ros (routing — exclude this provider), monitoring/alert-manager, desktop/workspace-panel, brain/provider-reliability-tracker

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider_id` | string | * | |
| `failure_count` | integer | * | Failures in the window that triggered the trip. |
| `failure_window_seconds` | integer | * | The measurement window. |
| `failure_threshold` | integer | * | The configured threshold that was exceeded. |
| `circuit_open_until` | ISO8601 | * | When the circuit will transition to HALF_OPEN for testing. |
| `last_error_code` | string | * | Error code of the final triggering failure. |
| `last_error_category` | enum(TIMEOUT, SERVER_ERROR, NETWORK_ERROR, AUTH_ERROR) | * | |
| `affected_models` | [string] | * | Models that are now unavailable. |
| `fallback_providers` | [string] | * | Currently healthy providers that will receive rerouted traffic. |

**Validation Rules:** `failure_count` >= `failure_threshold`  
**Ordering:** Guaranteed per provider_id. PV-001 precedes PV-003 for same provider.  
**Delivery:** At-least-once | **Retry:** 5 attempts | **DLQ:** `ai.provider.circuit_opened.dlq`  
**TTL:** 3 days | **Priority:** HIGH | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes

**Example Payload:**
```
event_type:    ai.provider.circuit_opened
schema_version: "1.0"
source:        platform/ai-ros/circuit-breaker
payload:
  provider_id:           "anthropic"
  failure_count:         5
  failure_window_seconds: 60
  failure_threshold:     5
  circuit_open_until:    "2026-06-28T15:31:00Z"
  last_error_code:       "PROVIDER_TIMEOUT"
  last_error_category:   "TIMEOUT"
  affected_models:       ["claude-sonnet-4-6", "claude-haiku-4-5-20251001"]
  fallback_providers:    ["openai", "claude_code"]
```

**Lifecycle:** ACTIVE

---

## PV-002: `ai.provider.circuit_half_opened`

**Description:** A provider's circuit breaker has transitioned to HALF_OPEN state. A probe request will be sent; success closes the circuit, failure re-opens it.

**Producer:** `platform/ai-ros/circuit-breaker`  
**Consumers:** platform/ai-ros (routing — send one probe), monitoring/alert-manager, desktop/workspace-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider_id` | string | * | |
| `open_duration_seconds` | integer | * | How long the circuit was OPEN. |
| `probe_timeout_seconds` | integer | * | If probe doesn't complete within this time, circuit re-opens. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 3 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## PV-003: `ai.provider.circuit_closed`

**Description:** A provider's circuit breaker has returned to CLOSED (normal) state. The provider is again eligible for request routing.

**Producer:** `platform/ai-ros/circuit-breaker`  
**Consumers:** platform/ai-ros (routing — restore to candidate pool), monitoring/alert-manager, desktop/workspace-panel, brain/provider-reliability-tracker

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider_id` | string | * | |
| `open_duration_seconds` | integer | * | Total time circuit was open. |
| `recovery_method` | enum(PROBE_SUCCESS, MANUAL_RESET) | * | |
| `recovered_at` | ISO8601 | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 3 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## PV-004: `ai.provider.health_degraded`

**Description:** A provider's health check returned DEGRADED status. The provider is still routing but with increased latency or elevated error rates.

**Producer:** `platform/ai-ros/health-monitor`  
**Consumers:** platform/ai-ros (lower routing priority), monitoring/alert-manager, desktop/workspace-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider_id` | string | * | |
| `latency_p50_ms` | float | * | |
| `latency_p99_ms` | float | * | |
| `error_rate_1m` | float | * | Error rate in the last minute. |
| `degradation_cause` | string | * | |
| `health_check_at` | ISO8601 | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 3 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## PV-005: `ai.provider.registered`

**Description:** A new AI provider adapter has been registered in the ProviderRegistry. Emitted at platform startup or when a provider is added dynamically.

**Producer:** `platform/ai-ros/registry`  
**Consumers:** monitoring/provider-catalog, desktop/workspace-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider_id` | string | * | |
| `display_name` | string | * | |
| `models` | [string] | * | Model IDs. |
| `supports_streaming` | boolean | * | |
| `supports_function_calling` | boolean | * | |
| `supports_vision` | boolean | * | |
| `initial_health` | enum(HEALTHY, DEGRADED, OFFLINE) | * | Health at registration time. |

**Delivery:** At-least-once | **Priority:** LOW | **TTL:** 3 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## PV-006: `ai.provider.rate_limited`

**Description:** A provider has returned a rate limit response (HTTP 429). AI ROS is backing off and honoring the provider's Retry-After.

**Producer:** `platform/ai-ros`  
**Consumers:** monitoring/alert-manager, brain/provider-reliability-tracker

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `provider_id` | string | * | |
| `model_id` | string | * | |
| `retry_after_seconds` | integer | * | From provider's Retry-After header. |
| `requests_queued` | integer | * | Requests queued behind this rate limit. |
| `rate_limit_type` | enum(REQUEST_LIMIT, TOKEN_LIMIT, CONCURRENT_LIMIT) | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 3 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

---

# Category 5: AI Model Events

**Stream:** AI  
**TTL (default):** 3 days  
**Priority (default):** STANDARD  
**Security Classification (default):** INTERNAL

AI model events describe execution-level outcomes — completions, cost, and budget status. These are the highest-volume events in the AI stream.

---

## AM-001: `ai.execution.started`

**Description:** An AI execution request has been dequeued from the scheduling queue and the provider API call has been initiated.

**Producer:** `platform/ai-ros`  
**Consumers:** monitoring/execution-tracker, billing/cost-accumulator (open billing context)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `execution_id` | UUID | * | |
| `product_id` | string | * | |
| `workflow_id` | UUID \| null | ? | If within a workflow. |
| `task_id` | string \| null | ? | |
| `provider_id` | string | * | |
| `model_id` | string | * | |
| `priority` | enum(URGENT, STANDARD, BATCH) | * | |
| `estimated_cost_usd` | float | * | Pre-execution estimate. |
| `queue_wait_ms` | integer | * | Time spent in scheduling queue before this call. |
| `has_tools` | boolean | * | |

**Delivery:** At-least-once | **Priority:** STANDARD | **TTL:** 3 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No (too high volume)  
**Lifecycle:** ACTIVE

---

## AM-002: `ai.execution.completed`

**Description:** An AI execution has completed successfully. This is the primary event for cost tracking and experience recording.

**Producer:** `platform/ai-ros`  
**Consumers:** billing/cost-accumulator, brain/experience-recorder, usage/token-tracker, monitoring/execution-tracker, desktop/cost-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `execution_id` | UUID | * | |
| `product_id` | string | * | |
| `workflow_id` | UUID \| null | ? | |
| `task_id` | string \| null | ? | |
| `provider_id` | string | * | |
| `model_id` | string | * | |
| `prompt_tokens` | integer | * | |
| `completion_tokens` | integer | * | |
| `cache_read_tokens` | integer | * | Prompt cache hits (0 if no caching). |
| `cache_write_tokens` | integer | * | |
| `total_tokens` | integer | * | |
| `cost_usd` | float | * | Actual cost. |
| `duration_ms` | integer | * | |
| `provider_latency_ms` | integer | * | |
| `queue_wait_ms` | integer | * | |
| `stop_reason` | enum(END_TURN, MAX_TOKENS, STOP_SEQUENCE, TOOL_USE) | * | |
| `retry_count` | integer | * | 0 if first attempt succeeded. |
| `tool_calls_count` | integer | * | |

**Delivery:** At-least-once | **Priority:** STANDARD | **TTL:** 3 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No (too high volume — billing events are audited instead)  
**Lifecycle:** ACTIVE

---

## AM-003: `ai.execution.failed`

**Description:** An AI execution has failed after all retries. The requesting caller has received an error.

**Producer:** `platform/ai-ros`  
**Consumers:** billing/cost-accumulator (record partial cost if any tokens consumed), monitoring/alert-manager, brain/failure-analyzer

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `execution_id` | UUID | * | |
| `product_id` | string | * | |
| `workflow_id` | UUID \| null | ? | |
| `task_id` | string \| null | ? | |
| `error_code` | string | * | |
| `error_category` | enum(BUDGET_EXCEEDED, QUOTA_EXCEEDED, NO_PROVIDER, TIMEOUT, VALIDATION_FAILED) | * | |
| `providers_attempted` | [string] | * | All providers tried before failure. |
| `total_attempts` | integer | * | |
| `partial_cost_usd` | float | * | Cost incurred before failure (may be 0). |
| `partial_tokens` | integer | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 3 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## AM-004: `ai.budget.threshold_reached`

**Description:** A product's AI spend has crossed the warning threshold (default 80% of daily limit). No requests are blocked yet — this is a warning.

**Producer:** `platform/ai-ros/budget-manager`  
**Consumers:** monitoring/alert-manager, desktop/cost-panel, billing/budget-monitor

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `daily_limit_usd` | float | * | |
| `daily_spend_usd` | float | * | |
| `threshold_pct` | float | * | The threshold that was crossed (e.g., 80.0). |
| `spend_pct` | float | * | Current spend as percentage of limit. |
| `estimated_limit_reached_at` | ISO8601 \| null | ? | Projected time to reach 100%, based on current burn rate. |
| `reset_at` | ISO8601 | * | When daily limit resets. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 3 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## AM-005: `ai.budget.limit_exceeded`

**Description:** A product's daily AI spend has reached 100% of its limit. All further AI requests from this product are rejected until the limit resets.

**Producer:** `platform/ai-ros/budget-manager`  
**Consumers:** monitoring/alert-manager (CRITICAL alert), desktop/cost-panel, billing/budget-monitor, platform/ai-ros (enforce rejection), product teams

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `daily_limit_usd` | float | * | |
| `daily_spend_usd` | float | * | |
| `excess_usd` | float | * | How much the rejected request would have cost beyond the limit. |
| `rejected_execution_id` | UUID | * | The execution that triggered this event (was rejected). |
| `reset_at` | ISO8601 | * | When requests will be allowed again. |
| `total_executions_today` | integer | * | |

**Delivery:** At-least-once | **Priority:** CRITICAL | **TTL:** 3 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## AM-006: `ai.quota.rate_limit_exceeded`

**Description:** A product has exceeded its per-minute AI request quota. The triggering request was rejected.

**Producer:** `platform/ai-ros/quota-manager`  
**Consumers:** monitoring/alert-manager, desktop/cost-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `quota_rpm` | integer | * | Configured limit (requests per minute). |
| `requests_this_minute` | integer | * | |
| `reset_at` | ISO8601 | * | When the minute window resets. |
| `rejected_execution_id` | UUID | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 3 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---



---

# Category 6: Knowledge Events

**Stream:** KNOWLEDGE  
**TTL (default):** 30 days  
**Priority (default):** LOW  
**Security Classification (default):** INTERNAL

Knowledge events describe changes to the Central Knowledge Graph and the Brain's pattern learning. These are relatively low-frequency but high-value events for AI behavior improvement.

---

## KN-001: `knowledge.node.added`

**Description:** A new node has been added to the knowledge graph.

**Producer:** `platform/knowledge`  
**Consumers:** brain/pattern-indexer, monitoring/knowledge-monitor

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `node_id` | string | * | |
| `node_type` | string | * | e.g., `"File"`, `"Module"`, `"Concept"`, `"Person"` |
| `label` | string | * | Human-readable label. |
| `properties_count` | integer | * | Number of properties set on the node. |
| `relationship_count` | integer | * | Number of existing relationships to/from this node at creation. |
| `workspace_id` | string \| null | ? | If workspace-scoped. |
| `added_by` | string | * | Module or identity. |

**Validation Rules:** `node_id` must be unique in the graph.  
**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## KN-002: `knowledge.node.updated`

**Description:** Properties on an existing knowledge graph node have been updated.

**Producer:** `platform/knowledge`  
**Consumers:** brain/pattern-indexer

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `node_id` | string | * | |
| `node_type` | string | * | |
| `properties_changed` | [string] | * | Keys of properties that changed. |
| `updated_by` | string | * | |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## KN-003: `knowledge.relationship.added`

**Description:** A directional relationship has been added between two knowledge graph nodes.

**Producer:** `platform/knowledge`  
**Consumers:** brain/pattern-indexer

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `relationship_id` | string | * | |
| `relationship_type` | string | * | e.g., `"DEPENDS_ON"`, `"AUTHORED_BY"`, `"RELATES_TO"` |
| `from_node_id` | string | * | |
| `to_node_id` | string | * | |
| `from_node_type` | string | * | |
| `to_node_type` | string | * | |
| `properties` | object | * | Key-value pairs on the relationship (empty object if none). |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## KN-004: `knowledge.pattern.indexed`

**Description:** The knowledge indexer has completed a pattern indexing pass. This indicates the graph is up-to-date for pattern queries.

**Producer:** `platform/knowledge/indexer`  
**Consumers:** brain/pattern-engine, monitoring/knowledge-monitor

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `index_run_id` | UUID | * | |
| `nodes_indexed` | integer | * | |
| `relationships_indexed` | integer | * | |
| `new_patterns_discovered` | integer | * | Patterns not seen in the previous run. |
| `duration_ms` | integer | * | |
| `index_size_mb` | float | * | |
| `backend` | enum(SQL, KUZU, NEO4J) | * | Current graph backend. |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## KN-005: `knowledge.memory.set`

**Description:** A persistent memory value has been stored in the knowledge system.

**Producer:** `platform/knowledge`  
**Consumers:** monitoring/knowledge-monitor (if memory exceeds size threshold)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `memory_key` | string | * | |
| `scope` | enum(GLOBAL, WORKSPACE, PRODUCT, SESSION) | * | |
| `scope_id` | string \| null | ? | Populated if scope is not GLOBAL. |
| `value_size_bytes` | integer | * | Size of the stored value. |
| `ttl_seconds` | integer \| null | ? | Null means no expiry. |
| `set_by` | string | * | |

**Validation Rules:** `value_size_bytes` must be <= 1MB (platform limit).  
**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

---

# Category 7: Authentication Events

**Stream:** SECURITY  
**TTL (default):** 90 days  
**Priority (default):** HIGH  
**Security Classification (default):** RESTRICTED

Authentication events are identity-lifecycle events. They are always written to the immutable audit log. Long retention (90 days) supports compliance and forensic investigation.

---

## AU-001: `security.auth.succeeded`

**Description:** A request was successfully authenticated. Published for API key authentication; not published for every request (that would be too high volume) — only for explicitly significant auth events per platform policy.

**Policy Note:** This event is published: (1) first authentication of a new API key; (2) first authentication after key was dormant for >24 hours; (3) authentication from a new IP address for a key.

**Producer:** `platform/security/auth-service`  
**Consumers:** monitoring/auth-tracker, brain/behavior-baseline

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `identity` | string | * | Hashed key identifier. Never the raw key. |
| `role` | enum(VIEWER, DEVELOPER, REVIEWER, ADMIN) | * | |
| `ip_hash` | string | * | SHA-256 of client IP. Never the raw IP. |
| `auth_reason` | enum(NEW_KEY, DORMANT_KEY, NEW_IP) | * | Why this auth event was published. |
| `product_id` | string \| null | ? | If the request was product-scoped. |

**Ordering:** Not guaranteed. Auth events may arrive out of order.  
**Delivery:** At-least-once | **TTL:** 90 days | **Priority:** HIGH | **Version:** 1.0  
**Security Classification:** RESTRICTED | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## AU-002: `security.auth.failed`

**Description:** Authentication failed. Published for every authentication failure.

**Producer:** `platform/security/auth-service`  
**Consumers:** monitoring/alert-manager (track failure rate; alert if spike), monitoring/audit-log, brain/threat-detector

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `failure_reason` | enum(KEY_NOT_FOUND, KEY_REVOKED, KEY_EXPIRED, MALFORMED_HEADER) | * | |
| `ip_hash` | string | * | SHA-256 of client IP. |
| `path` | string | * | Request path (not query string). |
| `method` | string | * | HTTP method. |
| `key_prefix` | string \| null | ? | First 8 chars of attempted key (for support diagnosis). Null for MALFORMED_HEADER. |

**Validation Rules:** No full key values, raw IPs, or tokens may appear in any field.  
**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** RESTRICTED | **Audit Required:** Yes

**Example Payload:**
```
event_type:    security.auth.failed
schema_version: "1.0"
source:        platform/security/auth-service
payload:
  failure_reason: "KEY_REVOKED"
  ip_hash:        "a3f2c1d5e8b7..."
  path:           "/api/v1/products"
  method:         "POST"
  key_prefix:     "sk_live_ab"
```

**Lifecycle:** ACTIVE

---

## AU-003: `security.permission.denied`

**Description:** An authenticated request was rejected because the identity lacked the required permission.

**Producer:** `platform/security/auth-service`  
**Consumers:** monitoring/alert-manager (track denial rate), monitoring/audit-log, brain/behavior-anomaly-detector

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `identity` | string | * | Hashed identifier. |
| `role` | enum(VIEWER, DEVELOPER, REVIEWER, ADMIN) | * | Role the identity holds. |
| `required_permission` | string | * | e.g., `"workflow:write"` |
| `path` | string | * | |
| `method` | string | * | |
| `resource_id` | string \| null | ? | If permission check was for a specific resource. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** RESTRICTED | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## AU-004: `security.api_key.created`

**Description:** A new API key has been created.

**Producer:** `platform/security/key-service`  
**Consumers:** monitoring/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `key_id` | UUID | * | Internal key identifier. Not the key value. |
| `key_prefix` | string | * | First 8 chars (for display only). |
| `role` | enum(VIEWER, DEVELOPER, REVIEWER, ADMIN) | * | |
| `created_by` | string | * | Identity that created the key. |
| `label` | string | * | Human-readable label. |
| `expires_at` | ISO8601 \| null | ? | Null means no expiry. |
| `scope` | enum(GLOBAL, PRODUCT) | * | |
| `product_id` | string \| null | ? | Populated if scope=PRODUCT. |

**Validation Rules:** Key value must never appear anywhere in this event.  
**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** CONFIDENTIAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## AU-005: `security.api_key.revoked`

**Description:** An API key has been revoked. All subsequent requests using this key will fail with AU-002.

**Producer:** `platform/security/key-service`  
**Consumers:** platform/security (invalidate key cache), monitoring/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `key_id` | UUID | * | |
| `key_prefix` | string | * | |
| `revoked_by` | string | * | |
| `revocation_reason` | enum(USER_REQUEST, SECURITY_INCIDENT, ADMIN_ACTION, KEY_COMPROMISED, EXPIRED) | * | |
| `was_active` | boolean | * | False if key was never used or already expired. |
| `last_used_at` | ISO8601 \| null | ? | |

**Delivery:** At-least-once | **Priority:** CRITICAL (must propagate before next request arrives) | **Retry:** 10 attempts  
**TTL:** 90 days | **Version:** 1.0 | **Security Classification:** CONFIDENTIAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## AU-006: `security.rate_limit.exceeded`

**Description:** A rate limit has been hit for an identity, path, or IP. The triggering request was rejected.

**Producer:** `platform/security/rate-limiter`  
**Consumers:** monitoring/alert-manager, monitoring/audit-log, brain/threat-detector

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `identity` | string \| null | ? | Hashed. Null if rate limit is IP-based before auth. |
| `ip_hash` | string | * | |
| `limit_type` | enum(GLOBAL_RPM, PER_KEY_RPM, PER_PATH_RPM, PER_IP_RPM) | * | |
| `limit_value` | integer | * | Configured limit. |
| `actual_rate` | integer | * | Observed rate at time of rejection. |
| `path` | string | * | |
| `retry_after_seconds` | integer | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** RESTRICTED | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

---

# Category 8: Security Events

**Stream:** SECURITY  
**TTL (default):** 90 days  
**Priority (default):** CRITICAL  
**Security Classification (default):** CONFIDENTIAL

Security events describe detected threats, policy violations, and security configuration changes. Distinct from authentication events: these are signals of potentially hostile or anomalous activity.

---

## SC-001: `security.prompt_injection.detected`

**Description:** The prompt validation system has detected a likely prompt injection attempt in a request payload.

**Producer:** `platform/security/prompt-validator`  
**Consumers:** monitoring/alert-manager (CRITICAL), monitoring/audit-log, brain/threat-analyzer

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `identity` | string | * | Hashed. |
| `path` | string | * | |
| `detection_method` | enum(PATTERN_MATCH, HEURISTIC, ML_MODEL) | * | |
| `confidence` | float | * | Confidence score [0.0, 1.0]. |
| `action_taken` | enum(BLOCKED, SANITIZED, LOGGED_ONLY) | * | |
| `pattern_id` | string \| null | ? | If detection was pattern-based. |
| `payload_hash` | string | * | SHA-256 of the suspicious content. Not the content itself. |

**Validation Rules:** Raw content must never appear in this event. Only the SHA-256 hash.  
**Delivery:** At-least-once | **Priority:** CRITICAL | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** CONFIDENTIAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## SC-002: `security.tool_sandbox.violation`

**Description:** A tool execution attempted an operation outside its declared sandbox (e.g., a tool declared as read-only attempted to write).

**Producer:** `platform/security/tool-sandbox`  
**Consumers:** monitoring/alert-manager (CRITICAL), monitoring/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `tool_name` | string | * | |
| `plugin_id` | string \| null | ? | If the tool belongs to a plugin. |
| `declared_permissions` | [string] | * | What the tool declared it would do. |
| `attempted_operation` | string | * | The operation that was blocked. |
| `blocked` | boolean | * | Always true. |
| `execution_id` | UUID \| null | ? | If the tool was invoked during an AI execution. |

**Delivery:** At-least-once | **Priority:** CRITICAL | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** CONFIDENTIAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## SC-003: `security.anomaly.detected`

**Description:** The behavior anomaly detector has identified a statistically unusual access pattern for an identity.

**Producer:** `platform/security/anomaly-detector`  
**Consumers:** monitoring/alert-manager, monitoring/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `identity` | string | * | Hashed. |
| `anomaly_type` | enum(UNUSUAL_VOLUME, UNUSUAL_PATHS, UNUSUAL_HOURS, UNUSUAL_IP, DATA_EXFILTRATION_PATTERN) | * | |
| `anomaly_score` | float | * | Score [0.0, 1.0]. 1.0 = maximum anomaly. |
| `baseline_rpm` | float | * | Identity's historical baseline. |
| `current_rpm` | float | * | Current observed rate. |
| `observation_window_minutes` | integer | * | |
| `recommendation` | enum(MONITOR, THROTTLE, INVESTIGATE, SUSPEND) | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** CONFIDENTIAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## SC-004: `security.config.changed`

**Description:** A security-sensitive configuration value has been changed (rate limits, budget limits, auth settings).

**Producer:** `platform/security/config-auditor`  
**Consumers:** monitoring/audit-log, monitoring/alert-manager

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `config_key` | string | * | The configuration key that changed. |
| `changed_by` | string | * | Identity. Must be ADMIN role. |
| `previous_value_hash` | string | * | SHA-256 of previous value. Not the value itself. |
| `new_value_hash` | string | * | SHA-256 of new value. |
| `change_reason` | string | * | Required for all security config changes. |

**Validation Rules:** No config values may appear. Only hashes.  
**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** CONFIDENTIAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

---

# Category 9: Desktop Events

**Stream:** DESKTOP  
**Retention Policy:** INTEREST (ephemeral — events retained only while consumers exist)  
**TTL (default):** 1 hour  
**Priority (default):** LOW  
**Security Classification (default):** INTERNAL

Desktop events are UI signals between the FastAPI backend and the PySide6 desktop application. They are delivered via WebSocket bridge (in-process EventBus → WebSocket → desktop). These events are NOT persisted to JetStream in the interest retention stream beyond active consumers — they are fire-and-forget UI update signals.

---

## DS-001: `desktop.ui.workflow_panel_update`

**Description:** The workflow panel should refresh its display. Emitted when a workflow state changes that a human user would want to see.

**Producer:** `platform/workflow-runtime` (via in-process EventBus)  
**Consumers:** desktop/workflow-panel (via WebSocket bridge)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `workflow_id` | UUID | * | |
| `update_type` | enum(STATE_CHANGE, TASK_PROGRESS, GATE_PENDING, GATE_RESOLVED, COST_UPDATE) | * | |
| `new_state` | string \| null | ? | If update_type=STATE_CHANGE. |
| `task_id` | string \| null | ? | If update_type=TASK_PROGRESS. |
| `task_progress_pct` | integer \| null | ? | If update_type=TASK_PROGRESS. |
| `gate_id` | string \| null | ? | If update_type=GATE_PENDING or GATE_RESOLVED. |

**Delivery:** At-most-once (WebSocket best-effort — desktop can miss updates and refresh on reconnect)  
**TTL:** 1 hour | **Priority:** LOW | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## DS-002: `desktop.ui.cost_panel_update`

**Description:** The cost panel should refresh its totals.

**Producer:** `platform/ai-ros/budget-manager`, `billing/cost-tracker`  
**Consumers:** desktop/cost-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string \| null | ? | Null means platform-wide cost update. |
| `daily_spend_usd` | float | * | Current day's spend. |
| `daily_limit_usd` | float | * | |
| `spend_pct` | float | * | |
| `update_cause` | enum(EXECUTION_COMPLETED, BUDGET_THRESHOLD, BUDGET_EXCEEDED, PERIODIC_REFRESH) | * | |

**Delivery:** At-most-once | **TTL:** 1 hour | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## DS-003: `desktop.ui.workspace_panel_update`

**Description:** The workspace panel should refresh its module health display.

**Producer:** `platform/workspace/health-monitor`  
**Consumers:** desktop/workspace-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `module_id` | string | * | |
| `new_status` | enum(HEALTHY, DEGRADED, OFFLINE) | * | |
| `previous_status` | enum(HEALTHY, DEGRADED, OFFLINE) | * | |

**Delivery:** At-most-once | **TTL:** 1 hour | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## DS-004: `desktop.ui.notification`

**Description:** A user-facing notification should be displayed in the desktop app.

**Producer:** Any platform module that needs to surface a message to the user.  
**Consumers:** desktop/notification-center

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | * | Short notification title. |
| `message` | string | * | |
| `severity` | enum(INFO, WARNING, ERROR, CRITICAL) | * | |
| `auto_dismiss_seconds` | integer \| null | ? | If null, notification requires manual dismissal. |
| `action_label` | string \| null | ? | Label for optional action button. |
| `action_event` | string \| null | ? | Event name to publish when action is clicked. |
| `source_module` | string | * | Which module sent this notification. |

**Delivery:** At-most-once | **TTL:** 1 hour | **Priority:** STANDARD | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## DS-005: `desktop.session.connected`

**Description:** A desktop client WebSocket session has connected to the platform.

**Producer:** `platform/desktop-bridge`  
**Consumers:** monitoring/desktop-monitor

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session_id` | UUID | * | |
| `identity` | string | * | Hashed. |
| `platform_version` | string | * | |
| `desktop_version` | string | * | Version of the PySide6 app. |
| `connection_type` | enum(LOCAL, REMOTE) | * | |

**Delivery:** At-most-once | **TTL:** 1 hour | **Priority:** LOW | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## DS-006: `desktop.session.disconnected`

**Description:** A desktop client WebSocket session has disconnected.

**Producer:** `platform/desktop-bridge`  
**Consumers:** monitoring/desktop-monitor

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `session_id` | UUID | * | |
| `duration_seconds` | integer | * | |
| `disconnect_reason` | enum(CLIENT_CLOSE, NETWORK_ERROR, SERVER_CLOSE, TIMEOUT) | * | |

**Delivery:** At-most-once | **TTL:** 1 hour | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

---

# Category 10: Marketplace Events

**Stream:** Not implemented yet — planned for v2.5 (multi-tenant)  
**Planned Stream:** MARKETPLACE  
**Status:** All events in this category are PROPOSED — not yet active.

Marketplace events will support the optional AI Studio Marketplace (product templates, prompt packs, plugin distributions). The marketplace is feature-flagged OFF at launch.

---

## MK-001: `marketplace.product.listed` (PROPOSED)

**Description:** A product template has been published to the marketplace.

**Producer:** `marketplace/listing-service`  
**Consumers:** Desktop/marketplace-panel, marketing/analytics

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `listing_id` | UUID | * | |
| `product_template_name` | string | * | |
| `category` | string | * | |
| `author_id` | string | * | Hashed author identity. |
| `price_type` | enum(FREE, PAID, SUBSCRIPTION) | * | |
| `price_usd` | float \| null | ? | Null for FREE. |
| `version` | string | * | |
| `tags` | [string] | * | |

**Lifecycle:** PROPOSED

---

## MK-002: `marketplace.product.installed` (PROPOSED)

**Description:** A user has installed a marketplace product template into their workspace.

**Producer:** `marketplace/install-service`  
**Consumers:** billing/marketplace-billing, brain/usage-tracker, workspace/configurator

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `installation_id` | UUID | * | |
| `listing_id` | UUID | * | |
| `workspace_id` | string | * | |
| `installed_by` | string | * | Hashed. |
| `license_key` | string \| null | ? | For paid listings. |

**Lifecycle:** PROPOSED

---

## MK-003: `marketplace.review.submitted` (PROPOSED)

**Description:** A user has submitted a review for a marketplace listing.

**Producer:** `marketplace/review-service`  
**Consumers:** marketplace/rating-calculator, brain/review-analyzer

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `review_id` | UUID | * | |
| `listing_id` | UUID | * | |
| `reviewer_id` | string | * | Hashed. |
| `rating` | integer | * | 1–5. |

**Note:** Review text is not included in the event payload. Consumers fetch it from the review service API.  
**Lifecycle:** PROPOSED

---

---

# Category 11: Plugin Events

**Stream:** PLUGINS  
**TTL (default):** 30 days  
**Priority (default):** STANDARD  
**Security Classification (default):** INTERNAL

Plugin events describe the lifecycle of third-party plugins installed in the platform. Plugins extend platform capabilities (workers, tools, prompt packs, knowledge providers).

---

## PL-001: `plugins.plugin.installed`

**Description:** A plugin has been installed and its sandboxed runtime has been initialized.

**Producer:** `platform/plugin-manager`  
**Consumers:** monitoring/plugin-monitor, security/plugin-auditor, desktop/workspace-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `plugin_id` | string | * | |
| `plugin_name` | string | * | |
| `version` | string | * | |
| `plugin_type` | enum(WORKER, TOOL, PROMPT, KNOWLEDGE) | * | |
| `installed_by` | string | * | |
| `declared_permissions` | [string] | * | Capabilities the plugin declared it needs. |
| `granted_permissions` | [string] | * | Permissions actually granted (admin may restrict). |
| `sandbox_config` | object | * | Sanitized sandbox configuration (no secrets). |
| `integrity_hash` | string | * | SHA-256 of the plugin bundle. |
| `signature_verified` | boolean | * | Whether the plugin's digital signature was verified. |

**Validation Rules:** `signature_verified` must be `true` for all production environments. Unsigned plugins may only be installed in local/dev environments.  
**Delivery:** At-least-once | **Priority:** STANDARD | **TTL:** 30 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## PL-002: `plugins.plugin.uninstalled`

**Description:** A plugin has been removed from the platform.

**Producer:** `platform/plugin-manager`  
**Consumers:** monitoring/plugin-monitor, desktop/workspace-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `plugin_id` | string | * | |
| `plugin_name` | string | * | |
| `version` | string | * | |
| `uninstalled_by` | string | * | |
| `reason` | enum(USER_REQUEST, ADMIN_ACTION, SECURITY_VIOLATION, INSTALL_FAILURE, UPGRADE) | * | |
| `uptime_seconds` | integer | * | |
| `execution_count` | integer | * | Total times the plugin was invoked while installed. |

**Delivery:** At-least-once | **TTL:** 30 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## PL-003: `plugins.plugin.enabled`

**Description:** A disabled plugin has been re-enabled.

**Producer:** `platform/plugin-manager`  
**Consumers:** monitoring/plugin-monitor, desktop/workspace-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `plugin_id` | string | * | |
| `enabled_by` | string | * | |
| `was_disabled_since` | ISO8601 | * | |

**Delivery:** At-least-once | **TTL:** 30 days | **Version:** 1.0 | **Lifecycle:** ACTIVE

---

## PL-004: `plugins.plugin.disabled`

**Description:** A plugin has been disabled (not uninstalled — it remains installed but will not receive invocations).

**Producer:** `platform/plugin-manager`  
**Consumers:** monitoring/plugin-monitor, desktop/workspace-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `plugin_id` | string | * | |
| `disabled_by` | string | * | |
| `reason` | enum(USER_REQUEST, ADMIN_ACTION, SECURITY_VIOLATION, RESOURCE_OVERUSE, HEALTH_FAILURE) | * | |
| `is_temporary` | boolean | * | True if disabled pending review (may be re-enabled). False if permanently disabled. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 30 days | **Version:** 1.0  
**Audit Required:** Yes | **Lifecycle:** ACTIVE

---

## PL-005: `plugins.plugin.execution_failed`

**Description:** A plugin invocation failed. Published when a plugin fails after all retries.

**Producer:** `platform/plugin-manager`  
**Consumers:** monitoring/alert-manager, monitoring/plugin-monitor

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `plugin_id` | string | * | |
| `invocation_id` | UUID | * | |
| `execution_id` | UUID \| null | ? | If the plugin was invoked during an AI execution. |
| `error_code` | string | * | |
| `error_category` | enum(SANDBOX_VIOLATION, TIMEOUT, EXCEPTION, RESOURCE_LIMIT) | * | |
| `retry_count` | integer | * | |
| `duration_ms` | integer | * | |
| `auto_disabled` | boolean | * | True if the platform auto-disabled the plugin after repeated failures. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 30 days | **Version:** 1.0  
**Audit Required:** Yes (SANDBOX_VIOLATION only) | **Lifecycle:** ACTIVE

---

## PL-006: `plugins.plugin.health_failed`

**Description:** A plugin's periodic health check has failed. The plugin may be approaching auto-disable.

**Producer:** `platform/plugin-manager/health-monitor`  
**Consumers:** monitoring/plugin-monitor, monitoring/alert-manager

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `plugin_id` | string | * | |
| `health_check_count` | integer | * | Total health checks run since install. |
| `consecutive_failures` | integer | * | Consecutive failures without a pass. |
| `auto_disable_threshold` | integer | * | How many consecutive failures before auto-disable. |
| `will_auto_disable` | boolean | * | True if this failure triggers auto-disable. |

**Delivery:** At-least-once | **TTL:** 30 days | **Version:** 1.0 | **Lifecycle:** ACTIVE

---



---

# Category 12: Billing Events

**Stream:** BILLING  
**TTL (default):** 365 days  
**Priority (default):** HIGH  
**Security Classification (default):** CONFIDENTIAL

Billing events are the financial record of platform usage. They have the longest retention period (365 days) to satisfy accounting and audit requirements. These events must never be dropped and must reach the audit log.

---

## BI-001: `billing.cost.recorded`

**Description:** A finalized cost entry has been recorded for a completed AI execution. This is the billing source of truth — downstream billing systems derive invoices from these events.

**Producer:** `platform/billing/cost-tracker`  
**Consumers:** billing/invoice-generator, billing/budget-monitor, monitoring/audit-log, brain/cost-analyzer

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `cost_record_id` | UUID | * | Idempotency key — same execution always produces same ID. |
| `execution_id` | UUID | * | The AIClient execution this cost entry covers. |
| `product_id` | string | * | |
| `workflow_id` | UUID \| null | ? | |
| `task_id` | string \| null | ? | |
| `provider_id` | string | * | |
| `model_id` | string | * | |
| `prompt_tokens` | integer | * | |
| `completion_tokens` | integer | * | |
| `cache_read_tokens` | integer | * | |
| `cost_usd` | float | * | Actual cost in USD (6 decimal places). |
| `billing_date` | ISO8601 | * | Date the cost is attributed to (UTC calendar day). |
| `billing_period` | string | * | ISO period string (e.g., `"2026-06"`). |

**Validation Rules:**
- `cost_usd` must be >= 0
- `cost_record_id` is deterministic: `sha256(execution_id + billing_date)`

**Ordering:** Not required. Invoice generator aggregates by `billing_date` and `billing_period`.  
**Delivery:** Exactly-once (idempotency via `cost_record_id` — duplicates silently deduped by consumer)  
**DLQ:** `billing.cost.recorded.dlq`  
**TTL:** 365 days | **Priority:** HIGH | **Version:** 1.0  
**Security Classification:** CONFIDENTIAL | **Audit Required:** Yes (every entry)  
**Lifecycle:** ACTIVE

---

## BI-002: `billing.budget.allocated`

**Description:** A daily AI spend budget has been allocated for a product. Emitted at midnight UTC when daily limits reset, and when a budget is first configured.

**Producer:** `platform/billing/budget-service`  
**Consumers:** platform/ai-ros/budget-manager (load new limits), monitoring/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `daily_limit_usd` | float | * | |
| `monthly_limit_usd` | float \| null | ? | |
| `budget_period` | string | * | ISO calendar date (e.g., `"2026-06-28"`) the budget applies to. |
| `allocated_by` | string | * | System (daily reset) or identity (manual change). |
| `previous_daily_limit_usd` | float \| null | ? | Null if first-time allocation. |

**Delivery:** At-least-once | **TTL:** 365 days | **Version:** 1.0  
**Security Classification:** CONFIDENTIAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## BI-003: `billing.invoice.generated`

**Description:** A billing invoice has been generated for a billing period.

**Producer:** `platform/billing/invoice-generator`  
**Consumers:** monitoring/audit-log, (future) finance/accounts-receivable

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `invoice_id` | UUID | * | |
| `billing_period` | string | * | e.g., `"2026-06"` |
| `generated_at` | ISO8601 | * | |
| `product_ids` | [string] | * | Products covered by this invoice. |
| `total_executions` | integer | * | |
| `total_tokens` | integer | * | |
| `total_cost_usd` | float | * | |
| `line_item_count` | integer | * | |
| `status` | enum(DRAFT, FINALIZED) | * | |

**Delivery:** At-least-once | **TTL:** 365 days | **Version:** 1.0  
**Security Classification:** CONFIDENTIAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## BI-004: `billing.budget.reset`

**Description:** The daily spending budget has been reset at midnight UTC for all products.

**Producer:** `platform/billing/budget-service` (scheduled daily)  
**Consumers:** platform/ai-ros/budget-manager, monitoring/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reset_date` | ISO8601 | * | The UTC date being reset to. |
| `products_reset` | integer | * | Number of product budgets reset. |
| `previous_day_total_spend_usd` | float | * | Platform-wide spend on the completed day. |

**Delivery:** At-least-once | **TTL:** 365 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## BI-005: `billing.anomaly.detected`

**Description:** A billing anomaly has been detected — a cost pattern that deviates significantly from historical baselines.

**Producer:** `platform/billing/anomaly-detector`  
**Consumers:** monitoring/alert-manager (HIGH alert), monitoring/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `product_id` | string | * | |
| `anomaly_type` | enum(SPEND_SPIKE, UNUSUAL_MODEL, UNUSUAL_VOLUME, SUSPICIOUS_PATTERN) | * | |
| `current_spend_usd` | float | * | Current period spend. |
| `baseline_spend_usd` | float | * | Historical average for comparison. |
| `deviation_pct` | float | * | Percentage above baseline. |
| `detection_window_hours` | integer | * | |
| `recommendation` | enum(MONITOR, REVIEW_IMMEDIATELY, SUSPEND_BILLING) | * | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 365 days | **Version:** 1.0  
**Security Classification:** CONFIDENTIAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

---

# Category 13: Usage Events

**Stream:** USAGE  
**TTL (default):** 30 days  
**Priority (default):** LOW  
**Security Classification (default):** INTERNAL

Usage events are aggregated metrics events for analytics and capacity planning. They are lower-fidelity than billing events (no per-execution granularity) but higher-volume and used for trend analysis and SLA reporting.

---

## US-001: `usage.tokens.hourly_summary`

**Description:** Hourly aggregated token usage summary. Published at the start of each hour for the prior hour.

**Producer:** `platform/usage/aggregator` (scheduled hourly)  
**Consumers:** brain/usage-analyzer, monitoring/capacity-planner, desktop/cost-panel (rolling stats)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `hour_utc` | ISO8601 | * | The hour being summarized (e.g., `"2026-06-28T14:00:00Z"`). |
| `total_prompt_tokens` | integer | * | |
| `total_completion_tokens` | integer | * | |
| `total_cache_tokens` | integer | * | |
| `total_tokens` | integer | * | |
| `total_cost_usd` | float | * | |
| `total_executions` | integer | * | |
| `executions_by_provider` | object | * | `{"anthropic": 150, "openai": 43, ...}` |
| `executions_by_model` | object | * | `{"claude-sonnet-4-6": 120, ...}` |
| `executions_by_product` | object | * | |
| `p50_latency_ms` | integer | * | |
| `p99_latency_ms` | integer | * | |
| `error_rate` | float | * | Failed executions / total executions. |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## US-002: `usage.requests.daily_summary`

**Description:** Daily aggregated platform request summary (all API calls, not just AI executions).

**Producer:** `platform/usage/aggregator` (scheduled daily, midnight UTC)  
**Consumers:** monitoring/capacity-planner, brain/usage-trend-analyzer

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `date_utc` | ISO8601 | * | The date being summarized (e.g., `"2026-06-28"`). |
| `total_requests` | integer | * | All HTTP requests to the platform. |
| `total_authenticated` | integer | * | |
| `total_rejected` | integer | * | Auth failures + rate limits. |
| `total_ai_executions` | integer | * | |
| `total_workflow_submissions` | integer | * | |
| `total_unique_identities` | integer | * | Count of distinct authenticated identities. |
| `peak_rpm` | integer | * | Peak requests per minute observed. |
| `peak_rpm_at` | ISO8601 | * | When the peak occurred. |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## US-003: `usage.workflow.daily_summary`

**Description:** Daily aggregated workflow statistics.

**Producer:** `platform/usage/aggregator`  
**Consumers:** brain/pattern-analyzer, monitoring/sla-reporter

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `date_utc` | ISO8601 | * | |
| `total_submitted` | integer | * | |
| `total_completed` | integer | * | |
| `total_failed` | integer | * | |
| `total_cancelled` | integer | * | |
| `sla_met_rate` | float | * | Percentage of completed workflows that met SLA. |
| `avg_completion_minutes` | float | * | |
| `avg_task_failure_rate` | float | * | Task failures per workflow. |
| `most_used_product_id` | string | * | |
| `gates_approved_count` | integer | * | |
| `gates_rejected_count` | integer | * | |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## US-004: `usage.brain.weekly_summary`

**Description:** Weekly aggregated Central Brain activity summary.

**Producer:** `platform/usage/aggregator` (scheduled weekly, Monday midnight UTC)  
**Consumers:** monitoring/brain-monitor, desktop/brain-stats-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `week_start_utc` | ISO8601 | * | |
| `experiences_recorded` | integer | * | |
| `patterns_discovered` | integer | * | |
| `patterns_updated` | integer | * | |
| `recommendations_made` | integer | * | |
| `recommendation_acceptance_rate` | float | * | |
| `knowledge_nodes_added` | integer | * | |
| `knowledge_relationships_added` | integer | * | |
| `brain_query_count` | integer | * | |
| `avg_query_latency_ms` | float | * | |

**Delivery:** At-least-once | **TTL:** 30 days | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## US-005: `usage.platform.capacity_warning`

**Description:** A usage aggregator has detected that current usage is approaching platform capacity limits.

**Producer:** `platform/usage/capacity-monitor`  
**Consumers:** monitoring/alert-manager, monitoring/capacity-planner

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `resource_type` | enum(AI_CONCURRENT_EXECUTIONS, WORKFLOW_QUEUE_DEPTH, NATS_STREAM_SIZE, DB_CONNECTIONS, KNOWLEDGE_INDEX_SIZE) | * | |
| `current_value` | float | * | Current measured value. |
| `capacity_limit` | float | * | Configured limit. |
| `utilization_pct` | float | * | current_value / capacity_limit * 100. |
| `warning_threshold_pct` | float | * | Threshold that was crossed (e.g., 80.0). |
| `trend` | enum(STABLE, INCREASING, DECREASING) | * | |
| `projected_breach_at` | ISO8601 \| null | ? | Projected time to exceed capacity at current trend rate. Null if trend is STABLE or DECREASING. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 30 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

---

# Category 14: Monitoring Events

**Stream:** MONITORING  
**TTL (default):** 7 days  
**Priority (default):** HIGH  
**Security Classification (default):** INTERNAL

Monitoring events are operational signals for the platform's health monitoring system. They describe SLA outcomes, health check results, and alert state transitions.

---

## MO-001: `monitoring.sla.breached`

**Description:** A workflow or task has exceeded its SLA and is considered in breach.

**Producer:** `platform/monitoring/sla-monitor`  
**Consumers:** monitoring/alert-manager (HIGH alert), brain/sla-tracker, desktop/workflow-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `sla_type` | enum(WORKFLOW_COMPLETION, TASK_COMPLETION, AI_EXECUTION, GATE_RESPONSE) | * | |
| `entity_id` | string | * | Workflow ID, task ID, etc. |
| `product_id` | string \| null | ? | |
| `sla_target_seconds` | integer | * | |
| `actual_seconds` | integer | * | |
| `breach_by_seconds` | integer | * | actual_seconds - sla_target_seconds. |
| `breach_pct` | float | * | breach_by_seconds / sla_target_seconds * 100. |
| `contributing_factors` | [string] | * | Known factors (e.g., `["provider_degraded", "high_queue_depth"]`). |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## MO-002: `monitoring.health_check.failed`

**Description:** A scheduled health check for a platform module has failed.

**Producer:** `platform/monitoring/health-checker`  
**Consumers:** workspace/health-monitor (may trigger WS-003 or WS-004), monitoring/alert-manager

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `module_id` | string | * | |
| `check_type` | enum(HTTP_LIVENESS, HTTP_READINESS, DB_PING, NATS_PING, CUSTOM) | * | |
| `check_endpoint` | string | * | URL or connection string checked. |
| `expected_status` | integer | * | Expected HTTP status or connection result. |
| `actual_status` | integer \| null | ? | Null if connection refused. |
| `latency_ms` | integer | * | How long the check took before timing out. |
| `consecutive_failures` | integer | * | |
| `error_detail` | string | * | Sanitized error description. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** No  
**Lifecycle:** ACTIVE

---

## MO-003: `monitoring.alert.fired`

**Description:** An alert has transitioned from INACTIVE to FIRING state.

**Producer:** `platform/monitoring/alert-manager`  
**Consumers:** monitoring/on-call-notifier, monitoring/audit-log, desktop/notification-center

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `alert_id` | UUID | * | |
| `alert_name` | string | * | Human-readable alert name. |
| `severity` | enum(INFO, WARNING, HIGH, CRITICAL) | * | |
| `source_event_type` | string | * | The event that triggered the alert. |
| `summary` | string | * | |
| `labels` | object | * | Key-value tags for routing (e.g., `{"product": "aisf", "module": "ai-ros"}`). |
| `fired_at` | ISO8601 | * | |
| `runbook_url` | string \| null | ? | |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 7 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes (CRITICAL only)  
**Lifecycle:** ACTIVE

---

## MO-004: `monitoring.alert.resolved`

**Description:** A previously firing alert has returned to INACTIVE state.

**Producer:** `platform/monitoring/alert-manager`  
**Consumers:** monitoring/on-call-notifier, desktop/notification-center

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `alert_id` | UUID | * | |
| `alert_name` | string | * | |
| `severity` | enum(INFO, WARNING, HIGH, CRITICAL) | * | |
| `fired_at` | ISO8601 | * | When the alert originally fired. |
| `resolved_at` | ISO8601 | * | |
| `duration_seconds` | integer | * | How long the alert was firing. |
| `resolution_cause` | enum(AUTO_RECOVERED, MANUAL_RESOLVE, TIMEOUT) | * | |

**Delivery:** At-least-once | **Priority:** STANDARD | **TTL:** 7 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## MO-005: `monitoring.platform.metrics_snapshot`

**Description:** A periodic snapshot of key platform metrics. Published every 60 seconds by the metrics aggregator.

**Producer:** `platform/monitoring/metrics-aggregator` (scheduled every 60s)  
**Consumers:** desktop/health-panel, monitoring/capacity-planner

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `snapshot_at` | ISO8601 | * | |
| `ai_executions_in_flight` | integer | * | |
| `ai_queue_depth` | object | * | `{"URGENT": 0, "STANDARD": 5, "BATCH": 12}` |
| `workflow_active_count` | integer | * | |
| `workflow_paused_count` | integer | * | |
| `nats_connected` | boolean | * | |
| `nats_pending_ack_count` | integer | * | |
| `db_pool_active` | integer | * | Active DB connections. |
| `db_pool_idle` | integer | * | |
| `db_pool_max` | integer | * | |
| `memory_used_mb` | float | * | Platform process memory. |
| `cpu_pct` | float | * | Platform process CPU. |
| `active_websocket_sessions` | integer | * | Connected desktop clients. |

**Delivery:** At-most-once (metrics are stateless — missing a snapshot is acceptable)  
**TTL:** 7 days | **Priority:** LOW | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

## MO-006: `monitoring.circuit_breaker.state_changed`

**Description:** Any circuit breaker on the platform has changed state. Unified event aggregating provider and internal circuit breaker state transitions.

**Producer:** `platform/ai-ros/circuit-breaker`, `platform/monitoring/circuit-tracker`  
**Consumers:** monitoring/alert-manager, desktop/health-panel

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `breaker_id` | string | * | Unique ID of the circuit breaker (e.g., `"provider:anthropic"`, `"nats:connection"`). |
| `breaker_type` | enum(PROVIDER, NATS, DATABASE, INTERNAL_SERVICE) | * | |
| `previous_state` | enum(CLOSED, OPEN, HALF_OPEN) | * | |
| `new_state` | enum(CLOSED, OPEN, HALF_OPEN) | * | |
| `failure_count` | integer | * | |
| `recovery_count` | integer | * | Successful probes (relevant in HALF_OPEN). |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 7 days | **Version:** 1.0  
**Lifecycle:** ACTIVE

---

---

# Category 15: System Events

**Stream:** SYSTEM  
**TTL (default):** 90 days  
**Priority (default):** HIGH  
**Security Classification (default):** INTERNAL

System events describe the platform's own lifecycle: startup, shutdown, configuration changes, and schema migrations. These are low-frequency, high-significance events and always written to the audit log.

---

## SY-001: `system.platform.started`

**Description:** The AI Studio platform has started successfully and all required modules are healthy.

**Producer:** `platform/startup-manager`  
**Consumers:** monitoring/audit-log, monitoring/alert-manager (resolves any pending OFFLINE alerts), desktop/notification-center

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `platform_version` | string | * | |
| `environment` | enum(local, staging, production) | * | |
| `startup_duration_ms` | integer | * | Time from process start to all health checks passing. |
| `modules_loaded` | [string] | * | All platform modules that were successfully initialized. |
| `providers_healthy` | [string] | * | AI providers that passed their initial health check. |
| `providers_degraded` | [string] | * | Providers that started in DEGRADED state. |
| `providers_offline` | [string] | * | Providers that could not be reached at startup. |
| `pending_migrations` | integer | * | DB migrations applied during this startup. 0 means schema was current. |
| `host` | string | * | Hostname/instance identifier. |

**Validation Rules:** `platform_version` must be valid SemVer.  
**Delivery:** At-least-once | **TTL:** 90 days | **Priority:** HIGH | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## SY-002: `system.platform.shutdown_initiated`

**Description:** The platform has received a shutdown signal and is beginning graceful shutdown.

**Producer:** `platform/startup-manager`  
**Consumers:** monitoring/audit-log, platform/workflow-runtime (drain queue), platform/ai-ros (drain in-flight executions), desktop/notification-center

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `platform_version` | string | * | |
| `environment` | string | * | |
| `shutdown_reason` | enum(SIGTERM, SIGINT, ADMIN_REQUEST, CRASH) | * | |
| `drain_timeout_seconds` | integer | * | How long the platform will wait for in-flight work to complete before forcing shutdown. |
| `active_workflows_count` | integer | * | Workflows that will be checkpointed. |
| `in_flight_executions_count` | integer | * | AI executions in progress. |

**Delivery:** At-least-once | **TTL:** 90 days | **Priority:** CRITICAL | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## SY-003: `system.migration.completed`

**Description:** A database migration has been applied successfully.

**Producer:** `platform/db/migration-runner`  
**Consumers:** monitoring/audit-log, monitoring/alert-manager (resolves any migration-pending alerts)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `migration_id` | string | * | Migration file identifier (e.g., `"0042_add_execution_index"`). |
| `migration_direction` | enum(UP, DOWN) | * | |
| `duration_ms` | integer | * | |
| `applied_at` | ISO8601 | * | |
| `schema_version_before` | string | * | DB schema version before this migration. |
| `schema_version_after` | string | * | |
| `rows_affected` | integer \| null | ? | Null if migration was DDL-only (no row modifications). |

**Delivery:** At-least-once | **TTL:** 90 days | **Priority:** HIGH | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## SY-004: `system.migration.failed`

**Description:** A database migration has failed. The platform may not be able to start.

**Producer:** `platform/db/migration-runner`  
**Consumers:** monitoring/audit-log, monitoring/alert-manager (CRITICAL)

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `migration_id` | string | * | |
| `migration_direction` | enum(UP, DOWN) | * | |
| `error_summary` | string | * | Sanitized error. No DB connection strings or credentials. |
| `schema_version_before` | string | * | DB was at this version before the attempted migration. |
| `rollback_attempted` | boolean | * | Whether the migration runner attempted to roll back. |
| `rollback_succeeded` | boolean \| null | ? | Null if rollback was not attempted. |

**Delivery:** At-least-once | **Priority:** CRITICAL | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## SY-005: `system.config.reloaded`

**Description:** The platform configuration has been reloaded (e.g., after a config file change or environment variable update).

**Producer:** `platform/config-manager`  
**Consumers:** all platform modules (via broadcast), monitoring/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `reload_id` | UUID | * | |
| `triggered_by` | enum(FILE_CHANGE, ENV_CHANGE, API_REQUEST, STARTUP) | * | |
| `changed_keys` | [string] | * | Configuration keys that changed. No values. |
| `modules_notified` | [string] | * | Modules that received the reload signal. |
| `hot_reload` | boolean | * | True if config applied without restart. False requires restart. |

**Validation Rules:** No configuration values may appear. Only key names.  
**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

## SY-006: `system.feature_flag.changed`

**Description:** A platform feature flag has been toggled.

**Producer:** `platform/config-manager`  
**Consumers:** all platform modules (via broadcast), monitoring/audit-log

**Payload Schema:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `flag_name` | string | * | |
| `previous_value` | boolean | * | |
| `new_value` | boolean | * | |
| `changed_by` | string | * | Identity (ADMIN required). |
| `change_reason` | string | * | Mandatory for feature flag changes. |
| `effective_immediately` | boolean | * | True if change takes effect without restart. |
| `expiry_date` | ISO8601 \| null | ? | When the flag is scheduled to be removed. |

**Delivery:** At-least-once | **Priority:** HIGH | **TTL:** 90 days | **Version:** 1.0  
**Security Classification:** INTERNAL | **Audit Required:** Yes  
**Lifecycle:** ACTIVE

---

---

# Appendix A: Event Quick Reference

## A.1 All Events by Category

| # | Subject | Category | Priority | Security | Audit |
|---|---------|----------|----------|----------|-------|
| WS-001 | `workspace.product.registered` | Workspace | LOW | INTERNAL | No |
| WS-002 | `workspace.product.deregistered` | Workspace | LOW | INTERNAL | No |
| WS-003 | `workspace.health.degraded` | Workspace | HIGH | INTERNAL | Yes |
| WS-004 | `workspace.health.offline` | Workspace | CRITICAL | INTERNAL | Yes |
| WS-005 | `workspace.health.recovered` | Workspace | HIGH | INTERNAL | Yes |
| WF-001 | `workflows.workflow.created` | Workflow | STANDARD | INTERNAL | No |
| WF-002 | `workflows.workflow.completed` | Workflow | STANDARD | INTERNAL | Yes |
| WF-003 | `workflows.workflow.failed` | Workflow | HIGH | INTERNAL | Yes |
| WF-004 | `workflows.workflow.cancelled` | Workflow | STANDARD | INTERNAL | Yes |
| WF-005 | `workflows.workflow.paused` | Workflow | STANDARD | INTERNAL | No |
| WF-006 | `workflows.workflow.resumed` | Workflow | STANDARD | INTERNAL | No |
| WF-007 | `workflows.task.started` | Workflow | STANDARD | INTERNAL | No |
| WF-008 | `workflows.task.completed` | Workflow | STANDARD | INTERNAL | No |
| WF-009 | `workflows.task.failed` | Workflow | HIGH | INTERNAL | Yes* |
| WF-010 | `workflows.task.stalled` | Workflow | HIGH | INTERNAL | No |
| WF-011 | `workflows.gate.approved` | Workflow | HIGH | RESTRICTED | Yes |
| WF-012 | `workflows.gate.rejected` | Workflow | HIGH | RESTRICTED | Yes |
| PM-001 | `prompts.template.registered` | Prompt | LOW | INTERNAL | No |
| PM-002 | `prompts.template.updated` | Prompt | LOW | INTERNAL | No |
| PM-003 | `prompts.template.submitted_for_review` | Prompt | LOW | INTERNAL | Yes |
| PM-004 | `prompts.template.activated` | Prompt | HIGH | INTERNAL | Yes |
| PM-005 | `prompts.template.deprecated` | Prompt | LOW | INTERNAL | Yes |
| PM-006 | `prompts.template.suspended` | Prompt | CRITICAL | RESTRICTED | Yes |
| PM-007 | `prompts.template.render_failed` | Prompt | HIGH | RESTRICTED | Yes |
| PM-008 | `prompts.ab_test.winner_declared` | Prompt | LOW | INTERNAL | No |
| PV-001 | `ai.provider.circuit_opened` | Provider | HIGH | INTERNAL | Yes |
| PV-002 | `ai.provider.circuit_half_opened` | Provider | HIGH | INTERNAL | No |
| PV-003 | `ai.provider.circuit_closed` | Provider | HIGH | INTERNAL | No |
| PV-004 | `ai.provider.health_degraded` | Provider | HIGH | INTERNAL | No |
| PV-005 | `ai.provider.registered` | Provider | LOW | INTERNAL | No |
| PV-006 | `ai.provider.rate_limited` | Provider | HIGH | INTERNAL | No |
| AM-001 | `ai.execution.started` | AI Model | STANDARD | INTERNAL | No |
| AM-002 | `ai.execution.completed` | AI Model | STANDARD | INTERNAL | No |
| AM-003 | `ai.execution.failed` | AI Model | HIGH | INTERNAL | No |
| AM-004 | `ai.budget.threshold_reached` | AI Model | HIGH | INTERNAL | No |
| AM-005 | `ai.budget.limit_exceeded` | AI Model | CRITICAL | INTERNAL | Yes |
| AM-006 | `ai.quota.rate_limit_exceeded` | AI Model | HIGH | INTERNAL | No |
| KN-001 | `knowledge.node.added` | Knowledge | LOW | INTERNAL | No |
| KN-002 | `knowledge.node.updated` | Knowledge | LOW | INTERNAL | No |
| KN-003 | `knowledge.relationship.added` | Knowledge | LOW | INTERNAL | No |
| KN-004 | `knowledge.pattern.indexed` | Knowledge | LOW | INTERNAL | No |
| KN-005 | `knowledge.memory.set` | Knowledge | LOW | INTERNAL | No |
| AU-001 | `security.auth.succeeded` | Auth | HIGH | RESTRICTED | Yes |
| AU-002 | `security.auth.failed` | Auth | HIGH | RESTRICTED | Yes |
| AU-003 | `security.permission.denied` | Auth | HIGH | RESTRICTED | Yes |
| AU-004 | `security.api_key.created` | Auth | HIGH | CONFIDENTIAL | Yes |
| AU-005 | `security.api_key.revoked` | Auth | CRITICAL | CONFIDENTIAL | Yes |
| AU-006 | `security.rate_limit.exceeded` | Auth | HIGH | RESTRICTED | Yes |
| SC-001 | `security.prompt_injection.detected` | Security | CRITICAL | CONFIDENTIAL | Yes |
| SC-002 | `security.tool_sandbox.violation` | Security | CRITICAL | CONFIDENTIAL | Yes |
| SC-003 | `security.anomaly.detected` | Security | HIGH | CONFIDENTIAL | Yes |
| SC-004 | `security.config.changed` | Security | HIGH | CONFIDENTIAL | Yes |
| DS-001 | `desktop.ui.workflow_panel_update` | Desktop | LOW | INTERNAL | No |
| DS-002 | `desktop.ui.cost_panel_update` | Desktop | LOW | INTERNAL | No |
| DS-003 | `desktop.ui.workspace_panel_update` | Desktop | LOW | INTERNAL | No |
| DS-004 | `desktop.ui.notification` | Desktop | STANDARD | INTERNAL | No |
| DS-005 | `desktop.session.connected` | Desktop | LOW | INTERNAL | No |
| DS-006 | `desktop.session.disconnected` | Desktop | LOW | INTERNAL | No |
| MK-001 | `marketplace.product.listed` | Marketplace | — | INTERNAL | No |
| MK-002 | `marketplace.product.installed` | Marketplace | — | INTERNAL | No |
| MK-003 | `marketplace.review.submitted` | Marketplace | — | INTERNAL | No |
| PL-001 | `plugins.plugin.installed` | Plugin | STANDARD | INTERNAL | Yes |
| PL-002 | `plugins.plugin.uninstalled` | Plugin | STANDARD | INTERNAL | Yes |
| PL-003 | `plugins.plugin.enabled` | Plugin | STANDARD | INTERNAL | No |
| PL-004 | `plugins.plugin.disabled` | Plugin | HIGH | INTERNAL | Yes |
| PL-005 | `plugins.plugin.execution_failed` | Plugin | HIGH | INTERNAL | Yes* |
| PL-006 | `plugins.plugin.health_failed` | Plugin | STANDARD | INTERNAL | No |
| BI-001 | `billing.cost.recorded` | Billing | HIGH | CONFIDENTIAL | Yes |
| BI-002 | `billing.budget.allocated` | Billing | HIGH | CONFIDENTIAL | Yes |
| BI-003 | `billing.invoice.generated` | Billing | HIGH | CONFIDENTIAL | Yes |
| BI-004 | `billing.budget.reset` | Billing | HIGH | INTERNAL | Yes |
| BI-005 | `billing.anomaly.detected` | Billing | HIGH | CONFIDENTIAL | Yes |
| US-001 | `usage.tokens.hourly_summary` | Usage | LOW | INTERNAL | No |
| US-002 | `usage.requests.daily_summary` | Usage | LOW | INTERNAL | No |
| US-003 | `usage.workflow.daily_summary` | Usage | LOW | INTERNAL | No |
| US-004 | `usage.brain.weekly_summary` | Usage | LOW | INTERNAL | No |
| US-005 | `usage.platform.capacity_warning` | Usage | HIGH | INTERNAL | No |
| MO-001 | `monitoring.sla.breached` | Monitoring | HIGH | INTERNAL | Yes |
| MO-002 | `monitoring.health_check.failed` | Monitoring | HIGH | INTERNAL | No |
| MO-003 | `monitoring.alert.fired` | Monitoring | HIGH | INTERNAL | Yes* |
| MO-004 | `monitoring.alert.resolved` | Monitoring | STANDARD | INTERNAL | No |
| MO-005 | `monitoring.platform.metrics_snapshot` | Monitoring | LOW | INTERNAL | No |
| MO-006 | `monitoring.circuit_breaker.state_changed` | Monitoring | HIGH | INTERNAL | No |
| SY-001 | `system.platform.started` | System | HIGH | INTERNAL | Yes |
| SY-002 | `system.platform.shutdown_initiated` | System | CRITICAL | INTERNAL | Yes |
| SY-003 | `system.migration.completed` | System | HIGH | INTERNAL | Yes |
| SY-004 | `system.migration.failed` | System | CRITICAL | INTERNAL | Yes |
| SY-005 | `system.config.reloaded` | System | HIGH | INTERNAL | Yes |
| SY-006 | `system.feature_flag.changed` | System | HIGH | INTERNAL | Yes |

*Conditional — see event definition.

**Total events: 84 ACTIVE, 3 PROPOSED**

---

## A.2 Events That Are Always Audited

The following events are unconditionally written to the immutable audit log. No configuration can disable audit for these events.

```
security.auth.failed                   (AU-002)
security.auth.succeeded                (AU-001)
security.permission.denied             (AU-003)
security.api_key.created               (AU-004)
security.api_key.revoked               (AU-005)
security.rate_limit.exceeded           (AU-006)
security.prompt_injection.detected     (SC-001)
security.tool_sandbox.violation        (SC-002)
security.anomaly.detected              (SC-003)
security.config.changed                (SC-004)
workspace.health.degraded              (WS-003)
workspace.health.offline               (WS-004)
workspace.health.recovered             (WS-005)
workflows.workflow.completed           (WF-002)
workflows.workflow.failed              (WF-003)
workflows.workflow.cancelled           (WF-004)
workflows.gate.approved                (WF-011)
workflows.gate.rejected                (WF-012)
prompts.template.submitted_for_review  (PM-003)
prompts.template.activated             (PM-004)
prompts.template.deprecated            (PM-005)
prompts.template.suspended             (PM-006)
prompts.template.render_failed         (PM-007)
ai.provider.circuit_opened             (PV-001)
ai.budget.limit_exceeded               (AM-005)
billing.cost.recorded                  (BI-001)
billing.budget.allocated               (BI-002)
billing.invoice.generated              (BI-003)
billing.budget.reset                   (BI-004)
billing.anomaly.detected               (BI-005)
monitoring.sla.breached                (MO-001)
monitoring.alert.fired (CRITICAL only) (MO-003)
plugins.plugin.installed               (PL-001)
plugins.plugin.uninstalled             (PL-002)
plugins.plugin.disabled                (PL-004)
system.platform.started                (SY-001)
system.platform.shutdown_initiated     (SY-002)
system.migration.completed             (SY-003)
system.migration.failed                (SY-004)
system.config.reloaded                 (SY-005)
system.feature_flag.changed            (SY-006)
```

---

## A.3 CRITICAL Priority Events

Events at CRITICAL priority bypass all backpressure mechanisms and are always delivered first.

| Event | Code | Why Critical |
|-------|------|-------------|
| `workspace.health.offline` | WS-004 | Platform incident |
| `prompts.template.suspended` | PM-006 | Emergency template block |
| `security.prompt_injection.detected` | SC-001 | Active attack |
| `security.tool_sandbox.violation` | SC-002 | Active sandbox escape |
| `ai.budget.limit_exceeded` | AM-005 | Must block requests before next API call |
| `security.api_key.revoked` | AU-005 | Must propagate before next request |
| `system.platform.shutdown_initiated` | SY-002 | Drain signal |
| `system.migration.failed` | SY-004 | Platform may not start |

---

## A.4 Security Classification Reference

| Classification | Who May Read | What it covers |
|---------------|-------------|----------------|
| PUBLIC | Anyone | Nothing on this platform — no platform events are public |
| INTERNAL | Any authenticated identity | Health, workflow, prompt, AI, knowledge, usage, monitoring |
| RESTRICTED | REVIEWER or ADMIN role only | Auth events, gate decisions, rate limit events |
| CONFIDENTIAL | ADMIN role only | API keys, billing, security violations, anomalies |

---

## A.5 Event Delivery Semantics Comparison

| Semantics | Events that use it | Guarantee |
|-----------|-------------------|-----------|
| **At-most-once** | Desktop UI events (DS-*) | May be lost; consumers tolerate gaps |
| **At-least-once** | Most platform events | May arrive duplicated; consumers must be idempotent |
| **Exactly-once** | `billing.cost.recorded` (BI-001) | Deduplication via `cost_record_id`; guaranteed once-and-only-once billing entry |

---

## A.6 Brain Consumer Map

The Central Brain (`platform/brain`) consumes events to build its knowledge and pattern base. This map defines which events the Brain subscribes to, and what it does with each.

| Event | Brain Action |
|-------|-------------|
| `workflows.workflow.completed` | Record success pattern; update model accuracy |
| `workflows.workflow.failed` | Record failure pattern; analyze root cause |
| `workflows.task.completed` | Record task execution sample; update timing model |
| `workflows.task.failed` | Record task failure; update retry prediction |
| `ai.execution.completed` | Record execution sample; update provider reliability model |
| `prompts.template.activated` | Record new template baseline; update prompt quality model |
| `prompts.ab_test.winner_declared` | Update prompt effectiveness model |
| `security.auth.failed` | Update anomaly baseline |
| `security.anomaly.detected` | Flag identity for elevated monitoring |
| `ai.provider.circuit_opened` | Update provider reliability score |
| `ai.provider.circuit_closed` | Update provider recovery score |
| `billing.cost.recorded` | Update cost prediction model |
| `knowledge.pattern.indexed` | Merge new patterns into recommendation engine |

---

## A.7 Consumer Idempotency Requirements

All consumers of at-least-once events must handle duplicate delivery. Standard idempotency patterns by consumer type:

**State machine consumers** (workflow-runtime, circuit-breaker): Natural idempotency — a state transition applied twice produces the same state.

**Accumulator consumers** (billing/cost-tracker, usage/aggregator): Use the event's `event_id` as a deduplication key in a seen-events table with the same TTL as the event stream.

**Cache invalidation consumers** (prompt-os, security): Invalidate-then-reload is idempotent by nature. Duplicate invalidation is harmless.

**Alert consumers** (alert-manager): Deduplicate by `event_id` with a 5-minute window. Beyond 5 minutes, treat as a new alert trigger.

**Audit log consumers**: Use `event_id` as the primary key. Duplicate inserts must be silently discarded (INSERT OR IGNORE / ON CONFLICT DO NOTHING).

---

## A.8 DLQ (Dead Letter Queue) Policy

Events that cannot be delivered after all retries are routed to a DLQ subject. DLQ subjects follow the naming pattern: `<original-subject>.dlq`.

| DLQ Subject | Source Events | Monitored by |
|-------------|---------------|-------------|
| `workspace.product.registered.dlq` | WS-001 | monitoring/dlq-monitor |
| `workspace.health.degraded.dlq` | WS-003 | monitoring/dlq-monitor |
| `workflows.dlq` | All workflow events | monitoring/dlq-monitor |
| `ai.provider.circuit_opened.dlq` | PV-001 | monitoring/dlq-monitor |
| `prompts.template.activated.dlq` | PM-004 | monitoring/dlq-monitor |
| `billing.cost.recorded.dlq` | BI-001 | monitoring/dlq-monitor + finance |

Events in a DLQ require manual investigation. The `monitoring/dlq-monitor` publishes `monitoring.alert.fired` when any DLQ receives a message. DLQ events retain the full original payload and envelope, plus additional fields:
- `dlq_arrival_at` — when the event arrived in the DLQ
- `delivery_attempts` — how many times delivery was attempted
- `last_error` — final error from the consumer

---

## A.9 Version Compatibility Matrix

| Schema Version | Interpretation |
|---------------|----------------|
| `"1.0"` | Initial version. All current events. |
| `"1.x"` | Minor update. New optional fields added. Backward-compatible. |
| `"2.0"` | Major update. Breaking change. Dual-publish period required. |

**Dual-publish protocol:** When a MAJOR version bump is required, the publisher emits both the old and new schema version for a minimum 4-week compatibility window. Consumers select the version they support via `schema_version` field. After the window, the old version is retired and publishing stops.

---

## A.10 Event Roadmap

Events planned for future platform versions. Not yet PROPOSED — included for reference only.

| Planned Version | Events |
|----------------|--------|
| v2.5 (multi-tenant) | `marketplace.*` events (MK-001 through MK-003 promoted to ACTIVE); `tenants.*` category; `billing.tenant.*` events |
| v3.0 (HA + SSO) | `system.replica.*` events (leader election, failover); `security.sso.*` events; `system.cluster.*` events |
| v3.x (AI Studio 3.0) | `ai.agent.*` events (autonomous agent lifecycle); `ai.swarm.*` events (multi-agent coordination) |

---

## A.11 Glossary

| Term | Definition |
|------|-----------|
| **Consumer** | A platform module or product that subscribes to an event. |
| **Dead Letter Queue (DLQ)** | A NATS subject that receives events that could not be delivered after all retries. |
| **Delivery Semantics** | The guarantee around how many times a consumer receives an event: at-most-once, at-least-once, or exactly-once. |
| **Domain** | The first segment of an event subject (e.g., `workflows`). Always plural. |
| **Envelope** | The universal fields present on every event (event_id, timestamp, correlation_id, etc.). |
| **Entity** | The second segment of an event subject (e.g., `task`, `workflow`). Always singular. |
| **Event** | A fact — something that happened on the platform, described in the past tense. |
| **Idempotency** | The property that applying an operation multiple times has the same effect as applying it once. |
| **JetStream** | NATS JetStream — the persistence layer for platform events. |
| **Lifecycle** | The governance state of an event: PROPOSED → ACTIVE → DEPRECATED → RETIRED. |
| **Producer** | The single platform module that publishes a given event. Every event has exactly one producer. |
| **Schema Version** | The version of the event payload schema, carried in the `schema_version` field. |
| **Stream** | A NATS JetStream stream — a durable, ordered collection of events for a given domain. |
| **Subject** | The NATS subject of an event — the routing address, three segments: `domain.entity.verb`. |
| **TTL** | Time to live — how long an event is retained in the JetStream stream. |

---

*End of EVENT-CATALOG v1.0.0*

*Document authority: Chief Software Architect*  
*Next review: 2026-09-28 (quarterly)*  
*Changes require: architect approval for ACTIVE events; CTO approval for CRITICAL events; no approval needed for PROPOSED events*

