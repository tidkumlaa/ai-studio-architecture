---
knowledge_id: KA-SPEC-018
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0.5
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-core
type: specification
implements:
  - KA-SPEC-015
depends_on:
  - id: KA-SPEC-017
    reason: "Complexity is analyzed per algorithm in this catalog"
  - id: KA-SPEC-016
    reason: "Space complexity is analyzed per data structure"
---

# Complexity Guide

## Time and Space Complexity for All Knowledge System Operations

---

## 1. Purpose

Performance guarantees require complexity analysis. This guide provides the time and space complexity for every significant operation in the knowledge system, enabling:
- Informed decisions about index design
- Performance budget validation (see [19-PERFORMANCE-BUDGET.md](19-PERFORMANCE-BUDGET.md))
- Identification of operations that will degrade at scale
- Correct algorithm selection for each use case

---

## 2. Notation

| Symbol | Meaning |
|--------|---------|
| `D` | Number of documents (current: ~786, target: ~5,000) |
| `V` | Number of graph vertices (≈ D × 1.2 including structural nodes) |
| `E` | Number of graph edges (relationships) |
| `C` | Number of capabilities |
| `N` | Catch-all for the dominant size parameter |
| `F` | Number of metadata fields per document |
| `k` | Number of results returned |
| `d` | Graph traversal depth |
| `O(1)` | Constant time (hash map lookup) |
| `O(log n)` | Logarithmic time (sorted set operations) |
| `O(n)` | Linear time |
| `O(n log n)` | Linearithmic (comparison sort) |
| `O(n²)` | Quadratic (pairwise comparison) |

---

## 3. Data Structure Operation Complexity

### DS-009 — KnowledgeRegistry

| Operation | Time | Space |
|-----------|------|-------|
| `get(id)` | O(1) | — |
| `put(obj)` | O(1) | O(C) amortized |
| `by_file(path)` | O(1) | — |
| `all_active()` | O(D) | O(D) |
| `next_id(type)` | O(1) | — |

### DS-006 — KnowledgeGraph

| Operation | Time | Space |
|-----------|------|-------|
| `add_vertex(obj)` | O(1) | O(1) |
| `add_edge(from, to, type)` | O(1) | O(1) |
| `get_vertex(id)` | O(1) | — |
| `outbound(id)` | O(k) where k=degree | — |
| `inbound(id)` | O(k) | — |
| `neighbors_filtered(id, types)` | O(k) | — |
| Full graph scan | O(V + E) | — |

### DS-010 — CapabilityIndex

| Operation | Time | Space |
|-----------|------|-------|
| `canonical(cap, type)` | O(1) | — |
| `all_in_capability(cap)` | O(1) | — |
| `missing_required(cap)` | O(1) | — |
| Build from registry | O(D) | O(D) |

### DS-011 — HealthIndex

| Operation | Time | Space |
|-----------|------|-------|
| `score(id)` | O(1) | — |
| `stale_top(n)` | O(n) | — |
| `critical_top(n)` | O(n) | — |
| `repository_score()` | O(1) | — |
| Insert/update score | O(log D) (SortedList) | — |

---

## 4. Algorithm Complexity

### ALG-001 — ID Generation

| Case | Time | Space |
|------|------|-------|
| No collision (normal) | O(1) | O(1) |
| k collisions (after manual edits) | O(k) | O(1) |

### ALG-003 — DFS Traversal

| Case | Time | Space |
|------|------|-------|
| Unlimited depth | O(V + E) | O(V) |
| Depth-limited (depth d) | O(b^d) where b=avg branching factor | O(b×d) |
| Expected (d=3, b=5) | O(125) ≈ O(1) at typical scale | O(15) |

**Note:** For a typical knowledge graph with V=1000, E=4000, unlimited DFS is O(5000) — fast even without optimization.

### ALG-004 — Tarjan's SCC

| Case | Time | Space |
|------|------|-------|
| All cases | O(V + E) | O(V) |
| Expected (no cycles) | O(V + E) early termination possible | O(V) |

### ALG-005 — Orphan Detection (Multi-Source BFS)

| Case | Time | Space |
|------|------|-------|
| All cases | O(V + E) | O(V) |
| Expected (few orphans) | O(V + E) | O(V) |

### ALG-006 — Shortest Path (BFS)

| Case | Time | Space |
|------|------|-------|
| Path exists, depth d | O(b^d) | O(b^d) |
| No path | O(V + E) | O(V) |

