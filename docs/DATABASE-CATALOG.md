# AI Studio Platform — Database Catalog

**Document ID:** DATABASE-CATALOG  
**Version:** 1.0.0  
**Date:** 2026-06-28  
**Status:** RATIFIED  
**Classification:** INTERNAL — PLATFORM GOVERNANCE  
**Authority:** Platform Data Architect  
**Predecessor:** PLATFORM-CONTRACTS.md, API-CATALOG.md, EVENT-CATALOG.md  
**Review Cycle:** Quarterly

---

## Catalog Authority Notice

> This catalog is the authoritative logical data architecture for the AI Studio Platform. It defines what data exists, where it lives, who owns it, how long it is retained, and how it evolves.
>
> No schema change may be implemented without a corresponding update to this catalog. No migration may be deployed without a corresponding entry in the Migration Ledger (Appendix A).
>
> This document is logical architecture — it does not contain SQL. Column types, constraint syntax, and DDL belong in the migration files. This catalog defines *what* and *why*; migrations define *how*.

---

## How to Read This Catalog

**Entity notation:** `schema.table_name` — e.g., `security.api_keys`

**Field notation in entity definitions:**

| Notation | Meaning |
|----------|---------|
| `(PK)` | Primary key |
| `(FK → table)` | Foreign key to the referenced table |
| `(UK)` | Unique constraint |
| `(IDX)` | Indexed field |
| `(PART)` | Partition key |
| `?` | Nullable |
| `*` | Required (NOT NULL) |
| `[json]` | Stored as JSON/JSONB |
| `[encrypted]` | Encrypted at rest |
| `[hashed]` | Stored as cryptographic hash, never plaintext |

**ER diagram notation:**
```
||──|| = one-to-one
||──|{ = one-to-many
}|──|{ = many-to-many (via junction table)
──── = soft reference (no FK constraint)
```

---

---

# Section 1: Database Inventory

## 1.1 Physical Database Topology

The AI Studio Platform uses two persistent storage backends and one in-memory store.

| Store | Technology | Version | Role | Persistence |
|-------|-----------|---------|------|-------------|
| **Primary Database** | PostgreSQL | 15.x | All relational data, audit log, billing, prompts, workflows, brain, knowledge (SQL phase) | Durable — WAL-backed |
| **Knowledge Graph** | Kuzu (planned: Neo4j) | 0.x → 4.x | Graph nodes, relationships, pattern traversal | Durable |
| **Event Bus** | NATS JetStream | 2.10.x | Platform events (see EVENT-CATALOG) | Durable — file-backed |
| **Cache** | In-process LRU | — | Prompt template cache (60s TTL), workspace registry (30s TTL) | Ephemeral — no persistence |

### What Lives Where

| Data Domain | Primary DB | Knowledge Graph | Event Bus | Notes |
|-------------|-----------|----------------|-----------|-------|
| Authentication & Keys | ✓ | | | |
| Workspace Registry | ✓ | | Events only | |
| Workflow Execution | ✓ | | Events only | |
| Prompt OS | ✓ | | Events only | |
| Central Brain | ✓ | ✓ (patterns) | Events only | Experiences in PG; pattern graph in KG |
| Knowledge Graph | ✓ (now) | ✓ (v1.1) | Events only | Full migration to Kuzu in v1.1 |
| Provider Registry | ✓ | | Events only | |
| Billing & Cost | ✓ | | Events only | |
| Usage Analytics | ✓ | | Events only | |
| Audit Log | ✓ | | | Immutable; append-only |
| Plugin Registry | ✓ | | Events only | |
| Marketplace | ✓ | | Events only | PROPOSED |
| Monitoring | ✓ | | Events only | Alerts, SLA, health checks |
| Secrets | ✓ (hashed only) | | | Raw secrets in env/vault only |

## 1.2 PostgreSQL Schema Inventory

PostgreSQL schema = a namespace within the database. Each platform domain owns one schema.

| Schema | Domain | Owner | Tables | Row Volume (est.) |
|--------|--------|-------|--------|------------------|
| `security` | Authentication, keys, audit | Platform Security | 3 tables | Medium (audit: very high) |
| `workspace` | Products, health, modules | Platform Workspace | 3 tables | Low |
| `workflow` | Workflows, tasks, gates, checkpoints | Platform Workflow Runtime | 5 tables | High |
| `prompt` | Templates, versions, governance, A/B | Platform Prompt OS | 6 tables | Low–Medium |
| `brain` | Experiences, patterns, recommendations | Platform Central Brain | 5 tables | High (experiences) |
| `knowledge` | Graph nodes, relationships, memory (SQL phase) | Platform Knowledge | 3 tables | Medium–High |
| `provider` | Provider registry, models, health | Platform AI ROS | 4 tables | Low |
| `billing` | Costs, budgets, invoices | Platform Billing | 5 tables | Very High (cost records) |
| `usage` | Aggregated analytics | Platform Usage Aggregator | 4 tables | Medium |
| `plugins` | Plugin registry, invocations | Platform Plugin Manager | 3 tables | Low–Medium |
| `marketplace` | Listings, installations, reviews | Platform Marketplace (PROPOSED) | 3 tables | Low |
| `monitoring` | Alerts, SLA, health checks | Platform Monitoring | 3 tables | Medium |

**Total: 12 schemas, 42 tables (38 active + 4 proposed)**

## 1.3 Knowledge Graph Inventory

| Graph | Backend (now) | Backend (v1.1) | Entities | Purpose |
|-------|--------------|----------------|---------|---------|
| `knowledge_graph` | PostgreSQL (SQL tables) | Kuzu | Nodes + Relationships | Workspace knowledge: files, modules, concepts, people |
| `brain_pattern_graph` | In-memory + PostgreSQL | Qdrant (vector) | Experience clusters | Brain pattern similarity matching |

---

---

# Section 2: Schema Ownership

Each schema is owned by exactly one platform module. Ownership means:
- Only the owning module may write to the schema
- No other module may reference schema tables via foreign key
- Cross-schema data access is always via API call, not SQL JOIN

| Schema | Owning Module | Path |
|--------|--------------|------|
| `security` | Security Service | `platform/security` |
| `workspace` | Workspace Manager | `platform/workspace` |
| `workflow` | Workflow Runtime | `platform/workflow-runtime` |
| `prompt` | Prompt OS | `platform/prompt-os` |
| `brain` | Central Brain | `platform/brain` |
| `knowledge` | Knowledge Service | `platform/knowledge` |
| `provider` | AI ROS / Provider Registry | `platform/ai-ros` |
| `billing` | Billing Service | `platform/billing` |
| `usage` | Usage Aggregator | `platform/usage` |
| `plugins` | Plugin Manager | `platform/plugin-manager` |
| `marketplace` | Marketplace Service (PROPOSED) | `platform/marketplace` |
| `monitoring` | Monitoring Service | `platform/monitoring` |

### Cross-Schema Access Rules

**Rule DB-OWN-1:** A module may only write to its own schema. All cross-schema data flows are via events (NATS) or API calls.

**Rule DB-OWN-2:** No cross-schema foreign keys. `workflow.workflows.product_id` is a string reference to a product — not a foreign key to `workspace.products`. This is intentional: schemas must be independently deployable and testable.

**Rule DB-OWN-3:** Read replicas may serve cross-schema reads for analytics dashboards only. All transactional reads go to the primary.

**Rule DB-OWN-4:** The `security.audit_log` schema may receive writes from any module via the audit log service interface — not direct SQL writes.

---

---

# Section 3: Bounded Context Mapping

A bounded context groups related entities that form a coherent domain model. Each context maps to one or more PostgreSQL schemas and has one aggregate root.

```
╔══════════════════════════════════════════════════════════════════════════════╗
║                    BOUNDED CONTEXTS — AI STUDIO PLATFORM                    ║
╠══════════════════════╦══════════════════════╦══════════════════════════════╗
║  IDENTITY CONTEXT    ║  EXECUTION CONTEXT   ║  INTELLIGENCE CONTEXT        ║
║                      ║                      ║                              ║
║  schema: security    ║  schema: workflow    ║  schema: brain               ║
║                      ║  schema: provider    ║  schema: knowledge           ║
║  ApiKey              ║                      ║                              ║
║  AuditEntry          ║  Workflow            ║  Experience                  ║
║  RateLimitBucket     ║  Task                ║  Pattern                     ║
║                      ║  Gate                ║  KnowledgeNode               ║
╠══════════════════════╬══════════════════════╬══════════════════════════════╣
║  CONTENT CONTEXT     ║  COMMERCIAL CONTEXT  ║  OPERATIONS CONTEXT          ║
║                      ║                      ║                              ║
║  schema: prompt      ║  schema: billing     ║  schema: workspace           ║
║                      ║  schema: usage       ║  schema: monitoring          ║
║  PromptTemplate      ║                      ║  schema: plugins             ║
║  PromptVersion       ║  CostRecord          ║  schema: marketplace*        ║
║  ABTest              ║  Budget              ║                              ║
║                      ║  Invoice             ║  Product                     ║
║                      ║                      ║  Plugin                      ║
║                      ║                      ║  Alert                       ║
╚══════════════════════╩══════════════════════╩══════════════════════════════╝
                                                           * PROPOSED
```

### Context Integration Points

Contexts integrate exclusively via events. Direct database access across context boundaries is forbidden.

| From Context | To Context | Integration Method | Trigger |
|-------------|-----------|-------------------|---------|
| Execution | Intelligence | `ai.execution.completed` event | Brain records experience |
| Execution | Commercial | `ai.execution.completed` event | Billing records cost |
| Execution | Content | `WorkflowClient.submit()` API call | Tasks request prompt renders |
| Identity | Operations | `security.api_key.created` event | Workspace tracks active keys |
| Intelligence | Content | `brain.pattern.discovered` event | Brain may recommend prompt template |
| Commercial | Execution | Budget check via `AIClient.execute()` | AI ROS blocks over-budget requests |
| Operations | Execution | `workspace.product.registered` event | Workflow validates product exists |

---

---

# Section 4: Entity Ownership

Every persistent entity is owned by exactly one aggregate root. Writes flow through the root; reads may bypass it.

| Entity | Schema.Table | Aggregate Root | Context |
|--------|-------------|---------------|---------|
| ApiKey | `security.api_keys` | ApiKey | Identity |
| AuditEntry | `security.audit_log` | AuditEntry (append-only) | Identity |
| RateLimitBucket | `security.rate_limit_buckets` | ApiKey | Identity |
| Product | `workspace.products` | Product | Operations |
| ProductHealthHistory | `workspace.product_health_history` | Product | Operations |
| PlatformModule | `workspace.platform_modules` | PlatformModule | Operations |
| Workflow | `workflow.workflows` | Workflow | Execution |
| WorkflowTask | `workflow.tasks` | Workflow | Execution |
| TaskOutput | `workflow.task_outputs` | Workflow | Execution |
| WorkflowCheckpoint | `workflow.checkpoints` | Workflow | Execution |
| WorkflowGate | `workflow.gates` | Workflow | Execution |
| PromptTemplate | `prompt.templates` | PromptTemplate | Content |
| PromptVersion | `prompt.versions` | PromptTemplate | Content |
| GovernanceTransition | `prompt.governance_log` | PromptTemplate | Content |
| ABTest | `prompt.ab_tests` | PromptTemplate | Content |
| ABTestResult | `prompt.ab_test_results` | ABTest | Content |
| Experience | `brain.experiences` | Experience | Intelligence |
| Pattern | `brain.patterns` | Pattern | Intelligence |
| ImprovementProposal | `brain.improvement_proposals` | Pattern | Intelligence |
| KnowledgeNode | `knowledge.nodes` | KnowledgeNode | Intelligence |
| KnowledgeRelationship | `knowledge.relationships` | KnowledgeNode (from) | Intelligence |
| KnowledgeMemory | `knowledge.memory` | KnowledgeMemory | Intelligence |
| Provider | `provider.providers` | Provider | Execution |
| ProviderModel | `provider.models` | Provider | Execution |
| ProviderHealthRecord | `provider.health_history` | Provider | Execution |
| CircuitBreakerEvent | `provider.circuit_events` | Provider | Execution |
| CostRecord | `billing.cost_records` | CostRecord (immutable) | Commercial |
| BudgetConfig | `billing.budget_configs` | BudgetConfig | Commercial |
| BudgetSnapshot | `billing.budget_snapshots` | BudgetConfig | Commercial |
| Invoice | `billing.invoices` | Invoice | Commercial |
| InvoiceLineItem | `billing.invoice_line_items` | Invoice | Commercial |
| HourlyTokenSummary | `usage.hourly_token_summaries` | (time series, no root) | Commercial |
| DailyRequestSummary | `usage.daily_request_summaries` | (time series) | Commercial |
| DailyWorkflowSummary | `usage.daily_workflow_summaries` | (time series) | Commercial |
| WeeklyBrainSummary | `usage.weekly_brain_summaries` | (time series) | Commercial |
| Plugin | `plugins.plugins` | Plugin | Operations |
| PluginHealthCheck | `plugins.health_checks` | Plugin | Operations |
| PluginInvocation | `plugins.invocations` | Plugin | Operations |
| Alert | `monitoring.alerts` | Alert | Operations |
| SLAMeasurement | `monitoring.sla_measurements` | (time series) | Operations |
| HealthCheckResult | `monitoring.health_check_results` | (time series) | Operations |
| MarketplaceListing | `marketplace.listings` | MarketplaceListing | Operations |
| MarketplaceInstallation | `marketplace.installations` | MarketplaceListing | Operations |
| MarketplaceReview | `marketplace.reviews` | MarketplaceListing | Operations |

