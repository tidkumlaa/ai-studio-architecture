# KNW-KE-ARCH-017 — Knowledge Scoring

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Complete scoring formulas for all 9 quality dimensions applied to Knowledge Ecosystem objects. This document is the authoritative reference for any implementation of `kos quality score`.

---

## Scoring Algorithm (A-04 applied to ecosystem)

```
INPUT:  object (any KnowledgeObject subtype)
OUTPUT: QualityScoreRecord

STEP 1: compute each dimension
  qd1 = completeness(object)
  qd2 = consistency(object)
  qd3 = coverage(object, registry)
  qd4 = freshness(object)
  qd5 = evidence_score(object)
  qd6 = confidence_score(object)
  qd7 = usage(object, registry)
  qd8 = reasoning(object)

STEP 2: compute weighted sum
  weights = [0.15, 0.15, 0.20, 0.10, 0.15, 0.10, 0.10, 0.05]
  weighted = sum(qdi * wi for qdi, wi in zip(dimensions, weights))

STEP 3: compute health penalty
  health = 1.0 - (broken_relations / max(1, total_active_relations))

STEP 4: overall
  overall = weighted * health

STEP 5: compute alerts
  alerts = [qa for qa in QA_RULES if qa.triggered(record)]

RETURN: QualityScoreRecord
```

---

## QD-1: Completeness Formula

```
required = base_required_fields + type_specific_required_fields

completeness = count(f for f in required if is_non_empty(f)) / len(required)

is_non_empty:
  str   → len(strip()) > 0
  list  → len > 0
  float → value is not None
  bool  → always non-empty (True/False both count)
  None  → empty
```

**Base required fields (8):** `knowledge_id, object_type, name, version, status, owner, description, tags`

**Type-specific required fields by type:**

| Type | Extra required |
|------|---------------|
| MODULE | `module_id`, `runtime_id`, `python_module` |
| SERVICE | `service_id`, `interface_path` |
| REQUIREMENT | `requirement_id`, `source` |
| DECISION | `decision_id`, `context`, `chosen_option`, `rationale` |
| API | (none — all fields have defaults) |
| BENCHMARK | `benchmark_id`, `operation` |
| PROVIDER | `provider_name` |
| AGENT | `agent_id`, `agent_type` |
| MARKET | `market_id`, `exchange` |
| DATASET | `dataset_id`, `provenance` |

---

## QD-2: Consistency Formula

```
checks = [
  C-01: knowledge_id matches regex
  C-02: version matches semver
  C-03: status is valid enum value
  C-04: confidence.declared in [0.0, 1.0]
  C-05: canonical_name matches computed name (if set)
  C-06: knowledge_uri matches computed URI (if set)
  C-07: no self-reference in traceability.satisfies
  C-08: no self-reference in traceability.implements
]

consistency = count(c for c in checks if c.passes(object)) / len(checks)
```

---

## QD-3: Coverage Formula (type-weighted)

```
weights_by_type = {
  REQUIREMENT:    {satisfies: 0.0, implements: 0.5, tests: 0.5},
  ARCHITECTURE:   {satisfies: 0.5, implements: 0.2, tests: 0.3},
  MODULE:         {satisfies: 0.3, implements: 0.4, tests: 0.3},
  SERVICE:        {satisfies: 0.3, implements: 0.4, tests: 0.3},
  DECISION:       {satisfies: 0.4, implements: 0.2, tests: 0.4},
  TEST:           {satisfies: 0.0, implements: 0.0, tests: 1.0},  # tests must test something
  BENCHMARK:      {satisfies: 0.5, implements: 0.0, tests: 0.5},
  DEFAULT:        {satisfies: 0.33, implements: 0.33, tests: 0.34},
}

w = weights_by_type.get(object.type, weights_by_type["DEFAULT"])
has_satisfies    = 1.0 if object.traceability.satisfies else 0.0
has_implements   = 1.0 if object.traceability.implements else 0.0
has_tests        = 1.0 if object.traceability.tests else 0.0

coverage = w["satisfies"] * has_satisfies
         + w["implements"] * has_implements
         + w["tests"] * has_tests
```

---

## QD-4: Freshness Formula

```
age_days = (now - object.metadata.updated_at).days
threshold_days = 180   # 6 months

if age_days <= threshold_days:
  freshness = 1.0
elif age_days <= 365:
  freshness = 1.0 - ((age_days - threshold_days) / (365 - threshold_days)) * 0.5
else:
  freshness = max(0.0, 0.5 - ((age_days - 365) / 365) * 0.5)
```

---

## QD-5: Evidence Score

```
evidence_score = object.evidence.evidence_score
```

Computed by Evidence Engine (A-05 freshness × type weight × count factor):
```
evidence_score = sum(
  e.weight * freshness(e) for e in object.evidence.items
) / max(1, len(object.evidence.items))  *  count_bonus
```

Where `count_bonus = min(1.0, 0.5 + 0.1 * len(items))`.

---

## QD-6: Confidence Score

```
confidence_score = object.confidence.declared
```

If `confidence.composite` is set (by Confidence Engine), use that:
```
confidence_score = object.confidence.composite or object.confidence.declared
```

---

## QD-7: Usage Score

```
inbound_refs = count of other objects that reference this one
usage = min(1.0, inbound_refs / 10.0)
```

Special case: `object_type in [ARCHITECTURE, REQUIREMENT]` → threshold is 5 (harder to use).

---

## QD-8: Reasoning Score

```
if object.type in [DECISION, REQUIREMENT]:
  has_context  = len(object.context.strip()) >= 10
  has_criteria = len(object.acceptance_criteria or object.rationale or "") > 0
  reasoning = 1.0 if (has_context and has_criteria) else 0.5 if has_context else 0.0
else:
  reasoning = 1.0
```

---

## QD-9: Health Penalty

```
broken = count active relations where source or target is ARCHIVED
total  = count all ACTIVE relations
penalty = 1.0 - (broken / max(1, total))

if broken == 0:
  health_penalty = 1.0
```

---

## Score Summary Formula

```python
def compute_score(obj, registry) -> QualityScoreRecord:
    dims = [
        completeness(obj),
        consistency(obj),
        coverage(obj, registry),
        freshness(obj),
        evidence_score(obj),
        confidence_score(obj),
        usage(obj, registry),
        reasoning(obj),
    ]
    weights = [0.15, 0.15, 0.20, 0.10, 0.15, 0.10, 0.10, 0.05]
    assert sum(weights) == 1.0

    weighted = sum(d * w for d, w in zip(dims, weights))
    penalty = health_penalty(obj, registry)
    overall = weighted * penalty

    return QualityScoreRecord(dims=dims, weighted=weighted, overall=overall)
```

---

## Cross-References

- Dimension weights from → Phase 3.0C `11-QUALITY-ENGINE`
- Quality gates using these scores → `16-KNOWLEDGE-QUALITY`
- Algorithm A-04 definition → Phase 3.0C `36-ALGORITHMS`
- DS-14 QualityScoreRecord → Phase 3.0C `35-DATA-STRUCTURES`