### ALG-007 — Health Score Computation

| Case | Time | Space |
|------|------|-------|
| Single document | O(L + R + E) | O(1) |
| All documents | O(D × (L + R + E)) | O(1) |
| L ≈ 50 links, R ≈ 10 rels, E ≈ 5 evidence | O(D × 65) = O(D) | O(1) |

### ALG-008 — Duplicate Canonical Detection

| Case | Time | Space |
|------|------|-------|
| All cases | O(D) | O(D) |

### ALG-010 — Supersession Chain Traversal

| Case | Time | Space |
|------|------|-------|
| Chain length c | O(c) | O(c) |
| Maximum chain (typical) | O(5) | O(5) |

---

## 5. Full Build Complexity

The full index build (ALG-009) has the following complexity breakdown:

| Phase | Time | Notes |
|-------|------|-------|
| File discovery | O(D) | Glob operation |
| Parsing | O(D × F) | F=fields per doc ≈ 20 |
| Schema validation | O(D × F) | Per-field validation |
| Registry build | O(D) | Hash map insertions |
| Graph build | O(D + E) | Vertex + edge insertion |
| Secondary indexes | O(D) | Classify each document |
| Health computation | O(D × (L+R+E)) ≈ O(D) | Per-document scoring |
| Aggregate health | O(D) | Rollup |
| Serialization | O(D + E) | JSON/YAML write |
| **Total** | **O(D × F + E)** | |

At D=5000, F=20, E=20000: **O(120,000)** operations — sub-second on modern hardware.

---

## 6. Incremental Update Complexity

| Phase | Time | Notes |
|-------|------|-------|
| Parse changed file | O(F) | Single document |
| Update registry | O(1) | |
| Remove old edges | O(k_old) | k_old=old degree |
| Add new edges | O(k_new) | k_new=new degree |
| Recompute affected scores | O(k_affected × F) | |
| **Total per file** | **O(k × F)** | k=affected neighbors |

For typical changes (1 file, 10 affected): **O(200)** operations — instant.

---

## 7. KQL Query Complexity

| Query Type | Time | Notes |
|-----------|------|-------|
| `WHERE id = X` | O(1) | Registry lookup |
| `WHERE type = T` | O(1) | Type index lookup |
| `WHERE capability = C` | O(1) | Capability index lookup |
| `WHERE health_score < N` | O(k) | HealthIndex scan |
| `WHERE review_date < D` | O(k) | Stale index scan |
| `TRAVERSE FROM X DEPTH d` | O(b^d) | BFS/DFS |
| `TRAVERSE FROM X UNLIMITED` | O(V + E) | |
| `FIND DUPLICATES` | O(D) | ALG-008 |
| `FIND ORPHANS` | O(V + E) | ALG-005 |
| Full text search | O(D × L) | Without search index |
| Full text search | O(1) + O(k) | With inverted index |

---

## 8. Scale Analysis

### 8.1 Growth Projections

| Milestone | Documents | Edge Count | Full Build | Health Report |
|-----------|-----------|-----------|-----------|--------------|
| Phase 2.0D.1 (current) | 786 | ~3,000 | < 1s | < 2s |
| Phase 2.0E (knowledge OS) | 2,000 | ~10,000 | < 3s | < 5s |
| Phase 3.0 (self-evolving) | 5,000 | ~25,000 | < 8s | < 15s |
| Enterprise scale | 20,000 | ~100,000 | < 30s | < 60s |

### 8.2 Operations That Degrade at Scale

| Operation | Current | At 20K docs | Mitigation |
|-----------|---------|-------------|-----------|
| Full-text search | O(D×L) = fast | Slow | Inverted index (Phase 2.0D.2) |
| Full graph rebuild | O(D×F) | 30s | Incremental updates |
| SCC detection | O(V+E) | Fast | None needed |
| Health computation | O(D×F) | 30s | Parallel computation |
| Semantic similarity | O(D²) if naive | Very slow | Embedding index (Phase 2.0E+) |

---

## References

- [17-ALGORITHMS-CATALOG.md](17-ALGORITHMS-CATALOG.md) — Algorithms analyzed here
- [16-DATA-STRUCTURE-CATALOG.md](16-DATA-STRUCTURE-CATALOG.md) — Structures analyzed here
- [19-PERFORMANCE-BUDGET.md](19-PERFORMANCE-BUDGET.md) — Performance targets that this analysis must satisfy
