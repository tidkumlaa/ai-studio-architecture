---
knowledge_id: KA-KIP-007
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.2
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-intelligence
type: specification
depends_on:
  - id: KA-KIP-003
    reason: "Reverse BFS on dependency graph"
---

# Impact Analysis Engine

## Change · Architecture · Runtime · Test · Document · Provider Impact

---

## 1. Purpose

Given a set of changed knowledge objects, the Impact Analysis Engine computes:
- What is **directly** affected (1-hop reverse dependencies)
- What is **transitively** affected (N-hop reverse traversal)
- What **tests** must be re-run
- What **architecture documents** are now stale
- What **providers** are implicated
- The **severity** of the change

---

## 2. Impact Categories

| Category | Definition | Graph Traversal |
|----------|-----------|-----------------|
| Direct | Immediately depend on changed object | Reverse BFS depth=1 |
| Transitive | Depend on direct dependents | Reverse BFS unlimited |
| Architecture | Affected architecture documents | Filter by type="architecture" |
| Runtime | Affected runtime specifications | Filter by type="runtime" |
| Test | Documents with tested_by relationships | Reverse via "tested_by" |
| Document | All affected documents (union of above) | All types |
| Provider | Affected provider integrations | Filter domain=DOM-PROVIDER |

---

## 3. Impact Report Structure

```python
@dataclass
class ImpactReport:
    changed_ids:          list[str]
    directly_affected:    list[ImpactedObject]
    transitively_affected: list[ImpactedObject]
    architecture_impact:  list[ImpactedObject]
    runtime_impact:       list[ImpactedObject]
    test_impact:          list[ImpactedObject]
    document_impact:      list[ImpactedObject]
    provider_impact:      list[ImpactedObject]
    severity:             RiskLevel
    generated:            str

@dataclass(frozen=True)
class ImpactedObject:
    knowledge_id: str
    title:        str
    path:         str
    impact_type:  str   # "direct", "transitive", "architecture", ...
    distance:     int   # Hops from changed object
    severity:     str   # "critical", "high", "medium", "low"
```

---

## 4. Core Algorithm

```
ANALYZE_IMPACT(changed_ids, graph, registry):
  directly_affected = []
  transitively_affected = []
  
  FOR changed_id IN changed_ids:
    # Reverse BFS: find everything that depends on this
    FOR (affected_id, depth) IN graph.bfs(
        start=changed_id,
        direction=REVERSE,
        edge_labels={"depends_on", "implements", "tested_by"}
    ):
      obj = registry.get(affected_id)
      IF obj is None: CONTINUE
      
      impact = ImpactedObject(
        knowledge_id = affected_id,
        title        = obj.title,
        path         = obj.path,
        impact_type  = "direct" if depth == 1 else "transitive",
        distance     = depth,
        severity     = classify_severity(obj, depth)
      )
      
      IF depth == 1:
        directly_affected.append(impact)
      ELSE:
        transitively_affected.append(impact)
  
  # Classify by type
  all_affected = directly_affected + transitively_affected
  architecture_impact = filter(all_affected, type="architecture")
  test_impact         = filter(all_affected, type in {"testing", "specification"})
  provider_impact     = filter(all_affected, domain="DOM-PROVIDER")
  
  severity = compute_overall_severity(all_affected, changed_ids)
  
  RETURN ImpactReport(...)

Complexity: O(V + E) per changed_id
```

---

## 5. Severity Classification

```
classify_severity(obj, depth):
  IF obj.type == "architecture" AND obj.status == "approved": RETURN "critical"
  IF obj.type == "specification" AND depth == 1:              RETURN "high"
  IF obj.domain in {DOM-AI, DOM-WORKFLOW}:                    RETURN "high"
  IF depth <= 2:                                              RETURN "medium"
  RETURN "low"
```

---

## 6. Change Analysis

A change analysis compares two registry states and identifies what changed:

```
ANALYZE_CHANGE(before: RegistrySnapshot, after: RegistrySnapshot):
  added   = after.ids - before.ids
  removed = before.ids - after.ids
  modified = {id for id in (before.ids & after.ids)
              if before.get(id).version != after.get(id).version}
  
  RETURN ChangeSet(added=added, removed=removed, modified=modified)
```

---

## 7. Interface

```python
class ImpactAnalyzer:
    def analyze(
        self,
        changed_ids: list[str],
        graph: KnowledgeGraph,
        registry: KnowledgeRegistry,
        max_depth: int = 10,
    ) -> ImpactReport

    def architecture_impact(self, changed_id: str, ...) -> list[ImpactedObject]
    def test_impact(self, changed_ids: list[str], ...) -> list[ImpactedObject]
    def provider_impact(self, changed_ids: list[str], ...) -> list[ImpactedObject]
    def analyze_change(self, before: dict, after: dict) -> ChangeSet

    def format_report(self, report: ImpactReport) -> str  # Markdown
    def kql_for_report(self, changed_id: str) -> str       # KQL that produces this report
```

---

## 8. Impact KQL

The engine can express its analysis as a KQL query for transparency:

```
-- Impact of changing KA-ARCH-001
SELECT id, title, type, status, capability
FROM knowledge
TRAVERSE REVERSE DEPTH 5
WHERE id = "KA-ARCH-001"
ORDER BY type ASC, status DESC
```

---

## References

- [02-KNOWLEDGE-GRAPH-RUNTIME.md](02-KNOWLEDGE-GRAPH-RUNTIME.md) — BFS used for traversal
- [KA-SPEC-011](../knowledge-core/11-KNOWLEDGE-TRACEABILITY-MODEL.md) — Traceability model
- [03-KNOWLEDGE-QUERY-ENGINE.md](03-KNOWLEDGE-QUERY-ENGINE.md) — KQL impact queries
