# AI Resource Database — Schema and Data Architecture

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

This document specifies the complete database schema for AI ROS. It defines every table, column, index, constraint, and partitioning strategy. It also documents the read/write model separation (CQRS), retention policies, and migration conventions.

---

## 2. Architecture

AI ROS uses PostgreSQL 15+ as its primary data store. The schema follows a CQRS pattern:

- **Write models (append-only ledgers and state tables):** Authoritative source of truth. Never updated after insert.
- **Read models (materialized views and summary tables):** Pre-aggregated for query performance. Rebuilt from write models on startup; kept in sync by background workers.
- **Redis:** In-memory cache for hot read models (quota positions, session metadata, provider health).

### Database Layout

| Database | Purpose |
|---------|---------|
| `ai_ros_main` | Core operational tables |
| `ai_ros_vault` | Credential storage (isolated; separate connection pool) |
| `ai_ros_analytics` | Benchmark and analytics data (read replica target) |
| `ai_ros_audit` | Immutable audit log (separate for compliance retention policy) |

---

## 3. Schema: Provider Registry

### ai_ros_providers

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `provider_id` | varchar(64) | PK | Stable identifier (e.g., `anthropic`) |
| `display_name` | varchar(255) | NOT NULL | |
| `adapter_class` | varchar(512) | NOT NULL | Python class path |
| `version` | varchar(32) | NOT NULL | Adapter version |
| `protocol_version` | varchar(32) | NOT NULL | AI ROS protocol version |
| `status` | varchar(32) | NOT NULL | `active` / `disabled` / `deprecated` |
| `requires_api_key` | boolean | NOT NULL DEFAULT true | |
| `supports_local_deployment` | boolean | NOT NULL DEFAULT false | |
| `manifest_json` | jsonb | | Full provider manifest |
| `registered_at` | timestamptz | NOT NULL DEFAULT now() | |
| `updated_at` | timestamptz | NOT NULL DEFAULT now() | |

**Indexes:** `provider_id` (PK), `status` (btree)

---

### ai_ros_provider_models

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `canonical_id` | varchar(256) | PK | e.g., `anthropic/claude/claude-opus-4-8` |
| `provider_id` | varchar(64) | FK → ai_ros_providers | |
| `provider_model_id` | varchar(256) | NOT NULL | Provider's native ID |
| `display_name` | varchar(255) | NOT NULL | |
| `family` | varchar(128) | NOT NULL | |
| `status` | varchar(32) | NOT NULL | `active` / `deprecated` / `beta` / `retired` |
| `successor_canonical_id` | varchar(256) | NULL FK | |
| `context_window_tokens` | integer | NOT NULL | |
| `max_output_tokens` | integer | NOT NULL | |
| `capabilities_json` | jsonb | NOT NULL | CapabilityFlags serialized |
| `pricing_json` | jsonb | NOT NULL | ModelPricing serialized |
| `latency_class` | varchar(32) | NOT NULL | |
| `quality_tier` | varchar(32) | NOT NULL | |
| `available_regions` | text[] | | |
| `knowledge_cutoff` | date | NULL | |
| `release_date` | date | NOT NULL | |
| `deprecation_date` | date | NULL | |
| `metadata_json` | jsonb | | |
| `updated_at` | timestamptz | NOT NULL DEFAULT now() | |

**Indexes:** `canonical_id` (PK), `provider_id` (btree), `status` (btree), `quality_tier` (btree), `(provider_id, status)` composite

---

### ai_ros_provider_health

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `health_id` | uuid | PK DEFAULT gen_random_uuid() | |
| `provider_id` | varchar(64) | NOT NULL FK | |
| `status` | varchar(32) | NOT NULL | `healthy` / `degraded` / `unavailable` |
| `latency_ms` | integer | | |
| `models_available` | text[] | | |
| `error_message` | text | NULL | |
| `checked_at` | timestamptz | NOT NULL | |
| `circuit_breaker_state` | varchar(32) | NOT NULL DEFAULT 'closed' | |

**Partitioned by:** `checked_at` (monthly range partitions)  
**Indexes:** `(provider_id, checked_at DESC)` for current health lookup

---

## 4. Schema: Requests and Executions