---

---

# Section 5: ER Diagrams

ASCII ER diagrams showing entity relationships within and across schemas. Cross-schema arrows are labeled as "soft ref" — no FK constraint enforced at database level.

---

## 5.1 Identity Context — Security Schema

```
┌──────────────────────────────┐
│  security.api_keys           │
├──────────────────────────────┤
│  key_id          (PK)        │
│  key_hash        * [hashed]  │──────────────┐
│  key_prefix      *           │              │
│  role            *           │              │
│  label           *           │              │
│  scope           *           │              │
│  product_id      ? (IDX)     │─ soft ref ──▶ workspace.products
│  expires_at      ?           │              │
│  allowed_ips     ? [json]    │              │
│  status          * (IDX)     │              │
│  created_by      * [hashed]  │              │
│  created_at      * (IDX)     │              │
│  revoked_at      ?           │              │
│  revocation_reason ?         │              │
│  last_used_at    ?           │              │
└──────────────────────────────┘              │
          ||                                  │
          ||  1                               │
          ||                                  │
          |{  N                               │
┌──────────────────────────────┐              │
│  security.rate_limit_buckets │              │
├──────────────────────────────┤              │
│  bucket_id       (PK)        │              │
│  key_id          * (FK)      │◀─────────────┘
│  bucket_type     * (IDX)     │
│  window_start    * (IDX)     │
│  request_count   *           │
│  updated_at      *           │
└──────────────────────────────┘

                    Separate entity (no FK to api_keys — audit is immutable)
┌──────────────────────────────────────────────────────┐
│  security.audit_log                                  │
├──────────────────────────────────────────────────────┤
│  entry_id         (PK)         — UUID                │
│  event_type       * (IDX)      — e.g. "api_key.created"
│  severity         * (IDX)      — LOW/MEDIUM/HIGH/CRITICAL
│  timestamp        * (IDX,PART) — partition by month  │
│  identity_hash    * (IDX)      — SHA-256 of key      │
│  ip_hash          ? [hashed]   — SHA-256 of IP       │
│  path             ?            — request path        │
│  method           ?            — HTTP method         │
│  resource_id      ? (IDX)      — affected resource   │
│  correlation_id   * (IDX)                            │
│  details_json     * [json]     — event-specific data │
│  product_id       ? (IDX)      — soft ref            │
└──────────────────────────────────────────────────────┘
  NOTE: audit_log is APPEND-ONLY. No UPDATE or DELETE permitted.
  Enforced by: PostgreSQL row-level security (RLS) — insert only.
```

---

## 5.2 Operations Context — Workspace Schema

```
┌───────────────────────────────────┐
│  workspace.products               │
├───────────────────────────────────┤
│  product_id         (PK)  *       │
│  product_name       *             │
│  version            * (IDX)       │
│  api_prefix         *             │
│  capabilities       * [json]      │
│  required_modules   * [json]      │
│  health_status      * (IDX)       │
│  registered_at      *             │
│  last_heartbeat_at  * (IDX)       │
│  registered_by      * [hashed]    │
└───────────────────────────────────┘
               ||
               || 1
               ||
               |{ N
┌───────────────────────────────────┐
│  workspace.product_health_history │
├───────────────────────────────────┤
│  record_id     (PK)  *            │
│  product_id    * (FK,IDX)         │
│  status        *                  │
│  changed_at    * (IDX,PART)       │   ← partition by month
│  error_code    ?                  │
│  error_summary ?                  │
└───────────────────────────────────┘

┌───────────────────────────────────┐
│  workspace.platform_modules       │
├───────────────────────────────────┤
│  module_id       (PK)  *          │
│  module_name     *                │
│  health_status   * (IDX)          │
│  last_check_at   * (IDX)          │
│  version         ?                │
│  error_summary   ?                │
└───────────────────────────────────┘
  NOTE: No FK relationship with products.
  Products reference modules by ID in their required_modules JSON array.
```

---

## 5.3 Execution Context — Workflow Schema

```
┌──────────────────────────────────────────┐
│  workflow.workflows                      │
├──────────────────────────────────────────┤
│  workflow_id      (PK)   * UUID          │
│  product_id       *  (IDX) soft ref      │──── soft ref ──▶ workspace.products
│  plan_name        *                      │
│  status           * (IDX)                │
│  priority         * (IDX)                │
│  task_count       *                      │
│  gate_count       *                      │
│  idempotency_key  ? (UK) [product_id+key]│
│  submitted_by     * [hashed]             │
│  submitted_at     * (IDX)                │
│  started_at       ?                      │
│  completed_at     ? (IDX)                │
│  total_cost_usd   ?                      │
│  total_tokens     ?                      │
│  sla_met          ?                      │
│  metadata         ? [json]               │
└──────────────────────────────────────────┘
     ||                ||                ||
     || 1              || 1              || 1
     ||                ||                ||
     |{ N              |{ N              |{ N
┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐
│ workflow.    │  │ workflow.    │  │ workflow.checkpoints  │
│ tasks        │  │ gates        │  ├──────────────────────┤
├──────────────┤  ├──────────────┤  │ checkpoint_id (PK) * │
│ task_row_id  │  │ gate_row_id  │  │ workflow_id * (FK,UK)│
│ (PK)         │  │ (PK)         │  │ completed_tasks [json]│
│ workflow_id  │  │ workflow_id  │  │ checkpoint_data [json]│
│ * (FK,IDX)   │  │ * (FK,IDX)   │  │ created_at *         │
│ task_id *    │  │ gate_id *    │  └──────────────────────┘
│ task_type *  │  │ after_task_id│    (one checkpoint per
│ agent_type * │  │ before_task_id    workflow, replaced
│ status * IDX │  │ required_role│    on each checkpoint)
│ priority *   │  │ description *│
│ timeout_s *  │  │ status * IDX │
│ max_retries *│  │ resolved_by  │
│ retry_count *│  │ [hashed] ?   │
│ started_at ? │  │ resolved_at? │
│ completed_at?│  │ comment ?    │
│ error_code ? │  └──────────────┘
│ error_cat ?  │
│ tools [json] │
│ depends_on   │
│ [json]       │
└──────┬───────┘
       || 1
       ||
       |{ 0..1  (only COMPLETED tasks have output)
┌──────────────────────────────┐
│  workflow.task_outputs       │
├──────────────────────────────┤
│  output_id         (PK)  *   │
│  workflow_id       * (FK,IDX)│ ─── also FK to workflow.workflows
│  task_id           * (IDX)   │
│  output_json       * [json]  │
│  artifact_keys     ? [json]  │
│  tool_calls        ? [json]  │
│  ai_execution_ids  ? [json]  │
│  created_at        *         │
└──────────────────────────────┘
  UK constraint: (workflow_id, task_id)
  — one output per task
```

---

## 5.4 Content Context — Prompt Schema

```
┌──────────────────────────────────────┐
│  prompt.templates                    │
├──────────────────────────────────────┤
│  template_id     (PK) * UUID         │
│  display_name    *                   │
│  owner           * (IDX)             │
│  created_by      * [hashed]          │
│  created_at      *                   │
│  active_version  ? (IDX)             │ ← null until first REVIEW→ACTIVE
│  tags            ? [json]            │
│  description     ?                   │
└──────────────────────────────────────┘
          ||
          || 1
          ||
          |{ N
┌──────────────────────────────────────┐
│  prompt.versions                     │
├──────────────────────────────────────┤
│  version_row_id   (PK)  *            │
│  template_id      * (FK,IDX)         │
│  version          * (IDX)            │
│  content          * [text]           │
│  system_prompt    ? [text]           │
│  variables        * [json]           │
│  state            * (IDX)            │
│  hmac_signature   * [hashed]         │
│  content_hash     * [hashed]         │
│  parent_version   ? (IDX)            │
│  created_by       * [hashed]         │
│  created_at       *                  │
│  change_notes     *                  │
└──────────────────────────────────────┘
  UK constraint: (template_id, version)

          ||
          || 1 (template)
          ||
          |{ N
┌──────────────────────────────────────┐
│  prompt.governance_log               │
├──────────────────────────────────────┤
│  transition_id      (PK) *           │
│  template_id        * (FK,IDX)       │
│  version            *                │
│  from_state         *                │
│  to_state           *                │
│  transitioned_by    * [hashed]       │
│  transitioned_at    * (IDX)          │
│  notes              ?                │
│  removal_after      ?                │
│  replacement_id     ? soft ref       │
└──────────────────────────────────────┘
  APPEND-ONLY: governance history is immutable.

┌──────────────────────────────────────┐
│  prompt.ab_tests                     │
├──────────────────────────────────────┤
│  test_id            (PK) *           │
│  template_id        * (FK,IDX)       │
│  variant_a_version  *                │
│  variant_b_version  *                │
│  started_at         *                │
│  ended_at           ?                │
│  winner_variant     ?                │
│  status             * (IDX)          │
│  sample_size        ?                │
│  confidence_pct     ?                │
└──────────────────────────────────────┘
          ||
          || 1
          ||
          |{ N
┌──────────────────────────────────────┐
│  prompt.ab_test_results              │
├──────────────────────────────────────┤
│  result_id       (PK)  *             │
│  test_id         * (FK,IDX)          │
│  render_id       * (UK)              │ ← idempotency: one result per render
│  variant         * (IDX)             │
│  success         *                   │
│  feedback        ?                   │
│  recorded_at     * (IDX)             │
└──────────────────────────────────────┘
```

---

## 5.5 Intelligence Context — Brain Schema

```
┌──────────────────────────────────────────┐
│  brain.experiences                       │
├──────────────────────────────────────────┤
│  experience_id      (PK)  * UUID         │
│  product_id         * (IDX)  soft ref    │
│  task_type          * (IDX)              │
│  agent_type         * (IDX)              │
│  outcome            * (IDX)              │
│  duration_ms        *                    │
│  total_tokens       *                    │
│  total_cost_usd     *                    │
│  retry_count        *                    │
│  tools_used         * [json]             │
│  prompt_template_id ? (IDX)  soft ref    │
│  workflow_id        ? (IDX)  soft ref    │
│  context_hash       * (IDX)              │ ← hash of context for grouping
│  context_embedding  ? [json]             │ ← reserved for Qdrant migration (v1.1)
│  failure_reason     ?                    │
│  notes              ?                    │
│  recorded_at        * (IDX,PART)         │ ← partition by month
└──────────────────────────────────────────┘
          }|
          }| N
          ||
          || M (via junction)
┌────────────────────────────┐
│  brain.pattern_experiences │   ← junction table
├────────────────────────────┤
│  pattern_id   * (FK,IDX)   │
│  experience_id* (FK,IDX)   │
│  similarity   *            │
└────────────────────────────┘
          ||
          || M
          ||
          |{ 1
┌──────────────────────────────────────────┐
│  brain.patterns                          │
├──────────────────────────────────────────┤
│  pattern_id            (PK) *            │
│  name                  *                 │
│  task_types            * [json]          │
│  experience_count      *                 │
│  avg_success_rate      *                 │
│  avg_duration_ms       *                 │
│  avg_cost_usd          *                 │
│  context_centroid      * [json]          │ ← centroid of context space
│  top_agents            * [json]          │ ← ranked list
│  top_tools             * [json]          │
│  discovered_at         *                 │
│  last_updated_at       * (IDX)           │
└──────────────────────────────────────────┘
          ||
          || 1
          ||
          |{ N
┌──────────────────────────────────────────┐
│  brain.improvement_proposals             │
├──────────────────────────────────────────┤
│  proposal_id         (PK) *              │
│  pattern_id          * (FK,IDX)          │
│  proposal_type       * (IDX)             │
│  description         *                   │
│  affected_task_type  * (IDX)             │
│  estimated_impr_pct  *                   │
│  confidence          *                   │
│  evidence_count      *                   │
│  status              * (IDX)             │
│  proposed_at         *                   │
│  resolved_at         ?                   │
│  resolved_by         ? [hashed]          │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  brain.recommendation_cache              │
├──────────────────────────────────────────┤
│  cache_id         (PK) *                 │
│  context_hash     * (UK,IDX)             │ ← cache key
│  product_id       * (IDX)               │
│  recommendations  * [json]               │
│  computed_at      * (IDX)               │
│  expires_at       * (IDX)               │
└──────────────────────────────────────────┘
  TTL enforced by: scheduled cleanup job (not DB expiry)
```

