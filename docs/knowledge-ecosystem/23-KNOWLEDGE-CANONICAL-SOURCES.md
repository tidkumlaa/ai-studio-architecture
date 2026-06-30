# KNW-KE-ARCH-023 — Knowledge Canonical Sources

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Every claim in every Knowledge Object must have a canonical source. This document defines the source hierarchy, source types, conflict resolution rules, and traceability from source to object.

---

## Source Hierarchy

Sources are ranked by authority. When two sources conflict, the higher-ranked source wins:

```
Level 1 (HIGHEST): Architecture Board Decision
  - Explicit ADR (Architecture Decision Record)
  - Architecture Freeze documents
  - Board meeting minutes

Level 2: Architecture Specification
  - Architecture documents in architecture/docs/
  - Phase specification documents
  - Universal Schema

Level 3: Empirical Evidence
  - Benchmark results (measured performance)
  - Test results (pass/fail)
  - Production monitoring data

Level 4: Implementation
  - Production code in platform/
  - Committed code in git

Level 5: Engineering Proposal
  - PR descriptions
  - Engineering design docs
  - Team consensus

Level 6 (LOWEST): Informal
  - Conversation notes
  - Comments in code
  - Unreviewed drafts
```

---

## Canonical Source Types

| Type | Code | Level | Weight | Freshness threshold |
|------|------|-------|--------|-------------------|
| Board Decision | EV-HAPPROVAL | 1 | 0.80 | 1 year |
| Architecture Doc | EV-DOC | 2 | 0.60 | 6 months |
| Benchmark Result | EV-BENCHMARK | 3 | 0.70 | 90 days |
| Test Result | EV-TEST | 3 | 0.75 | 30 days |
| Production Code | EV-CODE | 4 | 0.60 | 180 days |
| Monitoring Data | EV-MONITORING | 3 | 0.70 | 7 days |
| Engineering Proposal | EV-PROPOSAL | 5 | 0.40 | 30 days |
| User Feedback | EV-USER | 5 | 0.50 | 30 days |
| External Standard | EV-STANDARD | 2 | 0.80 | 1 year |
| Research Paper | EV-RESEARCH | 3 | 0.65 | 2 years |

---

## Source Rules

| Rule | Description |
|------|-------------|
| CS-001 | Every VERIFIED+ object must cite at least one source |
| CS-002 | Every CANONICAL object must cite at least one Level 1–4 source |
| CS-003 | Sources from Level 5–6 alone are insufficient for CANONICAL |
| CS-004 | Source conflict resolution always favours the higher-ranked source |
| CS-005 | When sources conflict at the same level, the newer source wins |
| CS-006 | Disputed sources must be flagged with `disputed: true` in the evidence record |
| CS-007 | No external URL as source without archiving it (use DOI or git snapshot) |

---

## Source Record Format

```yaml
# In an object's evidence.items list:
- evidence_type: EV-DOC
  source_id: "KNW-ARCH-ARCH-001"        # canonical reference to the source object
  source_url: "architecture/docs/knowledge-core/37-PERFORMANCE-BUDGET.md"
  description: "P99 < 5ms specified in Section 'Registry Budgets'"
  weight: 0.60
  level: 2
  captured_at: "2026-06-30T00:00:00Z"
  freshness_score: 1.0
  disputed: false
  dispute_reason: null
```

---

## Conflict Resolution

When two sources for the same claim differ:

```
STEP 1: Compare levels. Higher wins.
  EV-HAPPROVAL (level 1) vs EV-CODE (level 4) → EV-HAPPROVAL wins.

STEP 2: Same level — compare captured_at. Newer wins.
  EV-CODE from 2026-01-01 vs EV-CODE from 2026-06-30 → 2026-06-30 wins.

STEP 3: Same level, same date — flag as dispute.
  Add both as evidence records.
  Set disputed: true on the lower-confidence one.
  Open Architecture Board resolution.
```

---

## Source Traceability Chain

```
Architecture Board Decision (Level 1)
    └── Architecture Document (Level 2)
          └── Implementation in Code (Level 4)
                └── Test (Level 3)
                      └── Benchmark (Level 3)
```

Example chain for `KNW-PLT-MOD-001 Quota Manager`:

```
ADR-001: Use Python for platform (EV-HAPPROVAL, Level 1)
  → architecture/docs/knowledge-core/19-RUNTIME-REGISTRY (EV-DOC, Level 2)
    → platform/ai_runtime/quota/engine.py (EV-CODE, Level 4)
      → TEST: KNW-TEST-TST-001 (EV-TEST, Level 3)
        → BENCH: KNW-TEST-BENCH-001 (EV-BENCHMARK, Level 3)
```

---

## Canonical Source Registry

```yaml
# knowledge/registry/canonical-sources.yaml
version: "1.0.0"
sources:
  - source_id: "CS-001"
    type: EV-HAPPROVAL
    title: "KOS Phase 3.0C Architecture Freeze"
    location: "architecture/docs/knowledge-core/40-ARCHITECTURE-FREEZE"
    captured_at: "2026-06-30"
    level: 1
    objects_supported: []   # populated by kos catalog build

  - source_id: "CS-002"
    type: EV-DOC
    title: "Universal Knowledge Schema"
    location: "architecture/docs/knowledge-core/02-UNIVERSAL-SCHEMA"
    captured_at: "2026-06-30"
    level: 2
    objects_supported: []
```

---

## Source Freshness Policy

| Level | Freshness window | After expiry |
|-------|-----------------|--------------|
| Level 1 (Board) | 1 year | Must re-approve or extend |
| Level 2 (Arch doc) | 6 months | Flag for review; object loses freshness |
| Level 3 (Empirical) | 7–90 days (type-dependent) | Evidence marked STALE |
| Level 4 (Code) | 180 days | Check if code still reflects claim |
| Level 5–6 | 30 days | Evidence expires quickly |

---

## Cross-References

- Evidence types and weights → Phase 3.0C `10-EVIDENCE-ENGINE`
- Evidence freshness decay → Phase 3.0C `36-ALGORITHMS` A-05
- Source traceability → `29-KNOWLEDGE-TRACEABILITY`
- Dispute protocol → Phase 3.0C `10-EVIDENCE-ENGINE`
