---
knowledge_id: KNW-KOS-ARCH-008
title: "KOS Knowledge Query Engine"
status: AUTHORITATIVE
phase: "3.0A"
version: "1.0.0"
created: "2026-06-30"
purpose: "Specify the Knowledge Query Engine: search types, query language, traversal patterns, and API"
canonical_source: "architecture/docs/kos/08-QUERY-ENGINE.md"
dependencies:
  - "05-KNOWLEDGE-GRAPH.md"
  - "07-KNOWLEDGE-REGISTRY.md"
related_documents:
  - "09-REASONING-ENGINE.md"
  - "10-EXECUTION-ENGINE.md"
acceptance_criteria:
  - "All 8 query types are specified"
  - "Query language syntax is defined"
  - "Response format is standardized"
verification_checklist:
  - "[ ] Semantic search specified"
  - "[ ] Graph traversal query specified"
  - "[ ] Natural language query interface specified"
future_extensions:
  - "Vector embedding search for semantic similarity"
  - "Query caching and materialized views"
---

# KOS Knowledge Query Engine

## Supported Query Types

The Knowledge Query Engine supports all query patterns needed to navigate and
reason over the Knowledge Graph.

---

## Query Type 1 — ID Lookup

Direct lookup by Knowledge ID.

```python
result = query_engine.get("KNW-AI-MOD-quota_manager")
# Returns: KnowledgeObject or None
```

**CLI:**
```bash
platform-kos query --id KNW-AI-MOD-quota_manager
```

---

## Query Type 2 — Semantic Search

Full-text search over `name`, `description`, `tags`, and `evidence` fields.
Ranked by relevance score.

```python
results = query_engine.search("quota enforcement AI runtime")
# Returns: list[KnowledgeObject] ordered by relevance
```

**CLI:**
```bash
platform-kos query --search "quota enforcement"
platform-kos query --search "import rules" --type architecture
```

---

## Query Type 3 — Graph Traversal

Follow named edge types from a root node to a specified depth.

```python
# Find all modules a requirement is fulfilled by (transitively)
results = query_engine.traverse(
    start="KNW-REQ-001",
    edge_types=["FULFILLED_BY", "IMPLEMENTS"],
    direction="outbound",
    max_depth=5,
)
```

**CLI:**
```bash
platform-kos query --traverse KNW-REQ-001 --edge FULFILLED_BY,IMPLEMENTS --depth 5
```

---

## Query Type 4 — Relationship Query

Find all objects related to a given object by specific edge type.

```python
# What depends on quota_manager?
dependents = query_engine.relations(
    knowledge_id="KNW-AI-MOD-quota_manager",
    edge_type="DEPENDS_ON",
    direction="inbound",
)
# Returns: list of objects that DEPEND_ON quota_manager
```

**CLI:**
```bash
# What depends on this?
platform-kos query --dependents KNW-AI-MOD-quota_manager

# What does this depend on?
platform-kos query --dependencies KNW-AI-MOD-quota_manager
```

---

## Query Type 5 — Dependency Query

Compute the full transitive dependency closure.

```python
# All transitive dependencies of ai_runtime
all_deps = query_engine.dependency_closure("KNW-AI-RUNTIME-001")
```

**CLI:**
```bash
platform-kos query --closure KNW-AI-RUNTIME-001
platform-kos query --closure KNW-AI-RUNTIME-001 --format tree
```

---

## Query Type 6 — Coverage Query

Check what percentage of requirements have fulfilling implementations.

```python
coverage = query_engine.coverage(
    domain="ai",
    requirement_status=["CANONICAL"],
    fulfillment_status=["CANONICAL", "APPROVED"],
)
# Returns: CoverageReport
```

**CLI:**
```bash
platform-kos query --coverage --domain ai
platform-kos query --coverage --type requirement
```

---

## Query Type 7 — Reasoning Query

