---
knowledge_id: KA-SPEC-010
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
  - KA-SPEC-001
depends_on:
  - id: KA-SPEC-008
    reason: "Index engine builds and maintains the knowledge graph"
  - id: KA-IDX-001
    reason: "Index structures defined in Phase 2.0D.0"
---

# Knowledge Index Engine

## The Runtime Engine That Builds, Maintains, and Queries All Indexes

---

## 1. Purpose

The Knowledge Index Engine (KIE) is the runtime component responsible for:
- Building all knowledge indexes from source documents
- Maintaining index freshness as documents change
- Serving index queries to the Documentation Intelligence Platform
- Detecting inconsistencies and generating audit diagnostics

Without the KIE, every query requires a full file system scan. With the KIE, every query resolves in O(1) for direct lookups and O(V+E) for graph traversals.

---

## 2. Engine Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Knowledge Index Engine                      │
│                                                             │
│  ┌──────────────┐    ┌────────────────┐    ┌────────────┐  │
│  │   Watcher    │    │  Index Builder │    │   Indexer  │  │
│  │  (fs events) │───▶│  (full rebuild)│───▶│  (stores)  │  │
│  └──────────────┘    └────────────────┘    └────────────┘  │
│                                                             │
│  ┌──────────────┐    ┌────────────────┐    ┌────────────┐  │
│  │ Incremental  │    │  Graph Builder │    │   Query    │  │
│  │  Updater     │───▶│  (edges/nodes) │    │   Engine   │  │
│  └──────────────┘    └────────────────┘    └────────────┘  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │                    Index Store                        │  │
│  │  Registry | Domain | Capability | Type | Tech         │  │
│  │  Product  | Relationship | Graph | Health             │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Index Types and Data Structures

### 3.1 Registry Index

**Purpose:** O(1) lookup of any KnowledgeObject by ID.

**Data structure:** Hash Map

```
RegistryIndex: HashMap<KnowledgeID, KnowledgeObject>

Operations:
  get(id)           → KnowledgeObject | null    O(1)
  put(id, obj)      → void                      O(1)
  remove(id)        → void                      O(1)
  contains(id)      → boolean                   O(1)
  all()             → KnowledgeObject[]         O(n)
```

### 3.2 Domain Index

**Purpose:** Retrieve all knowledge objects for a domain.

**Data structure:** Hash Map of Sorted Set

```
DomainIndex: HashMap<DomainID, SortedSet<KnowledgeID>>

Operations:
  by_domain(domain)             → KnowledgeID[]   O(1) lookup + O(k) result
  add(domain, id)               → void             O(log k)
  remove(domain, id)            → void             O(log k)
```

### 3.3 Capability Index

**Purpose:** Retrieve all documents for a capability, organized by type.

**Data structure:** Hash Map of Hash Map of List

```
CapabilityIndex: HashMap<CapabilityID, HashMap<KnowledgeType, KnowledgeID[]>>

Operations:
  by_capability(cap)            → Map<Type, ID[]>  O(1)
  by_capability_type(cap, type) → KnowledgeID[]   O(1)
  canonical(cap, type)          → KnowledgeID | null  O(1)
```

### 3.4 Relationship Index

**Purpose:** Graph adjacency lists for relationship traversal.

**Data structure:** Two adjacency lists (outbound + inbound)

```
RelationshipIndex:
  adj_out: HashMap<KnowledgeID, List<(EdgeType, KnowledgeID, EdgeProperties)>>
  adj_in:  HashMap<KnowledgeID, List<(EdgeType, KnowledgeID, EdgeProperties)>>

Operations:
  outbound(id)                        → EdgeList   O(1) + O(k)
  inbound(id)                         → EdgeList   O(1) + O(k)
  outbound_by_type(id, type)          → EdgeList   O(k)
  inbound_by_type(id, type)           → EdgeList   O(k)
  degree(id)                          → Integer    O(1)
```

### 3.5 Health Index

**Purpose:** Precomputed health scores for fast retrieval.

**Data structure:** Hash Map + Priority Queue

```
HealthIndex:
  scores:          HashMap<KnowledgeID, HealthScore>
  stale:           SortedSet<(OverdueDays, KnowledgeID)>  # Sorted by overdue
  critical:        SortedSet<(HealthScore, KnowledgeID)>  # Sorted by score
  by_domain:       HashMap<DomainID, AggregateHealthScore>
  by_capability:   HashMap<CapabilityID, AggregateHealthScore>

Operations:
  score(id)        → HealthScore  O(1)
  stale_top(n)     → KnowledgeID[] O(n log n) initial, O(n) subsequent
  critical_top(n)  → KnowledgeID[] O(n)
```

---

## 4. Index Build Algorithm

### 4.1 Full Build (Phase 2.0D.1 — Initial Build)

