---
knowledge_id: KA-KIP-004
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
  - id: KA-SPEC-007
    reason: "KQL language specification"
  - id: KA-KIP-003
    reason: "TRAVERSE and PATHS require the graph runtime"
---

# Knowledge Query Engine

## KQL Parser → AST → Planner → Optimizer → Executor

---

## 1. Pipeline

```
KQL string
    │
    ▼ Lexer
Token stream
    │
    ▼ Parser (recursive descent)
AST (KQLQuery node tree)
    │
    ▼ Semantic Analyzer
Validated AST + symbol resolution
    │
    ▼ Planner
QueryPlan (ordered execution steps)
    │
    ▼ Optimizer
Optimized QueryPlan (predicate push-down, index selection)
    │
    ▼ Executor
QueryResult (list[dict], metadata)
```

---

## 2. KQL Grammar (EBNF Subset)

```
Query      ::= Clause+
Clause     ::= Select | From | Where | Traverse | Paths
             | Neighbors | OrderBy | Limit | GroupBy | Having

Select     ::= "SELECT" (Field ("," Field)* | "*")
From       ::= "FROM" Source
Where      ::= "WHERE" Predicate
Traverse   ::= "TRAVERSE" Direction? ("DEPTH" Integer)?
Paths      ::= "PATHS" "FROM" Id "TO" Id
Neighbors  ::= "NEIGHBORS" "OF" Id ("TYPE" TypeList)?
OrderBy    ::= "ORDER" "BY" Field ("ASC" | "DESC")?
Limit      ::= "LIMIT" Integer
GroupBy    ::= "GROUP" "BY" Field
Having     ::= "HAVING" Predicate

Source     ::= "knowledge" | "capability" "(" String ")"
             | "domain" "(" String ")" | "knowledge.archive"

Predicate  ::= Comparison (("AND" | "OR") Comparison)* | "NOT" Predicate
Comparison ::= Field Op Value | Field "IN" "(" ValueList ")"
             | Field "LIKE" String | Field "IS" ("NULL" | "NOT" "EMPTY")
             | Field "CONTAINS" String

Direction  ::= "FORWARD" | "REVERSE" | "BOTH"
Op         ::= "=" | "!=" | "<" | ">" | "<=" | ">="
```

---

## 3. AST Node Hierarchy

```python
class ASTNode:  pass

class KQLQuery(ASTNode):
    select:    SelectNode | None
    from_:     FromNode
    where:     WhereNode | None
    traverse:  TraverseNode | None
    paths:     PathsNode | None
    neighbors: NeighborsNode | None
    order_by:  OrderByNode | None
    limit:     LimitNode | None
    group_by:  GroupByNode | None
    having:    HavingNode | None

class FromNode(ASTNode):
    source: str            # "knowledge", "capability", "domain"
    args:   list[str]      # capability name, domain name, etc.

class WhereNode(ASTNode):
    predicate: PredicateNode

class PredicateNode(ASTNode):  pass
class EqPredicate(PredicateNode):    field: str; value: Any
class InPredicate(PredicateNode):    field: str; values: list[Any]
class CmpPredicate(PredicateNode):   field: str; op: str; value: Any
class LikePredicate(PredicateNode):  field: str; pattern: str
class ContainsPredicate(PredicateNode): field: str; value: str
class NullPredicate(PredicateNode):  field: str; is_null: bool
class AndPredicate(PredicateNode):   left: PredicateNode; right: PredicateNode
class OrPredicate(PredicateNode):    left: PredicateNode; right: PredicateNode
class NotPredicate(PredicateNode):   operand: PredicateNode

class TraverseNode(ASTNode):
    direction: str    # "forward", "reverse", "both"
    depth: int | None # None = unlimited
    from_id: str | None

class PathsNode(ASTNode):
    from_id: str; to_id: str

class NeighborsNode(ASTNode):
    id: str; types: list[str]
```

---

## 4. Query Plan

```python
class PlanStep:
    op: str       # "scan", "filter", "index_lookup", "traverse", "sort", "limit"
    args: dict

class QueryPlan:
    steps: list[PlanStep]
    estimated_cost: float
    index_used: str | None
```

Planner rules:
- `WHERE id = X` → `index_lookup(registry, id=X)` O(1)
- `WHERE type = T` → `index_lookup(catalog.by_type, type=T)` O(1)
- `WHERE capability = C` → `index_lookup(catalog.by_capability, cap=C)` O(1)
- `WHERE health_score < N` → `health_index_scan(N)` O(k)
- `TRAVERSE FROM X DEPTH d` → `bfs(graph, start=X, depth=d)` O(b^d)
- Predicates pushed before traversal to reduce graph search space

---

## 5. Executor Examples

```python
# Example 1: Find all approved architectures in workflow domain
SELECT * FROM knowledge
WHERE domain = "DOM-WORKFLOW" AND type = "architecture" AND status = "approved"

# Example 2: Find what depends on this document
SELECT id, title, capability
FROM knowledge
WHERE depends_on CONTAINS "KA-SPEC-001"

# Example 3: Traverse impact
SELECT id, title
FROM knowledge
TRAVERSE REVERSE DEPTH 3
WHERE id = "KA-ARCH-001"

# Example 4: Find stale documents
SELECT id, title, review_date
FROM knowledge
WHERE review_date < "2026-01-01"
ORDER BY review_date ASC
LIMIT 20

# Example 5: Paths between two capabilities
PATHS FROM "KA-ARCH-001" TO "KA-SPEC-020"
```

---

## 6. QueryResult Structure

```python
@dataclass
class QueryResult:
    rows:             list[dict]
    count:            int
    execution_time_ms: float
    query:            str
    plan_used:        str
    index_used:       str | None
    warnings:         list[str]
```

---

## 7. Complexity

| Query Pattern | Index Used | Time |
|--------------|-----------|------|
| `WHERE id = X` | Registry HashMap | O(1) |
| `WHERE type = T` | Catalog HashMap | O(1) |
| `WHERE capability = C` | Catalog HashMap | O(1) |
| `WHERE health_score < N` | HealthIndex SortedList | O(k) |
| `TRAVERSE DEPTH d` | Graph adjacency | O(b^d) |
| `TRAVERSE unlimited` | Full graph BFS | O(V+E) |
| `PATHS FROM A TO B` | BFS shortest path | O(V+E) |
| `WHERE LIKE pattern` | Inverted index | O(k) |

---

## References

- [KA-SPEC-007](../knowledge-core/07-KNOWLEDGE-QUERY-LANGUAGE.md) — Full KQL specification
- [02-KNOWLEDGE-GRAPH-RUNTIME.md](02-KNOWLEDGE-GRAPH-RUNTIME.md) — TRAVERSE executor backend
- [04-KNOWLEDGE-SEARCH.md](04-KNOWLEDGE-SEARCH.md) — LIKE query backend
