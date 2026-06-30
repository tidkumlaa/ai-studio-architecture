# KNW-FINAL-018 — Knowledge Quality Gate

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines the KOS v1.0 quality gate — the complete, non-negotiable set of requirements that must all be satisfied before the knowledge base is considered complete for any certification level.

---

## Quality Gate Dimensions

The quality gate has 9 dimensions. All must reach 100% at Gold level.

| Dimension | Target (Gold) | Metric |
|-----------|--------------|--------|
| Repository Coverage | 100% | All 9 namespaces populated |
| Metadata Completeness | 100% | All CANONICAL objects have all required fields |
| Relationship Coverage | 100% | All bidirectional links verified |
| Traceability Coverage | 100% | All CANONICAL objects traced |
| Evidence Coverage | 100% | All CANONICAL objects have ≥1 evidence item |
| Namespace Coverage | 100% | All 9 namespaces declared and used |
| Package Coverage | 100% | All 9 packages installed and valid |
| Template Coverage | 100% | All 33 types have a canonical template |
| Acceptance Tests | 100% | All 10 KATs pass at declared level |

---

## Dimension 1: Repository Coverage

All 9 namespaces must have ≥ 5 CANONICAL objects:

| Namespace | Min CANONICAL | Status check |
|-----------|---------------|-------------|
| plt | ≥ 5 | `kos query --namespace plt --state CANONICAL` |
| rt | ≥ 5 | |
| prov | ≥ 5 | |
| alg | ≥ 5 | |
| pat | ≥ 5 | |
| api | ≥ 5 | |
| test | ≥ 5 | |
| fin | ≥ 5 | |
| meta | ≥ 5 | |

---

## Dimension 2: Metadata Completeness

For every CANONICAL object, all required fields must be non-null:

```
Required fields for CANONICAL:
  knowledge_id, object_type, canonical_name, object_uri,
  name, version, namespace, domain,
  lifecycle.state = CANONICAL,
  lifecycle.version,
  ownership.owner,
  description,
  tags (≥1),
  evidence.items (≥1)
```

Pass: Zero CANONICAL objects with any required field missing.

---

## Dimension 3: Relationship Coverage

For every object that has relationships:
- All referenced IDs exist in registry
- All relationships are bidirectional

Pass: Zero dangling references. Zero one-directional relationships.

---

## Dimension 4: Traceability Coverage

For CANONICAL MODULE, SERVICE, and REQUIREMENT objects:
- Must have `traceability.satisfies` OR `traceability.tested_by` populated
- All cited IDs must exist

Pass: Zero CANONICAL modules with no traceability links.

---

## Dimension 5: Evidence Coverage

For every CANONICAL object:
- Must have ≥ 1 evidence item
- Must have ≥ 1 Level 1–4 evidence item (not all Level 5–6)
- No evidence item past freshness threshold without flag

Pass: Zero CANONICAL objects with evidence_score = 0.

---

## Dimension 6: Namespace Coverage

All 9 namespaces declared in workspace.yaml and installed.

Pass: `kos list` shows all 9 packages; zero unknown namespaces.

---

## Dimension 7: Package Coverage

All 9 packages:
- Have valid package.yaml
- Pass `kos verify {package}`
- Quality gate `min_overall_score` met
- Lock file up to date

Pass: `kos verify --all` returns zero errors.

---

## Dimension 8: Template Coverage

All 33 KnowledgeObjectTypes have:
- A template file in `knowledge/templates/{type}.template.yaml`
- Template validates against base-schema.json
- Template used to generate at least 1 example in golden dataset

Pass: 33 templates present and valid.

---

## Dimension 9: Acceptance Tests

All 10 KAT tests pass at the declared certification level:

| Level | Required KAT pass rate |
|-------|----------------------|
| Bronze | ≥ 0.70 on all 10 KATs |
| Silver | ≥ 0.78 on all 10 KATs |
| Gold | ≥ 0.86 on all 10 KATs |
| Enterprise | ≥ 0.93 on all 10 KATs |
| Research | ≥ 0.97 on all 10 KATs |

---

## Quality Gate Report Format

```json
{
  "quality_gate": "KOS-v1.0",
  "timestamp": "2026-06-30T00:00:00Z",
  "dimensions": {
    "repository_coverage": {
      "pass": true,
      "namespaces_populated": 9,
      "min_canonical_met": true
    },
    "metadata_completeness": {
      "pass": true,
      "canonical_objects": 200,
      "missing_required_fields": 0
    },
    "relationship_coverage": {
      "pass": true,
      "dangling_references": 0,
      "unidirectional_relationships": 0
    },
    "traceability_coverage": {
      "pass": false,
      "canonical_modules": 42,
      "modules_with_traceability": 39,
      "rate": 0.929
    },
    "evidence_coverage": {
      "pass": true,
      "canonical_objects": 200,
      "zero_evidence_count": 0
    },
    "namespace_coverage": {"pass": true, "namespaces": 9},
    "package_coverage": {"pass": true, "packages": 9},
    "template_coverage": {"pass": true, "templates": 33},
    "acceptance_tests": {
      "pass": false,
      "kat_results": {
        "KAT-001": 0.89, "KAT-002": 0.87, "KAT-003": 0.82,
        "KAT-004": 0.91, "KAT-005": 0.88, "KAT-006": 0.84,
        "KAT-007": 0.86, "KAT-008": 0.85, "KAT-009": 0.87,
        "KAT-010": 0.83
      },
      "min_rate": 0.82,
      "required": 0.86
    }
  },
  "gate_pass": false,
  "blocking_dimension": "traceability_coverage",
  "overall_quality_score": 0.886
}
```

---

## CLI

```bash
kos quality gate                         # run all 9 dimensions
kos quality gate --dimension evidence    # single dimension
kos quality gate --report reports/quality-gate.json
kos quality gate --level gold            # check against Gold targets
```

---

## Cross-References

- Quality dimensions → Phase 3.0C.5 `16-KNOWLEDGE-QUALITY`
- Acceptance tests → `17-KNOWLEDGE-ACCEPTANCE-TEST`
- Certification scoring → Phase 3.0D.0 `27-OVERALL-SCORING`
- Coverage certification → Phase 3.0D.0 `26-COVERAGE-CERTIFICATION`