### ai_ros_requests

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `request_id` | uuid | PK | |
| `org_id` | uuid | NOT NULL | |
| `project_id` | uuid | NULL | |
| `user_id` | uuid | NULL | |
| `session_id` | uuid | NULL FK → ai_ros_conversations | |
| `product_id` | varchar(128) | NOT NULL | |
| `task_category` | varchar(128) | NULL | |
| `mode` | varchar(32) | NOT NULL | `immediate` / `queued` / `batch` / `scheduled` |
| `priority` | integer | NOT NULL DEFAULT 50 | |
| `status` | varchar(32) | NOT NULL | See request lifecycle |
| `provider_id` | varchar(64) | NULL FK | |
| `canonical_model_id` | varchar(256) | NULL FK | |
| `request_payload_ref` | varchar(512) | NULL | Object storage key for large payloads |
| `submitted_at` | timestamptz | NOT NULL DEFAULT now() | |
| `dispatched_at` | timestamptz | NULL | |
| `completed_at` | timestamptz | NULL | |
| `attempts` | integer | NOT NULL DEFAULT 0 | |
| `last_error_code` | varchar(64) | NULL | |
| `queue_wait_ms` | integer | NULL | |
| `execution_ms` | integer | NULL | |
| `input_tokens` | integer | NULL | |
| `output_tokens` | integer | NULL | |
| `total_cost_usd` | numeric(12,8) | NULL | |
| `routing_rationale` | text | NULL | |
| `metadata_json` | jsonb | | |

**Partitioned by:** `submitted_at` (monthly range partitions; retain 13 months hot, archive thereafter)  
**Indexes:** `request_id` (PK), `(org_id, submitted_at DESC)`, `(session_id, submitted_at)`, `(status, priority, submitted_at)` for queue dequeue

---

## 5. Schema: Conversations

### ai_ros_conversations

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `session_id` | uuid | PK | |
| `org_id` | uuid | NOT NULL | |
| `project_id` | uuid | NULL | |
| `user_id` | uuid | NULL | |
| `product_id` | varchar(128) | NOT NULL | |
| `status` | varchar(32) | NOT NULL DEFAULT 'active' | |
| `parent_session_id` | uuid | NULL FK self-ref | |
| `branch_point_message_id` | uuid | NULL | |
| `title` | varchar(512) | NULL | |
| `system_prompt` | text | NULL | |
| `config_json` | jsonb | NOT NULL DEFAULT '{}' | SessionConfig |
| `message_count` | integer | NOT NULL DEFAULT 0 | Cached; updated on insert |
| `total_input_tokens` | bigint | NOT NULL DEFAULT 0 | |
| `total_output_tokens` | bigint | NOT NULL DEFAULT 0 | |
| `total_cost_usd` | numeric(14,8) | NOT NULL DEFAULT 0 | |
| `created_at` | timestamptz | NOT NULL DEFAULT now() | |
| `last_active_at` | timestamptz | NOT NULL DEFAULT now() | |
| `expires_at` | timestamptz | NULL | |
| `archived_at` | timestamptz | NULL | |
| `metadata_json` | jsonb | | |

**Indexes:** `session_id` (PK), `(org_id, status, last_active_at DESC)`, `(user_id, status)`, `(expires_at)` for expiry sweep

---

### ai_ros_messages

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `message_id` | uuid | PK | |
| `session_id` | uuid | NOT NULL FK → ai_ros_conversations | |
| `sequence_number` | integer | NOT NULL | Monotonically increasing per session |
| `role` | varchar(32) | NOT NULL | |
| `content_json` | jsonb | NOT NULL | List of ContentBlocks |
| `content_storage_ref` | varchar(512) | NULL | Object storage key if content > 1MB |
| `token_count` | integer | NULL | |
| `linked_request_id` | uuid | NULL FK → ai_ros_requests | |
| `canonical_model_id` | varchar(256) | NULL FK | |
| `latency_ms` | integer | NULL | |
| `is_compressed_summary` | boolean | NOT NULL DEFAULT false | |
| `compressed_from_range` | jsonb | NULL | `{start_seq, end_seq}` |
| `is_archived` | boolean | NOT NULL DEFAULT false | Set when replaced by compression |
| `created_at` | timestamptz | NOT NULL DEFAULT now() | |
| `metadata_json` | jsonb | | |

