# KNW-CERT-ARCH-006 — Metadata Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that all metadata fields on Knowledge Objects are correctly structured, validated, indexed, and queryable.

---

## Metadata Field Checks

### Identity Metadata

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| `knowledge_id` format valid for all objects | MC-001 | CRITICAL | All match `^KNW-[A-Z][A-Z0-9]*-[A-Z][A-Z0-9]*-[A-Za-z0-9_-]+$` |
| `canonical_name` format valid | MC-002 | CRITICAL | All match `^[a-z][a-z0-9]*\.[a-z][a-z0-9]*\.[a-z][a-z0-9-]*$` |
| `object_uri` format valid | MC-003 | MAJOR | All match `knw://{ns}/{type}/{slug}@{version}` |
| `knowledge_id` is unique across all objects | MC-004 | CRITICAL | Zero duplicates |
| `canonical_name` is unique per namespace | MC-005 | CRITICAL | Zero duplicates within namespace |

### Classification Metadata

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| `object_type` is one of 33 canonical types | MC-006 | CRITICAL | Zero unknown types |
| `namespace` matches package namespace | MC-007 | MAJOR | All match declared package namespace |
| `domain` matches `knowledge_id` prefix | MC-008 | MAJOR | All consistent |
| All tags in canonical tag registry | MC-009 | MINOR | Zero unknown tags |
| Labels are valid key:value format | MC-010 | MINOR | Zero malformed labels |

### Lifecycle Metadata

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| `lifecycle.state` is valid enum value | MC-011 | CRITICAL | All in {DRAFT, PROPOSED, VERIFIED, CANONICAL, DEPRECATED, ARCHIVED} |
| `lifecycle.version` is semver | MC-012 | MAJOR | All match `\d+\.\d+\.\d+` |
| `created_at` ≤ `updated_at` | MC-013 | MAJOR | No objects with updated < created |
| `deprecated_at` present when state=DEPRECATED | MC-014 | MAJOR | |
| `successor_id` present when deprecated | MC-015 | MINOR | |

### Ownership Metadata

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| `owner` present for VERIFIED+ objects | MC-016 | MAJOR | Zero VERIFIED/CANONICAL with no owner |
| `owner` format valid (`team:X` or `user:X`) | MC-017 | MAJOR | |

### Operational Metadata

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| `quality_score` ∈ [0.0, 1.0] | MC-018 | MAJOR | |
| `checksum` present and SHA-256 format | MC-019 | MAJOR | All match `sha256:[0-9a-f]{64}` |
| Checksum verifies (recompute and compare) | MC-020 | CRITICAL | Zero checksum mismatches |

---

## Search Index Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| All indexed fields searchable | MC-021 | MAJOR | Every indexed field returns correct results |
| Range query on `quality_score` correct | MC-022 | MINOR | Quality range filter returns exact set |
| Multi-value filter on `tags` correct | MC-023 | MINOR | Tag filter returns exact set |

---

## Dataset

Metadata certification runs on all objects in the golden dataset (200 objects) plus the full installed packages (all CANONICAL objects in workspace).

---

## Metrics

| Metric | Bronze | Silver | Gold |
|--------|--------|--------|------|
| Field validation pass rate | ≥ 0.95 | ≥ 0.98 | = 1.0 |
| Zero CRITICAL check failures | required | required | required |
| Checksum verification pass rate | ≥ 0.99 | = 1.0 | = 1.0 |
| Index correctness | ≥ 0.95 | ≥ 0.98 | = 1.0 |

---

## Report Format

```json
{
  "domain": "metadata",
  "objects_tested": 200,
  "checks": {
    "MC-001": {"status": "PASS", "failures": 0},
    "MC-004": {"status": "PASS", "failures": 0},
    "MC-020": {"status": "FAIL", "failures": 3, "ids": ["KNW-PLT-MOD-007", "..."]}
  },
  "field_validation_pass_rate": 0.985,
  "checksum_pass_rate": 0.985,
  "critical_pass": false,
  "domain_score": 0.931,
  "level_achieved": "Bronze"
}
```

---

## CLI

```bash
kos-cert metadata                        # all objects in workspace
kos-cert metadata --package kos.platform.package
kos-cert metadata --check MC-020        # single check
kos-cert metadata --output reports/metadata.json
```

---

## Cross-References

- Metadata field spec → Phase 3.0C.5 `28-KNOWLEDGE-METADATA`
- Naming rules → Phase 3.0C.5 `05-KNOWLEDGE-NAMING-STANDARD`
- Linter rules KL-001–KL-005 → Phase 3.0C.5 `06-KNOWLEDGE-LINTER`