Delegated to the Reasoning Engine (09-REASONING-ENGINE.md). The Query Engine
provides the data access layer; the Reasoning Engine adds interpretation.

```python
result = query_engine.reason("What is the impact of changing quota_manager?")
# Delegates to KnowledgeReasoningEngine
```

---

## Query Type 8 — Natural Language Query

Transform natural language into a structured graph query.
Powered by the AI Runtime's intelligence engine.

```python
result = query_engine.ask("Which modules depend on the event bus?")
# Pipeline:
#   1. Parse natural language to structured query
#   2. Execute graph traversal
#   3. Format result as natural language answer
```

**CLI:**
```bash
platform-kos ask "Which modules depend on the event bus?"
platform-kos ask "What changed in the AI runtime since version 2.1?"
platform-kos ask "Show the traceability chain for quota enforcement"
```

---

## KQL — Knowledge Query Language

A minimal structured query language for complex queries:

```
# Syntax: [SELECT|TRAVERSE|FIND] [object_type] [WHERE conditions] [VIA edge_types] [DEPTH N]

# Find all modules owned by AI Runtime Team
SELECT module WHERE owner = "AI Runtime Team"

# Find all requirements not yet fulfilled
SELECT requirement WHERE FULFILLED_BY.count = 0

# Traverse graph from requirement to deployment
TRAVERSE FROM KNW-REQ-001 VIA FULFILLED_BY, IMPLEMENTS, EXPOSES, TESTED_BY, VALIDATES DEPTH 10

# Find all tests that cover quota-related requirements
SELECT test WHERE COVERS.tags CONTAINS "quota"

# Find all objects with quality_score < 0.7
SELECT * WHERE quality_score < 0.7 AND status = "CANONICAL"

# Find impact of changing a module
FIND DEPENDENTS OF KNW-AI-MOD-quota_manager DEPTH 5
```

---

## Query Engine Interface

```python
class KnowledgeQueryEngine(Protocol):
    def get(self, knowledge_id: str) -> KnowledgeObject | None: ...

    def search(
        self,
        query: str,
        object_types: list[KnowledgeObjectType] | None = None,
        status: KnowledgeLifecycleStatus | None = None,
        limit: int = 20,
    ) -> list[KnowledgeObject]: ...

    def traverse(
        self,
        start: str,
        edge_types: list[str],
        direction: str = "outbound",
        max_depth: int = 10,
    ) -> list[KnowledgeObject]: ...

    def relations(
        self,
        knowledge_id: str,
        edge_type: str,
        direction: str = "outbound",
    ) -> list[KnowledgeObject]: ...

    def dependency_closure(self, knowledge_id: str) -> set[str]: ...
    # Returns set of all transitive dependency knowledge_ids

    def coverage(
        self,
        domain: str | None = None,
        requirement_status: list[str] | None = None,
        fulfillment_status: list[str] | None = None,
    ) -> CoverageReport: ...

    def kql(self, query: str) -> list[KnowledgeObject]: ...
    # Execute a KQL query string

    def ask(self, question: str) -> NaturalLanguageAnswer: ...
    # Natural language query (delegates to ReasoningEngine)
```

---

## CoverageReport

```python
@dataclass(frozen=True)
class CoverageReport:
    domain: str
    total_requirements: int
    fulfilled_requirements: int
    unfulfilled_requirements: int
    coverage_pct: float                      # 0.0–100.0
    unfulfilled_ids: list[str]
    test_coverage_pct: float
    generated_at: datetime
```

---

## Performance Targets

| Query Type | P50 | P99 |
|------------|-----|-----|
| ID Lookup | < 1 ms | < 5 ms |
| Semantic Search (top 20) | < 100 ms | < 500 ms |
| Graph Traversal (depth 5) | < 50 ms | < 200 ms |
| Dependency Closure | < 200 ms | < 1 s |
| Coverage Report | < 500 ms | < 2 s |
| Natural Language Query | < 2 s | < 5 s |