---

## 5.6 Intelligence Context — Knowledge Schema

```
┌──────────────────────────────────────────┐
│  knowledge.nodes                         │
├──────────────────────────────────────────┤
│  node_id        (PK)  * string (UUID)    │
│  node_type      * (IDX)                  │
│  label          * (IDX)  [full-text]     │
│  properties     * [json]                 │
│  workspace_id   ? (IDX)  soft ref        │
│  created_by     * (IDX)                  │
│  created_at     *                        │
│  updated_at     * (IDX)                  │
└──────────────────────────────────────────┘
          ||
          || 1 (from)
          ||
          |{ N
┌──────────────────────────────────────────┐
│  knowledge.relationships                 │
├──────────────────────────────────────────┤
│  relationship_id   (PK) *                │
│  relationship_type * (IDX)               │
│  from_node_id      * (FK→nodes, IDX)     │
│  to_node_id        * (FK→nodes, IDX)     │
│  properties        ? [json]              │
│  created_at        *                     │
└──────────────────────────────────────────┘
          NOTE: Both FKs reference knowledge.nodes.
          Both are indexed for bidirectional traversal.

┌──────────────────────────────────────────┐
│  knowledge.memory                        │
├──────────────────────────────────────────┤
│  memory_id      (PK) *                   │
│  memory_key     * (IDX)                  │
│  scope          * (IDX)                  │
│  scope_id       ? (IDX)                  │
│  value_json     * [json]                 │
│  value_size_bytes *                      │
│  ttl_seconds    ?                        │
│  expires_at     ? (IDX)                  │ ← null if no expiry
│  set_by         *                        │
│  set_at         *                        │
│  updated_at     *                        │
└──────────────────────────────────────────┘
  UK constraint: (memory_key, scope, scope_id)
  — one value per (key, scope, scope_id) combination
```

---

## 5.7 Execution Context — Provider Schema

```
┌──────────────────────────────────────────┐
│  provider.providers                      │
├──────────────────────────────────────────┤
│  provider_id          (PK) *             │
│  display_name         *                  │
│  health_status        * (IDX)            │
│  circuit_state        * (IDX)            │
│  circuit_open_until   ?                  │
│  registered_at        *                  │
│  configuration_hash   * [hashed]         │ ← hash to detect config drift
└──────────────────────────────────────────┘
          ||
          || 1
          ||
          |{ N
┌──────────────────────────────────────────┐
│  provider.models                         │
├──────────────────────────────────────────┤
│  model_row_id          (PK) *            │
│  provider_id           * (FK,IDX)        │
│  model_id              * (IDX)           │
│  display_name          *                 │
│  context_window_tokens *                 │
│  max_output_tokens     *                 │
│  supports_streaming    *                 │
│  supports_fn_calling   *                 │
│  supports_vision       *                 │
│  input_cost_per_1k     *                 │
│  output_cost_per_1k    *                 │
│  active                * (IDX)           │
│  effective_from        *                 │ ← pricing is versioned
│  effective_until       ?                 │
└──────────────────────────────────────────┘
  UK constraint: (provider_id, model_id, effective_from)

          ||
          || 1
          ||
          |{ N
┌──────────────────────────────────────────┐
│  provider.health_history                 │
├──────────────────────────────────────────┤
│  record_id       (PK)  *                 │
│  provider_id     * (FK,IDX)              │
│  health_status   *                       │
│  circuit_state   *                       │
│  latency_p50_ms  ?                       │
│  latency_p99_ms  ?                       │
│  error_rate      ?                       │
│  checked_at      * (IDX,PART)            │ ← partition by day
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  provider.circuit_events                 │
├──────────────────────────────────────────┤
│  event_id        (PK) *                  │
│  provider_id     * (FK,IDX)              │
│  from_state      *                       │
│  to_state        *                       │
│  failure_count   ?                       │
│  failure_reason  ?                       │
│  occurred_at     * (IDX)                 │
│  reset_by        ? [hashed]              │ ← non-null if manually reset
└──────────────────────────────────────────┘
```

---

## 5.8 Commercial Context — Billing Schema

```
┌──────────────────────────────────────────────────┐
│  billing.cost_records                            │
├──────────────────────────────────────────────────┤
│  cost_record_id   (PK) * UUID  [deterministic]   │ ← SHA-256(execution_id+billing_date)
│  execution_id     * (UK,IDX)  soft ref           │ ← unique: one record per execution
│  product_id       * (IDX)     soft ref           │
│  workflow_id      ? (IDX)     soft ref           │
│  task_id          ? (IDX)                        │
│  provider_id      * (IDX)                        │
│  model_id         * (IDX)                        │
│  prompt_tokens    *                              │
│  completion_tokens *                             │
│  cache_read_tokens *                             │
│  cache_write_tokens *                            │
│  total_tokens     *                              │
│  cost_usd         *           (6 decimal places) │
│  billing_date     * (IDX,PART)                   │ ← partition by day
│  billing_period   * (IDX)     format: "2026-06"  │
│  recorded_at      *                              │
└──────────────────────────────────────────────────┘
  IMMUTABLE: No UPDATE or DELETE. Corrections via reversal entries.
  Retention: 365 days minimum (see Section 11)

┌──────────────────────────────────────────────────┐
│  billing.budget_configs                          │
├──────────────────────────────────────────────────┤
│  config_id        (PK) *                         │
│  product_id       * (IDX)    soft ref            │
│  daily_limit_usd  *                              │
│  monthly_limit_usd ?                             │
│  effective_date   * (IDX)                        │
│  configured_by    * [hashed]                     │
│  created_at       *                              │
└──────────────────────────────────────────────────┘
  UK: (product_id, effective_date)
  Active budget = MAX(effective_date) <= today

          ||
          || 1
          ||
          |{ N
┌──────────────────────────────────────────────────┐
│  billing.budget_snapshots                        │
├──────────────────────────────────────────────────┤
│  snapshot_id       (PK)  *                       │
│  product_id        * (FK,IDX)  soft ref          │
│  billing_date      * (IDX)                       │
│  daily_spend_usd   *                             │
│  daily_limit_usd   *                             │
│  snapshot_at       *                             │
└──────────────────────────────────────────────────┘
  UK: (product_id, billing_date) — one snapshot per product per day

┌──────────────────────────────────────────────────┐
│  billing.invoices                                │
├──────────────────────────────────────────────────┤
│  invoice_id        (PK) *                        │
│  billing_period    * (UK)  format: "2026-06"     │
│  status            * (IDX) DRAFT / FINALIZED     │
│  total_cost_usd    *                             │
│  total_executions  *                             │
│  total_tokens      *                             │
│  generated_at      *                             │
│  finalized_at      ?                             │
└──────────────────────────────────────────────────┘
          ||
          || 1
          ||
          |{ N
┌──────────────────────────────────────────────────┐
│  billing.invoice_line_items                      │
├──────────────────────────────────────────────────┤
│  line_id           (PK) *                        │
│  invoice_id        * (FK,IDX)                    │
│  product_id        * (IDX)                       │
│  provider_id       * (IDX)                       │
│  model_id          * (IDX)                       │
│  execution_count   *                             │
│  total_tokens      *                             │
│  cost_usd          *                             │
└──────────────────────────────────────────────────┘
```

---

## 5.9 Operations Context — Plugin Schema

```
┌──────────────────────────────────────────┐
│  plugins.plugins                         │
├──────────────────────────────────────────┤
│  plugin_id             (PK) *            │
│  plugin_name           *                 │
│  version               * (IDX)           │
│  plugin_type           * (IDX)           │
│  status                * (IDX)           │
│  granted_permissions   * [json]          │
│  configuration         ? [json]          │ ← sanitized, no secrets
│  integrity_hash        * [hashed]        │
│  signature_verified    *                 │
│  bundle_url_hash       * [hashed]        │ ← hash of bundle URL, not URL itself
│  installed_by          * [hashed]        │
│  installed_at          *                 │
│  disabled_at           ?                 │
│  disable_reason        ?                 │
│  uninstalled_at        ?                 │
│  invocation_count      *                 │
│  last_invoked_at       ?                 │
└──────────────────────────────────────────┘
          ||
          || 1
          ||
          |{ N
┌──────────────────────────────────────────┐
│  plugins.health_checks                   │
├──────────────────────────────────────────┤
│  check_id             (PK) *             │
│  plugin_id            * (FK,IDX)         │
│  passed               *                  │
│  consecutive_failures *                  │
│  error_detail         ?                  │
│  checked_at           * (IDX,PART)       │ ← partition by day
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  plugins.invocations                     │
├──────────────────────────────────────────┤
│  invocation_id    (PK) *                 │
│  plugin_id        * (FK,IDX)             │
│  execution_id     ? (IDX)  soft ref      │
│  status           * (IDX)                │
│  duration_ms      *                      │
│  error_code       ?                      │
│  error_category   ?                      │
│  invoked_at       * (IDX,PART)           │ ← partition by day
└──────────────────────────────────────────┘
  Retention: 7 days (high volume; analytics use usage schema summaries)
```

---

## 5.10 Operations Context — Monitoring Schema

```
┌──────────────────────────────────────────┐
│  monitoring.alerts                       │
├──────────────────────────────────────────┤
│  alert_id          (PK) *                │
│  alert_name        * (IDX)               │
│  severity          * (IDX)               │
│  summary           *                     │
│  labels            * [json]              │
│  source_event_type * (IDX)               │
│  source_event_id   ? (IDX)               │
│  fired_at          * (IDX)               │
│  resolved_at       ? (IDX)               │
│  resolution_cause  ?                     │
│  runbook_url       ?                     │
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  monitoring.sla_measurements             │
├──────────────────────────────────────────┤
│  measurement_id    (PK) *                │
│  sla_type          * (IDX)               │
│  entity_id         * (IDX)               │
│  product_id        ? (IDX)  soft ref     │
│  sla_target_secs   *                     │
│  actual_secs       *                     │
│  breach            *  boolean            │
│  measured_at       * (IDX,PART)          │ ← partition by day
└──────────────────────────────────────────┘

┌──────────────────────────────────────────┐
│  monitoring.health_check_results         │
├──────────────────────────────────────────┤
│  check_id          (PK) *                │
│  module_id         * (IDX)               │
│  check_type        *                     │
│  expected_status   *                     │
│  actual_status     ?                     │
│  latency_ms        *                     │
│  error_detail      ?                     │
│  checked_at        * (IDX,PART)          │ ← partition by day
└──────────────────────────────────────────┘
```


---

---

# Section 6: Aggregate Roots

An aggregate root is the entry point for all writes to a cluster of related entities. No entity within the cluster may be modified except through its root.

---

## 6.1 ApiKey (Identity Context)

**Root:** `security.api_keys`  
**Owns:** `security.rate_limit_buckets`  
**Invariants:**
- A key can only be created by a principal whose role is >= the new key's role
- The last ADMIN key in the system cannot be revoked
- Key hash is computed once at creation and never updated
- Revocation is irreversible — no un-revoke operation

