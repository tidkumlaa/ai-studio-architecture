# KNW-KE-ARCH-020 — Knowledge Verification

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Knowledge Verification suite checks that the entire ecosystem is internally consistent, complete, and correct. All checks are automated and machine-executable.

---

## Verification Command

```bash
kos verify                          # run all checks
kos verify --suite schema           # schema checks only
kos verify --suite references       # reference integrity
kos verify --suite packages         # package rules
kos verify --suite golden-dataset   # golden dataset
kos verify --check KV-015           # single check
kos verify --output reports/        # write report
```

---

## Verification Checks

### Suite 1: Schema Verification (KV-001–KV-010)

| Check | Description | Pass Condition |
|-------|-------------|----------------|
| KV-001 | All YAML files are valid YAML | Parse succeeds |
| KV-002 | All object files validate against `base-schema.json` | Zero schema errors |
| KV-003 | Each object file validates against its type-specific schema | Zero type errors |
| KV-004 | All schema files are valid JSON Schema (draft-2020-12) | Meta-validate passes |
| KV-005 | Schema registry lists all 33 object types | Count == 33 |
| KV-006 | All `object_type` values match KnowledgeObjectType enum | Zero mismatches |
| KV-007 | No unknown fields in any object file | Zero unknown fields |
| KV-008 | `knowledge_id` is unique across the entire repository | Zero duplicates |
| KV-009 | `canonical_name` is unique within each namespace | Zero namespace duplicates |
| KV-010 | `knowledge_uri` is unique across the repository | Zero URI duplicates |

---

### Suite 2: Reference Verification (KV-011–KV-020)

| Check | Description | Pass Condition |
|-------|-------------|----------------|
| KV-011 | All `traceability.satisfies` IDs exist in repository | Zero dangling refs |
| KV-012 | All `traceability.implements` IDs exist in repository | Zero dangling refs |
| KV-013 | All `traceability.tests` IDs exist in repository | Zero dangling refs |
| KV-014 | All relationship `source_id` values exist | Zero dangling sources |
| KV-015 | All relationship `target_id` values exist | Zero dangling targets |
| KV-016 | Relationship types are all valid (from 24 canonical types) | Zero invalid types |
| KV-017 | No cyclic DEPENDS_ON relationships | Zero cycles |
| KV-018 | No object's `traceability.satisfies` contains its own knowledge_id | Zero self-refs |
| KV-019 | Alias targets exist in registry | Zero broken aliases |
| KV-020 | Cross-reference map matches actual relationships | Zero inconsistencies |

---

### Suite 3: Package Verification (KV-021–KV-030)

| Check | Description | Pass Condition |
|-------|-------------|----------------|
| KV-021 | All objects in a package have matching domain_code in `knowledge_id` | Zero mismatches |
| KV-022 | Package `namespace` values are globally unique | Zero duplicates |
| KV-023 | Package `domain_code` values are globally unique | Zero duplicates |
| KV-024 | No object belongs to two packages | Zero double-registrations |
| KV-025 | All package `dependencies` exist in package registry | Zero missing deps |
| KV-026 | No circular package dependencies | Zero cycles |
| KV-027 | All packages have a `package.yaml` at root | Count == 9 |
| KV-028 | All packages have a `docs/README.md` | Zero missing READMEs |
| KV-029 | Lock file checksums match dependency packages | Zero checksum failures |
| KV-030 | Package `object_types` list matches actual object types found | Zero mismatches |

---

### Suite 4: Golden Dataset Verification (KV-031–KV-040)

| Check | Description | Pass Condition |
|-------|-------------|----------------|
| KV-031 | All golden objects pass `kos lint` with zero errors | Zero lint errors |
| KV-032 | All golden objects have `status` ≥ VERIFIED | Zero below-VERIFIED |
| KV-033 | All golden objects have `quality.overall_score` ≥ 0.75 | Zero below-threshold |
| KV-034 | All golden relationships reference existing golden objects | Zero dangling |
| KV-035 | Object count per type matches `12-GOLDEN-DATASET` specification | Exact match |
| KV-036 | Regression dataset objects still cause expected linter failures | All regressions fire |
| KV-037 | Benchmark objects cover all 19 algorithms | Count == 19 |
| KV-038 | AI eval dataset covers all quality score bands [0.1–0.9] in 0.2 steps | 5 bands covered |
| KV-039 | Performance dataset generation is reproducible (same seed → same output) | Checksum match |
| KV-040 | Golden dataset passes `kos verify --suite references` | Zero violations |

---

### Suite 5: Repository Verification (KV-041–KV-050)

| Check | Description | Pass Condition |
|-------|-------------|----------------|
| KV-041 | `knowledge/registry/index.yaml` lists all objects in packages | Count match |
| KV-042 | `knowledge/registry/fingerprints.yaml` has no duplicates | Zero duplicate fingerprints |
| KV-043 | `knowledge/catalog/catalog.yaml` is up to date with repository | Max 1h stale |
| KV-044 | Linter config lists no disabled rules (unless explicitly justified) | Zero unjustified disabled |
| KV-045 | All 32 linter rules have a rule YAML file in `lint/rules/` | Count == 32 |
| KV-046 | All 33 schemas exist in `schemas/by-type/` | Count == 33 |
| KV-047 | All 9 packages have entries in `knowledge/registry/index.yaml` | Count == 9 |
| KV-048 | SDK contract files exist for Python, Java, TypeScript | 3 files exist |
| KV-049 | `knowledge/reports/` has at least one quality report | At least 1 file |
| KV-050 | No DRAFT objects in `knowledge/reference/` | Zero DRAFT in reference |

---

## Verification Report

```yaml
# knowledge/reports/verification-{date}.yaml
run_date: "2026-06-30T00:00:00Z"
total_checks: 50
passed: 50
failed: 0
errors: 0

suites:
  schema:
    checks: 10
    passed: 10
    failed: 0

  references:
    checks: 10
    passed: 10
    failed: 0

  packages:
    checks: 10
    passed: 10
    failed: 0

  golden-dataset:
    checks: 10
    passed: 10
    failed: 0

  repository:
    checks: 10
    passed: 10
    failed: 0

failures: []
```

---

## Cross-References

- Linter checks (subset of verification) → `06-KNOWLEDGE-LINTER`
- Architecture verification → `21-KNOWLEDGE-ARCHITECTURE-TESTS`
- CI runs verification → `08-KNOWLEDGE-CI`
- Golden dataset specification → `12-KNOWLEDGE-GOLDEN-DATASET`