**UNIQUE constraint:** `(session_id, sequence_number)`  
**Indexes:** `message_id` (PK), `(session_id, sequence_number)`, `(session_id, is_archived, sequence_number)` for context assembly

---

## 6. Schema: Quota

### ai_ros_quota_rules

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `rule_id` | uuid | PK | |
| `org_id` | uuid | NOT NULL | |
| `project_id` | uuid | NULL | |
| `user_id` | uuid | NULL | |
| `provider_id` | varchar(64) | NOT NULL | |
| `model_id` | varchar(256) | NULL | |
| `resource` | varchar(64) | NOT NULL | |
| `window` | varchar(32) | NOT NULL | |
| `limit_val` | bigint | NOT NULL | Renamed from `limit` (SQL reserved word) |
| `warning_threshold_pct` | integer | NOT NULL DEFAULT 80 | |
| `critical_threshold_pct` | integer | NOT NULL DEFAULT 95 | |
| `on_exhaustion` | varchar(32) | NOT NULL DEFAULT 'reject' | |
| `source` | varchar(32) | NOT NULL | `provider_imposed` / `org_configured` |
| `created_at` | timestamptz | NOT NULL DEFAULT now() | |
| `updated_at` | timestamptz | NOT NULL DEFAULT now() | |

---

### ai_ros_quota_ledger

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `ledger_id` | uuid | PK | |
| `rule_id` | uuid | NOT NULL FK | |
| `request_id` | uuid | NOT NULL FK | |
| `entry_type` | varchar(32) | NOT NULL | `deduction` / `refund` / `reset` |
| `amount` | bigint | NOT NULL | Units (negative for refunds) |
| `window_start` | timestamptz | NOT NULL | Which window this belongs to |
| `recorded_at` | timestamptz | NOT NULL DEFAULT now() | |

**Partitioned by:** `recorded_at` (monthly)  
**No UPDATE or DELETE allowed on this table** — enforced by row-level security policy.

---

## 7. Schema: Budget

### ai_ros_budget_rules

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `rule_id` | uuid | PK | |
| `org_id` | uuid | NOT NULL | |
| `project_id` | uuid | NULL | |
| `department_id` | varchar(256) | NULL | |
| `user_id` | uuid | NULL | |
| `provider_id` | varchar(64) | NULL | |
| `period` | varchar(32) | NOT NULL | |
| `currency` | varchar(8) | NOT NULL DEFAULT 'USD' | |
| `limit_usd` | numeric(14,4) | NOT NULL | |
| `warning_threshold_pct` | integer | NOT NULL DEFAULT 70 | |
| `critical_threshold_pct` | integer | NOT NULL DEFAULT 90 | |
| `on_exhaustion` | varchar(32) | NOT NULL | |
| `approval_required_above_usd` | numeric(10,4) | NULL | |
| `rollover_enabled` | boolean | NOT NULL DEFAULT false | |
| `created_at` | timestamptz | NOT NULL DEFAULT now() | |

---

### ai_ros_cost_ledger

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `record_id` | uuid | PK | |
| `request_id` | uuid | NOT NULL FK | |
| `org_id` | uuid | NOT NULL | |
| `project_id` | uuid | NULL | |
| `department_id` | varchar(256) | NULL | |
| `user_id` | uuid | NULL | |
| `provider_id` | varchar(64) | NOT NULL | |
| `canonical_model_id` | varchar(256) | NOT NULL | |
| `input_tokens` | integer | NOT NULL DEFAULT 0 | |
| `output_tokens` | integer | NOT NULL DEFAULT 0 | |
| `cache_read_tokens` | integer | NOT NULL DEFAULT 0 | |
| `cache_write_tokens` | integer | NOT NULL DEFAULT 0 | |
| `reasoning_tokens` | integer | NOT NULL DEFAULT 0 | |
| `image_count` | integer | NOT NULL DEFAULT 0 | |
| `audio_seconds` | integer | NOT NULL DEFAULT 0 | |
| `input_cost_usd` | numeric(12,8) | NOT NULL DEFAULT 0 | |
| `output_cost_usd` | numeric(12,8) | NOT NULL DEFAULT 0 | |
| `cache_cost_usd` | numeric(12,8) | NOT NULL DEFAULT 0 | |
| `image_cost_usd` | numeric(12,8) | NOT NULL DEFAULT 0 | |
| `audio_cost_usd` | numeric(12,8) | NOT NULL DEFAULT 0 | |
| `total_cost_usd` | numeric(12,8) | NOT NULL DEFAULT 0 | |
| `subscription_covered` | boolean | NOT NULL DEFAULT false | |
| `task_type` | varchar(128) | NULL | |
| `outcome_quality_score` | real | NULL | |
| `recorded_at` | timestamptz | NOT NULL DEFAULT now() | |