**State Machine:**

```
ACTIVE ──────────────────────────────▶ REVOKED
  │                                      ▲
  │                                      │
  └──▶ EXPIRED (when expires_at < now) ──┘
         (auto-transition by TTL check)
```

**Consistency boundary:** A `rate_limit_bucket` is always consistent with its `api_key` within the same transaction. Buckets are created lazily on first use.

---

## 6.2 Workflow (Execution Context)

**Root:** `workflow.workflows`  
**Owns:** `workflow.tasks`, `workflow.gates`, `workflow.checkpoints`, `workflow.task_outputs`  
**Invariants:**
- A Workflow must have at least one Task
- A Task's `depends_on` list may only reference task_ids within the same Workflow
- The `dag_edges` must form a directed acyclic graph — no cycles
- A Gate's `after_task_id` must be a valid task_id in the same Workflow
- No task may enter IN_PROGRESS unless all its `depends_on` tasks are COMPLETED
- A Workflow in COMPLETED, FAILED, or CANCELLED state is immutable
- `auto_approve_after_seconds` is not a valid field — presence returns 400

**State Machine:**

```
                    ┌──────────────────────┐
                    │  QUEUED              │
                    └──────────┬───────────┘
                               │ first task dispatched
                               ▼
                    ┌──────────────────────┐
              ┌────▶│  ACTIVE              │
              │     └──┬──────┬────────────┘
              │        │      │ gate reached / pause()
              │        │      ▼
              │        │   ┌──────────┐
              │        │   │  PAUSED  │◀──────────────────────┐
              │        │   └────┬─────┘                       │
              │        │        │ resume() / gate approved     │
              └────────┘        └─────────────────────────────┘
     (resume sends back to ACTIVE)
              
              │ all tasks COMPLETED    │ task final-fail   │ cancel()
              ▼                        ▼                   ▼
        ┌──────────┐           ┌────────────┐    ┌──────────────┐
        │ COMPLETED│           │   FAILED   │    │  CANCELLED   │
        └──────────┘           └────────────┘    └──────────────┘
```

**Checkpoint policy:** A checkpoint is written after every completed task. The checkpoint is a single row (UK on workflow_id) that is replaced on each write. This is safe because task completion is idempotent — replaying from the last checkpoint yields the same result.

---

## 6.3 PromptTemplate (Content Context)

**Root:** `prompt.templates`  
**Owns:** `prompt.versions`, `prompt.governance_log`, `prompt.ab_tests`, `prompt.ab_test_results`  
**Invariants:**
- A Template may have only one ACTIVE version at a time
- A new version always starts in DRAFT state
- State transitions must follow the governance FSM exactly
- HMAC of the version content is computed at create time and verified before every render
- A SUSPENDED template blocks all renders immediately — no exceptions
- `governance_log` is append-only; no governance transition may be deleted or modified

**Governance FSM:**

```
  DRAFT ──────────────────▶ REVIEW ──────────────────▶ ACTIVE
    ▲                                                     │
    │  (new version inherits nothing from parent)         │ deprecated
    │                                                     ▼
    │                                              DEPRECATED ──▶ ARCHIVED
    │
    └── ACTIVE ──▶ SUSPENDED ──▶ ACTIVE  (emergency suspend/lift)
```

---

## 6.4 Experience (Intelligence Context)

**Root:** `brain.experiences`  
**Owns:** Nothing directly (Pattern is a separate aggregate that references experiences)  
**Note:** Experience is write-once. Once recorded, it is never modified. Pattern generation is a read-only projection over experiences, not a mutation of them.

---

## 6.5 Pattern (Intelligence Context)

**Root:** `brain.patterns`  
**Owns:** `brain.pattern_experiences` (junction), `brain.improvement_proposals`  
**Invariants:**
- A Pattern is generated by the Brain from a cluster of Experiences — not by API callers
- Pattern statistics are recomputed when new experiences are added to the cluster
- An ImprovementProposal belongs to exactly one Pattern

---

## 6.6 Provider (Execution Context)

**Root:** `provider.providers`  
**Owns:** `provider.models`, `provider.health_history`, `provider.circuit_events`  
**Invariants:**
- A Provider may have multiple Models, but a Model belongs to exactly one Provider
- Model pricing is versioned by `effective_from` / `effective_until` — cost records must reference the rate in effect at execution time
- Circuit state transitions follow the 3-state FSM: CLOSED → OPEN → HALF_OPEN → CLOSED/OPEN

---

## 6.7 CostRecord (Commercial Context)

**Root:** `billing.cost_records`  
**Owns:** Nothing (CostRecord is a terminal entity)  
**Invariants:**
- CostRecord is immutable once written. No UPDATE. No DELETE.
- `cost_record_id` is deterministic: `SHA-256(execution_id + billing_date)`. Duplicate inserts are silently rejected (ON CONFLICT DO NOTHING).
- Billing corrections are made by writing a reversal record (negative cost_usd) with a reference to the original record.

---

## 6.8 Invoice (Commercial Context)

**Root:** `billing.invoices`  
**Owns:** `billing.invoice_line_items`  
**Invariants:**
- An Invoice may not be modified after `status = FINALIZED`
- Line items are derived from `billing.cost_records` — they are projections, not source of truth
- One Invoice per billing_period (UK enforced)

---

## 6.9 Plugin (Operations Context)

**Root:** `plugins.plugins`  
**Owns:** `plugins.health_checks`, `plugins.invocations`  
**Invariants:**
- A Plugin in UNINSTALLED state has its invocation history retained for the retention period
- `invocation_count` is an approximate counter (async increment) — not used for billing
- A Plugin's `granted_permissions` are a subset of (or equal to) its `declared_permissions`

---

---

# Section 7: Read Models

Read models are projections optimized for query patterns. They are derived from write models via event projection or scheduled aggregation. Read models may be slightly stale.

---

## 7.1 WorkflowSummaryView

**Source:** `workflow.workflows` JOIN `workflow.tasks`  
**Consumer:** `GET /api/v1/workflows` list endpoint  
**Staleness:** Real-time (view over live tables, not a materialized cache)

| Field | Derived From |
|-------|-------------|
| `workflow_id` | `workflows.workflow_id` |
| `product_id` | `workflows.product_id` |
| `plan_name` | `workflows.plan_name` |
| `status` | `workflows.status` |
| `tasks_completed` | COUNT(tasks WHERE status=COMPLETED) |
| `tasks_failed` | COUNT(tasks WHERE status=FAILED) |
| `task_count` | `workflows.task_count` |
| `submitted_at` | `workflows.submitted_at` |
| `completed_at` | `workflows.completed_at` |
| `total_cost_usd` | `workflows.total_cost_usd` |

**Index requirement:** `(product_id, status, submitted_at DESC)` composite index on `workflow.workflows` to support the common query pattern: "give me ACTIVE workflows for product X, newest first."

---

## 7.2 BudgetStatusView

**Source:** `billing.budget_configs` + `billing.budget_snapshots`  
**Consumer:** `GET /api/v1/monitoring/usage`, AI ROS budget check  
**Staleness:** Budget snapshots are updated on every cost record write — < 1s stale

| Field | Derived From |
|-------|-------------|
| `product_id` | `budget_configs.product_id` |
| `billing_date` | today (UTC) |
| `daily_limit_usd` | latest `budget_configs.daily_limit_usd` for today |
| `daily_spend_usd` | `budget_snapshots.daily_spend_usd` for today |
| `spend_pct` | `daily_spend_usd / daily_limit_usd * 100` |
| `reset_at` | midnight UTC tomorrow |

---

## 7.3 ProviderCatalogView

**Source:** `provider.providers` JOIN `provider.models` (WHERE active=true)  
**Consumer:** `GET /api/v1/providers`, AI ROS routing  
**Staleness:** Real-time (view)

Fields: all provider fields + active models with current pricing.

---

## 7.4 WorkspaceHealthView

**Source:** `workspace.products` + `workspace.platform_modules`  
**Consumer:** `GET /api/v1/workspace`, Desktop  
**Staleness:** 30s (in-process LRU cache; read-through on cache miss)

| Field | Derived From |
|-------|-------------|
| `overall_health` | MIN(product.health_status, module.health_status) across all |
| `products` | `workspace.products` WHERE health_status != OFFLINE |
| `module_health` | map of module_id → health_status |

---

## 7.5 PromptActiveVersionView

**Source:** `prompt.templates` JOIN `prompt.versions` WHERE `version = templates.active_version`  
**Consumer:** `POST /api/v1/prompts/render` (after cache miss)  
**Staleness:** In-process cache; 60s TTL; invalidated on ACTIVE/SUSPENDED transition

| Field | Derived From |
|-------|-------------|
| `template_id` | `templates.template_id` |
| `version` | `templates.active_version` |
| `content` | `versions.content` |
| `system_prompt` | `versions.system_prompt` |
| `variables` | `versions.variables` |
| `hmac_signature` | `versions.hmac_signature` |
| `state` | `versions.state` |

---

## 7.6 BrainRecommendationView

**Source:** `brain.recommendation_cache`  
**Consumer:** `POST /api/v1/brain/recommend`  
**Staleness:** Up to TTL of recommendation cache entry (configurable, default 1 hour)  
**Cache miss behavior:** Recompute from `brain.experiences` via Jaccard similarity, write to cache

---

## 7.7 DailyCostRollupView

**Source:** `billing.cost_records` grouped by `billing_date, product_id, provider_id, model_id`  
**Consumer:** Invoice generation, usage endpoint  
**Staleness:** Real-time aggregation (GROUP BY on live table — indexed on billing_date)

---

## 7.8 PatternSummaryView

**Source:** `brain.patterns` JOIN `brain.pattern_experiences` JOIN `brain.experiences`  
**Consumer:** `GET /api/v1/brain/patterns`  
**Staleness:** Patterns are recomputed by the Brain background worker; up to 15 minutes stale

---

---

# Section 8: Write Models

Write models define the canonical shape of data at the point of write. Each write model has a single owner, clear invariants, and passes through its aggregate root.

---

## 8.1 WorkflowSubmit Write Model

**Triggered by:** `POST /api/v1/workflows`  
**Writes to:** `workflow.workflows`, `workflow.tasks`, `workflow.gates`  
**Transaction scope:** All three tables in a single database transaction.

**Write sequence:**
1. Validate plan (DAG check, task type check, gate validity)
2. Resolve idempotency key — if matched, return existing workflow (no write)
3. INSERT `workflow.workflows` (status=QUEUED)
4. INSERT all `workflow.tasks` (status=PENDING)
5. INSERT all `workflow.gates` (status=PENDING)
6. COMMIT
7. Publish `workflows.workflow.created` event (outside transaction)

**Idempotency enforcement:** `(product_id, idempotency_key)` unique index with `WHERE idempotency_key IS NOT NULL`. Handles the race condition where two concurrent submissions with the same key arrive simultaneously — only one INSERT succeeds; the other detects the conflict and reads the existing row.

---

## 8.2 TaskComplete Write Model

**Triggered by:** WorkflowRuntime completing a task  
**Writes to:** `workflow.tasks`, `workflow.task_outputs`, `workflow.checkpoints`, `workflow.workflows`  
**Transaction scope:** All four in a single transaction.

**Write sequence:**
1. UPDATE `workflow.tasks` SET status=COMPLETED, completed_at=now WHERE task_id=X
2. INSERT `workflow.task_outputs` for the completed task
3. UPSERT `workflow.checkpoints` (single row per workflow — replace on conflict)
4. Check if all tasks are COMPLETED → if yes, UPDATE `workflow.workflows` status=COMPLETED
5. COMMIT
6. Publish `workflows.task.completed` event

---

## 8.3 CostRecord Write Model

**Triggered by:** AI execution completion event (`ai.execution.completed`)  
**Writes to:** `billing.cost_records`, `billing.budget_snapshots`  
**Transaction scope:** Both in a single transaction.

**Idempotency enforcement:** `cost_record_id` = `SHA-256(execution_id + billing_date)`. INSERT with ON CONFLICT DO NOTHING. The event consumer checks `affected_rows` — if 0, the record already exists and is silently skipped.

**Budget snapshot update:** UPSERT `budget_snapshots` — increment `daily_spend_usd` by the new cost. This is a read-modify-write operation; the transaction prevents double-counting.

