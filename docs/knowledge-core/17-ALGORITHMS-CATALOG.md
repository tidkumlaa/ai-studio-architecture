---
knowledge_id: KA-SPEC-017
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
  - id: KA-SPEC-016
    reason: "Algorithms operate on these data structures"
  - id: KA-SPEC-008
    reason: "Graph algorithms are core to the knowledge system"
---

# Algorithms Catalog

## All Algorithms Used in the Knowledge System

---

## 1. Identity Algorithms

### ALG-001 — Knowledge ID Generation

**Purpose:** Generate the next available unique Knowledge ID for a given type.

```python
ALGORITHM generate_id(type_code: str, registry: KnowledgeRegistry) → str:

  last_id = registry.next_available.get(type_code, None)

  IF last_id is None:
    candidate = f"KA-{type_code}-001"
  ELSE:
    num = extract_number(last_id)
    candidate = f"KA-{type_code}-{num + 1:03d}"

  -- Collision check (defense against manual edits breaking next_available)
  WHILE candidate in registry:
    num += 1
    candidate = f"KA-{type_code}-{num:03d}"

  registry.set_next_available(type_code, candidate)
  RETURN candidate

Time: O(c) where c = collision count (expected O(1) with maintained registry)
Space: O(1)
```

### ALG-002 — Knowledge ID Validation

**Purpose:** Validate that a string is a correctly formatted Knowledge ID.

```python
ALGORITHM validate_id(candidate: str) → ValidationResult:
  pattern = r'^KA-[A-Z]{2,5}-\d{3,}$'
  IF NOT re.match(pattern, candidate):
    RETURN Invalid(f"Does not match pattern {pattern}")

  parts = candidate.split('-')
  type_code = parts[1]
  IF type_code NOT IN VALID_TYPE_CODES:
    RETURN Invalid(f"Unknown type code: {type_code}")

  RETURN Valid()

Time: O(1)
Space: O(1)
```

---

## 2. Graph Algorithms

### ALG-003 — Depth-First Traversal

**Purpose:** Traverse the knowledge graph from a start node following specified edge types.

```python
ALGORITHM dfs_traverse(
  graph: KnowledgeGraph,
  start: str,
  edge_types: List[RelationshipType],
  direction: Direction,           # FORWARD | REVERSE | BOTH
  max_depth: int = UNLIMITED
) → List[TraversalPath]:

  visited = Set[str]()
  results = List[TraversalPath]()
  stack = [(start, TraversalPath([start], []), 0)]

  WHILE stack not empty:
    (current, path, depth) = stack.pop()

    IF current in visited: CONTINUE
    IF depth > max_depth: CONTINUE
    visited.add(current)

    neighbors = get_neighbors(graph, current, edge_types, direction)
    FOR (neighbor, edge) in neighbors:
      new_path = TraversalPath(
        path.nodes + [neighbor],
        path.edges + [edge],
        depth + 1
      )
      results.append(new_path)
      stack.push((neighbor, new_path, depth + 1))

  RETURN results

Time: O(V + E) for unlimited depth
Space: O(V) for visited set + O(depth) for recursion stack
```

### ALG-004 — Tarjan's SCC (Cycle Detection)

**Purpose:** Detect cycles in the dependency or supersession graph.

```python
ALGORITHM tarjan_scc(graph: KnowledgeGraph, edge_type: RelationshipType):
  index_counter = [0]
  stack = []
  lowlink = {}
  index = {}
  on_stack = {}
  sccs = []

  FUNCTION strongconnect(v):
    index[v] = lowlink[v] = index_counter[0]
    index_counter[0] += 1
    stack.append(v)
    on_stack[v] = True

    FOR (w, edge) in graph.adj_out[v] filtered by edge_type:
      IF w not in index:
        strongconnect(w)
        lowlink[v] = min(lowlink[v], lowlink[w])
      ELIF on_stack[w]:
        lowlink[v] = min(lowlink[v], index[w])

    IF lowlink[v] == index[v]:
      scc = []
      WHILE True:
        w = stack.pop()
        on_stack[w] = False
        scc.append(w)
        IF w == v: BREAK
      sccs.append(scc)

  FOR v in graph.vertices:
    IF v not in index:
      strongconnect(v)

  -- SCCs with size > 1 are cycles
  cycles = [scc for scc in sccs IF len(scc) > 1]
  RETURN cycles

Time: O(V + E)
Space: O(V)
```

### ALG-005 — Multi-Source BFS (Orphan Detection)

**Purpose:** Identify KnowledgeObjects unreachable from known entry points.

```python
ALGORITHM find_orphans(
  graph: KnowledgeGraph,
  roots: List[str],           # Entry points: domain nodes, index nodes
  edge_types: List[RelationshipType]
) → Set[str]:

  reachable = Set[str]()
  queue = deque(roots)

  WHILE queue not empty:
    current = queue.popleft()
    IF current in reachable: CONTINUE
    reachable.add(current)

    FOR (neighbor, edge) in graph.adj_out[current] filtered by edge_types:
      IF neighbor not in reachable:
        queue.append(neighbor)

  all_active = Set(id for id, obj in graph.vertices.items()
                   WHERE obj.status == APPROVED)

  orphans = all_active - reachable
  RETURN orphans

Time: O(V + E)
Space: O(V)
```

### ALG-006 — Shortest Path (BFS)

**Purpose:** Find the shortest relationship path between two KnowledgeObjects.

