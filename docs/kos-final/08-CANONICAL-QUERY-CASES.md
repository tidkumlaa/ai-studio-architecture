# KNW-FINAL-008 — Canonical Query Cases

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines 50 canonical query patterns covering all query types. Each case includes the query, expected result type, and pass criteria. Used as the authoritative query test set.

---

## Filter Queries (QF-001–QF-015)

| ID | Query | Expected Result | Pass Criteria |
|----|-------|----------------|---------------|
| QF-001 | `type=MODULE, state=CANONICAL` | All CANONICAL MODULE objects | Exact set |
| QF-002 | `namespace=plt` | All objects in plt namespace | Exact set |
| QF-003 | `quality_score >= 0.80` | Objects with high quality | All have score ≥ 0.80 |
| QF-004 | `owner=team:platform` | Platform-owned objects | Exact set |
| QF-005 | `tags contains quota` | Tagged objects | All include quota tag |
| QF-006 | `state=DEPRECATED` | Deprecated objects | Exact count |
| QF-007 | `type=ALGORITHM AND namespace=alg` | Algorithm objects | Exact set |
| QF-008 | `created_at >= 2026-01-01` | Recently created | Date filter correct |
| QF-009 | `version >= 1.0.0` | V1+ objects | Correct semver comparison |
| QF-010 | `type=BENCHMARK` | All benchmarks | Count = 19+ |
| QF-011 | `state=ARCHIVED` | Archived objects (excluded by default) | Empty set (default) |
| QF-012 | `type=MODULE AND quality_score >= 0.90` | High-quality modules | Correct intersection |
| QF-013 | `domain=PLATFORM OR domain=RUNTIME` | Multi-domain filter | Correct union |
| QF-014 | `knowledge_id = KNW-PLT-MOD-001` | Exact ID lookup | Returns exact object |
| QF-015 | `canonical_name = plt.module.quota-manager` | Canonical name lookup | Same as QF-014 |

---

## Aggregation Queries (QA-001–QA-010)

| ID | Query | Expected Result |
|----|-------|----------------|
| QA-001 | COUNT by type | Dictionary: type → count |
| QA-002 | COUNT by namespace | Dictionary: ns → count |
| QA-003 | COUNT by state | Lifecycle state distribution |
| QA-004 | MEAN quality_score | Float ∈ [0.0, 1.0] |
| QA-005 | MAX quality_score | Float ∈ [0.0, 1.0] |
| QA-006 | MIN quality_score for CANONICAL | Float ≥ 0.80 (gate) |
| QA-007 | COUNT by owner | Team distribution |
| QA-008 | Quality score histogram (10 bins) | Bin counts summing to total |
| QA-009 | COUNT total relationships | Integer |
| QA-010 | MEAN evidence_score for CANONICAL | Float ∈ [0.0, 1.0] |

---

## Graph Traversal Queries (QG-001–QG-015)

| ID | Query | Expected Result |
|----|-------|----------------|
| QG-001 | DEPENDS_ON successors of KNW-PLT-MOD-001 | Direct dependencies |
| QG-002 | DEPENDS_ON predecessors of KNW-RT-RT-001 | Objects that depend on RT |
| QG-003 | All ancestors of KNW-PLT-MOD-001 (transitive) | Transitive dependency chain |
| QG-004 | All descendants of KNW-META-DEC-001 | Everything derived from ADR |
| QG-005 | Objects within 2 hops of KNW-PLT-MOD-001 | 2-hop neighborhood |
| QG-006 | Shortest path KNW-PLT-MOD-001 → KNW-ALG-ALG-007 | Path list |
| QG-007 | Is KNW-PLT-MOD-001 reachable from KNW-RT-RT-001? | Boolean |
| QG-008 | Strongly connected components containing KNW-PLT-MOD-001 | SCC list |
| QG-009 | Topological sort of all modules | Ordered list |
| QG-010 | Objects with no outgoing DEPENDS_ON (leaf nodes) | Leaf set |
| QG-011 | Objects with no incoming DEPENDS_ON (root nodes) | Root set |
| QG-012 | Impact: remove KNW-PLT-MOD-001 — what breaks? | Dependent set |
| QG-013 | All objects KNW-PLT-MOD-001 IMPLEMENTS | Implementation chain |
| QG-014 | All tests for KNW-PLT-MOD-001 (TESTED_BY) | Test set |
| QG-015 | Dependency closure of kos.platform.package | All transitive deps |

---

## Compound Queries (QC-001–QC-010)

| ID | Query | Expected Result |
|----|-------|----------------|
| QC-001 | High-quality platform modules (type=MODULE AND ns=plt AND quality≥0.80) | Filtered, ordered |
| QC-002 | Canonical algorithms with benchmark coverage | ALG objects with BENCH |
| QC-003 | Deprecated objects with no successor | Broken deprecations |
| QC-004 | CANONICAL objects with no evidence | Zero (gold target) |
| QC-005 | Objects owned by team:platform, ordered by quality DESC | Paginated result |
| QC-006 | Missing traceability: CANONICAL modules without test | Coverage gap |
| QC-007 | High-impact objects (depended on by ≥ 10 others) | Critical path objects |
| QC-008 | Stale evidence items (older than 90 days) | Freshness report |
| QC-009 | Cross-package dependencies from plt → rt | External dependencies |
| QC-010 | Objects modified in last 7 days | Recent changes |

---

## Expected Output Format (per query)

```json
{
  "query_id": "QF-001",
  "query": {"type": "MODULE", "state": "CANONICAL"},
  "result_count": 42,
  "result_type": "object_list",
  "sample": ["KNW-PLT-MOD-001", "KNW-RT-MOD-001"],
  "execution_ms": 1.2,
  "ground_truth_count": 42
}
```

---

## Cross-References

- Query certification → Phase 3.0D.0 `12-QUERY-CERTIFICATION`
- Graph algorithm specs → Phase 3.0C `25-GRAPH-ALGORITHMS`
- Search cases → `10-CANONICAL-SEARCH-CASES`