---

## 8.4 PromptVersion Write Model

**Triggered by:** `POST /api/v1/prompts/templates` and `PUT /api/v1/prompts/templates/{id}/versions`  
**Writes to:** `prompt.templates`, `prompt.versions`

**On new template:**
1. INSERT `prompt.templates` (active_version=null)
2. INSERT `prompt.versions` (version=1, state=DRAFT)
3. COMMIT

**On new version:**
1. SELECT MAX(version) FROM `prompt.versions` WHERE template_id=X FOR UPDATE
2. INSERT `prompt.versions` (version=MAX+1, state=DRAFT)
3. COMMIT

The SELECT FOR UPDATE prevents two concurrent version updates from getting the same version number.

---

## 8.5 GovernanceTransition Write Model

**Triggered by:** `POST /api/v1/prompts/templates/{id}/transition`  
**Writes to:** `prompt.versions`, `prompt.templates`, `prompt.governance_log`

**On REVIEW→ACTIVE transition:**
1. BEGIN TRANSACTION
2. UPDATE `prompt.versions` SET state=ACTIVE WHERE (template_id, version)
3. UPDATE previous ACTIVE version (if any): SET state=DEPRECATED (previous version auto-deprecated)
4. UPDATE `prompt.templates` SET active_version=new_version
5. INSERT `prompt.governance_log` entry
6. COMMIT
7. Invalidate prompt template cache entry

**On ACTIVE→SUSPENDED transition:**
1. UPDATE `prompt.versions` SET state=SUSPENDED
2. UPDATE `prompt.templates` SET active_version=null
3. INSERT `prompt.governance_log` entry
4. COMMIT
5. Invalidate prompt template cache entry
6. Publish `prompts.template.suspended` event (CRITICAL priority)

---

## 8.6 Experience Write Model

**Triggered by:** `POST /api/v1/brain/experiences`  
**Writes to:** `brain.experiences`  
**Write is async:** The HTTP response is returned before database write completes. The write is queued to a background worker that batches experience inserts.

**Why async:** Experience recording is on the hot path (after every task completion). Making the caller wait for a DB write is unnecessary latency. Loss of a single experience record is acceptable; the Brain is a statistical system.

**Pattern update trigger:** The Brain's pattern worker checks for new experiences every 15 minutes and updates `brain.patterns` and `brain.pattern_experiences` accordingly.

---

## 8.7 KeyRevocation Write Model

**Triggered by:** `DELETE /api/v1/auth/keys/{key_id}`  
**Writes to:** `security.api_keys`, `security.audit_log`

**Write sequence:**
1. SELECT api_keys WHERE key_id=X FOR UPDATE
2. Validate: not the last ADMIN key
3. UPDATE `security.api_keys` SET status=REVOKED, revoked_at=now, revocation_reason=?
4. INSERT `security.audit_log` entry
5. COMMIT
6. Invalidate in-process key cache (immediate)
7. Publish `security.api_key.revoked` event (CRITICAL priority — consumers must receive this before the next request arrives)

**Cache invalidation:** The in-process auth cache has a 5-second TTL. After revocation, the key remains cacheable for up to 5 seconds. Acceptable trade-off for single-instance deployment. For multi-instance: the `security.api_key.revoked` event triggers cache invalidation in all instances via subscription.

---

---

# Section 9: Audit Tables

The audit log is the platform's immutable record of all security-significant events. It is the only table in the platform where data may never be deleted, even after retention period — it is archived, not purged.

---

## 9.1 security.audit_log — Design

**Append-only enforcement:** PostgreSQL row-level security (RLS) policy restricts all writes to INSERT. UPDATE and DELETE are blocked at the database level, not application level.

```
RLS Policy Name: audit_log_insert_only
Command:         ALL
Role:            audit_writer (the only role that can write)
USING:           false    ← no existing row is accessible for UPDATE/DELETE
WITH CHECK:      true     ← all INSERTs are permitted
```

**Write path:** No module writes to `security.audit_log` directly. All audit writes go through the Audit Log Service interface (`platform/security/audit-log-service`). The service validates the entry shape and performs the INSERT.

**Who writes to the audit log:**

| Event Type | Written By |
|-----------|-----------|
| `api_key.created` | Security Service |
| `api_key.revoked` | Security Service |
| `auth.failed` | Security Service |
| `auth.succeeded` (selective) | Security Service |
| `permission.denied` | Security Service |
| `rate_limit.exceeded` | Security Service |
| `prompt_injection.detected` | Prompt OS |
| `tool_sandbox.violation` | Plugin Manager |
| `config.changed` | Config Manager |
| `workflow.gate.approved` | Workflow Runtime |
| `workflow.gate.rejected` | Workflow Runtime |
| `workflow.cancelled` | Workflow Runtime |
| `prompt.template.suspended` | Prompt OS |
| `plugin.installed` | Plugin Manager |
| `plugin.uninstalled` | Plugin Manager |
| `plugin.disabled` | Plugin Manager |
| `billing.limit_exceeded` | Billing Service |
| `system.platform.started` | Startup Manager |
| `system.migration.completed` | Migration Runner |
| `feature_flag.changed` | Config Manager |
| `audit_log.accessed` | Security Service (self-audit) |

---

## 9.2 Audit Entry Schema

| Field | Description |
|-------|-------------|
| `entry_id` | UUID primary key. Never sequential — prevents enumeration. |
| `event_type` | Dot-notation event name (matches EVENT-CATALOG naming). |
| `severity` | `LOW` \| `MEDIUM` \| `HIGH` \| `CRITICAL` |
| `timestamp` | UTC. Microsecond precision. |
| `identity_hash` | SHA-256 of the authenticated identity. Never the raw key. |
| `ip_hash` | SHA-256 of the client IP. Never the raw IP. |
| `path` | Request path. No query string (may contain filter values). |
| `method` | HTTP method. |
| `resource_id` | The resource affected (workflow_id, key_id, template_id, etc.). |
| `correlation_id` | From the request's X-Request-Id header. |
| `details_json` | Event-specific structured details. Never contains secrets, raw IPs, or key values. |
| `product_id` | Populated when the event is product-scoped. |

**Partition:** By month on `timestamp`. Each month is its own partition. Partitions older than the retention window are compressed to cold storage (not deleted — see Section 11).

---

## 9.3 Audit Log Access

**Who may query:** ADMIN role only, via `GET /api/v1/admin/audit-log` or `GET /api/v1/auth/audit-log`.

**Access is self-audited:** Every query to the audit log generates its own audit entry with `event_type = "audit_log.accessed"`. This creates a chain of custody: every read of audit records is itself audited.

**Export:** Audit log may be exported by ADMIN for compliance. Export operations generate an audit entry. Export format: JSONL (one entry per line). No binary formats — audit data must be human-readable.

---

---

# Section 10: Usage Tables

Usage tables store time-series aggregations for analytics and capacity planning. They are derived from raw events — not from querying operational tables directly.

---

## 10.1 usage.hourly_token_summaries

**Written by:** Usage Aggregator (scheduled: every hour, 5 minutes after the hour)  
**Source:** Consumes `ai.execution.completed` events from NATS, aggregated by hour  
**Consumer:** Usage analytics API, Desktop cost panel

| Field | Type | Notes |
|-------|------|-------|
| `summary_id` | UUID PK | |
| `hour_utc` | timestamp (UK) | Partitioned by month |
| `total_prompt_tokens` | bigint | |
| `total_completion_tokens` | bigint | |
| `total_cache_tokens` | bigint | |
| `total_tokens` | bigint | |
| `total_cost_usd` | numeric(12,6) | |
| `total_executions` | integer | |
| `failed_executions` | integer | |
| `p50_latency_ms` | integer | Approximate percentile |
| `p99_latency_ms` | integer | |
| `error_rate` | numeric(5,4) | |
| `executions_by_provider` | jsonb | `{"anthropic": 150, ...}` |
| `executions_by_model` | jsonb | |
| `executions_by_product` | jsonb | |

**Partition:** By month on `hour_utc`. Retention: 30 days (see Section 11).

---

## 10.2 usage.daily_request_summaries

**Written by:** Usage Aggregator (scheduled: daily, 00:05 UTC)  
**Source:** Aggregation of hourly summaries + auth event counts from NATS

| Field | Type | Notes |
|-------|------|-------|
| `summary_id` | UUID PK | |
| `date_utc` | date (UK) | |
| `total_requests` | integer | All HTTP requests |
| `total_authenticated` | integer | |
| `total_rejected` | integer | Auth failures + rate limits |
| `total_ai_executions` | integer | |
| `total_workflow_submissions` | integer | |
| `total_unique_identities` | integer | Distinct hashed identities |
| `peak_rpm` | integer | |
| `peak_rpm_at` | timestamp | |
| `sla_met_rate` | numeric(5,4) | Platform-wide SLA compliance |

**Retention:** 30 days.

---

## 10.3 usage.daily_workflow_summaries

**Written by:** Usage Aggregator (scheduled: daily, 00:10 UTC)  
**Source:** `workflow.workflows` aggregate query for the prior day

| Field | Type | Notes |
|-------|------|-------|
| `summary_id` | UUID PK | |
| `date_utc` | date (UK) | |
| `total_submitted` | integer | |
| `total_completed` | integer | |
| `total_failed` | integer | |
| `total_cancelled` | integer | |
| `completion_rate` | numeric(5,4) | |
| `sla_met_rate` | numeric(5,4) | |
| `avg_completion_minutes` | numeric(8,2) | |
| `avg_task_failure_rate` | numeric(5,4) | |
| `gates_approved_count` | integer | |
| `gates_rejected_count` | integer | |

**Retention:** 30 days.

---

## 10.4 usage.weekly_brain_summaries

**Written by:** Usage Aggregator (scheduled: weekly, Monday 00:15 UTC)  
**Source:** `brain.experiences`, `brain.patterns` aggregate queries for the prior week

| Field | Type | Notes |
|-------|------|-------|
| `summary_id` | UUID PK | |
| `week_start_utc` | date (UK) | Always a Monday |
| `experiences_recorded` | integer | |
| `patterns_discovered` | integer | |
| `patterns_updated` | integer | |
| `recommendations_made` | integer | |
| `recommendation_acceptance_rate` | numeric(5,4) | |
| `knowledge_nodes_added` | integer | |
| `brain_query_count` | integer | |
| `avg_query_latency_ms` | integer | |

**Retention:** 30 days.

---

---

# Section 11: Storage Designs by Domain

---

## 11.1 Workflow Storage

**Schema:** `workflow`  
**Volume:** High (potentially thousands of workflows per day with complex task trees)  
**Access patterns:**
- Hot path (read): Get workflow status by ID — O(1) by PK
- Hot path (write): Task status updates — O(1) by (workflow_id, task_id)
- List path: Workflows by product + status — covered by composite index
- Analytics: Count/aggregate by status, product, date — covered by usage schema

**Key design decisions:**

`dag_edges` is stored as `depends_on` per-task (array of task_ids the task depends on), not as a separate edges table. This reduces join complexity for the common case (get a task and its dependencies in one read).

`task_outputs` is a separate table from `tasks` because output JSON can be large (up to 500KB per task). Keeping it separate prevents row bloat on the `tasks` table, which is accessed on every status poll.

`checkpoints` uses UPSERT (ON CONFLICT ON CONSTRAINT workflow_id DO UPDATE) — one checkpoint row per workflow, overwritten on each task completion. This avoids checkpoint table bloat.

**Workflow archive policy:** Workflows in terminal state (COMPLETED, FAILED, CANCELLED) older than 30 days are moved to `workflow.workflows_archive`. The archive table has the same schema but no indexes (append-only cold store). Task outputs older than 7 days are eligible for compression. Task outputs older than 30 days are deleted (the workflow summary in `workflow.workflows_archive` is retained).

---

## 11.2 Prompt Storage

**Schema:** `prompt`  
**Volume:** Low (templates change rarely; render volume is handled by events, not DB writes)  
**Access patterns:**
- Ultra-hot path (read): Render — reads `PromptActiveVersionView` (in-process cache, 60s TTL)
- Admin path (read): List templates, get diff — uncached
- Write path: New version or governance transition — infrequent

**Key design decision:** Template `content` is stored as plain text (not JSON). The HMAC is computed over the raw text content, not a JSON encoding of it. This avoids encoding normalization bugs where identical content produces different JSON representations.

