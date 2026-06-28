# AI Policy Engine — Routing Policies, Compliance, and Governance

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

The Policy Engine is the governance layer of AI ROS. Every request passes through it before any provider is contacted. It evaluates routing policies, business rules, compliance constraints, security restrictions, regional data residency requirements, model allow/deny lists, and human approval triggers. It is the enforcement point for organizational AI governance.

---

## 2. Responsibilities

- Evaluate all active policies for each request (routing, compliance, security, approval)
- Enforce model allow/deny lists per organization, project, and user
- Enforce regional data residency restrictions
- Enforce content restrictions (sensitive data in prompt, restricted task types)
- Trigger the Approval Engine for high-stakes requests
- Provide a policy audit trail (every evaluation recorded)
- Support policy lifecycle: draft, active, suspended, archived
- Enable policy testing mode (simulate without blocking)

---

## 3. Architecture

```
Routing Engine
      │ evaluate_policies(policy_context)
      ▼
┌──────────────────────────────────────────────────────────────────────┐
│                          Policy Engine                                 │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │                     Policy Registry                                │  │
│  │  load_active_policies(org_id, project_id) → List[Policy]         │  │
│  └──────────────────────────────┬───────────────────────────────────┘  │
│                                  │                                      │
│  ┌──────────────────────────────▼───────────────────────────────────┐  │
│  │                     Policy Evaluator                               │  │
│  │                                                                    │  │
│  │  For each active policy (ordered by priority):                    │  │
│  │    result = policy.evaluate(context)                              │  │
│  │    if result.action == DENY → stop; return DENIED                 │  │
│  │    if result.action == REQUIRE_APPROVAL → collect approval reqs   │  │
│  │    if result.action == CONSTRAIN → narrow candidate set           │  │
│  │    if result.action == ALLOW → continue                           │  │
│  └──────────────────────────────┬───────────────────────────────────┘  │
│                                  │                                      │
│  ┌──────────────────────────────▼───────────────────────────────────┐  │
│  │                     Audit Logger                                   │  │
│  │  Record all evaluations (pass/fail/deny) in ai_ros_audit_log     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
         │ (if REQUIRE_APPROVAL)
         ▼
  Approval Engine
```

---

## 4. Policy Data Model

### 4.1 Policy

| Field | Type | Description |
|-------|------|-------------|
| `policy_id` | `UUID` | Primary key |
| `org_id` | `UUID` | Organization scope |
| `project_id` | `UUID \| None` | Optional project scope |
| `name` | `str` | Human-readable name |
| `description` | `str` | Purpose and rationale |
| `policy_type` | `PolicyType` | Category (see section 5) |
| `status` | `PolicyStatus` | `DRAFT` / `ACTIVE` / `SUSPENDED` / `ARCHIVED` |
| `priority` | `int` | Evaluation order (lower = higher priority) |
| `conditions` | `List[PolicyCondition]` | When this policy applies |
| `action` | `PolicyAction` | What to do when conditions match |
| `action_parameters` | `dict` | Action-specific parameters |
| `test_mode` | `bool` | If True: evaluate but don't enforce (log only) |
| `created_at` | `datetime` | |
| `created_by` | `UUID` | |
| `effective_from` | `datetime \| None` | Time-bounded policies |
| `effective_until` | `datetime \| None` | |

### 4.2 PolicyCondition

Conditions are ANDed within a policy.

| Field | Type | Description |
|-------|------|-------------|
| `field` | `str` | Context field to evaluate (see section 6) |
| `operator` | `ConditionOperator` | `EQ`, `NEQ`, `IN`, `NOT_IN`, `CONTAINS`, `GT`, `LT`, `REGEX`, `EXISTS` |
| `value` | `Any` | Target value |

### 4.3 PolicyAction

| Action | Parameters | Behavior |
|--------|-----------|---------|
| `ALLOW` | None | Explicitly permit; useful to override higher-priority DENY |
| `DENY` | `reason: str` | Block request; return structured error |
| `REQUIRE_APPROVAL` | `approver_roles: List[str]`, `reason: str` | Trigger approval workflow |
| `CONSTRAIN_PROVIDERS` | `allowed_providers: List[str]` | Narrow provider candidate set |
| `CONSTRAIN_MODELS` | `allowed_models: List[str]` | Narrow model candidate set |
| `FORCE_PROVIDER` | `provider_id: str` | Override routing: use this provider only |
| `FORCE_MODEL` | `canonical_model_id: str` | Override routing: use this model only |
| `REQUIRE_LOCAL` | None | Force routing to local provider (Ollama, LM Studio) |
| `ADD_METADATA` | `metadata: dict` | Attach metadata to request for downstream use |
| `REDACT_FIELD` | `fields: List[str]` | Redact specified prompt fields before sending |

---

## 5. Policy Types