```
ALGORITHM full_build(source_root: Path):
  
  Phase 1 — Discovery:
    files = glob(source_root, "**/*.md")
    yaml_files = glob(source_root, "**/index.yaml")

  Phase 2 — Parsing:
    FOR each file in files:
      (frontmatter, body) = parse_markdown(file)
      obj = KnowledgeObject.from_frontmatter(frontmatter)
      obj.body = body
      raw_objects.append(obj)

  Phase 3 — Validation:
    FOR each obj in raw_objects:
      violations = schema_validator.validate(obj)
      diagnostics.extend(violations)
      IF not violations.has_errors():
        valid_objects.append(obj)

  Phase 4 — Registry Build:
    FOR each obj in valid_objects:
      registry_index.put(obj.id, obj)

  Phase 5 — Graph Build:
    FOR each obj in valid_objects:
      graph.add_vertex(obj)
      FOR each relationship in obj.relationships:
        IF registry_index.contains(relationship.target_id):
          graph.add_edge(obj.id, relationship.target_id, relationship.type)
        ELSE:
          diagnostics.add_warning(BROKEN_REFERENCE, obj.id, relationship.target_id)

  Phase 6 — Secondary Index Build:
    FOR each obj in valid_objects:
      domain_index.add(obj.domain, obj.id)
      capability_index.add(obj.capability, obj.type, obj.id)
      IF obj.canonical:
        canonical_index.set(obj.capability, obj.type, obj.id)

  Phase 7 — Health Computation:
    FOR each obj in valid_objects:
      score = health_scorer.compute(obj, graph)
      health_index.scores.put(obj.id, score)
      IF score < 60:
        health_index.critical.add((score, obj.id))
      IF obj.is_stale():
        health_index.stale.add((obj.days_overdue(), obj.id))

  Phase 8 — Aggregate Health:
    FOR each capability in capability_index.keys():
      agg = compute_capability_health(capability, health_index)
      health_index.by_capability.put(capability, agg)

    FOR each domain in domain_index.keys():
      agg = compute_domain_health(domain, capability_index, health_index)
      health_index.by_domain.put(domain, agg)

  Phase 9 — Serialization:
    write_json("architecture/graph.json", graph)
    write_yaml("architecture/index.yaml", build_master_index())
    write_yaml("architecture/health-report.yaml", health_report())
    write_markdown("architecture/HEALTH-REPORT.md", health_report_markdown())

  RETURN diagnostics

Complexity: O(D × F + E) where D=documents, F=fields, E=relationships
```

### 4.2 Incremental Update

```
ALGORITHM incremental_update(changed_files: Path[]):

  FOR each file in changed_files:
    old_obj = registry_index.get_by_file(file)
    new_obj = parse_and_validate(file)

    IF new_obj.id != old_obj.id:
      -- ID changed: this is an error (IDs are immutable)
      diagnostics.add_error(ID_CHANGED, file)
      CONTINUE

    -- Remove old edges
    FOR each edge in graph.outbound(old_obj.id):
      graph.remove_edge(edge)

    -- Add new edges
    FOR each rel in new_obj.relationships:
      IF registry_index.contains(rel.target_id):
        graph.add_edge(new_obj.id, rel.target_id, rel.type)

    -- Update registry
    registry_index.put(new_obj.id, new_obj)

    -- Recompute health for affected objects
    affected = {new_obj.id} ∪ graph.inbound(new_obj.id).targets
    FOR each id in affected:
      score = health_scorer.compute(registry_index.get(id), graph)
      health_index.scores.put(id, score)

Complexity: O(k × F + k × E_local) where k=changed files
```

---

## 5. Index Freshness Protocol

### 5.1 Staleness Detection

The KIE detects stale indexes by comparing source file modification times:

```
FUNCTION is_stale():
  index_mtime = file_modified_time("architecture/index.yaml")
  FOR each .md file in scope:
    IF file_modified_time(file) > index_mtime:
      RETURN true
  RETURN false
```

### 5.2 Freshness Guarantee

The KIE guarantees that queries always operate on fresh indexes if:
1. The pre-commit hook runs `ka-compile --validate --update-indexes` before every commit
2. Or the KIE watch mode is running during development

In Phase 2.0D.2, the Documentation Intelligence Platform runs the KIE in watch mode as a background service.

---

## 6. Query Interface

The KIE exposes a query interface consumed by the KQL engine:

```python
class KnowledgeIndexEngine:

  def get(self, id: KnowledgeID) → KnowledgeObject | None
  def get_uri(self, uri: KnowledgeURI) → KnowledgeObject | None
  def by_domain(self, domain: DomainID) → List[KnowledgeObject]
  def by_capability(self, cap: CapabilityID) → Dict[KnowledgeType, List[KnowledgeObject]]
  def canonical(self, cap: CapabilityID, type: KnowledgeType) → KnowledgeObject | None
  def outbound(self, id: KnowledgeID, types: List[RelType]=None) → List[Edge]
  def inbound(self, id: KnowledgeID, types: List[RelType]=None) → List[Edge]
  def health_score(self, id: KnowledgeID) → HealthScore
  def stale_documents(self, min_days: int=0) → List[KnowledgeObject]
  def search(self, query: KQLQuery) → QueryResult
```

---

## 7. Cache Strategy

| Layer | Cache Target | TTL | Invalidation |
|-------|-------------|-----|-------------|
| L1 — Memory | All index structures | Session lifetime | On incremental update |
| L2 — Disk | graph.json, index.yaml | Until source changes | On rebuild |
| L3 — Query | KQL query result cache | 60 seconds | On any index update |

Cache invalidation follows a **write-through** strategy: updates to source documents invalidate affected cache entries immediately via the incremental update algorithm.

---

## References

- [08-KNOWLEDGE-GRAPH-MODEL.md](08-KNOWLEDGE-GRAPH-MODEL.md) — Graph that the KIE builds
- [07-KNOWLEDGE-QUERY-LANGUAGE.md](07-KNOWLEDGE-QUERY-LANGUAGE.md) — KQL that runs on the KIE
- [16-DATA-STRUCTURE-CATALOG.md](16-DATA-STRUCTURE-CATALOG.md) — Data structures used by the KIE
- [18-COMPLEXITY-GUIDE.md](18-COMPLEXITY-GUIDE.md) — Complexity analysis for KIE operations
- [19-PERFORMANCE-BUDGET.md](19-PERFORMANCE-BUDGET.md) — Performance targets for the KIE