**Version retention:** All versions of all templates are retained indefinitely. Prompt governance history is a compliance asset — it proves what prompt was used for any given AI execution.

**Render log:** The `prompt.render_log` table (not shown in ER diagrams) stores a lightweight record of each render: render_id, template_id, version, caller identity hash, timestamp, token estimate. No rendered content is stored. Retention: 7 days. This table is write-heavy and is truncated via partition drop, not DELETE.

---

## 11.3 Knowledge Storage

**Schema:** `knowledge`  
**Phase:** Currently SQL (PostgreSQL). Migration to Kuzu in v1.1.

**Current SQL design:**

`knowledge.nodes` stores nodes with a JSONB `properties` column. The JSONB column has a GIN index enabling `@>` containment queries on property key-value pairs. The `label` field has a `tsvector` full-text search index.

`knowledge.relationships` stores directional edges. Both `from_node_id` and `to_node_id` are indexed independently (not a composite) to enable both "outbound edges from node X" and "inbound edges to node X" queries efficiently.

`knowledge.memory` uses a scoped key-value model. The unique constraint `(memory_key, scope, scope_id)` ensures one value per key per scope context. A background worker runs every 60 seconds to DELETE rows WHERE `expires_at < now()` (TTL enforcement).

**v1.1 Migration to Kuzu:**
- `knowledge.nodes` → Kuzu node table (typed by `node_type`)
- `knowledge.relationships` → Kuzu relationship table (typed by `relationship_type`)
- `knowledge.memory` stays in PostgreSQL (memory is not graph data)
- Migration is a dual-write period: writes go to both stores; reads switch to Kuzu once parity is verified

---

## 11.4 Secrets Storage

**What is stored:** Only cryptographic hashes and metadata. Never raw secrets.

| Secret Type | What's Stored in DB | Where Raw Value Lives |
|-------------|--------------------|-----------------------|
| API key value | SHA-256 hash (`key_hash`) | Shown once to user at creation; user's responsibility |
| Client IP | SHA-256 hash (`ip_hash`) | Discarded after hashing |
| Provider API keys | Hash of value only | Environment variables / vault |
| HMAC signing key | Never stored | Environment variable `AISF_PROMPT_HMAC_KEY` |
| DB password | Never in DB | Environment variable / vault |
| Plugin bundle URL | SHA-256 hash of URL | Hashed immediately on receipt |

**Hashing contract:**
- Algorithm: SHA-256
- Salt: None (salting would prevent deduplication which is needed for key lookup)
- Purpose: Identification, not password hashing. API key lookup is: `SELECT * FROM api_keys WHERE key_hash = SHA256(incoming_key)`
- The SHA-256 pre-image resistance property is sufficient for this purpose: knowing the hash does not reveal the key value

**What is encrypted at rest:** PostgreSQL storage encryption is handled at the infrastructure layer (disk encryption). No column-level encryption is used in the current schema. CONFIDENTIAL-classified fields (billing amounts, identity hashes) are protected by:
1. Disk encryption at rest
2. Network TLS in transit
3. Schema-level ownership (no cross-schema access)
4. PostgreSQL RLS policies on `security.audit_log`

---

## 11.5 Provider Storage

**Schema:** `provider`  
**Volume:** Low (handful of providers; model catalog changes infrequently)  
**Key design:** Provider model pricing is versioned with `effective_from` / `effective_until`. When a provider changes pricing:
1. Set `effective_until = change_date` on the current model row
2. INSERT a new row with `effective_from = change_date` and the new prices
3. `billing.cost_records` records the actual cost at execution time — so historical records are always correct regardless of future price changes

**Circuit breaker state:** `provider.providers.circuit_state` is updated in place. The historical state transitions are in `provider.circuit_events`. This separation keeps the hot path (routing query: "which providers are CLOSED?") to a single-column read on a tiny table.

---

## 11.6 Plugin Storage

**Schema:** `plugins`  
**Volume:** Low (few plugins installed at any time)  
**Key design:** Plugin `configuration` is stored as JSONB but never contains secrets. Secret values needed by plugins are injected at runtime via environment variable injection into the sandbox — not read from the database.

`plugins.invocations` is a time-series table partitioned by day. It is write-heavy (every plugin invocation writes a record) and read-rarely (only for debugging). Retention is 7 days, enforced by partition drop. The `plugins.plugins.invocation_count` counter is incremented asynchronously after the fact — it is an approximate count, not used for billing.

---

## 11.7 Marketplace Storage (PROPOSED)

**Schema:** `marketplace`  
**Status:** Tables exist but are empty. Feature flag `marketplace=false` prevents any writes.  
**Activation:** v2.5 (multi-tenant release)

`marketplace.listings` stores listing metadata. The actual product template bundle is not stored in the database — it is stored in object storage (S3-compatible) and the `marketplace.listings` table stores a reference URL and content hash.

`marketplace.installations` tracks which workspace installed which listing, and when. Uninstallation sets `uninstalled_at` but does not delete the row (audit trail).

`marketplace.reviews` stores review metadata but not review text. Review text is stored in a separate full-text search index (not designed in this version).

---


---

---

# Section 12: Migration Strategy

---

## 12.1 Principles

**P-MIG-1: Migrations are atomic.** Each migration is a single transaction. If any step fails, the entire migration rolls back. No partial migrations.

**P-MIG-2: Migrations are single-purpose.** One migration does one thing. A migration that adds a column, creates an index, and backfills data is three migrations.

**P-MIG-3: Every migration has a down-migration.** The down-migration must be verified to work in CI before the up-migration is approved. A down-migration that truncates data is not acceptable — it must reverse the schema change without data loss.

**P-MIG-4: No blocking DDL on large tables.** `ADD COLUMN`, `CREATE INDEX`, and `ALTER TABLE` acquire locks. For tables with > 100K rows, use non-blocking alternatives:
- `ADD COLUMN` with a DEFAULT: Use `ADD COLUMN x type DEFAULT val` (PostgreSQL 11+ adds metadata-only for NOT NULL with default)
- Index creation: Use `CREATE INDEX CONCURRENTLY` (does not lock reads or writes)
- Constraint addition: Add as NOT VALID first, then VALIDATE CONSTRAINT separately

**P-MIG-5: Migrations deploy before code.** The migration runs first; new code deploys second. The migration must be backward-compatible with the old code. Old code reading a table with a new nullable column works fine. Old code writing a row without the new column — the column must have a safe default.

**P-MIG-6: Backfill is a separate migration.** Adding a column and populating it are two separate migrations. The backfill migration updates in batches (never a single UPDATE of millions of rows).

---

## 12.2 Migration File Naming Convention

```
{sequence:04d}_{schema}_{description}.sql
```

Examples:
```
0001_security_create_api_keys.sql
0002_security_create_audit_log.sql
0003_security_create_rate_limit_buckets.sql
0004_workflow_create_workflows.sql
0005_workflow_create_tasks.sql
0006_workflow_create_task_outputs.sql
0007_prompt_create_templates.sql
...
```

Sequence is global across all schemas — schema changes interleave in chronological order, not grouped by schema.

---

## 12.3 Migration Ledger

Every executed migration is recorded in `platform.schema_migrations` — a dedicated table in a `platform` meta-schema.

| Field | Description |
|-------|-------------|
| `migration_id` | The filename without `.sql` extension |
| `direction` | `UP` or `DOWN` |
| `applied_at` | UTC timestamp |
| `duration_ms` | How long the migration took |
| `schema_version_before` | Schema version before this migration |
| `schema_version_after` | Schema version after this migration |
| `checksum` | SHA-256 of the migration file content |
| `environment` | `local` / `staging` / `production` |

The `checksum` is used to detect if a migration file was modified after deployment. A modified migration file is a platform violation — immutable once deployed.

---

## 12.4 Large-Table Migration Playbook

For any table exceeding 1 million rows:

**Adding a column:**
1. Migration N: `ALTER TABLE ... ADD COLUMN x type` (no default — adds instantly as nullable)
2. Migration N+1: Backfill in batches: `UPDATE ... WHERE id BETWEEN $start AND $end AND x IS NULL`
3. Migration N+2: `ALTER TABLE ... ALTER COLUMN x SET NOT NULL` (after all rows backfilled)
4. Migration N+3: `ALTER TABLE ... ALTER COLUMN x SET DEFAULT 'value'` (if a default is needed for future inserts)

**Adding an index:**
1. Migration N: `CREATE INDEX CONCURRENTLY idx_name ON table(column)` — non-blocking, may take minutes
2. Note: `CREATE INDEX CONCURRENTLY` cannot run in a transaction. The migration runner must detect this and execute it outside the transaction block.

**Dropping a column:**
1. Code must stop reading the column first (deploy first, migrate after)
2. Migration N: `ALTER TABLE ... ALTER COLUMN x DROP NOT NULL` (make nullable — backward safe)
3. Migration N+1 (next deploy cycle): `ALTER TABLE ... DROP COLUMN x`

---

## 12.5 Knowledge Graph Migration (SQL → Kuzu)

The migration from PostgreSQL knowledge tables to Kuzu is the most complex schema migration in the platform roadmap. It is a live migration across two storage backends.

**Phase 1 — Dual Write (v1.0.5):**
- All writes go to both PostgreSQL (`knowledge.nodes`, `knowledge.relationships`) and Kuzu
- All reads come from PostgreSQL
- Duration: 2 weeks minimum

**Phase 2 — Read Switch (v1.1.0):**
- All writes continue to both stores
- All reads switch to Kuzu
- Parity validation: random sample of 1000 nodes/relationships compared between stores daily
- Duration: 2 weeks minimum

**Phase 3 — PostgreSQL Decommission (v1.1.2):**
- Stop writing to PostgreSQL `knowledge.*` tables
- Archive PostgreSQL tables (rename to `knowledge_legacy.*`)
- Read from Kuzu only

**Rollback:** At any phase, revert by switching reads back to PostgreSQL (which was dual-written throughout). The dual-write window is the rollback window.

---

---

# Section 13: Retention Policy

---

## 13.1 Retention by Table

| Schema | Table | Retention | Enforcement | After Retention |
|--------|-------|-----------|-------------|----------------|
| `security` | `api_keys` | Indefinite (until status=REVOKED, then 365d) | Scheduled cleanup | DELETE |
| `security` | `audit_log` | **Never deleted** | — | Archive to cold storage |
| `security` | `rate_limit_buckets` | 2 days | Scheduled cleanup | DELETE |
| `workspace` | `products` | Indefinite (active) / 90d (deregistered) | Scheduled cleanup | DELETE |
| `workspace` | `product_health_history` | 30 days | Partition drop | DROP PARTITION |
| `workspace` | `platform_modules` | Indefinite | — | — |
| `workflow` | `workflows` | 90 days (terminal) | Archival job | Move to archive |
| `workflow` | `tasks` | 90 days | Cascade from workflows | Move to archive |
| `workflow` | `task_outputs` | 30 days | Scheduled cleanup | DELETE |
| `workflow` | `checkpoints` | 7 days after terminal state | Scheduled cleanup | DELETE |
| `workflow` | `gates` | 90 days | Cascade from workflows | Move to archive |
| `prompt` | `templates` | Indefinite | — | — |
| `prompt` | `versions` | Indefinite (governance asset) | — | — |
| `prompt` | `governance_log` | Indefinite (compliance) | — | — |
| `prompt` | `ab_tests` | 365 days | Scheduled cleanup | DELETE |
| `prompt` | `ab_test_results` | 30 days | Partition drop | DROP PARTITION |
| `brain` | `experiences` | 365 days | Partition drop | DROP PARTITION |
| `brain` | `patterns` | Indefinite | — | — |
| `brain` | `pattern_experiences` | Cascade from experiences | — | — |
| `brain` | `improvement_proposals` | 365 days | Scheduled cleanup | DELETE |
| `brain` | `recommendation_cache` | TTL per entry (default 1h) | Background job | DELETE WHERE expires_at < now |
| `knowledge` | `nodes` | Indefinite | — | — |
| `knowledge` | `relationships` | Indefinite | — | — |
| `knowledge` | `memory` | Per `ttl_seconds` field | Background job | DELETE WHERE expires_at < now |
| `provider` | `providers` | Indefinite | — | — |
| `provider` | `models` | Indefinite (historical pricing) | — | — |
| `provider` | `health_history` | 30 days | Partition drop | DROP PARTITION |
| `provider` | `circuit_events` | 90 days | Scheduled cleanup | DELETE |
| `billing` | `cost_records` | **365 days minimum** | — | Archive to cold storage |
| `billing` | `budget_configs` | Indefinite (historical rates) | — | — |
| `billing` | `budget_snapshots` | 90 days | Scheduled cleanup | DELETE |
| `billing` | `invoices` | **Indefinite (financial records)** | — | Archive to cold storage |
| `billing` | `invoice_line_items` | **Indefinite** | — | Archive to cold storage |
| `usage` | `hourly_token_summaries` | 30 days | Partition drop | DROP PARTITION |
| `usage` | `daily_request_summaries` | 30 days | Scheduled cleanup | DELETE |
| `usage` | `daily_workflow_summaries` | 30 days | Scheduled cleanup | DELETE |
| `usage` | `weekly_brain_summaries` | 30 days | Scheduled cleanup | DELETE |
| `plugins` | `plugins` | Indefinite | — | — |
| `plugins` | `health_checks` | 7 days | Partition drop | DROP PARTITION |
| `plugins` | `invocations` | 7 days | Partition drop | DROP PARTITION |
| `monitoring` | `alerts` | 90 days | Scheduled cleanup | DELETE |
| `monitoring` | `sla_measurements` | 90 days | Partition drop | DROP PARTITION |
| `monitoring` | `health_check_results` | 7 days | Partition drop | DROP PARTITION |