```python
ALGORITHM shortest_path(
  graph: KnowledgeGraph,
  start: str,
  end: str,
  edge_types: List[RelationshipType]
) → Optional[TraversalPath]:

  IF start == end: RETURN TraversalPath([start], [], 0)

  visited = {start}
  queue = deque([(start, TraversalPath([start], [], 0))])

  WHILE queue not empty:
    (current, path) = queue.popleft()

    FOR (neighbor, edge) in graph.adj_out[current] filtered by edge_types:
      IF neighbor == end:
        RETURN TraversalPath(path.nodes + [neighbor], path.edges + [edge])
      IF neighbor not in visited:
        visited.add(neighbor)
        queue.append((neighbor, extend(path, neighbor, edge)))

  RETURN None  -- No path exists

Time: O(V + E)
Space: O(V)
```

---

## 3. Health Algorithms

### ALG-007 — Document Health Score Computation

**Purpose:** Compute the 0–100 health score for a single KnowledgeObject.

```python
ALGORITHM compute_health_score(obj: KnowledgeObject, graph: KnowledgeGraph) → HealthScore:

  -- Dimension 1: Structural Completeness (0-25)
  structural = sum(
    score for (field, score) in REQUIRED_FIELDS.items()
    IF getattr(obj, field) is not None
  )

  -- Dimension 2: Relationship Completeness (0-20)
  relationship = 0
  IF obj.relationships.implements: relationship += 4
  IF obj.relationships.depends_on: relationship += 4
  IF obj.relationships.implemented_by: relationship += 4
  IF obj.relationships.decided_by: relationship += 4
  IF graph.inbound(obj.id): relationship += 4

  -- Dimension 3: Freshness (0-20)
  IF obj.review_date is None: freshness = 0
  ELSE:
    days_overdue = max(0, (today() - obj.review_date).days)
    freshness = FRESHNESS_TABLE[days_overdue]   -- Lookup table

  -- Dimension 4: Evidence (0-15)
  evidence = 0
  IF obj.evidence: evidence += 5
  verified = [e for e in obj.evidence IF e.verified]
  IF verified: evidence += 5
  recent = [e for e in verified IF (today() - e.last_verified).days <= 90]
  IF recent: evidence += 5

  -- Dimension 5: Coverage (0-10)
  coverage = 0
  IF obj.coverage.architecture >= 80: coverage += 3
  ELIF obj.coverage.architecture >= 50: coverage += 1
  IF obj.coverage.implementation >= 70: coverage += 3
  ELIF obj.coverage.implementation >= 40: coverage += 1
  IF obj.coverage.testing >= 60: coverage += 4
  ELIF obj.coverage.testing >= 30: coverage += 2

  -- Dimension 6: Consistency (0-10)
  consistency = 10
  FOR link in extract_all_links(obj.body):
    IF NOT file_exists(link): consistency = max(0, consistency - 2)
  IF has_duplicate_canonical(obj.id, obj.capability, obj.type):
    consistency = max(0, consistency - 5)
  IF count_lines(obj.body) > 500: consistency = max(0, consistency - 5)

  total = structural + relationship + freshness + evidence + coverage + consistency

  RETURN HealthScore(
    total=total,
    structural=structural,
    relationship=relationship,
    freshness=freshness,
    evidence=evidence,
    coverage=coverage,
    consistency=consistency,
    computed_at=now()
  )

Time: O(L + R + E) where L=links in body, R=relationship count, E=evidence count
Space: O(1)
```

### ALG-008 — Duplicate Canonical Detection

**Purpose:** Find capability-type pairs with multiple canonical documents.

```python
ALGORITHM find_duplicate_canonicals(registry: KnowledgeRegistry) → List[DuplicateConflict]:

  canonical_map: Dict[(str, KnowledgeType), List[str]] = defaultdict(list)

  FOR id, obj in registry.all_approved():
    IF obj.canonical:
      canonical_map[(obj.capability, obj.type)].append(id)

  conflicts = [
    DuplicateConflict(cap, type, ids)
    FOR (cap, type), ids in canonical_map.items()
    IF len(ids) > 1
  ]

  RETURN conflicts

Time: O(N) where N = approved documents
Space: O(N)
```

---

## 4. Index Algorithms

### ALG-009 — Index Generation

**Purpose:** Build all seven knowledge indexes from the registry and graph.

Described in detail in [10-KNOWLEDGE-INDEX-ENGINE.md](10-KNOWLEDGE-INDEX-ENGINE.md) (ALG-009 references the full build algorithm there).

### ALG-010 — Supersession Chain Traversal

**Purpose:** Follow the supersession chain from any document to find the current canonical.

```python
ALGORITHM follow_supersession_chain(
  start: str,
  registry: KnowledgeRegistry,
  max_hops: int = 100
) → str:  -- Returns the current canonical ID

  current = start
  visited = Set()

  FOR _ in range(max_hops):
    IF current in visited:
      RAISE KnowledgeCycleError(visited + [current])
    visited.add(current)

    obj = registry.get(current)
    IF obj is None: RAISE KnowledgeNotFoundError(current)
    IF obj.superseded_by is None: RETURN current   -- Terminal node

    current = obj.superseded_by

  RAISE KnowledgeError("Supersession chain exceeded max_hops")

Time: O(c) where c = chain length
Space: O(c) for cycle detection
```

---

## References

- [16-DATA-STRUCTURE-CATALOG.md](16-DATA-STRUCTURE-CATALOG.md) — Structures these algorithms operate on
- [18-COMPLEXITY-GUIDE.md](18-COMPLEXITY-GUIDE.md) — Full complexity analysis
- [08-KNOWLEDGE-GRAPH-MODEL.md](08-KNOWLEDGE-GRAPH-MODEL.md) — Graph model for graph algorithms
- [10-KNOWLEDGE-INDEX-ENGINE.md](10-KNOWLEDGE-INDEX-ENGINE.md) — Index build algorithm
