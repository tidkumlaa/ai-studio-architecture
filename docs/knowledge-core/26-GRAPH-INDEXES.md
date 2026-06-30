# KNW-KC-ARCH-026 — Graph Indexes

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Graph Indexes enable sub-second lookup of nodes and edges without full graph scans. Every access pattern has a corresponding index. All indexes are maintained incrementally — no full rebuild on every write.

---

## Node Indexes

| Index Name | Key | Value | Lookup Complexity | Use Case |
|-----------|-----|-------|-----------------|---------|
| `node_by_id` | `knowledge_id` | `KnowledgeNode` | O(1) | Primary lookup |
| `node_by_uri` | `knowledge_uri` | `knowledge_id` | O(1) | URI resolution |
| `node_by_fingerprint` | `fingerprint` | `knowledge_id` | O(1) | Deduplication |
| `node_by_canonical_name` | `canonical_name` | `knowledge_id` | O(1) | Name lookup |
| `nodes_by_type` | `KnowledgeObjectType` | `list[knowledge_id]` | O(1) | Type queries |
| `nodes_by_status` | `LifecycleStatus` | `list[knowledge_id]` | O(1) | Lifecycle queries |
| `nodes_by_namespace` | `namespace` | `list[knowledge_id]` | O(1) | Namespace scoping |
| `nodes_by_owner` | `owner` | `list[knowledge_id]` | O(1) | Ownership queries |
| `nodes_by_tag` | `tag` | `list[knowledge_id]` | O(1) | Tag filtering |
| `nodes_by_quality` | sorted by `quality_score` | `knowledge_id` | O(log N) | Quality ranking |

---

## Edge Indexes

| Index Name | Key | Value | Lookup Complexity | Use Case |
|-----------|-----|-------|-----------------|---------|
| `edge_by_id` | `relation_id` | `KnowledgeEdge` | O(1) | Primary lookup |
| `out_edges` | `(source_id, rel_type)` | `list[edge_id]` | O(1) | Forward traversal |
| `in_edges` | `(target_id, rel_type)` | `list[edge_id]` | O(1) | Reverse traversal |
| `edges_by_type` | `RelationType` | `list[edge_id]` | O(1) | Type queries |
| `edges_by_source` | `source_id` | `list[edge_id]` | O(1) | All outbound edges |
| `edges_by_target` | `target_id` | `list[edge_id]` | O(1) | All inbound edges |
| `edges_by_pair` | `(source_id, target_id)` | `list[edge_id]` | O(1) | Duplicate check |

---

## Derived Indexes (Computed, Cached)

| Index Name | Description | Invalidated By |
|-----------|-------------|----------------|
| `dependency_closure` | For each node: full transitive `depends_on` set | Any new `depends_on` edge |
| `ancestor_index` | For each node: all ancestors | Any new edge in ancestor rel types |
| `descendant_index` | For each node: all descendants | Any new edge in descendant rel types |
| `traceability_index` | REQ → full forward trace | Any new impl/test/deploy edge |
| `impact_index` | For each node: blast radius set | Any new `depends_on`/`uses` edge |

---

## Index Storage Format

```yaml
# indexes/out_edges.yaml
KNW-KOS-ARCH-001:
  implements:
    - REL-IMPL-KOS-001
    - REL-IMPL-KOS-002
  documents:
    - REL-DOC-KOS-001
  references:
    - REL-REF-KOS-001

KNW-PLAT-SVC-001:
  depends_on:
    - REL-DEP-SVC-KERN-001
  uses:
    - REL-USE-SVC-API-001
```

---

## Incremental Index Maintenance

```
ON ADD_NODE(node):
  node_by_id[node.knowledge_id] = node
  node_by_uri[node.knowledge_uri] = node.knowledge_id
  node_by_fingerprint[node.fingerprint] = node.knowledge_id
  node_by_canonical_name[node.canonical_name] = node.knowledge_id
  nodes_by_type[node.object_type].append(node.knowledge_id)
  nodes_by_status[node.status].append(node.knowledge_id)
  nodes_by_namespace[node.namespace].append(node.knowledge_id)
  nodes_by_owner[node.owner].append(node.knowledge_id)
  for tag in node.tags:
    nodes_by_tag[tag].append(node.knowledge_id)
  # sorted quality index: insert in O(log N)

ON ADD_EDGE(edge):
  edge_by_id[edge.edge_id] = edge
  out_edges[(edge.source, edge.label)].append(edge.edge_id)
  in_edges[(edge.target, edge.label)].append(edge.edge_id)
  edges_by_type[edge.label].append(edge.edge_id)
  edges_by_source[edge.source].append(edge.edge_id)
  edges_by_target[edge.target].append(edge.edge_id)
  edges_by_pair[(edge.source, edge.target)].append(edge.edge_id)
  # Update derived indexes incrementally (see 25-GRAPH-ALGORITHMS)

COMPLEXITY PER WRITE: O(log N) for quality sort; O(1) for all hash indexes
```

---

## Index Rebuild Protocol

When indexes become corrupt or out of sync:

```
REBUILD_INDEX(index_name):
  1. Lock writes (pause new node/edge registrations)
  2. Clear the named index
  3. Scan all nodes/edges from primary storage
  4. Rebuild index entries from scan
  5. Validate rebuild: spot-check 1% sample
  6. Unlock writes
  7. Emit registry.index.rebuilt event

FULL REBUILD COMPLEXITY: O(N log N) for quality sort; O(N+E) for all others
Only required on data corruption — not on normal operation.
```

---

## Index Health Monitoring

```yaml
index_health:
  computed_at: datetime
  indexes:
    node_by_id:
      count: 0
      last_updated: datetime
      integrity: VALID|CORRUPT|REBUILDING
    out_edges:
      count: 0
      last_updated: datetime
      integrity: VALID
    dependency_closure:
      count: 0
      last_updated: datetime
      integrity: VALID
      stale_ratio: 0.00     # fraction of entries that are outdated
```

---

## Performance Requirements

| Index Operation | P50 | P99 |
|----------------|-----|-----|
| Hash index lookup | < 0.1ms | < 0.5ms |
| Sorted index range query | < 1ms | < 5ms |
| Incremental update (single node/edge) | < 1ms | < 5ms |
| Full index rebuild (1M nodes) | < 30s | < 120s |

---

## Cross-References

- Graph model defines what is indexed → `24-KNOWLEDGE-GRAPH-MODEL`
- Algorithms use these indexes → `25-GRAPH-ALGORITHMS`
- Query Engine relies on indexes for sub-second performance → `27-QUERY-LANGUAGE`
- Registry also maintains its own indexes → `14-REGISTRY-ARCHITECTURE`
