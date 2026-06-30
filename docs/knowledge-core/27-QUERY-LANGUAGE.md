# KNW-KC-ARCH-027 — Query Language

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Knowledge Query Language (KQL) is the canonical query interface for all Knowledge Objects, relationships, and graph traversals. KQL is declarative, strongly typed, and compiles to index lookups and graph algorithms.

---

## Query Types (8)

| Type | Code | Description |
|------|------|-------------|
| Identity Query | `ID` | Retrieve an object by knowledge_id, URI, or canonical name |
| Relationship Query | `REL` | Find all relationships of a given type from/to an object |
| Dependency Query | `DEP` | Compute dependency closure for an object |
| Impact Query | `IMP` | Find all objects impacted by a change to the target |
| Reasoning Query | `RSN` | Ask why/how/what-if questions about objects |
| Coverage Query | `COV` | Compute traceability coverage for a namespace or requirement set |
| Evidence Query | `EV` | Retrieve evidence records for an object or set |
| Natural Language Query | `NL` | Translate free-text question into KQL (semantic layer) |

---

## KQL Syntax

```
STATEMENT := SELECT CLAUSE [WHERE CLAUSE] [TRAVERSE CLAUSE] [LIMIT CLAUSE]

SELECT CLAUSE:
  SELECT *
  SELECT {field, field, ...}
  SELECT COUNT(*)
  SELECT DISTINCT namespace

WHERE CLAUSE:
  WHERE knowledge_id = "KNW-KOS-ARCH-001"
  WHERE namespace = "kos"
  WHERE object_type IN [ARCHITECTURE, DECISION]
  WHERE lifecycle.status = "CANONICAL"
  WHERE quality.overall_score >= 0.80
  WHERE owner = "team:architecture-board"
  WHERE "kos" IN tags
  WHERE created_at >= "2026-01-01"
  WHERE knowledge_uri MATCHES "knw://kos/*"

TRAVERSE CLAUSE:
  TRAVERSE depends_on DEPTH 3
  TRAVERSE implements INVERSE
  TRAVERSE [depends_on, imports] FORWARD DEPTH 5

LIMIT CLAUSE:
  LIMIT 100 OFFSET 0
  ORDER BY quality.overall_score DESC
```

---

## KQL Examples

### Identity Queries
```kql
-- Get by ID
SELECT * WHERE knowledge_id = "KNW-KOS-ARCH-001"

-- Get by URI
SELECT * WHERE knowledge_uri = "knw://kos/arch/vision"

-- Get by canonical name
SELECT * WHERE canonical_name = "kos.arch.vision"

-- Get all CANONICAL architectures
SELECT knowledge_id, name, version, quality.overall_score
WHERE object_type = ARCHITECTURE
AND lifecycle.status = "CANONICAL"
ORDER BY quality.overall_score DESC
LIMIT 20
```

### Relationship Queries
```kql
-- All objects that KNW-AI-MOD-001 depends on
SELECT * FROM KNW-AI-MOD-001
TRAVERSE depends_on FORWARD DEPTH 1

-- All modules that depend on service.auth
SELECT * FROM KNW-PLAT-SVC-001
TRAVERSE depends_on INVERSE DEPTH 1

-- Full dependency closure (transitive)
SELECT * FROM KNW-AI-RUN-001
TRAVERSE depends_on FORWARD DEPTH 10
```

### Dependency Queries
```kql
-- All direct and transitive dependencies of the AI runtime
DEPENDENCY_CLOSURE KNW-AI-RUN-001 USING depends_on

-- Dependency cycle check
CYCLE_CHECK namespace="platform" relation_type=depends_on
```

### Impact Queries
```kql
-- What will break if I deprecate service.auth?
IMPACT KNW-PLAT-SVC-001 RELATION depends_on,uses DEPTH 5

-- Impact sorted by severity
IMPACT KNW-PLAT-SVC-001 ORDER BY blast_radius DESC
```

### Coverage Queries
```kql
-- Traceability coverage for KOS namespace
COVERAGE namespace="kos" TYPE traceability

-- Which requirements have no implementation?
COVERAGE namespace="kos" TYPE traceability GAPS_ONLY

-- Quality coverage across all modules
COVERAGE namespace="platform" TYPE quality MIN_SCORE 0.70
```

### Evidence Queries
```kql
-- All evidence for an object
EVIDENCE KNW-KOS-ARCH-001

-- Stale evidence across namespace
EVIDENCE namespace="kos" WHERE freshness_score < 0.30

-- Objects missing required evidence for CANONICAL
EVIDENCE_GAP namespace="kos" TARGET_STATUS CANONICAL
```

---

## KQL Execution Model

```
PARSE(kql_string) → AST
  1. Lexer: tokenise keywords, identifiers, literals
  2. Parser: validate grammar, build AST
  3. Semantic analysis: resolve object_types, rel_types, namespaces

PLAN(AST) → ExecutionPlan
  1. SELECT → identify output fields
  2. WHERE → choose index(es) to satisfy predicates
  3. TRAVERSE → choose DFS or BFS based on depth
  4. LIMIT → apply before sorting if COUNT-only

EXECUTE(ExecutionPlan) → ResultSet
  1. Index lookups → O(1) per predicate
  2. Set intersection for multiple predicates
  3. Graph traversal using 25-GRAPH-ALGORITHMS
  4. Apply ORDER BY (sort)
  5. Apply LIMIT/OFFSET
  6. Return ResultSet{rows, total_count, execution_ms}

COMPLEXITY: O(T + N*) where T=token count, N*=matching nodes
```

---

## KQL Result Schema

```python
@dataclass
class KQLResult:
    query: str
    rows: list[dict]
    total_count: int
    returned_count: int
    execution_ms: float
    index_hits: list[str]
    plan: str
    warnings: list[str]
```

---

## KQL Query Protocol

```python
class KnowledgeQueryEngine(Protocol):
    def execute(self, kql: str) -> KQLResult: ...
    def parse(self, kql: str) -> KQLAST: ...
    def plan(self, ast: KQLAST) -> ExecutionPlan: ...
    def dependency_closure(self, knowledge_id: str) -> set[str]: ...
    def impact(self, knowledge_id: str, depth: int) -> ImpactReport: ...
    def coverage(self, namespace: str, coverage_type: str) -> CoverageReport: ...
    def evidence_gap(self, namespace: str, target_status: str) -> list[str]: ...
    def natural_language(self, question: str) -> KQLResult: ...
```

---

## Performance Requirements

| Query Type | P50 | P99 |
|-----------|-----|-----|
| ID lookup | < 1ms | < 5ms |
| Namespace scan (1000 objects) | < 10ms | < 50ms |
| 1-hop traversal | < 2ms | < 10ms |
| 5-hop traversal | < 20ms | < 100ms |
| Dependency closure | < 50ms | < 200ms |
| Natural language query | < 500ms | < 2s |

---

## CLI Integration

```bash
platform-kos query "SELECT * WHERE namespace = 'kos' AND status = 'CANONICAL'"
platform-kos query --file queries/coverage.kql
platform-kos impact KNW-PLAT-SVC-001
platform-kos closure KNW-AI-RUN-001
```

---

## Cross-References

- Indexes used for query execution → `26-GRAPH-INDEXES`
- Graph algorithms for traversal → `25-GRAPH-ALGORITHMS`
- Natural language layer → `29-SEMANTIC-LAYER`
- Reasoning queries → `30-REASONING-MODEL`
