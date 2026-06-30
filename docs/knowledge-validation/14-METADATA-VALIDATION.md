# KVF-DOC-014 — Metadata Validation

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Metadata Validation verifies the completeness, accuracy, and consistency of
all metadata fields across the corpus. Poor metadata directly degrades search
quality, quality scoring, and AI readiness assessment.

---

## Metadata Field Categories

```
MV-CAT-1: Core Identity Fields
  knowledge_id, canonical_name, object_type, namespace, version

MV-CAT-2: Lifecycle Fields
  metadata.state, metadata.created_at, metadata.updated_at,
  metadata.author, metadata.reviewers

MV-CAT-3: Quality Fields
  metadata.quality_score, metadata.lint_issues,
  metadata.last_validated_at

MV-CAT-4: AI Intelligence Fields
  intelligence.ai_readiness_score, intelligence.confidence.overall_score,
  intelligence.confidence.level

MV-CAT-5: Search Fields
  intelligence.search.keyword.primary_terms,
  intelligence.search.vector.embedding_model,
  intelligence.search.vector.last_updated
```

---

## Metadata Completeness Checks

```
For each CANONICAL or ACTIVE object:

MV-1.01: knowledge_id is present and matches pattern
MV-1.02: canonical_name is non-empty and unique
MV-1.03: object_type is from declared vocabulary
MV-1.04: namespace is from declared namespaces
MV-1.05: version is valid semver

MV-2.01: metadata.state is one of 6 valid states
MV-2.02: metadata.created_at is present and valid ISO 8601
MV-2.03: metadata.updated_at is >= created_at
MV-2.04: metadata.author is present (may be "system")

MV-3.01: metadata.quality_score is in [0.0, 1.0]
MV-3.02: quality_score equals the computed formula value (±0.01 tolerance)
MV-3.03: metadata.last_validated_at is present for CANONICAL objects

MV-4.01: intelligence.ai_readiness_score is in [0.0, 1.0] (if KIL block present)
MV-4.02: intelligence.confidence.overall_score is in [0.0, 1.0]
MV-4.03: confidence.level matches score boundary

MV-5.01: search.keyword.primary_terms is non-empty
MV-5.02: search.vector.embedding_model is specified
MV-5.03: search.vector.last_updated is <= object.metadata.updated_at + 24h
```

---

## Metadata Staleness Check

```
For each CANONICAL/ACTIVE object:

  staleness_days = (now - metadata.updated_at).days

  ALERT if staleness_days > 180 (6 months without update)
  ERROR if staleness_days > 365 (1 year)

  Vector embedding staleness:
  ALERT if search.vector.last_updated < metadata.updated_at - 24h
  ERROR if gap > 7 days (embedding was not refreshed after object update)
```

---

## Quality Score Accuracy Check

```
For each object with quality_score:

  recomputed = compute_quality_score(object)  # using Phase 3.0B formula
  declared = object.metadata.quality_score

  accuracy = 1 - abs(recomputed - declared)
  Pass if: abs(recomputed - declared) <= 0.01

  quality_score_accuracy = count(pass) / count(objects)
  Target: >= 0.98
```

---

## Metadata Validation Result Schema

```yaml
metadata_validation_result:
  objects_checked: integer

  completeness:
    by_category:
      - category: string              # MV-CAT-1 through MV-CAT-5
        fields_required: integer
        fields_present: integer
        completeness_rate: float
    overall_completeness_rate: float

  staleness:
    stale_objects_6mo: integer
    stale_objects_12mo: integer
    stale_embeddings: integer

  quality_score_accuracy: float
  quality_score_out_of_tolerance: integer

  confidence_level_accuracy: float    # does level match score?

  critical_failures:
    - type: string                    # e.g., MISSING_KNOWLEDGE_ID
      count: integer
      sample_ids: [string]
```

---

## Metadata Targets

| Metric | BRONZE | GOLD |
|--------|--------|------|
| Core identity completeness | ≥ 0.98 | 1.00 |
| Lifecycle completeness | ≥ 0.95 | ≥ 0.99 |
| Quality field accuracy | ≥ 0.90 | ≥ 0.98 |
| Stale objects (>6mo) | < 20% | < 5% |
| Embedding staleness (>7d) | < 10% | < 1% |

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-066 | Missing knowledge_id or canonical_name is always a critical failure |
| KVF-067 | Quality score recomputation must use the Phase 3.0B formula — no shortcuts |
| KVF-068 | Embedding staleness check must compare update timestamps, not just check presence |
| KVF-069 | Stale objects > 20% is a SILVER-blocking issue — indicates maintenance failure |
| KVF-070 | Confidence level accuracy < 0.95 triggers KIL confidence block review |

---

## Cross-References

- Knowledge conformance → `02-KNOWLEDGE-CONFORMANCE`
- KIL quality model → Phase 3.0D.0.6 `14-KNOWLEDGE-QUALITY-MODEL`
- KOS quality model → Phase 3.0B documentation