---

## 13.2 Retention Enforcement Mechanisms

**Partition Drop:** Tables with `(PART)` on a date/timestamp column are partitioned by day, month, or week. Dropping a partition is instantaneous and non-locking — it does not perform row-by-row deletion.

**Scheduled Cleanup:** A platform background job (`platform/maintenance/data-cleaner`) runs nightly at 02:00 UTC and executes bounded DELETE queries. "Bounded" means: DELETE with a LIMIT (e.g., 10,000 rows per pass) and multiple passes if needed. This prevents large DELETE operations from locking the table.

**Archival:** Data that must be retained for compliance but is no longer needed for operational queries is moved to cold storage (JSONL files in object storage). The archival job runs weekly. Archived data is immutable and not queryable by the platform — it exists for compliance audit only.

---

## 13.3 Immutable Tables

The following tables may never have rows deleted, even after the retention period:

| Table | Reason | Archive Mechanism |
|-------|--------|-------------------|
| `security.audit_log` | Compliance requirement; chain of custody | Monthly JSONL export to object storage; PostgreSQL partitions retained indefinitely |
| `billing.cost_records` | Financial record; accounting compliance | After 365 days: move to archive partition (remains queryable for invoice reconciliation) |
| `billing.invoices` | Financial record | Indefinite retention in primary database |
| `billing.invoice_line_items` | Financial record | Indefinite retention in primary database |
| `prompt.governance_log` | Prompt governance chain of custody | Indefinite retention |

---

---

# Section 14: Backup Policy

---

## 14.1 Backup Types

| Type | Frequency | Method | Retention |
|------|-----------|--------|-----------|
| **WAL Archiving** | Continuous | PostgreSQL WAL → object storage | 30 days |
| **Base Backup** | Daily | `pg_basebackup` | 7 days |
| **Schema-Only Backup** | On every migration | `pg_dump --schema-only` | 90 days |
| **Audit Log Export** | Monthly | JSONL export → object storage | Indefinite |
| **Billing Records Export** | Monthly | JSONL export → object storage | Indefinite |
| **Knowledge Graph Backup** | Daily | Kuzu checkpoint copy | 7 days |

---

## 14.2 Recovery Time Objectives

| Scenario | RTO | RPO | Recovery Method |
|----------|-----|-----|----------------|
| Single table corruption | 30 min | Last WAL flush (~1s) | Point-in-time recovery via WAL |
| Database server failure | 2 hours | Last WAL flush (~1s) | Restore base backup + replay WAL |
| Accidental bulk delete | 15 min | Time of delete | Point-in-time recovery via WAL |
| Full data centre loss | 4 hours | Last backup + WAL from replica | Restore from off-site backup |

---

## 14.3 Backup Validation

**Weekly:** A restore drill is performed in an isolated environment. The drill verifies:
1. Base backup can be restored
2. WAL can be replayed to a specific point in time
3. Schema matches expected structure
4. Row counts are within expected ranges for key tables

**Monthly:** A full production restore is tested in staging. The staging environment is rebuilt from the production backup + WAL. Application smoke tests are run against the restored database.

---

---

# Section 15: Index Strategy

---

## 15.1 Index Design Principles

**P-IDX-1: Index every foreign key.** Foreign keys without indexes cause sequential scans on the referencing table during JOIN and CASCADE operations.

**P-IDX-2: Index every column used in WHERE clauses on large tables.** "Large" means > 10,000 rows at expected peak volume.

**P-IDX-3: Composite indexes satisfy the most common multi-column query.** The leftmost column(s) of a composite index must match the query's equality filters.

**P-IDX-4: Partial indexes for status-filtered queries.** If 90% of queries filter by `status = 'ACTIVE'`, a partial index `WHERE status = 'ACTIVE'` is smaller and faster than a full index.

**P-IDX-5: GIN for JSONB.** Any JSONB column queried with `@>` (containment) or `->` (key access) needs a GIN index.

**P-IDX-6: tsvector for full-text search.** Store a `tsvector` computed column for any text column that is full-text searched.

**P-IDX-7: Don't over-index write-heavy tables.** Each index on a write-heavy table adds write overhead. `billing.cost_records` and `brain.experiences` are insert-heavy — only add indexes that serve specific query patterns.

---

## 15.2 Critical Indexes

### security.api_keys
| Index | Columns | Type | Why |
|-------|---------|------|-----|
| `idx_api_keys_hash` | `key_hash` | B-tree (UK) | Auth lookup on every request |
| `idx_api_keys_status` | `status` | Partial B-tree WHERE status='ACTIVE' | List active keys |
| `idx_api_keys_product` | `product_id` | B-tree | List keys for a product |

### workflow.workflows
| Index | Columns | Type | Why |
|-------|---------|------|-----|
| `idx_workflows_product_status` | `(product_id, status)` | Composite B-tree | Most common list query |
| `idx_workflows_status_submitted` | `(status, submitted_at DESC)` | Composite B-tree | List by status, newest first |
| `idx_workflows_idempotency` | `(product_id, idempotency_key)` | Partial UK WHERE `idempotency_key IS NOT NULL` | Idempotency check |
| `idx_workflows_completed_at` | `completed_at` | B-tree WHERE `completed_at IS NOT NULL` | Retention query |

### workflow.tasks
| Index | Columns | Type | Why |
|-------|---------|------|-----|
| `idx_tasks_workflow_status` | `(workflow_id, status)` | Composite B-tree | Get all tasks for a workflow |
| `idx_tasks_status_active` | `status` | Partial B-tree WHERE status IN ('PENDING','READY','IN_PROGRESS') | Find active tasks |

### billing.cost_records
| Index | Columns | Type | Why |
|-------|---------|------|-----|
| `idx_cost_records_execution` | `execution_id` | UK | Idempotency deduplication |
| `idx_cost_records_billing_date` | `(billing_date, product_id)` | Composite B-tree | Budget calculation query |
| `idx_cost_records_product_period` | `(product_id, billing_period)` | Composite B-tree | Invoice generation |

### prompt.versions
| Index | Columns | Type | Why |
|-------|---------|------|-----|
| `idx_versions_template_version` | `(template_id, version)` | UK | Version lookup |
| `idx_versions_template_state` | `(template_id, state)` | Composite B-tree | Find ACTIVE version for a template |

### brain.experiences
| Index | Columns | Type | Why |
|-------|---------|------|-----|
| `idx_experiences_task_type` | `(task_type, outcome)` | Composite B-tree | Pattern matching query |
| `idx_experiences_context_hash` | `context_hash` | B-tree | Similarity lookup by context hash |
| `idx_experiences_recorded_at` | `recorded_at` | B-tree (PART) | Partition-aware retention |

### knowledge.nodes
| Index | Columns | Type | Why |
|-------|---------|------|-----|
| `idx_nodes_type` | `node_type` | B-tree | Filter nodes by type |
| `idx_nodes_label_fts` | `label` (as tsvector) | GIN | Full-text search |
| `idx_nodes_properties_gin` | `properties` | GIN | Property containment queries |

### knowledge.relationships
| Index | Columns | Type | Why |
|-------|---------|------|-----|
| `idx_rels_from_node` | `from_node_id` | B-tree | Outbound traversal |
| `idx_rels_to_node` | `to_node_id` | B-tree | Inbound traversal |
| `idx_rels_type` | `relationship_type` | B-tree | Filter by relationship type |

### security.audit_log
| Index | Columns | Type | Why |
|-------|---------|------|-----|
| `idx_audit_timestamp` | `timestamp` | B-tree (PART) | Range queries |
| `idx_audit_identity` | `identity_hash` | B-tree | Filter by identity |
| `idx_audit_event_type` | `event_type` | B-tree | Filter by event type |
| `idx_audit_correlation` | `correlation_id` | B-tree | Trace lookup |

---

---

# Section 16: Partition Strategy

---

## 16.1 Partitioning Overview

PostgreSQL declarative partitioning (PARTITION BY RANGE) is used for time-series tables. Partitioning provides:
- Fast data deletion (DROP PARTITION instead of DELETE)
- Query pruning (scans only the relevant partition)
- Easier maintenance (VACUUM runs per-partition)

---

## 16.2 Partition Definitions

| Table | Partition Key | Partition Interval | Max Partitions Active |
|-------|--------------|-------------------|-----------------------|
| `security.audit_log` | `timestamp` | Monthly | Indefinite (never dropped) |
| `workflow.workflows` | `submitted_at` | Monthly | 3 months active + archive |
| `workflow.task_outputs` | `created_at` | Monthly | 1 month active |
| `brain.experiences` | `recorded_at` | Monthly | 12 months active |
| `provider.health_history` | `checked_at` | Daily | 30 partitions |
| `billing.cost_records` | `billing_date` | Daily | 365 partitions |
| `plugins.invocations` | `invoked_at` | Daily | 7 partitions |
| `plugins.health_checks` | `checked_at` | Daily | 7 partitions |
| `monitoring.sla_measurements` | `measured_at` | Daily | 90 partitions |
| `monitoring.health_check_results` | `checked_at` | Daily | 7 partitions |
| `workspace.product_health_history` | `changed_at` | Monthly | 1 month active |
| `prompt.ab_test_results` | `recorded_at` | Monthly | 2 months active |
| `usage.hourly_token_summaries` | `hour_utc` | Monthly | 1 month active |

---

## 16.3 Partition Creation and Maintenance

**Pre-creation:** Partitions are created 1 week in advance by a scheduled maintenance job. If the job fails and the next-day partition doesn't exist, inserts fail — pre-creation provides headroom.

**Naming convention:** `{table_name}_{year}{month}` for monthly, `{table_name}_{year}{month}{day}` for daily.

Examples:
```
billing.cost_records_20260628    (daily partition)
brain.experiences_202606         (monthly partition)
security.audit_log_202606        (monthly partition — never dropped)
```

**Default partition:** Each partitioned table has a default partition (`{table_name}_default`) that catches any rows that don't match a defined partition range. This prevents insert failures. A monitoring alert fires if any row lands in the default partition — it indicates either a partition creation failure or a timestamp outside expected bounds.

---

## 16.4 Workflow Archive Partition

Workflows older than 90 days are moved to `workflow.workflows_archive` — a separate table with the same schema but:
- No foreign keys (tasks/outputs may already be deleted)
- No indexes except PK and `(product_id, submitted_at)`
- Partitioned by month with no upper bound

This separation keeps the active `workflow.workflows` table small and fast while retaining historical data.

---

---

# Section 17: Sharding Strategy

---

## 17.1 Current State: No Sharding

The current platform is single-instance PostgreSQL. Sharding is not required at current scale. The decision to not shard is explicit and revisited quarterly.

**When to shard:** Sharding is warranted when any of these conditions is met:
- Write throughput exceeds 10,000 writes/second sustained on any single table
- Table size exceeds 500GB on any single table
- Query latency at P99 exceeds 2x the latency target for any endpoint after index and partition optimization

