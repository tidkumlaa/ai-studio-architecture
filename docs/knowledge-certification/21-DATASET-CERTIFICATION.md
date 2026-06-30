# KNW-CERT-ARCH-021 — Dataset Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the certification dataset itself is complete, consistent, correctly structured, and covers all object types and relationship types required by the certification suite.

---

## Dataset Requirements

The certification suite requires 5 datasets:

| Dataset | Objects | Relationships | Used by |
|---------|---------|---------------|---------|
| Golden | 200 curated objects | 500 relationships | Search, Quality, Reasoning |
| Regression | 50 fixed objects (subset) | 200 relationships | Regression suite |
| Performance | Synthetic 1K/10K/100K/1M | Generated | Performance suite |
| Traceability | 500 requirements with chains | — | Traceability suite |
| Adversarial | 100 malformed objects | 50 broken relationships | Security suite |

---

## Golden Dataset Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Exactly 200 curated objects | DC-001 | CRITICAL | Count = 200 |
| Exactly 500 curated relationships | DC-002 | CRITICAL | Count = 500 |
| All 33 object types represented | DC-003 | MAJOR | Each type has ≥ 1 object |
| All 10 relationship types represented | DC-004 | MAJOR | Each type has ≥ 1 relationship |
| All 9 packages represented | DC-005 | MAJOR | Each namespace has ≥ 1 object |
| All lifecycle states represented | DC-006 | MINOR | DRAFT, PROPOSED, VERIFIED, CANONICAL, DEPRECATED, ARCHIVED |
| All objects schema-valid | DC-007 | CRITICAL | Zero validation failures |
| All objects pass `kos lint` | DC-008 | CRITICAL | Zero lint failures |
| Checksum for each object stored | DC-009 | MAJOR | 200 checksums in dataset manifest |
| Ground-truth answers pre-computed | DC-010 | CRITICAL | Search and reasoning ground-truth files exist |

---

## Regression Dataset Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Regression dataset is a subset of Golden | DC-011 | CRITICAL | All 50 IDs exist in Golden |
| Regression dataset version-pinned | DC-012 | CRITICAL | Hash matches declared version |
| Regression results pre-computed | DC-013 | CRITICAL | Expected results file present for each check |

---

## Performance Dataset Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| 1K dataset generates correctly | DC-014 | MAJOR | Deterministic with seed 42 |
| 10K dataset generates correctly | DC-015 | MAJOR | |
| 100K dataset generates correctly | DC-016 | MAJOR | |
| 1M dataset generates correctly | DC-017 | MINOR | |
| Relationship ratio maintained | DC-018 | MINOR | ≈ 5 relationships per object |

---

## Traceability Dataset Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| 500 requirements present | DC-019 | CRITICAL | Count = 500 |
| Every requirement has known chain | DC-020 | CRITICAL | Ground-truth chain stored |
| Chains span all 9 levels | DC-021 | MAJOR | Each level represented |
| 50 incomplete chains for gap testing | DC-022 | MAJOR | Known-broken chains present |

---

## Adversarial Dataset Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| 100 malformed objects | DC-023 | CRITICAL | Count = 100 |
| Covers all 7 security attack types | DC-024 | MAJOR | Each attack type has ≥ 10 examples |
| Expected error type documented | DC-025 | CRITICAL | Error mapping file present |

---

## Dataset Manifest Format

```yaml
# benchmarks/kos/datasets/manifest.yaml
version: "1.0.0"
created_at: "2026-06-30T00:00:00Z"
seed: 42

datasets:
  golden:
    path: "benchmarks/kos/datasets/golden/"
    objects: 200
    relationships: 500
    checksum: "sha256:abc123"
    ground_truth: "benchmarks/kos/datasets/golden/ground-truth/"

  regression:
    path: "benchmarks/kos/datasets/regression/"
    objects: 50
    relationships: 200
    checksum: "sha256:def456"
    expected_results: "benchmarks/kos/datasets/regression/expected/"

  performance:
    sizes: [1000, 10000, 100000, 1000000]
    seed: 42
    relationship_ratio: 5

  traceability:
    path: "benchmarks/kos/datasets/traceability/"
    requirements: 500
    complete_chains: 450
    incomplete_chains: 50
    checksum: "sha256:ghi789"

  adversarial:
    path: "benchmarks/kos/datasets/adversarial/"
    malformed_objects: 100
    broken_relationships: 50
    attack_types: 7
    checksum: "sha256:jkl012"
```

---

## Report Format

```json
{
  "domain": "dataset",
  "checks": {
    "DC-001": "PASS", "DC-007": "PASS", "DC-008": "PASS",
    "DC-010": "PASS", "DC-019": "PASS", "DC-023": "PASS"
  },
  "golden_valid": true,
  "regression_pinned": true,
  "adversarial_coverage": 1.0,
  "domain_score": 0.971,
  "level_achieved": "Enterprise"
}
```

---

## CLI

```bash
kos-cert dataset                         # verify all datasets
kos-cert dataset --type golden           # golden dataset only
kos-cert dataset --type performance --size 1M
kos-cert dataset --generate --seed 42    # regenerate all datasets
kos-cert dataset --output reports/dataset.json
```

---

## Cross-References

- Golden dataset spec → Phase 3.0C.5 `12-KNOWLEDGE-GOLDEN-DATASET`
- Benchmark objects → Phase 3.0C.5 `13-KNOWLEDGE-BENCHMARKS`
- Adversarial inputs → `18-SECURITY-CERTIFICATION`