### 5.1 ROUTING_POLICY
Controls which providers and models may be used.

Examples:
- "AI Software Factory project may only use Anthropic Claude"
- "Content Factory may use any provider"
- "Mythic Realms game logic must use FRONTIER quality models"
- "ERP Generator must not use local providers for financial data"

### 5.2 MODEL_ALLOWLIST / MODEL_DENYLIST
Explicit allow or deny lists for models.

Examples:
- "Organization allows only: claude-opus-4-8, claude-sonnet-4-6, claude-haiku-4-5"
- "Project 'prototype' denies: o3 (too expensive for experiments)"
- "User 'intern-01' is limited to EFFICIENT tier models only"

### 5.3 DATA_RESIDENCY
Ensures data stays in configured geographic regions.

Examples:
- "EU organization: providers must be in eu-west-1 or eu-central-1"
- "HIPAA project: only approved US regions"
- "Local-only project: must use local/ollama or local/lm-studio"

### 5.4 CONTENT_RESTRICTION
Restricts request content or output.

Examples:
- "PII detection: if prompt contains email/SSN pattern, require approval"
- "No external data: block requests with web_search capability"
- "Code execution disabled for untrusted projects"

### 5.5 SPEND_POLICY
Spending controls beyond the Budget Engine.

Examples:
- "Any single request estimated > $5.00 requires manager approval"
- "Weekend requests auto-downgrade to EFFICIENT models"
- "Batch jobs must use LOW priority and ECONOMY models"

### 5.6 COMPLIANCE_POLICY
Regulatory and legal compliance rules.

Examples:
- "GDPR compliance: no user PII to non-EU providers"
- "SOC2 compliance: all provider responses must be logged for 7 years"
- "HIPAA compliance: no PHI to provider without BAA — Claude and Azure OpenAI only"

### 5.7 RATE_POLICY
Custom rate limits applied by the Policy Engine (supplemental to Scheduler rate limits).

Examples:
- "Development environment: max 10 requests/minute per user"
- "Free tier: max 1,000 tokens per request"

---

## 6. Policy Context

The `PolicyContext` object is assembled by the Routing Engine and passed to the Policy Engine for evaluation.

| Field | Type | Description |
|-------|------|-------------|
| `request_id` | `UUID` | |
| `org_id` | `UUID` | |
| `project_id` | `UUID \| None` | |
| `user_id` | `UUID \| None` | |
| `user_roles` | `List[str]` | User's roles within the org |
| `product_id` | `str` | Which AI Studio product originated this request |
| `task_type` | `str \| None` | Application-level task category |
| `provider_candidates` | `List[str]` | Providers under consideration |
| `model_candidates` | `List[str]` | Models under consideration |
| `required_capabilities` | `List[str]` | Capabilities the request needs |
| `estimated_cost_usd` | `Decimal` | From Budget Engine pre-computation |
| `estimated_input_tokens` | `int` | |
| `has_tool_calls` | `bool` | |
| `has_web_search` | `bool` | |
| `has_code_execution` | `bool` | |
| `prompt_pii_detected` | `bool` | From PII scanner |
| `prompt_sensitive_keywords` | `List[str]` | From content scanner |
| `data_classification` | `str \| None` | `PUBLIC` / `INTERNAL` / `CONFIDENTIAL` / `RESTRICTED` |
| `request_region` | `str` | AWS region / geographic origin |
| `environment` | `str` | `development` / `staging` / `production` |
| `scheduling_mode` | `str` | `IMMEDIATE` / `QUEUED` / `BATCH` |

---

## 7. Policy Evaluation Algorithm

```
evaluate_policies(context: PolicyContext) → PolicyEvaluationResult

1. Load all active policies for (org_id, project_id), ordered by priority ASC
2. For each policy:
   a. Check effective_from / effective_until (skip if not in range)
   b. Evaluate all conditions (AND logic)
   c. If conditions match:
      - If test_mode: log match, continue (do not enforce)
      - If DENY: record evaluation, return DENIED immediately
      - If REQUIRE_APPROVAL: add to approval_requirements list
      - If CONSTRAIN_*: narrow candidate sets
      - If FORCE_*: record forced selection
      - If ALLOW: mark as explicitly allowed (prevents lower-priority DENY)
3. Return PolicyEvaluationResult
```

**PolicyEvaluationResult:**

| Field | Type | Description |
|-------|------|-------------|
| `allowed` | `bool` | Whether request may proceed |
| `denial_reason` | `str \| None` | Set if allowed=False |
| `denial_policy_id` | `UUID \| None` | Which policy denied |
| `requires_approval` | `bool` | |
| `approval_requirements` | `List[ApprovalRequirement]` | |
| `constrained_providers` | `List[str] \| None` | Narrowed from original candidates |
| `constrained_models` | `List[str] \| None` | |
| `forced_provider` | `str \| None` | |
| `forced_model` | `str \| None` | |
| `matched_policies` | `List[UUID]` | All policies that matched (for audit) |
| `test_mode_hits` | `List[UUID]` | Policies in test mode that would have triggered |

