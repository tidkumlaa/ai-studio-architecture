# KVF-DOC-002 — Knowledge Conformance

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Knowledge Conformance validates that every KnowledgeObject in the corpus
conforms to the KOS schema v1.0 and the KIL intelligence extension schema.
This is the foundation check — an implementation that fails conformance
fails everything.

---

## Conformance Check Groups

```
CG-1: Identity Conformance (KOS Core)
CG-2: State Machine Conformance
CG-3: Quality Score Conformance
CG-4: Relationship Conformance
CG-5: Intelligence Block Conformance (KIL)
CG-6: DNA / Genome Conformance (KIL)
CG-7: Compression Block Conformance (KIL)
CG-8: Index Entry Conformance
```

---

## CG-1: Identity Conformance

```
For every KnowledgeObject in corpus:

  KC-1.01: knowledge_id matches pattern /^KNW-[A-Z]{3}-[A-Z]{3}-\d{3}$/
  KC-1.02: canonical_name is non-empty string
  KC-1.03: object_type is one of declared KOS types
  KC-1.04: namespace exists in namespace registry
  KC-1.05: version matches semver format
  KC-1.06: metadata.created_at is valid ISO 8601
  KC-1.07: metadata.updated_at >= metadata.created_at
  KC-1.08: metadata.quality_score is float in [0.0, 1.0]
```

---

## CG-2: State Machine Conformance

```
  KC-2.01: metadata.state is one of {DRAFT, EXPERIMENTAL, ACTIVE, CANONICAL, DEPRECATED, ARCHIVED}
  KC-2.02: No ARCHIVED object has state transition after archive date
  KC-2.03: CANONICAL objects have quality_score >= 0.80
  KC-2.04: DEPRECATED objects have a deprecation date set
  KC-2.05: State transitions follow the declared state machine (Phase 3.0C)
```

---

## CG-3: Quality Score Conformance

```
  KC-3.01: metadata.quality_score equals the computed KOS quality formula
  KC-3.02: All quality sub-dimensions (Phase 3.0B) are present
  KC-3.03: kil_overall_score (if KIL present) >= 0.80 for CANONICAL
  KC-3.04: quality_score == metadata.quality_score (KIL cross-check: KIL-086)
```

---

## CG-4: Relationship Conformance

```
  KC-4.01: All relationship.target_id values exist in the corpus
  KC-4.02: Relationship types are from declared KOS vocabulary
  KC-4.03: HARD DEPENDS_ON relationships have no cycles (acyclic check)
  KC-4.04: Bidirectional relationships are declared on both sides
  KC-4.05: Relationship counts match index entry relationship_count
```

---

## CG-5: Intelligence Block Conformance (KIL)

```
  KC-5.01: If intelligence: block present, all 20 sub-blocks have correct structure
  KC-5.02: Rules KIL-001 through KIL-149 — all applicable rules pass
  KC-5.03: intelligence.ai_readiness_score is in [0.0, 1.0]
  KC-5.04: AIRS level matches score boundary (KIL-135):
    AIRS >= 0.90 → AI-NATIVE; >= 0.80 → AI-READY; etc.
  KC-5.05: All 10 compression levels (L1–L5 + nano/micro/mini/standard/machine) have content
```

---

## CG-6: DNA / Genome Conformance

```
  KC-6.01: dna.dna_hash == sha256(dna.identity + dna.purpose + dna.core_behavior
                                   + dna.primary_interfaces + dna.fundamental_constraints)
  KC-6.02: CANONICAL objects have sealed DNA (dna.sealed_at is set)
  KC-6.03: genome.genome_fingerprint == sha256(all 8 genome fields)
  KC-6.04: No DNA field change on CANONICAL without new object creation
```

---

## CG-7: Compression Block Conformance

```
  KC-7.01: compression.nano token count <= 15
  KC-7.02: compression.micro token count <= 50
  KC-7.03: compression.mini token count <= 200
  KC-7.04: compression.standard token count <= 500
  KC-7.05: compression.machine token count <= 2000
  KC-7.06: compression.machine content is valid JSON
  KC-7.07: Higher compression levels contain superset of lower level content
```

---

## CG-8: Index Entry Conformance

```
  KC-8.01: Every CANONICAL/ACTIVE object has entry in all 10 KIL indexes (IDX-001–IDX-010)
  KC-8.02: Index entry knowledge_id matches object knowledge_id
  KC-8.03: Index entry quality_score matches object metadata.quality_score
  KC-8.04: Index entry ai_readiness_score matches object intelligence.ai_readiness_score
  KC-8.05: Search index has populated embedding vector for objects with AIRS >= 0.70
```

---

## Conformance Result Schema

```yaml
conformance_result:
  total_objects_checked: integer
  total_checks_run: integer            # objects × applicable checks

  by_group:
    - group: string                    # CG-1 through CG-8
      checks_run: integer
      checks_passed: integer
      critical_failures: integer
      failure_rate: float

  object_conformance_distribution:
    fully_conformant: integer           # 0 failures
    minor_violations: integer          # 1–3 failures, none critical
    major_violations: integer          # 4+ failures or any critical
    non_conformant: integer            # critical CG-1 or CG-2 failure

  conformance_score: float             # checks_passed / checks_run
  critical_failure_count: integer

  sample_violations:
    - knowledge_id: string
      check_id: string
      expected: string
      actual: string
```

---

## Conformance Score Thresholds

| Score | Level |
|-------|-------|
| ≥ 0.99 | CONFORMANT — eligible for Gold+ certification |
| ≥ 0.95 | MOSTLY CONFORMANT — eligible for Silver |
| ≥ 0.90 | PARTIALLY CONFORMANT — eligible for Bronze |
| < 0.90 | NON-CONFORMANT — certification blocked |

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-006 | CG-1 failures are critical — any CG-1 failure means the object is non-conformant |
| KVF-007 | CG-6 DNA hash check must be cryptographically verified, not assumed |
| KVF-008 | KC-7.06 (machine JSON validity) must use strict JSON parse, not lenient |
| KVF-009 | CG-4 cycle detection must check the full transitive closure, not direct edges only |
| KVF-010 | Conformance score must be computed per-object, then aggregated — not per-check across all objects |

---

## Cross-References

- KOS schema → Phase 3.0A core documentation
- KIL rules → Phase 3.0D.0.6 `30-KNOWLEDGE-FREEZE.md`
- Runtime conformance → `03-RUNTIME-CONFORMANCE`
- Certification scoring → `27-CERTIFICATION-SCORING`
