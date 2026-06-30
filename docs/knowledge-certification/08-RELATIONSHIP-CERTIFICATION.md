# KNW-CERT-ARCH-008 — Relationship Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the relationship engine correctly stores, validates, traverses, and enforces bidirectionality for all relationship types.

---

## Relationship Types Under Test

All relationship types from Phase 3.0C `06-RELATIONSHIP-ENGINE`:

| Type | Direction | Checked |
|------|-----------|---------|
| DEPENDS_ON | directed | Yes |
| IMPLEMENTS | directed | Yes |
| USES | directed | Yes |
| EXTENDS | directed | Yes |
| TESTS | directed | Yes |
| PROVIDES | directed | Yes |
| RELATED_TO | undirected | Yes |
| REFERENCES | directed | Yes |
| SUPERSEDES | directed | Yes |
| OWNED_BY | directed | Yes |

---

## Structural Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Every relationship has source + target + type | RLC-001 | CRITICAL | Zero incomplete relationships |
| Source object exists in registry | RLC-002 | CRITICAL | Zero dangling sources |
| Target object exists in registry | RLC-003 | CRITICAL | Zero dangling targets |
| Relationship type is a valid enum value | RLC-004 | CRITICAL | Zero unknown types |
| Bidirectional relationships stored in both directions | RLC-005 | CRITICAL | |
| No self-referential relationships | RLC-006 | MAJOR | Except where spec allows |
| Relationship ID is unique | RLC-007 | CRITICAL | Zero duplicate relation_ids |

---

## Bidirectionality Checks (500 random relationships)

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| For every R(A→B), inverse R(B←A) exists | RLC-008 | CRITICAL | All 500 pass |
| Traversal A.successors includes B | RLC-009 | MAJOR | |
| Traversal B.predecessors includes A | RLC-010 | MAJOR | |
| Inverse direction correctly labelled | RLC-011 | MAJOR | DEPENDS_ON inverse = DEPENDENCY_OF |

---

## CRUD Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Add relationship: source gets updated | RLC-012 | MAJOR | |
| Add relationship: target gets updated | RLC-013 | MAJOR | |
| Delete relationship: both sides updated | RLC-014 | MAJOR | |
| Delete relationship: not found error if not exists | RLC-015 | MAJOR | |
| Update relationship strength | RLC-016 | MINOR | |

---

## Version Constraint Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Version-constrained relationship verified | RLC-017 | MINOR | Constraint stored and queryable |
| Constraint violation detected | RLC-018 | MINOR | Warning emitted when target version out of range |

---

## Performance

| Operation | P50 | P99 | Dataset |
|-----------|-----|-----|---------|
| Add relationship | ≤ 2ms | ≤ 10ms | 100K objects |
| Traverse successors | ≤ 1ms | ≤ 5ms | 10M relationships |
| Bidirectionality check | ≤ 1ms | ≤ 5ms | per relationship |

---

## Metrics

| Metric | Bronze | Silver | Gold |
|--------|--------|--------|------|
| Structural validity | = 1.0 | = 1.0 | = 1.0 |
| Bidirectionality coverage | ≥ 0.95 | ≥ 0.98 | = 1.0 |
| CRUD correctness | ≥ 0.95 | ≥ 0.99 | = 1.0 |

---

## Report Format

```json
{
  "domain": "relationship",
  "relationships_tested": 500,
  "checks": {
    "RLC-001": "PASS", "RLC-005": "PASS", "RLC-008": "PASS",
    "RLC-009": "FAIL", "RLC-011": "PASS"
  },
  "bidirectionality_coverage": 0.972,
  "structural_validity": 1.0,
  "domain_score": 0.944,
  "level_achieved": "Silver"
}
```

---

## CLI

```bash
kos-cert relationship                    # 500 random relationships
kos-cert relationship --relationships 2000
kos-cert relationship --check-bidi-only
kos-cert relationship --output reports/relationship.json
```

---

## Cross-References

- Relationship engine → Phase 3.0C `06-RELATIONSHIP-ENGINE`
- Dependency model → Phase 3.0C.5 `27-KNOWLEDGE-DEPENDENCY-MODEL`
- Traceability links → Phase 3.0C.5 `29-KNOWLEDGE-TRACEABILITY`