---

## 8. Approval Engine Integration

When `requires_approval=True`, the Approval Engine is invoked before dispatching:

```
ApprovalRequirement:
  policy_id:         UUID
  reason:            str
  approver_roles:    List[str]   e.g., ["org_admin", "project_manager"]
  timeout_minutes:   int         default: 60
  on_timeout:        str         "DENY" | "AUTO_APPROVE"
  context_summary:   str         Human-readable summary for approver
```

The request is held in `PENDING_APPROVAL` status. The approver receives a notification (via event → Desktop notification or email). On approval: request proceeds. On rejection or timeout: request is denied.

---

## 9. PII Detection

The Policy Engine includes a lightweight PII scanner that runs on the assembled prompt before evaluation:

| Pattern | Detection Method |
|---------|----------------|
| Email addresses | Regex |
| Phone numbers | Regex (international formats) |
| SSN (US) | Regex |
| Credit card numbers | Regex + Luhn check |
| AWS access keys | Regex |
| API key patterns | Regex (common formats) |
| Names + dates (potential patient data) | Heuristic |

Detection results are added to `PolicyContext.prompt_pii_detected` and `prompt_sensitive_keywords`. CONTENT_RESTRICTION policies can then enforce approval or denial.

PII detection is optional (configurable per organization) and does not store the detected values — only boolean flags are set.

---

## 10. Policy Testing Mode

Organizations can deploy policies in `test_mode=True` to evaluate impact before enforcement:

- Policies are evaluated fully
- Matches are logged to `ai_ros_policy_test_log`
- No requests are denied or constrained
- Test mode results visible in Desktop Policy dashboard
- After review, policy is promoted to enforcement by toggling `test_mode=False`

---

## 11. Policy Versioning

Every change to a policy creates a new version record. The current active version is always used for evaluation. Rollback to previous version available via admin API.

| Field | Type | Description |
|-------|------|-------------|
| `policy_version_id` | `UUID` | |
| `policy_id` | `UUID` | |
| `version_number` | `int` | Incrementing integer |
| `snapshot` | `JSON` | Full policy state at this version |
| `changed_by` | `UUID` | |
| `changed_at` | `datetime` | |
| `change_reason` | `str` | |

---

## 12. Data Model

### ai_ros_policies
Policy configuration. See `AI-RESOURCE-DATABASE.md`.

### ai_ros_policy_evaluations
Audit log of every evaluation.

| Column | Type | Notes |
|--------|------|-------|
| `evaluation_id` | UUID | PK |
| `request_id` | UUID | |
| `policy_id` | UUID | |
| `policy_version` | int | |
| `matched` | bool | Whether conditions matched |
| `action_taken` | str | `ALLOW` / `DENY` / `CONSTRAIN` / `SKIPPED` |
| `test_mode` | bool | |
| `evaluated_at` | timestamptz | |

---

## 13. Events Published

| Event | Trigger |
|-------|---------|
| `policy.evaluation.passed` | All policies passed |
| `policy.evaluation.denied` | Request denied by policy |
| `policy.evaluation.approval_required` | Approval workflow triggered |
| `policy.test_mode.hit` | Test-mode policy would have triggered |
| `policy.created` | New policy created |
| `policy.modified` | Policy changed |
| `policy.activated` | Policy status → ACTIVE |
| `policy.suspended` | Policy suspended |

---

## 14. Security

- Policy configuration writable only by org admin and policy admin roles
- All policy changes are immutably versioned and audit-logged
- PII scanner results never stored in persistent log (only boolean flags)
- Approval notifications sent to approver only — request content is not included without explicit approval from data owner

---

## 15. Scalability

- Active policies are cached in-memory per (org_id, project_id) with TTL 30 seconds
- Policy evaluation is typically < 1ms for the common case (< 20 active policies)
- PII scanner is designed to run in < 10ms on prompts up to 100K tokens

---

## 16. Future Evolution

| Feature | Notes |
|---------|-------|
| Policy-as-code (YAML import/export) | Version-control policies in Git |
| ML-based anomaly detection | Detect unusual usage patterns |
| Cross-organization policy templates | Share compliance policy templates |
| Real-time policy streaming | Policy changes take effect within 1 second |

---

## 17. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-SCHEDULER.md | Pre-dispatch gate |
| AI-BUDGET-ENGINE.md | Spend policies coordinate with budget checks |
| AI-RESOURCE-SECURITY.md | Compliance and data residency policy foundations |
| AI-RESOURCE-EVENTS.md | Event schemas |
| AI-RESOURCE-DATABASE.md | Table schemas |
| AI-RESOURCE-DESKTOP.md | Policy management UI |
