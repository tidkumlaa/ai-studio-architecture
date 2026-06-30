# KNW-CERT-ARCH-004 — Graph Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies correctness and performance of the Knowledge Graph traversal and analysis algorithms. Graph certification verifies that all 7 Phase 3.0C graph algorithms (G-01 through G-07) are implemented correctly.

---

## Graph Test Data

| Dataset | Nodes | Edges | Description |
|---------|-------|-------|-------------|
| G-TINY | 50 | 200 | Smoke test |
| G-SMALL | 500 | 2,000 | Bronze baseline |
| G-MEDIUM | 5,000 | 25,000 | Gold baseline |
| G-LARGE | 50,000 | 250,000 | Enterprise baseline |
| G-XLARGE | 500,000 | 2,500,000 | Research baseline |

All datasets have pre-computed correct answers for every algorithm.

---

## Algorithm Checks

### G-01: BFS (Breadth-First Search)

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| BFS from each of 100 random nodes | GC-001 | CRITICAL | Visited nodes match ground truth |
| BFS respects edge direction | GC-002 | MAJOR | No reverse traversal |
| BFS handles disconnected graph | GC-003 | MAJOR | Returns only reachable nodes |
| BFS on single node | GC-004 | CRITICAL | Returns [node] |

### G-02: DFS (Depth-First Search)

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| DFS from each of 100 random nodes | GC-005 | CRITICAL | Visited node set matches BFS set |
| DFS detects back edges | GC-006 | MAJOR | Back edge flag set on cycles |

### G-03: Shortest Path

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Shortest path between 500 random pairs | GC-007 | CRITICAL | Path length matches ground truth |
| Shortest path with no path | GC-008 | CRITICAL | Returns None (not error) |
| Shortest path = direct edge | GC-009 | MAJOR | Single-hop case returns 1 |

### G-04: Topological Sort

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Topo sort on DAG | GC-010 | CRITICAL | All edges go forward in sort order |
| Topo sort rejects cyclic graph | GC-011 | CRITICAL | Returns CycleDetectedError |
| Topo sort unique per consistent graph | GC-012 | MINOR | Same input → same output |

### G-05: Tarjan SCC (Strongly Connected Components)

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| SCC count matches ground truth | GC-013 | CRITICAL | |
| Every node assigned to exactly one SCC | GC-014 | CRITICAL | |
| SCC on DAG returns N singletons | GC-015 | MAJOR | Each node is its own SCC |

### G-06: Cycle Detection

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Detect direct cycle (A→B→A) | GC-016 | CRITICAL | Returns cycle path |
| Detect indirect cycle (A→B→C→A) | GC-017 | CRITICAL | Returns cycle path |
| Detect no cycle on DAG | GC-018 | CRITICAL | Returns None |
| Detect self-loop (A→A) | GC-019 | MAJOR | Returns [A] |

### G-07: Reachability / Ancestor / Descendant

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Ancestor set matches ground truth | GC-020 | MAJOR | |
| Descendant set matches ground truth | GC-021 | MAJOR | |
| Reachability matrix spot-check (100 pairs) | GC-022 | MAJOR | |
| Impact analysis: remove node X, find affected | GC-023 | MAJOR | |
| Dependency closure correct | GC-024 | MAJOR | Transitive closure = ground truth |
| Transitive closure correct | GC-025 | MAJOR | |

---

## Random Traversal Stress

Beyond algorithm correctness, run 1,000 random traversals on G-MEDIUM:

| Traversal Type | Count | Pass Criteria |
|----------------|-------|---------------|
| Random BFS | 200 | All complete within 100ms P99 |
| Random DFS | 200 | All complete within 100ms P99 |
| Random shortest path | 200 | All complete within 50ms P99 |
| Random ancestor | 200 | All complete within 50ms P99 |
| Random descendant | 200 | All complete within 50ms P99 |

---

## Latency Targets (G-MEDIUM, 5K nodes / 25K edges)

| Operation | P50 | P95 | P99 |
|-----------|-----|-----|-----|
| BFS | ≤ 10ms | ≤ 30ms | ≤ 100ms |
| DFS | ≤ 10ms | ≤ 30ms | ≤ 100ms |
| Shortest Path | ≤ 5ms | ≤ 20ms | ≤ 50ms |
| Topo Sort | ≤ 20ms | ≤ 50ms | ≤ 100ms |
| Tarjan SCC | ≤ 50ms | ≤ 100ms | ≤ 200ms |
| Cycle Detection | ≤ 5ms | ≤ 20ms | ≤ 50ms |
| Ancestor/Descendant | ≤ 10ms | ≤ 30ms | ≤ 80ms |

---

## Metrics Reported

```json
{
  "domain": "graph",
  "dataset": "G-MEDIUM",
  "algorithms": {
    "bfs": {"checks_pass": 4, "checks_total": 4, "p99_ms": 87.3},
    "dfs": {"checks_pass": 2, "checks_total": 2, "p99_ms": 91.2},
    "shortest_path": {"checks_pass": 3, "checks_total": 3, "p99_ms": 41.0},
    "topo_sort": {"checks_pass": 3, "checks_total": 3, "p99_ms": 78.5},
    "tarjan_scc": {"checks_pass": 3, "checks_total": 3, "p99_ms": 155.0},
    "cycle_detection": {"checks_pass": 4, "checks_total": 4, "p99_ms": 22.3},
    "reachability": {"checks_pass": 6, "checks_total": 6, "p99_ms": 65.4}
  },
  "random_traversal_p99_ms": 88.2,
  "domain_score": 0.974,
  "level_achieved": "Enterprise"
}
```

---

## CLI

```bash
kos-cert graph                           # G-SMALL (default)
kos-cert graph --dataset G-MEDIUM
kos-cert graph --dataset G-LARGE --algo bfs,scc
kos-cert graph --random-traversals 1000
kos-cert graph --output reports/graph.json
```

---

## Cross-References

- Graph algorithms specification → Phase 3.0C `25-GRAPH-ALGORITHMS` (G-01–G-07)
- Graph at scale → `15-SCALABILITY-CERTIFICATION`
- Overall score weight (10%) → `27-OVERALL-SCORING`
