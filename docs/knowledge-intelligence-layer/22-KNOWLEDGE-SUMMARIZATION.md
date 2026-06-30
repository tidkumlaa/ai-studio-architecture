# KNW-KIL-DOC-022 — Knowledge Summarization

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Summarization defines the rules and strategies for generating compressed
representations of Knowledge Objects. While Compression (doc 05) defines
the target formats and token budgets, Summarization defines *how* to produce
those compressed forms correctly.

Correct summarization is non-trivial: a wrong summary that omits a critical
constraint is worse than no summary — it actively misleads.

This document defines the summarization rules, mandatory inclusions, forbidden
omissions, and quality checks for all compression levels.

---

## Summarization Rules

### SR-001: Lead with Primary Capability

The first sentence of any summary must state the object's primary capability.
It must not begin with the object's name alone.

```
✓ "Enforces per-tenant resource quotas with P99 < 5ms."
✗ "Quota Manager is a module in the platform namespace."
```

### SR-002: Include Primary Constraint

Any summary ≥ L2 (Micro) must include the most important constraint.

```
✓ "...returns ALLOW/DENY; P99 < 5ms required."
✗ "...returns ALLOW or DENY decision."
```

### SR-003: Include Scope Boundary

Any summary ≥ L3 (Mini) must explicitly state one NOT-in-scope item when a
common confusion exists.

```
✓ "...does NOT configure quotas (Admin API responsibility)."
✗ (omission when confusion is known)
```

### SR-004: Cite Critical Dependencies

Any summary ≥ L3 must name HARD dependencies.

```
✓ "...HARD dependency on KNW-RT-RT-001 Registry."
✗ (omission of hard dependency)
```

### SR-005: No Hallucination Expansion

Summaries must never add information not present in the source object.
Truncation is allowed; invention is not.

### SR-006: No Capability Omission for Primary Capabilities

A summary must not omit the object's primary capability to meet token budget.
Reduce detail, not capabilities.

### SR-007: Confidence Propagation

If `confidence.level` is MEDIUM or below, the summary must include a hedge.
Do not summarize a LOW-confidence object as if it were certain.

---

## Summarization Strategy by Level

### L1 Nano (≤ 15 tokens)

```
Format:  {knowledge_id} {name}
Example: "KNW-PLT-MOD-001 Quota Manager"

Rules:
  - Always: knowledge_id + name
  - Never: description, constraint, capability
```

### L2 Micro (≤ 50 tokens)

```
Format:  {name}: {primary_capability}, {primary_constraint}.
Example: "Quota Manager: enforces per-tenant CPU/Memory/Request/Storage
          quotas; returns ALLOW/DENY; P99 < 5ms."

Rules:
  - Always: name, primary capability, primary constraint
  - Optional: namespace, version, state
  - Never: dependencies, evidence, history
```

### L3 Mini (≤ 200 tokens)

```
Format:
  {knowledge_id} {name} ({canonical_name} {version})
  Purpose: {purpose}
  Capabilities: {cap_1} ({category}), {cap_2}
  Critical constraint: {most_critical_constraint}
  Depends on: {HARD_dependencies}
  Provides: {primary_output}
  [NOT: {primary_scope_boundary}]
  State: {state}. Quality: {quality_score}.

Rules:
  - Always: all fields in Format above
  - If confidence < 0.70: add hedge before capabilities
  - If has common_confusion: add NOT scope boundary
```

### L4 Standard (≤ 500 tokens)

```
Format: YAML structured block (see compression.standard schema)

Mandatory inclusions:
  - id, name, type, namespace
  - purpose (full statement)
  - all capabilities
  - all HARD dependencies
  - all primary constraints
  - quality_score, state
  - key_decisions (top 3)

Optional (include if budget allows):
  - evidence summary
  - relationship count
  - version history
```

### L5 Machine (structure only)

```
Format: JSON (see compression.machine schema)

Always include: id, type, version, capabilities[], provides[],
                consumes[], depends_on[], implements[],
                quality_score, confidence_score, ai_readiness_score

Purpose: Machine consumption only — no prose
```

---

## Forbidden Omissions

The following must NEVER be omitted regardless of compression level:

| Level | Must Never Omit |
|-------|----------------|
| L2+ | Primary capability |
| L2+ | Primary constraint (if constraint is fundamental) |
| L3+ | HARD dependencies |
| L3+ | Primary scope boundary (if common confusion exists) |
| L4+ | Quality score and lifecycle state |
| All | Confidence hedge when level < MEDIUM |

---

## Summarization Quality Check

A summarization must pass these checks before being stored:

| Check | Test |
|-------|------|
| SC-01 | L1 token count ≤ 15 |
| SC-02 | L2 token count ≤ 50 |
| SC-03 | L3 token count ≤ 200 |
| SC-04 | L4 token count ≤ 500 |
| SC-05 | Primary capability appears in L2 |
| SC-06 | Primary constraint appears in L2 |
| SC-07 | All HARD dependencies appear in L3 |
| SC-08 | Hedge present in L3 when confidence < 0.70 |
| SC-09 | Scope boundary appears in L3 if common_confusion exists |
| SC-10 | L5 machine form parses as valid JSON |

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-123 | All 10 summarization quality checks (SC-01–SC-10) must pass |
| KIL-124 | Summarization must be regenerated whenever `self_describing.purpose` or `dna.core_behavior` changes |
| KIL-125 | `compression.nano` is always derivable from `knowledge_id` + `name` — it must never be hand-crafted |
| KIL-126 | Summaries are DERIVED artifacts — the source of truth is always the full object |

---

## Cross-References

- Compression levels → `05-KNOWLEDGE-COMPRESSION`
- AI context summaries → `04-AI-CONTEXT-LAYER`
- Explanation layer → `21-KNOWLEDGE-EXPLANATION-LAYER`
- AI readiness score → `27-KNOWLEDGE-AI-READINESS`
