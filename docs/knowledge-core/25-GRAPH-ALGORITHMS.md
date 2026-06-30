# KNW-KC-ARCH-025 — Graph Algorithms

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

This document specifies all graph algorithms required by the Knowledge Graph. Each algorithm has an algorithm ID, pseudocode, complexity analysis, and use case.

---

## Algorithm Catalog

### ALG-KC-011 — Depth-First Search (DFS)

**Use:** Traversal, cycle detection, dependency closure  
**Complexity:** Time O(N+E), Space O(N)

```
DFS(G, start, rel_types, direction, max_depth, visited=∅):
  if start ∈ visited OR max_depth == 0: return []
  visited.add(start)
  result = [start]
  for edge in get_edges(G, start, rel_types, direction):
    result += DFS(G, edge.target, rel_types, direction, max_depth-1, visited)
  return result
```

---

### ALG-KC-012 — Breadth-First Search (BFS)

**Use:** Shortest path finding, level-order traversal, impact radius  
**Complexity:** Time O(N+E), Space O(N)

```
BFS(G, start, rel_types, direction, max_depth):
  queue = [(start, 0)]
  visited = {start}
  result = []
  while queue:
    node, depth = queue.pop(0)
    result.append((node, depth))
    if depth < max_depth:
      for edge in get_edges(G, node, rel_types, direction):
        if edge.target not in visited:
          visited.add(edge.target)
          queue.append((edge.target, depth+1))
  return result
```

---

### ALG-KC-013 — Tarjan's Strongly Connected Components (SCC)

**Use:** Cycle detection in `depends_on` graphs, circular dependency identification  
**Complexity:** Time O(N+E), Space O(N)

```
TARJAN_SCC(G):
  index_counter = [0]
  stack = []
  lowlinks = {}
  index = {}
  on_stack = {}
  sccs = []

  STRONGCONNECT(v):
    index[v] = lowlinks[v] = index_counter[0]
    index_counter[0] += 1
    stack.append(v)
    on_stack[v] = True

    for edge in G.get_out_edges(v):
      w = edge.target
      if w not in index:
        STRONGCONNECT(w)
        lowlinks[v] = min(lowlinks[v], lowlinks[w])
      elif on_stack[w]:
        lowlinks[v] = min(lowlinks[v], index[w])

    if lowlinks[v] == index[v]:
      scc = []
      while True:
        w = stack.pop()
        on_stack[w] = False
        scc.append(w)
        if w == v: break
      if len(scc) > 1:   # cycle detected
        sccs.append(scc)

  for v in G.nodes:
    if v not in index: STRONGCONNECT(v)
  return sccs                         # each list with >1 element is a cycle
```

---

### ALG-KC-014 — Topological Sort

**Use:** Boot ordering, compilation ordering, execution plan scheduling  
**Complexity:** Time O(N+E), Space O(N)

```
TOPOLOGICAL_SORT(G, rel_types):
  in_degree = {v: 0 for v in G.nodes}
  for edge in G.edges where edge.label in rel_types:
    in_degree[edge.target] += 1

  queue = [v for v in G.nodes if in_degree[v] == 0]
  result = []
  while queue:
    v = queue.pop(0)
    result.append(v)
    for edge in G.get_out_edges(v):
      in_degree[edge.target] -= 1
      if in_degree[edge.target] == 0:
        queue.append(edge.target)

  if len(result) != len(G.nodes):
    raise CycleDetectedError(G.nodes - set(result))
  return result
```

---

### ALG-KC-015 — Shortest Path (Dijkstra)

**Use:** Minimum-weight dependency path, impact minimization  
**Complexity:** Time O((N+E) log N), Space O(N)

```
SHORTEST_PATH(G, source, target, weight_fn):
  dist = {v: ∞ for v in G.nodes}
  dist[source] = 0
  prev = {}
  pq = [(0, source)]

  while pq:
    d, v = heappop(pq)
    if v == target: break
    if d > dist[v]: continue
    for edge in G.get_out_edges(v):
      w = edge.target
      cost = d + (1.0 / weight_fn(edge))    # lower weight = shorter
      if cost < dist[w]:
        dist[w] = cost
        prev[w] = (v, edge.edge_id)
        heappush(pq, (cost, w))

  return reconstruct_path(prev, source, target), dist[target]
```

---

### ALG-KC-016 — Dependency Closure (Transitive Closure)

**Use:** Full dependency set computation, impact analysis  
**Complexity:** Time O(N+E), Space O(N)

```
DEPENDENCY_CLOSURE(G, start, rel_type):
  # DFS collecting all reachable nodes
  closure = set()
  DFS_COLLECT(G, start, rel_type, closure)
  closure.discard(start)   # exclude the start node itself
  return closure

# Incremental update on ADD_EDGE(A → B):
INCREMENTAL_CLOSURE_UPDATE(G, new_edge, closures):
  # All nodes that can reach A now also can reach B's closure
  for node in ancestors_of(G, new_edge.source):
    closures[node] |= closures[new_edge.target]
    closures[node].add(new_edge.target)
```

---

### ALG-KC-022 — Graph Diff

**Use:** Snapshot comparison, impact of changes between versions  
**Complexity:** Time O(N+E), Space O(N)

```
GRAPH_DIFF(G1, G2):
  nodes_1 = set(G1.nodes)
  nodes_2 = set(G2.nodes)
  edges_1 = set(G1.edges)
  edges_2 = set(G2.edges)

  return GraphDiff(
    added_nodes   = nodes_2 - nodes_1,
    removed_nodes = nodes_1 - nodes_2,
    modified_nodes = {n for n in nodes_1 & nodes_2
                     if G1.node(n).properties != G2.node(n).properties},
    added_edges   = edges_2 - edges_1,
    removed_edges = edges_1 - edges_2,
  )
```

---

## Reachability, Ancestor & Descendant Search

```
REACHABLE(G, start, rel_types, direction, max_depth=∞):
  = set from DFS/BFS from start

ANCESTORS(G, target, rel_types):
  = REACHABLE(G, target, rel_types, INVERSE)

DESCENDANTS(G, source, rel_types):
  = REACHABLE(G, source, rel_types, FORWARD)

IMPACT_GRAPH(G, target):
  = ANCESTORS(G, target, [depends_on, imports, uses])
  # Who depends on target? Those are impacted.
```

---

## Complexity Summary

| Algorithm | Time | Space | Use Case |
|-----------|------|-------|---------|
| DFS | O(N+E) | O(N) | Traversal, cycle check |
| BFS | O(N+E) | O(N) | Shortest path, impact radius |
| Tarjan SCC | O(N+E) | O(N) | Cycle detection |
| Topological Sort | O(N+E) | O(N) | Ordering |
| Shortest Path | O((N+E)logN) | O(N) | Minimum weight path |
| Dependency Closure | O(N+E) | O(N) | Full dep set |
| Graph Diff | O(N+E) | O(N) | Change analysis |

---

## Cross-References

- Algorithms registered in → `21-ALGORITHM-REGISTRY`
- Graph model → `24-KNOWLEDGE-GRAPH-MODEL`
- Graph indexes used during algorithms → `26-GRAPH-INDEXES`
- Traceability uses these algorithms → `13-TRACEABILITY`
