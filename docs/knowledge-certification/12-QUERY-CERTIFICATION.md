# KNW-CERT-ARCH-012 — Query Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the Knowledge Query Engine returns correct, complete, and consistent results for structured queries against the registry and graph. This is distinct from semantic search (02-SEARCH-CERTIFICATION) — it covers structured filter and lookup queries.

---

## Query Types

| Type | Description | Checks |
|------|-------------|--------|
| Filter | Field equality / range queries | QR-001–QR-010 |
| Projection | Select specific fields | QR-011–QR-015 |
| Aggregation | Count, group-by, stats | QR-016–QR-020 |
| Graph traversal | Relationship-based queries | QR-021–QR-030 |
| Compound | Multiple predicates combined | QR-031–QR-035 |

---

## Filter Query Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Filter by `object_type` = MODULE | QR-001 | CRITICAL | Returns all and only MODULE objects |
| Filter by `lifecycle.state` = CANONICAL | QR-002 | CRITICAL | Correct set |
| Filter by `namespace` = plt | QR-003 | MAJOR | Correct set |
| Filter by `quality_score` ≥ 0.80 | QR-004 | MAJOR | Correct set |
| Filter by `tags` contains "quota" | QR-005 | MAJOR | All tagged objects returned |
| Filter by `owner` = team:platform | QR-006 | MAJOR | Correct set |
| Filter by multiple predicates (AND) | QR-007 | MAJOR | Intersection correct |
| Filter by multiple predicates (OR) | QR-008 | MINOR | Union correct |
| Filter returns empty set correctly | QR-009 | CRITICAL | [] not error |
| Filter on non-indexed field (full-scan) | QR-010 | MINOR | Correct but may be slow |

---

## Projection Query Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Project `knowledge_id` only | QR-011 | MAJOR | Returns only IDs, no other fields |
| Project `knowledge_id` + `name` | QR-012 | MAJOR | Returns exactly those two fields |
| Project all fields (wildcard) | QR-013 | MAJOR | Returns full object |
| Projection with filter combined | QR-014 | MAJOR | |
| Project non-existent field: error | QR-015 | MINOR | Returns FieldNotFoundError |

---

## Aggregation Query Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Count all objects | QR-016 | MAJOR | Matches registry stats |
| Count by object_type | QR-017 | MAJOR | Group-by counts correct |
| Count by namespace | QR-018 | MAJOR | Correct |
| Mean quality_score | QR-019 | MINOR | Within 0.001 of ground truth |
| Quality score histogram (10 bins) | QR-020 | MINOR | Bin counts correct |

---

## Graph Traversal Query Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Find all objects that DEPEND_ON KNW-X | QR-021 | MAJOR | |
| Find all objects that KNW-X DEPENDS_ON | QR-022 | MAJOR | |
| Find objects within 2 hops of KNW-X | QR-023 | MAJOR | |
| Find all ancestors of KNW-X | QR-024 | MAJOR | Matches graph cert result |
| Find objects in same SCC as KNW-X | QR-025 | MINOR | |
| Find shortest path KNW-X to KNW-Y | QR-026 | MAJOR | Matches graph cert result |
| Impact analysis: what breaks if KNW-X removed | QR-027 | MAJOR | Returns dependent objects |
| Find objects with no outgoing dependencies | QR-028 | MINOR | Leaf nodes |
| Find objects with no incoming dependencies | QR-029 | MINOR | Root nodes |
| Transitive closure of IMPLEMENTS | QR-030 | MINOR | |

---

## Compound Query Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Filter by type AND traverse graph | QR-031 | MAJOR | |
| Aggregate with graph filter | QR-032 | MINOR | |
| Paginated query (page 2 of 10) | QR-033 | MAJOR | Correct items at correct offset |
| Sorted result (by quality_score DESC) | QR-034 | MINOR | Descending order correct |
| Query with limit=0 returns all | QR-035 | MINOR | |

---

## Performance

| Query Type | P50 | P99 | At |
|------------|-----|-----|----|
| Filter (indexed) | ≤ 1ms | ≤ 5ms | 100K objects |
| Filter (unindexed) | ≤ 50ms | ≤ 200ms | 100K objects |
| Aggregation (count) | ≤ 5ms | ≤ 20ms | 100K objects |
| Graph traversal (2 hops) | ≤ 10ms | ≤ 50ms | 10M edges |

---

## Report Format

```json
{
  "domain": "query",
  "checks": {
    "QR-001": "PASS", "QR-009": "PASS",
    "QR-027": "PASS", "QR-033": "FAIL"
  },
  "filter_correctness": 0.941,
  "aggregation_correctness": 0.800,
  "graph_query_correctness": 0.933,
  "domain_score": 0.887,
  "level_achieved": "Silver"
}
```

---

## CLI

```bash
kos-cert query                           # full query certification
kos-cert query --type filter             # filter checks only
kos-cert query --type graph              # graph traversal checks only
kos-cert query --output reports/query.json
```

---

## Cross-References

- Registry → `03-REGISTRY-CERTIFICATION`
- Graph algorithms → `04-GRAPH-CERTIFICATION`
- Search (semantic) → `02-SEARCH-CERTIFICATION`
