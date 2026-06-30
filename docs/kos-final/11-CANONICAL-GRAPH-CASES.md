# KNW-FINAL-011 — Canonical Graph Cases

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines canonical test cases for all 7 graph algorithms (G-01 through G-07). Each case includes the fixture graph, query, and exact expected result.

---

## Reference Graph Fixture

The canonical graph fixture used in all graph cases:

```
Nodes (9 objects):
  A = KNW-PLT-MOD-001  (Quota Manager)
  B = KNW-RT-RT-001    (AI Runtime)
  C = KNW-PROV-SVC-001 (Provider Registry)
  D = KNW-ALG-ALG-007  (BM25 Ranker)
  E = KNW-PLT-MOD-002  (AI Router)
  F = KNW-RT-SVC-007   (AI Router Service)
  G = KNW-META-DEC-001 (ADR-001)
  H = KNW-TEST-TST-001 (Quota Test)
  I = KNW-ALG-ALG-009  (Tarjan SCC)

Edges (DEPENDS_ON):
  A → B  (Quota Manager depends on AI Runtime)
  B → C  (AI Runtime depends on Provider Registry)
  E → D  (AI Router depends on BM25 Ranker)
  F → B  (AI Router Service depends on AI Runtime)
  F → E  (AI Router Service depends on AI Router)
  H → A  (Quota Test tests Quota Manager)
  G → A  (ADR references Quota Manager)

Aliases: TESTS and REFERENCES treated as directed edges
```

---

## G-01: BFS Cases

| ID | Start | Expected Visited Set | Expected Order |
|----|-------|---------------------|----------------|
| GC-BFS-001 | A | {A, B, C} | Level by level |
| GC-BFS-002 | F | {F, B, E, C, D} | Level by level |
| GC-BFS-003 | G | {G, A, B, C} | Level by level |
| GC-BFS-004 | D | {D} | Single node (no outgoing) |
| GC-BFS-005 | I | {I} | Disconnected — only self |

---

## G-02: DFS Cases

| ID | Start | Expected Visited Set |
|----|-------|---------------------|
| GC-DFS-001 | A | {A, B, C} |
| GC-DFS-002 | F | {F, B, C, E, D} |
| GC-DFS-003 | D | {D} |

---

## G-03: Shortest Path Cases

| ID | Source | Target | Expected Path | Length |
|----|--------|--------|--------------|--------|
| GC-SP-001 | A | C | A → B → C | 2 |
| GC-SP-002 | F | C | F → B → C | 2 |
| GC-SP-003 | G | C | G → A → B → C | 3 |
| GC-SP-004 | D | B | None | ∞ (no path) |
| GC-SP-005 | A | A | [A] | 0 |
| GC-SP-006 | H | C | H → A → B → C | 3 |

---

## G-04: Topological Sort Cases

### Case GC-TOPO-001 (DAG — valid)
Input: all 9 nodes, 7 edges above  
Expected: Any ordering where for every edge (u→v), u appears before v.

Valid orderings include:
```
[G, H, I, A, E, F, B, D, C]  ← one valid order
[I, G, H, E, A, F, D, B, C]  ← another valid order
```

### Case GC-TOPO-002 (Cycle — invalid)
Input: Add edge C → A (creating cycle A → B → C → A)  
Expected: `CycleDetectedError` with cycle path `[A, B, C, A]`

---

## G-05: Tarjan SCC Cases

### Case GC-SCC-001 (DAG — all singletons)
Input: 9 nodes, original 7 edges (no cycle)  
Expected: 9 SCCs, each containing exactly 1 node.

### Case GC-SCC-002 (With cycle)
Input: Add edges C → A and A → F (two cycles created):
- Cycle 1: A → B → C → A
- Cycle 2: A → F (if F → A added, separate cycle)

For cycle C → A added only:  
Expected SCCs: [{A, B, C}, {D}, {E}, {F}, {G}, {H}, {I}]

### Case GC-SCC-003 (Self-loop)
Input: Add edge D → D  
Expected: {D} is still a singleton SCC (self-loops are trivial SCCs)

---

## G-06: Cycle Detection Cases

| ID | Graph | Expected |
|----|-------|----------|
| GC-CYCLE-001 | Original (no cycle) | None |
| GC-CYCLE-002 | Add C → A | [A, B, C, A] |
| GC-CYCLE-003 | Add D → D (self-loop) | [D] |
| GC-CYCLE-004 | Add A → G (making G → A → G) | [G, A, G] |
| GC-CYCLE-005 | 10-node cycle (A→B→...→J→A) | Full cycle path |

---

## G-07: Ancestor / Descendant / Reachability Cases

| ID | Query | Node | Expected |
|----|-------|------|---------|
| GC-ANCS-001 | Ancestors of C | C | {B, A, F, E, H, G} (all that reach C) |
| GC-ANCS-002 | Ancestors of B | B | {A, F, H, G, E} |
| GC-DESC-001 | Descendants of A | A | {B, C} |
| GC-DESC-002 | Descendants of F | F | {B, C, E, D} |
| GC-REACH-001 | Is A reachable from H? | H → A? | Yes |
| GC-REACH-002 | Is D reachable from A? | A → D? | No (A depends on B, not D) |
| GC-IMPACT-001 | Impact: remove B | Who breaks? | A, F, H, G (all depend on B transitively) |
| GC-TRANS-001 | Transitive closure DEPENDS_ON | A | {B, C} |
| GC-TRANS-002 | Transitive closure DEPENDS_ON | F | {B, C, E, D} |

---

## Performance Cases

For each algorithm, benchmark on G-MEDIUM (5K nodes, 25K edges):

| Algorithm | P99 Target |
|-----------|-----------|
| BFS from random 100 nodes | ≤ 100ms |
| DFS from random 100 nodes | ≤ 100ms |
| Shortest path (random 500 pairs) | ≤ 50ms |
| Topological sort | ≤ 100ms |
| Tarjan SCC | ≤ 200ms |
| Cycle detection | ≤ 50ms |
| Ancestor/Descendant (random 200) | ≤ 80ms |

---

## Cross-References

- Graph certification → Phase 3.0D.0 `04-GRAPH-CERTIFICATION`
- Graph algorithm spec → Phase 3.0C `25-GRAPH-ALGORITHMS`
- Query graph cases → `08-CANONICAL-QUERY-CASES` (QG-001–015)
