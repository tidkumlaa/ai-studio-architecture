# KNW-KE-ARCH-016 — Knowledge Quality

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Quality gates and dimension specifications applied to Knowledge Ecosystem objects. Every object in every package must meet the minimum quality threshold for its lifecycle state.

---

## Quality Dimensions (from Phase 3.0C `11-QUALITY-ENGINE`)

| Dim | Name | Weight | Focus |
|-----|------|--------|-------|
| QD-1 | Completeness | 0.15 | All required fields populated |
| QD-2 | Consistency | 0.15 | Internal rules satisfied |
| QD-3 | Coverage | 0.20 | Relationships and traceability |
| QD-4 | Freshness | 0.10 | Evidence and content recency |
| QD-5 | Evidence | 0.15 | Evidence score |
| QD-6 | Confidence | 0.10 | Declared and composite confidence |
| QD-7 | Usage | 0.10 | Referential usage by other objects |
| QD-8 | Reasoning | 0.05 | Decision/requirement rationale |
| QD-9 | Health | — | Multiplier; penalises broken relations |

`overall_score = weighted_sum(QD-1..QD-8) × health_penalty`

---

## Quality Gates by Lifecycle State

| State | Min Overall | Min Evidence | Min Confidence | Additional |
|-------|------------|-------------|----------------|-----------|
| DRAFT | none | none | none | none |
| REVIEW | ≥ 0.50 | none | none | completeness ≥ 0.60 |
| VERIFIED | ≥ 0.65 | ≥ 0.55 | ≥ 0.50 | ≥ 1 evidence record |
| APPROVED | ≥ 0.75 | ≥ 0.65 | ≥ 0.65 | `approved_by` non-empty |
| CANONICAL | ≥ 0.80 | ≥ 0.70 | ≥ 0.80 | ≥ 1 EV-CODE or EV-TEST |
| DEPRECATED | (no gate — already canonical) | | | `lifecycle.successor_id` set |
| ARCHIVED | (no gate — read-only) | | | |

---

## Quality Gate Enforcement

Quality gates are checked:
1. By `kos lint` (rules KL-022, KL-023)
2. By CI quality-gate step (`kos quality-gate`)
3. By lifecycle engine before any status transition (Phase 3.0D)

**Not enforced for:**
- `knowledge/dataset/regression/` — may contain intentionally low-quality objects
- `knowledge/examples/` — must meet VERIFIED quality minimum
- `knowledge/templates/` — quality N/A (these are templates, not objects)

---

## Quality Computation for Ecosystem Objects

### QD-1: Completeness

```
completeness = non_empty_required_fields / total_required_fields
```

Required fields for all types (base): `knowledge_id, object_type, name, version, status, owner, description, tags`

Type-specific required fields are defined in `03-KNOWLEDGE-TEMPLATES`.

### QD-2: Consistency

Checks:
- `version` format valid
- `knowledge_id` format valid
- `knowledge_uri` matches computed URI
- `canonical_name` matches computed name
- `confidence.declared` in [0, 1]
- No circular `traceability.satisfies`

```
consistency = passed_checks / total_checks
```

### QD-3: Coverage

```
coverage = (satisfies_filled + implements_filled + tests_filled) / 3.0
```

Where `_filled` = 1.0 if non-empty, 0.0 otherwise. Weighted by object type:
- REQUIREMENT: satisfies weight 0 (requirements don't satisfy other reqs)
- MODULE: implements weight 1.5 (must implement something)
- TEST: tests weight 1.5 (must test something)

### QD-4: Freshness

```
freshness = min(1.0, 1.0 - (days_since_update / 365))
```

Cap at 1.0 for objects updated within a year. Penalty increases linearly after.

### QD-5: Evidence

```
evidence = object.evidence.evidence_score   (computed by Evidence Engine)
```

### QD-6: Confidence

```
confidence_dim = object.confidence.declared
```

### QD-7: Usage

```
usage = min(1.0, inbound_reference_count / 10.0)
```

Objects referenced by 10+ other objects score 1.0.

### QD-8: Reasoning

```
if object_type in [DECISION, REQUIREMENT]:
  reasoning = 1.0 if (context non-empty AND chosen_option non-empty) else 0.0
else:
  reasoning = 1.0  # N/A for other types
```

### QD-9: Health Penalty

```
broken_relations = count of ACTIVE rels pointing to ARCHIVED objects
total_relations  = count of all ACTIVE relations
health_penalty = 1.0 - (broken_relations / max(1, total_relations))
```

---

## Quality Alerts

| Alert | Condition | Severity |
|-------|-----------|---------|
| QA-001 | overall_score < 0.50 | ERROR |
| QA-002 | completeness < 0.60 | WARNING |
| QA-003 | evidence_score < 0.30 | WARNING |
| QA-004 | confidence.declared < 0.40 | WARNING |
| QA-005 | freshness < 0.30 (> 8 months old) | INFO |
| QA-006 | health_penalty < 0.90 | ERROR |

---

## Quality Commands

```bash
kos quality score KNW-PLT-MOD-001        # compute quality for one object
kos quality score --namespace plt        # compute for all objects in namespace
kos quality gate --status CANONICAL      # check CANONICAL quality gate
kos quality report --output reports/     # generate quality report
kos quality freshness                    # check evidence freshness
```

---

## Cross-References

- Dimension formulas → Phase 3.0C `11-QUALITY-ENGINE`
- Evidence engine → Phase 3.0C `10-EVIDENCE-ENGINE`
- Scoring detail → `17-KNOWLEDGE-SCORING`
- Linter checks → `06-KNOWLEDGE-LINTER` KL-022, KL-023
