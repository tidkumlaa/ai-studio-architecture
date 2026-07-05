---
knowledge_id: KA-KIP-006
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
    reason: "Graph traversal for propagation algorithms"
  - id: KA-SPEC-011
    reason: "Traceability model defines inference chains"
---

# Knowledge Reasoning

## Inference · Relationship Inference · Evidence Propagation · Confidence Propagation

---

## 1. Reasoning Scope

The Reasoning Engine derives knowledge that is not explicitly declared:

| Capability | Input | Output |
|-----------|-------|--------|
| Relationship Inference | Graph topology | Implied relationships |
| Capability Inference | Object types + domains | Capability membership |
| Evidence Propagation | Evidence graph | Inferred coverage |
| Confidence Propagation | Declared confidence + evidence | Derived confidence |
| Coverage Propagation | Implementation presence | Coverage scores |
| Dependency Reasoning | depends_on chains | Transitive dependencies |

---

## 2. Inference Rules

Rules are data-driven, not hardcoded. Each rule has:

```python
@dataclass(frozen=True)
class InferenceRule:
    rule_id:     str      # e.g. "IR-001"
    name:        str
    antecedent:  str      # Pattern that triggers the rule
    consequent:  str      # What is inferred
    confidence:  float    # Confidence of the inferred relationship
    invertible:  bool     # Whether reverse also applies
```

**Built-in rules:**

| ID | Antecedent | Consequent | Confidence |
|----|-----------|-----------|-----------|
| IR-001 | A implements B | B implemented_by A | 0.95 |
| IR-002 | A depends_on B, B depends_on C | A transitively_depends_on C | 0.80 |
| IR-003 | A supersedes B | B superseded_by A | 0.95 |
| IR-004 | A implemented_by B, B tested_by C | A has_test_coverage C | 0.75 |
| IR-005 | A and B are in same capability | A related_to B | 0.60 |
| IR-006 | A has no inbound edges | A is_orphan | 1.00 |
| IR-007 | A.status=approved AND evidence missing | A has_coverage_gap | 0.90 |
| IR-008 | A.depends_on B AND B.status=deprecated | A has_stale_dependency | 1.00 |

---

## 3. Relationship Inference Algorithm

```
INFER_RELATIONSHIPS(graph, rules):
  inferred = []
  
  FOR rule IN rules:
    matches = find_antecedent_matches(graph, rule)
    FOR match IN matches:
      new_rel = apply_rule(rule, match)
      IF new_rel NOT ALREADY IN graph.edges:
        inferred.append(InferredRelationship(
          from_id=new_rel.from_id,
          to_id=new_rel.to_id,
          rel_type=new_rel.type,
          confidence=rule.confidence,
          rule_id=rule.rule_id
        ))
        IF rule.invertible:
          inferred.append(reverse(new_rel, confidence=rule.confidence))
  
  RETURN inferred

Complexity: O(R × E) where R = rule count, E = edge count
```

---

## 4. Evidence Propagation

Evidence propagates from implementation/test documents to architecture documents
following the traceability chain: Architecture ← Implementation ← Test ← Evidence.

```
PROPAGATE_EVIDENCE(graph, evidence_registry):
  confidence_map = {}
  
  FOR arch_doc IN registry.by_type("architecture"):
    impl_docs = graph.neighbors(arch_doc.id, labels={"implemented_by"})
    
    evidence_scores = []
    FOR impl IN impl_docs:
      evidence = evidence_registry.get(impl)
      IF evidence:
        evidence_scores.append(evidence.confidence)
    
    IF evidence_scores:
      propagated = mean(evidence_scores) * 0.85  # Decay factor
      confidence_map[arch_doc.id] = propagated
    ELSE:
      confidence_map[arch_doc.id] = 0.0
  
  RETURN confidence_map
```

---

## 5. Confidence Propagation

Confidence flows from evidence through the traceability chain:

```
PROPAGATE_CONFIDENCE(graph, base_confidence):
  visited = {}
  
  FOR root IN topological_sort(graph):
    declared = base_confidence.get(root, None)
    IF declared is not None:
      visited[root] = declared
      CONTINUE
    
    # Derive from predecessors
    preds = graph.edges_to(root)
    IF NOT preds: visited[root] = 0.0; CONTINUE
    
    pred_confidences = [visited.get(e.from_id, 0.0) for e in preds]
    # Conservative: min of predecessors with decay
    visited[root] = min(pred_confidences) * 0.90
  
  RETURN visited

Complexity: O(V + E)  (topological traversal)
```

---

## 6. Dependency Reasoning

Computes all transitive dependencies:

```
TRANSITIVE_DEPS(id, graph, max_depth=10):
  deps = set()
  queue = deque([(id, 0)])
  WHILE queue:
    current, depth = queue.popleft()
    IF depth >= max_depth: CONTINUE
    FOR edge IN graph.edges_from(current):
      IF edge.label == "depends_on":
        IF edge.to_id NOT IN deps:
          deps.add(edge.to_id)
          queue.append((edge.to_id, depth + 1))
  RETURN deps
```

---

## 7. Capability Inference

Infers which capability a document belongs to when not explicitly declared:

```
INFER_CAPABILITY(obj, graph):
  # Method 1: Majority neighbor vote
  neighbors = graph.neighbors(obj.id)
  cap_votes = Counter(registry.get(n).capability for n in neighbors if registry.get(n))
  IF cap_votes: RETURN cap_votes.most_common(1)[0][0], confidence=0.70
  
  # Method 2: Path-based inference
  path_parts = obj.path.split("/")
  FOR known_cap IN catalog.all_capabilities():
    IF any(part.replace("-","_") == known_cap for part in path_parts):
      RETURN known_cap, confidence=0.80
  
  RETURN None, confidence=0.0
```

---

## 8. InferredRelationship Model

```python
@dataclass(frozen=True)
class InferredRelationship:
    from_id:          str
    to_id:            str
    relationship_type: str
    confidence:       float
    rule_id:          str
    basis:            str    # Human-readable explanation
```

---

## References

- [KA-SPEC-011](../knowledge-core/11-KNOWLEDGE-TRACEABILITY-MODEL.md) — Traceability chain
- [02-KNOWLEDGE-GRAPH-RUNTIME.md](02-KNOWLEDGE-GRAPH-RUNTIME.md) — Graph for traversal
- [08-EVIDENCE-ENGINE.md](08-EVIDENCE-ENGINE.md) — Evidence input to propagation
