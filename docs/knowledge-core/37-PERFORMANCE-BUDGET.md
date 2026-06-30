# KNW-KC-ARCH-037 — Performance Budget

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Every operation in the Knowledge Core has a defined P50 and P99 latency budget. These budgets are hard contracts — implementations that violate them fail CI. No exceptions without Architecture Board approval.

---

## Scale Targets

| Metric | Target |
|--------|--------|
| Knowledge Objects | 1,000,000 |
| Relationships | 10,000,000 |
| Namespaces | 20 |
| Registries | 9 |
| Graph nodes | 1,000,000 |
| Graph edges | 10,000,000 |
| Search index size | 1,000,000 documents |
| Concurrent users | 100 |
| Events per second | 10,000 |

---

## Identity Engine Budgets

| Operation | P50 | P99 |
|-----------|-----|-----|
| ID generation | < 1ms | < 5ms |
| Checksum computation (1KB object) | < 1ms | < 5ms |
| Checksum computation (10KB object) | < 5ms | < 20ms |
| Fingerprint computation | < 1ms | < 2ms |
| Duplicate detection (fingerprint lookup) | < 1ms | < 5ms |
| Alias resolution | < 1ms | < 5ms |

---

## Registry Budgets

| Operation | P50 | P99 |
|-----------|-----|-----|
| Register single object | < 10ms | < 50ms |
| Update single object | < 10ms | < 50ms |
| Get by knowledge_id | < 1ms | < 5ms |
| Get by URI | < 1ms | < 5ms |
| List by type (100 results) | < 10ms | < 50ms |
| List by namespace (1000 results) | < 50ms | < 200ms |
| Bulk register (100 objects) | < 200ms | < 1s |
| Bulk register (1000 objects) | < 2s | < 10s |
| Index rebuild (1M objects) | < 30s | < 120s |
| Registry stats computation | < 100ms | < 500ms |

---

## Graph Budgets

| Operation | P50 | P99 |
|-----------|-----|-----|
| Add node | < 1ms | < 5ms |
| Add edge | < 1ms | < 5ms |
| Single-hop traversal | < 1ms | < 5ms |
| 3-hop traversal | < 10ms | < 50ms |
| 5-hop traversal | < 20ms | < 100ms |
| 10-hop traversal | < 50ms | < 200ms |
| Dependency closure (avg 50 nodes) | < 20ms | < 100ms |
| Dependency closure (max 500 nodes) | < 100ms | < 500ms |
| Cycle detection (full graph) | < 1s | < 5s |
| Topological sort (full graph) | < 2s | < 10s |
| Graph diff (1M nodes) | < 30s | < 120s |
| Incremental node/edge update | < 5ms | < 20ms |
| Full graph cold-load (1M nodes) | < 30s | < 120s |

---

## Query Engine Budgets

| Operation | P50 | P99 |
|-----------|-----|-----|
| Identity query (ID lookup) | < 1ms | < 5ms |
| Structured WHERE query (1000 results) | < 10ms | < 50ms |
| Relationship query (1-hop) | < 2ms | < 10ms |
| Dependency query (closure) | < 50ms | < 200ms |
| Impact query (blast radius ≤ 100) | < 50ms | < 200ms |
| Coverage query (100 requirements) | < 100ms | < 500ms |
| Natural language query | < 500ms | < 2s |

---

## Search Engine Budgets

| Operation | P50 | P99 |
|-----------|-----|-----|
| Structured search | < 5ms | < 20ms |
| Full-text search (10K docs) | < 30ms | < 100ms |
| Full-text search (1M docs) | < 100ms | < 500ms |
| Semantic search (1M docs) | < 50ms | < 200ms |
| Hybrid search (1M docs) | < 100ms | < 500ms |
| Index single object | < 10ms | < 50ms |
| Query suggestion | < 10ms | < 50ms |

---

## Evidence & Quality Budgets

| Operation | P50 | P99 |
|-----------|-----|-----|
| Freshness computation | < 0.1ms | < 0.5ms |
| Evidence score computation | < 1ms | < 5ms |
| Quality score computation (all 9 dims) | < 10ms | < 50ms |
| Confidence decay computation | < 1ms | < 5ms |
| Path confidence computation | < 1ms | < 5ms |
| Bulk quality computation (100 objects) | < 500ms | < 2s |

---

## Snapshot Budgets

| Operation | P50 | P99 |
|-----------|-----|-----|
| Single-object snapshot | < 5ms | < 20ms |
| Namespace snapshot (1000 objects) | < 500ms | < 2s |
| Full registry snapshot (1M objects) | < 60s | < 300s |
| Restore single object | < 10ms | < 50ms |
| Restore namespace (1000 objects) | < 1s | < 5s |
| Snapshot diff (1M nodes) | < 30s | < 120s |
| Snapshot verification | < 100ms | < 500ms |

---

## Reasoning Engine Budgets

| Operation | P50 | P99 |
|-----------|-----|-----|
| Why reasoning | < 100ms | < 500ms |
| Impact reasoning (blast radius ≤ 100) | < 50ms | < 200ms |
| Risk reasoning | < 100ms | < 500ms |
| Alternatives (top 5) | < 200ms | < 1s |
| Tradeoff comparison | < 200ms | < 1s |
| History (last 10 versions) | < 50ms | < 200ms |

---

## Memory Budgets

| Component | Resident Memory Budget |
|-----------|----------------------|
| Registry index (1M objects) | ≤ 2 GB |
| Graph in-memory (1M nodes, 10M edges) | ≤ 8 GB |
| Search full-text index (1M docs) | ≤ 4 GB |
| Search vector index (1M docs, 768 dims) | ≤ 6 GB |
| Evidence cache | ≤ 512 MB |
| Version record cache | ≤ 256 MB |
| Query result cache | ≤ 512 MB |

---

## Budget Enforcement

Budgets are enforced by:
1. `BenchmarkObject` for each operation (registered in `21-ALGORITHM-REGISTRY`)
2. CI benchmark suite that fails if P99 is exceeded
3. Platform dashboard alerts on degradation (P99 > 80% of budget)

---

## Cross-References

- Benchmark objects → Phase 3.0B `knowledge_runtime/objects/quality.py`
- Algorithm complexities → `36-ALGORITHMS`
- Graph operation complexities → `25-GRAPH-ALGORITHMS`
- Test strategy enforcing budgets → `39-TEST-STRATEGY`