**Partitioned by:** `recorded_at` (monthly)  
**No UPDATE or DELETE** — append-only.

---

## 8. Schema: Policies

### ai_ros_policies

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `policy_id` | uuid | PK | |
| `org_id` | uuid | NOT NULL | |
| `project_id` | uuid | NULL | |
| `name` | varchar(512) | NOT NULL | |
| `description` | text | | |
| `policy_type` | varchar(64) | NOT NULL | |
| `status` | varchar(32) | NOT NULL DEFAULT 'draft' | |
| `priority` | integer | NOT NULL DEFAULT 100 | |
| `conditions_json` | jsonb | NOT NULL DEFAULT '[]' | |
| `action` | varchar(64) | NOT NULL | |
| `action_params_json` | jsonb | NOT NULL DEFAULT '{}' | |
| `test_mode` | boolean | NOT NULL DEFAULT false | |
| `effective_from` | timestamptz | NULL | |
| `effective_until` | timestamptz | NULL | |
| `current_version` | integer | NOT NULL DEFAULT 1 | |
| `created_at` | timestamptz | NOT NULL DEFAULT now() | |
| `created_by` | uuid | NOT NULL | |
| `updated_at` | timestamptz | NOT NULL DEFAULT now() | |

---

### ai_ros_policy_versions

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `version_id` | uuid | PK | |
| `policy_id` | uuid | NOT NULL FK | |
| `version_number` | integer | NOT NULL | |
| `snapshot_json` | jsonb | NOT NULL | Full policy at this version |
| `changed_by` | uuid | NOT NULL | |
| `changed_at` | timestamptz | NOT NULL DEFAULT now() | |
| `change_reason` | text | | |

---

## 9. Schema: Audit Log

### ai_ros_audit_log

Located in dedicated `ai_ros_audit` database.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `audit_id` | uuid | PK | |
| `event_type` | varchar(128) | NOT NULL | e.g., `credential.accessed` |
| `actor_id` | uuid | NOT NULL | User or service that caused the event |
| `actor_type` | varchar(32) | NOT NULL | `user` / `service` / `system` |
| `org_id` | uuid | NOT NULL | |
| `resource_type` | varchar(64) | NOT NULL | `credential` / `policy` / `request` / etc. |
| `resource_id` | varchar(256) | NOT NULL | |
| `action` | varchar(64) | NOT NULL | `read` / `create` / `modify` / `delete` |
| `outcome` | varchar(32) | NOT NULL | `success` / `failure` / `denied` |
| `details_json` | jsonb | | Non-sensitive event details |
| `ip_address` | inet | NULL | |
| `user_agent` | text | NULL | |
| `occurred_at` | timestamptz | NOT NULL DEFAULT now() | |

**Partitioned by:** `occurred_at` (monthly)  
**No UPDATE or DELETE** — tamper-evident by append-only constraint + row hash chain.  
**Retention:** 7 years (compliance requirement); older partitions archived to object storage.

---

## 10. Schema: Benchmark / Analytics

### ai_ros_execution_outcomes (in ai_ros_analytics database)

| Column | Type | Notes |
|--------|------|-------|
| `outcome_id` | uuid | PK |
| `request_id` | uuid | Reference (no FK across databases) |
| `org_id` | uuid | |
| `provider_id` | varchar(64) | |
| `canonical_model_id` | varchar(256) | |
| `task_category` | varchar(128) | |
| `latency_ms` | integer | |
| `ttft_ms` | integer | NULL |
| `input_tokens` | integer | |
| `output_tokens` | integer | |
| `total_cost_usd` | numeric(12,8) | |
| `stop_reason` | varchar(32) | |
| `error_code` | varchar(64) | NULL |
| `retry_count` | integer | |
| `automated_quality_score` | real | NULL |
| `human_quality_score` | real | NULL |
| `recorded_at` | timestamptz | |

