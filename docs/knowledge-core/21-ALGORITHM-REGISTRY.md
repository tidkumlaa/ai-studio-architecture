# KNW-KC-ARCH-021 — Algorithm Registry

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Algorithm Registry catalogs all named algorithms used in KOS and the platform. Every algorithm has a formal Big-O complexity specification, pseudocode, and reference to its implementation module.

---

## Registered Algorithms

### Platform Algorithms (from Phase 2.1D.0 21-ALGORITHMS)

| ID | Name | Time | Space | Module |
|----|------|------|-------|--------|
| `ALG-AI-001` | Workload Detection | O(W×S) | O(1) | `ai_runtime.ri2.workload_profiler` |
| `ALG-AI-002` | Complexity Scoring | O(P) | O(1) | `ai_runtime.ri2.task_complexity` |
| `ALG-AI-003` | EMA Learning Rate | O(1) | O(N) | `ai_runtime.ri2.learning_engine` |
| `ALG-AI-004` | Cost Prediction | O(1) | O(1) | `ai_runtime.ri2.cost_predictor` |
| `ALG-AI-005` | Provider Selection | O(P) | O(P) | `ai_runtime.ri2.provider_selector` |
| `ALG-AI-006` | Model Selection | O(M) | O(1) | `ai_runtime.ri2.provider_selector` |
| `ALG-AI-007` | Execution Complexity | O(1) | O(1) | `ai_runtime.ri2.execution_planner` |
| `ALG-AI-008` | Accuracy Trend | O(N) | O(N) | `ai_runtime.ri2.learning_engine` |
| `ALG-PLAT-009` | Import Rewrite | O(F×L) | O(1) | `tools.migration.import_rewriter` |
| `ALG-PLAT-010` | Manifest Checksum | O(N log N) | O(N) | `tools.migration.verifier` |

### Knowledge Core Algorithms (Phase 3.0C)

| ID | Name | Time | Space | Document |
|----|------|------|-------|---------|
| `ALG-KC-011` | DFS Traversal | O(N+E) | O(N) | 25-GRAPH-ALGORITHMS |
| `ALG-KC-012` | BFS Traversal | O(N+E) | O(N) | 25-GRAPH-ALGORITHMS |
| `ALG-KC-013` | Tarjan SCC | O(N+E) | O(N) | 25-GRAPH-ALGORITHMS |
| `ALG-KC-014` | Topological Sort | O(N+E) | O(N) | 25-GRAPH-ALGORITHMS |
| `ALG-KC-015` | Shortest Path | O((N+E) log N) | O(N) | 25-GRAPH-ALGORITHMS |
| `ALG-KC-016` | Dependency Closure | O(N+E) | O(N) | 25-GRAPH-ALGORITHMS |
| `ALG-KC-017` | Fingerprint Hash | O(L) | O(1) | 01-IDENTITY-ENGINE |
| `ALG-KC-018` | Quality Scoring | O(D) | O(1) | 11-QUALITY-ENGINE |
| `ALG-KC-019` | Evidence Freshness | O(1) | O(1) | 10-EVIDENCE-ENGINE |
| `ALG-KC-020` | Confidence Decay | O(1) | O(1) | 12-CONFIDENCE-MODEL |
| `ALG-KC-021` | Traceability Coverage | O(N+E) | O(N) | 13-TRACEABILITY |
| `ALG-KC-022` | Graph Diff | O(N+E) | O(N) | 25-GRAPH-ALGORITHMS |
| `ALG-KC-023` | KQL Parse & Execute | O(T + N) | O(N) | 27-QUERY-LANGUAGE |
| `ALG-KC-024` | Semantic Similarity | O(D) | O(D) | 29-SEMANTIC-LAYER |

*W=workload features, S=signals, P=providers, M=models, N=nodes, E=edges, F=files, L=lines, D=dimensions, T=query tokens*

---

## Algorithm Entry Format

```yaml
# knowledge/registry/algorithms/ALG-KC-011.yaml
identity:
  knowledge_id: "KNW-KC-ALG-011"
  canonical_name: "kc.algorithm.dfs_traversal"
  knowledge_uri: "knw://kos/algorithm/dfs-traversal"
  namespace: "kos"
  version: "1.0.0"
  owner: "team:architecture-board"

algorithm_spec:
  algorithm_id: "ALG-KC-011"
  time_complexity: "O(N+E)"
  space_complexity: "O(N)"
  inputs:
    - "start_node: KnowledgeObject"
    - "relation_type: RelationType"
    - "direction: TraversalDirection"
    - "max_depth: int"
  outputs:
    - "list[TraversalResult]"
  constants: {}
  implemented_in: []            # future: module knowledge_ids
  pseudocode: |
    DFS(start, type, direction, depth_limit, visited={}):
      if start in visited or depth == 0: return
      visited.add(start)
      results.append(start)
      for edge in get_edges(start, type, direction):
        DFS(edge.target, type, direction, depth - 1, visited)
      return results

lifecycle:
  status: CANONICAL
```

---

## Algorithm Registry Protocol

```python
class AlgorithmRegistry(KnowledgeRegistryProtocol):
    def get_by_complexity(self, max_time_complexity: str) -> list[AlgorithmObject]: ...
    def get_implementations(self, algorithm_id: str) -> list[str]: ...
    def get_for_operation(self, operation: str) -> AlgorithmObject | None: ...
    def validate_complexity_claim(self, algorithm_id: str) -> bool: ...
```

---

## Cross-References

- Registry base → `14-REGISTRY-ARCHITECTURE`
- Graph algorithms in detail → `25-GRAPH-ALGORITHMS`
- EMA constant (α=0.3) → Phase 2.1D.0 `21-ALGORITHMS`
- Performance budgets for algorithms → `37-PERFORMANCE-BUDGET`