---

## 17.2 Shard Key Design (Pre-planned)

When sharding is needed, the following shard keys are pre-designated:

| Schema | Shard Key | Rationale |
|--------|-----------|-----------|
| `workflow` | `product_id` | Workflows are always queried by product. Sharding by product keeps all a product's workflows on one shard. |
| `billing` | `billing_date` | Cost records are always queried by date range. Date sharding enables range-based query routing. |
| `brain` | `task_type` | Experiences are queried by task type for pattern matching. Sharding by task type collocates similar experiences. |
| `security.audit_log` | `timestamp` | Already partitioned by month — each month becomes a candidate shard. |
| `knowledge` | `node_type` | After Kuzu migration, knowledge partitioning by node type distributes the graph naturally. |

---

## 17.3 Shard-Unsafe Patterns to Avoid

These patterns prevent sharding without application changes:

**Cross-shard JOINs:** If `billing.cost_records` and `workflow.workflows` were on different shards, a JOIN between them would require cross-shard communication. Design avoids this: all cross-domain reads go through APIs, not JOINs.

**Global sequences:** The platform uses UUIDs for all primary keys. No global sequences (SERIAL, BIGSERIAL) exist. UUID generation is distributed-safe.

**Cross-schema foreign keys:** Explicitly prohibited (Section 2). This ensures schemas can be split across database instances independently.

---

---

# Section 18: Future Expansion

---

## 18.1 v1.1: Knowledge Graph Migration (Kuzu)

**New storage:** Kuzu graph database  
**Impact on schema:** `knowledge.nodes` and `knowledge.relationships` migrate to Kuzu. `knowledge.memory` stays in PostgreSQL.  
**New capability:** Cypher-like path queries; cycle detection; graph algorithms (shortest path, centrality)  
**New tables required:** None in PostgreSQL (Kuzu is a separate data store)  
**Migration:** Dual-write period per Section 12.5

---

## 18.2 v1.1: Brain Vector Search (Qdrant)

**New storage:** Qdrant vector database  
**Replaces:** Jaccard similarity in `brain.pattern_experiences` (O(n²) Jaccard on SQL → ANN search on Qdrant)  
**New fields in brain.experiences:** `context_embedding` (float vector, 1536 dimensions) — already reserved as nullable JSONB in current schema  
**Impact on schema:** `brain.recommendation_cache` becomes less critical (Qdrant queries are fast enough to not require aggressive caching)

---

## 18.3 v2.5: Multi-Tenant Support

**New schemas required:**
- `tenants` schema: `tenants`, `tenant_products`, `tenant_memberships`

**Schema changes to existing tables:**

| Table | Change | Why |
|-------|--------|-----|
| `security.api_keys` | Add `tenant_id` column | Keys are scoped to tenants |
| `workflow.workflows` | Add `tenant_id` (PART) | Tenant-level workflow isolation |
| `billing.cost_records` | Add `tenant_id` | Tenant billing separation |
| `billing.budget_configs` | Add `tenant_id` | Per-tenant budgets |
| `prompt.templates` | Add `tenant_id` | Tenant-scoped templates |
| `knowledge.nodes` | Add `tenant_id` | Tenant knowledge graphs |

**Sharding:** v2.5 is the target release for introducing read replicas (not full sharding). Each tenant's workload is isolated by `product_id` already — tenant_id adds a second isolation level.

---

## 18.4 v3.0: High Availability

**Change:** PostgreSQL moves from single primary to primary + read replica pair  
**New storage component:** pgBouncer (connection pooling) in front of PostgreSQL  
**Schema changes:** None — all schemas remain identical on primary and replica  
**Read routing:** Analytics queries (`monitoring.sla`, `usage.*`, `billing.invoice_line_items`) route to the read replica. All write and transactional read queries go to the primary.

**Impact on indexes:** Read replica carries all the same indexes as the primary. No index differences between primary and replica.

---

## 18.5 v3.0: SSO and Identity Expansion

**Current model:** API keys only. No sessions, no OAuth, no user accounts.

**Future model:** SSO integration adds:
- `security.user_accounts` — user profile (hashed email, display name, tenant_id)
- `security.sso_sessions` — ephemeral session tokens (short TTL, aggressive pruning)
- `security.oauth_connections` — OAuth provider links (provider, external_id_hash)

**Impact on api_keys:** `api_keys.created_by` becomes a reference to either a `user_accounts` row or another `api_keys` row (machine-to-machine).

---

## 18.6 Capacity Projections

| Table | Current Est. | 12-Month Projection | 24-Month Projection |
|-------|-------------|--------------------|--------------------|
| `billing.cost_records` | 10K rows/day | 50K rows/day | 200K rows/day |
| `brain.experiences` | 5K rows/day | 25K rows/day | 100K rows/day |
| `workflow.tasks` | 50K rows/day | 250K rows/day | 1M rows/day |
| `security.audit_log` | 1K rows/day | 5K rows/day | 20K rows/day |
| `plugins.invocations` | 500 rows/day | 5K rows/day | 50K rows/day |

**At 200K cost records/day (24-month):** ~73M rows/year. At ~500 bytes/row average, that is ~36GB/year in `billing.cost_records`. With partitioning and a 365-day rolling window, the active table stays at ~36GB. This is well within single-instance PostgreSQL capacity.

**Trigger for sharding review:** 500M rows on any single table, or 500GB total database size. At current projections, this is not reached before v3.0.

---

---

# Appendix A: Migration Ledger (Initial Migrations)

| # | File | Schema | Description |
|---|------|--------|-------------|
| 0001 | `0001_platform_create_schema_migrations.sql` | platform | Create migration tracking table |
| 0002 | `0002_security_create_schemas.sql` | security | Create security schema |
| 0003 | `0003_security_create_api_keys.sql` | security | api_keys table + indexes |
| 0004 | `0004_security_create_audit_log.sql` | security | audit_log partitioned table + RLS |
| 0005 | `0005_security_create_rate_limit_buckets.sql` | security | rate_limit_buckets table |
| 0006 | `0006_workspace_create_schema.sql` | workspace | Create workspace schema |
| 0007 | `0007_workspace_create_products.sql` | workspace | products table + health_history (partitioned) |
| 0008 | `0008_workspace_create_platform_modules.sql` | workspace | platform_modules table |
| 0009 | `0009_workflow_create_schema.sql` | workflow | Create workflow schema |
| 0010 | `0010_workflow_create_workflows.sql` | workflow | workflows table (partitioned) + indexes |
| 0011 | `0011_workflow_create_tasks.sql` | workflow | tasks table + indexes |
| 0012 | `0012_workflow_create_task_outputs.sql` | workflow | task_outputs table (partitioned) |
| 0013 | `0013_workflow_create_checkpoints.sql` | workflow | checkpoints table |
| 0014 | `0014_workflow_create_gates.sql` | workflow | gates table |
| 0015 | `0015_prompt_create_schema.sql` | prompt | Create prompt schema |
| 0016 | `0016_prompt_create_templates.sql` | prompt | templates table |
| 0017 | `0017_prompt_create_versions.sql` | prompt | versions table + indexes |
| 0018 | `0018_prompt_create_governance_log.sql` | prompt | governance_log (append-only) + RLS |
| 0019 | `0019_prompt_create_ab_tests.sql` | prompt | ab_tests + ab_test_results (partitioned) |
| 0020 | `0020_brain_create_schema.sql` | brain | Create brain schema |
| 0021 | `0021_brain_create_experiences.sql` | brain | experiences table (partitioned) + indexes |
| 0022 | `0022_brain_create_patterns.sql` | brain | patterns + pattern_experiences + improvement_proposals |
| 0023 | `0023_brain_create_recommendation_cache.sql` | brain | recommendation_cache |
| 0024 | `0024_knowledge_create_schema.sql` | knowledge | Create knowledge schema |
| 0025 | `0025_knowledge_create_nodes.sql` | knowledge | nodes table + GIN index + FTS |
| 0026 | `0026_knowledge_create_relationships.sql` | knowledge | relationships table + dual indexes |
| 0027 | `0027_knowledge_create_memory.sql` | knowledge | memory table + unique constraint |
| 0028 | `0028_provider_create_schema.sql` | provider | Create provider schema |
| 0029 | `0029_provider_create_providers.sql` | provider | providers table |
| 0030 | `0030_provider_create_models.sql` | provider | models table + pricing versioning |
| 0031 | `0031_provider_create_health_history.sql` | provider | health_history (partitioned) |
| 0032 | `0032_provider_create_circuit_events.sql` | provider | circuit_events |
| 0033 | `0033_billing_create_schema.sql` | billing | Create billing schema |
| 0034 | `0034_billing_create_cost_records.sql` | billing | cost_records (partitioned, immutable) |
| 0035 | `0035_billing_create_budget_configs.sql` | billing | budget_configs + snapshots |
| 0036 | `0036_billing_create_invoices.sql` | billing | invoices + line_items |
| 0037 | `0037_usage_create_schema.sql` | usage | Create usage schema |
| 0038 | `0038_usage_create_summaries.sql` | usage | All 4 summary tables |
| 0039 | `0039_plugins_create_schema.sql` | plugins | Create plugins schema |
| 0040 | `0040_plugins_create_plugins.sql` | plugins | plugins + health_checks (partitioned) + invocations (partitioned) |
| 0041 | `0041_marketplace_create_schema.sql` | marketplace | Create marketplace schema (tables empty) |
| 0042 | `0042_marketplace_create_tables.sql` | marketplace | listings + installations + reviews |
| 0043 | `0043_monitoring_create_schema.sql` | monitoring | Create monitoring schema |
| 0044 | `0044_monitoring_create_alerts.sql` | monitoring | alerts |
| 0045 | `0045_monitoring_create_sla.sql` | monitoring | sla_measurements (partitioned) |
| 0046 | `0046_monitoring_create_health_checks.sql` | monitoring | health_check_results (partitioned) |

---

## Appendix B: Schema Dependency Order

The schemas must be created in this order (later schemas may reference earlier ones via soft references):

```
1. platform   (meta — migration tracking)
2. security   (auth — first to exist; everything needs it)
3. workspace  (product registry — referenced by many)
4. workflow   (execution — references workspace softly)
5. prompt     (content — references brain softly)
6. brain      (intelligence — references prompt, workflow softly)
7. knowledge  (intelligence — independent)
8. provider   (execution — independent)
9. billing    (commercial — references provider, workflow softly)
10. usage     (commercial — references billing softly)
11. plugins   (operations — independent)
12. monitoring (operations — references workspace, workflow softly)
13. marketplace (operations — independent; PROPOSED)
```

---

## Appendix C: Glossary

| Term | Definition |
|------|-----------|
| **Aggregate Root** | The entity through which all writes to a cluster of related entities must pass. |
| **Bounded Context** | A coherent domain model boundary. Data within a context is owned by that context; data crossing boundaries travels via events or APIs. |
| **Deterministic UUID** | A UUID computed from input data (e.g., SHA-256 of execution_id). Same input always produces the same UUID. Used for idempotency. |
| **Dual-Write** | A migration pattern where writes go to both the old and new storage simultaneously. Enables zero-downtime migration. |
| **Immutable Table** | A table where rows may only be inserted, never updated or deleted. Enforced via PostgreSQL RLS or application-level contract. |
| **Partition** | A PostgreSQL child table that stores a subset of its parent table's rows (e.g., one month's worth of data). |
| **Partition Drop** | Removing a partition by DROP TABLE — instantaneous, non-locking. More efficient than DELETE for large time-series retention. |
| **Read Model** | A projection or view optimized for reads. May be stale. Derives from write models. |
| **Schema Ownership** | The principle that each PostgreSQL schema is written to exclusively by one platform module. |
| **Soft Reference** | A column that stores the value of another table's primary key, but without a foreign key constraint. Allows cross-schema references without tight coupling. |
| **Write Model** | The canonical schema for data as it is written. Normalized, consistent, owned by one aggregate root. |

---

*End of DATABASE-CATALOG v1.0.0*

*Document authority: Platform Data Architect*  
*Next review: 2026-09-28 (quarterly)*  
*Schema changes require: Architect approval + migration in this catalog's ledger*  
*Breaking schema changes (column removal, type change) require: CTO approval + migration plan*