**Partitioned by:** `recorded_at` (monthly)

---

### ai_ros_benchmark_scores

| Column | Type | Notes |
|--------|------|-------|
| `score_id` | uuid | PK |
| `canonical_model_id` | varchar(256) | |
| `task_category` | varchar(128) | |
| `sample_size` | integer | |
| `success_rate` | real | |
| `avg_quality_score` | real | NULL |
| `p50_latency_ms` | integer | |
| `p95_latency_ms` | integer | |
| `p99_latency_ms` | integer | |
| `avg_cost_usd` | numeric(12,8) | |
| `composite_score` | real | |
| `window_start` | timestamptz | |
| `window_end` | timestamptz | |
| `computed_at` | timestamptz | |

**Unique index:** `(canonical_model_id, task_category, window_start)`

---

## 11. Schema: Credential Vault

Located in dedicated `ai_ros_vault` database with strict connection controls.

### ai_ros_credentials

| Column | Type | Notes |
|--------|------|-------|
| `credential_id` | uuid | PK |
| `org_id` | uuid | NOT NULL |
| `user_id` | uuid | NULL |
| `provider_id` | varchar(64) | NOT NULL |
| `credential_type` | varchar(32) | NOT NULL |
| `encrypted_value` | bytea | NOT NULL (AES-256-GCM) |
| `key_id` | varchar(128) | NOT NULL |
| `alias` | varchar(512) | NOT NULL |
| `status` | varchar(32) | NOT NULL |
| `expiry` | timestamptz | NULL |
| `last_used_at` | timestamptz | NULL |
| `last_validated_at` | timestamptz | NULL |
| `rotation_policy_json` | jsonb | NULL |
| `created_at` | timestamptz | NOT NULL DEFAULT now() |
| `created_by` | uuid | NOT NULL |

**No SELECT permission** for non-vault service roles — credential reads go through Account Manager API only.

---

## 12. Partitioning Strategy

| Table | Partition Key | Partition Size | Hot Retention | Archive |
|-------|--------------|----------------|--------------|---------|
| ai_ros_requests | submitted_at | Monthly | 13 months | Object storage |
| ai_ros_messages | created_at | Monthly | 13 months | Object storage |
| ai_ros_quota_ledger | recorded_at | Monthly | 6 months | Object storage |
| ai_ros_cost_ledger | recorded_at | Monthly | 13 months | Object storage |
| ai_ros_audit_log | occurred_at | Monthly | 7 years | Object storage (compliance) |
| ai_ros_execution_outcomes | recorded_at | Monthly | 13 months | Object storage |
| ai_ros_provider_health | checked_at | Monthly | 6 months | Drop |

---

## 13. Migrations

- Tool: Alembic
- All migrations stored in `ai_ros/alembic/versions/`
- Naming: `{revision}_{short_description}.py`
- Zero-downtime migrations: always additive first (add column nullable), then backfill, then add constraint
- Destructive migrations (drop column/table) require separate migration after 2 major releases

---

## 14. Retention and Archival

| Data Class | Hot Retention | Archive Medium | Archive Retention |
|-----------|--------------|----------------|-----------------|
| Request/response metadata | 13 months | S3-compatible | 7 years |
| Message content | 13 months | S3-compatible (encrypted) | Per org policy |
| Cost records | 13 months | S3-compatible | 7 years (financial) |
| Audit log | 7 years | S3-compatible (WORM) | 7 years (compliance) |
| Health metrics | 6 months | Drop | — |
| Benchmark data | 13 months | S3-compatible | 3 years |

---

## 15. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-PROVIDER-CONTRACTS.md | Data models for requests/responses |
| AI-QUOTA-ENGINE.md | Quota tables detailed here |
| AI-BUDGET-ENGINE.md | Budget/cost tables detailed here |
| AI-ACCOUNT-MANAGER.md | Vault schema |
| AI-RESOURCE-SECURITY.md | Encryption of vault columns |
| AI-RESOURCE-EVENTS.md | Events triggered by database writes |
