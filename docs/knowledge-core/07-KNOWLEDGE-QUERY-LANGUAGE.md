---
knowledge_id: KA-SPEC-007
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
  - id: KA-SPEC-004
    reason: "KQL uses Knowledge URIs as operands"
  - id: KA-SPEC-008
    reason: "KQL traverses the Knowledge Graph"
---

# Knowledge Query Language

## KQL — A Declarative Language for Querying the Knowledge Graph

---

## 1. Purpose

The Knowledge Query Language (KQL) enables declarative, structured queries over the AI Studio knowledge graph. It answers questions that cannot be answered by reading a single file:

- What capabilities depend on provider-framework?
- Which architecture documents have no evidence?
- What changed between Phase 2.0D.0 and Phase 2.0D.1?
- What is the complete impact of deprecating KA-ARCH-005?
- Which documents are stale and owned by a specific role?

KQL is the query interface for the Knowledge Index Engine and Documentation Intelligence Platform.

---

## 2. Design Principles

| Principle | Description |
|-----------|-------------|
| **Declarative** | Express what you want, not how to get it |
| **Graph-aware** | First-class support for relationship traversal |
| **Composable** | Queries compose with subqueries |
| **Safe** | Read-only — no mutations via KQL |
| **Explainable** | Every result includes its derivation path |
| **Type-safe** | Results are typed KnowledgeObjects or projections |

---

## 3. Basic Query Syntax

### 3.1 SELECT

```kql
-- Select all approved architecture documents
SELECT *
FROM knowledge
WHERE type = 'architecture'
  AND status = 'approved'

-- Select specific fields
SELECT knowledge_id, title, capability, health_score
FROM knowledge
WHERE domain = 'DOM-WORKFLOW'
  AND status = 'approved'
ORDER BY health_score DESC

-- Count by type
SELECT type, COUNT(*) as count
FROM knowledge
WHERE status = 'approved'
GROUP BY type
ORDER BY count DESC
```

### 3.2 FROM Clauses

```kql
-- Query all knowledge objects
FROM knowledge

-- Query specific capability
FROM capability('workflow-runtime')

-- Query specific domain
FROM domain('DOM-WORKFLOW')

-- Query archived objects
FROM knowledge.archive

-- Query specific version history
FROM knowledge.history('KA-ARCH-001')
```

### 3.3 WHERE Predicates

```kql
-- Equality
WHERE type = 'architecture'
WHERE status = 'approved'
WHERE canonical = true

-- Set membership
WHERE type IN ('architecture', 'api', 'database')
WHERE domain IN ('DOM-WORKFLOW', 'DOM-AI')

-- Comparison
WHERE health_score < 60
WHERE review_date < '2026-01-01'

-- Pattern matching
WHERE capability LIKE 'provider-*'
WHERE tags CONTAINS 'workflow'

-- Null checks
WHERE superseded_by IS NULL
WHERE evidence IS NOT EMPTY

-- Compound conditions
WHERE status = 'approved'
  AND health_score < 60
  AND review_date < TODAY() - DAYS(30)
```

---

## 4. Graph Traversal Syntax

### 4.1 TRAVERSE

```kql
-- Find all documents that depend on KA-ARCH-005
TRAVERSE depends_on
  FROM KA-ARCH-005
  DIRECTION reverse         -- Follow inbound edges (who depends on me)
  DEPTH unlimited
  RETURN node

-- Find 3-hop dependency chain from workflow-runtime
TRAVERSE depends_on
  FROM capability('workflow-runtime')
  DIRECTION forward
  DEPTH 3
  RETURN path

-- Impact analysis: what would be affected if KA-ARCH-001 is deprecated?
TRAVERSE [depends_on, implemented_by, related_apis]
  FROM KA-ARCH-001
  DIRECTION reverse
  DEPTH unlimited
  RETURN { node, path_length, relationship_type }
```

### 4.2 PATHS

```kql
-- Find all paths between two knowledge objects
PATHS
  FROM KA-ARCH-001
  TO KA-ARCH-005
  VIA [depends_on, implements]
  MAX_LENGTH 5

-- Shortest path
SHORTEST PATH
  FROM capability('workflow-runtime')
  TO capability('provider-framework')
  VIA depends_on
```

### 4.3 NEIGHBORS

```kql
-- Immediate neighbors (1 hop)
NEIGHBORS OF KA-ARCH-001
  RELATIONSHIPS [depends_on, implements, implemented_by]

-- All relationships for an object
RELATIONSHIPS OF KA-ARCH-001
  DIRECTION both
  RETURN { type, target, direction }
```

---

## 5. Aggregation and Analysis

### 5.1 Health Analysis

```kql
-- Domain health scorecard
SELECT domain, AVG(health_score) as avg_score, MIN(health_score) as min_score,
       COUNT(*) as doc_count,
       COUNT(*) FILTER (WHERE health_score < 60) as unhealthy_count
FROM knowledge
WHERE status = 'approved'
GROUP BY domain
ORDER BY avg_score ASC

-- Stale documents
SELECT knowledge_id, title, owner, review_date,
       DAYS_SINCE(review_date) as days_overdue
FROM knowledge
WHERE status = 'approved'
  AND review_date < TODAY()
  AND type NOT IN ('adr', 'history')
ORDER BY days_overdue DESC
```

### 5.2 Coverage Analysis

```kql
-- Capabilities missing required documents
SELECT capability,
       MISSING_REQUIRED_FILES() as missing
FROM capability(*)
WHERE MISSING_REQUIRED_FILES() IS NOT EMPTY

-- Architecture documents without evidence
SELECT knowledge_id, title, capability, confidence
FROM knowledge
WHERE type = 'architecture'
  AND status = 'approved'
  AND (evidence IS EMPTY OR evidence[?(@.verified==true)] IS EMPTY)
ORDER BY capability
```

### 5.3 Duplicate Detection

```kql
-- Find duplicate canonical conflicts
SELECT capability, type, COLLECT(knowledge_id) as ids
FROM knowledge
WHERE status = 'approved'
  AND canonical = true
GROUP BY capability, type
HAVING COUNT(*) > 1

-- Find content similarity (fuzzy — runs semantic comparison)
FIND SIMILAR TO KA-ARCH-001
  IN domain('DOM-WORKFLOW')
  SIMILARITY_THRESHOLD 0.85
  RETURN { id, title, similarity_score }
```

### 5.4 Orphan and Dead Document Detection

```kql
-- Orphans: approved documents with no relationships
SELECT knowledge_id, title, capability, type
FROM knowledge
WHERE status = 'approved'
  AND INBOUND_COUNT() = 0
  AND OUTBOUND_COUNT() = 0
  AND type NOT IN ('overview', 'vision')

-- Dead documents: not in any index
SELECT knowledge_id, title
FROM knowledge.all   -- Includes non-indexed
WHERE knowledge_id NOT IN (
  SELECT knowledge_id FROM knowledge
)
```

---

## 6. Evolution Queries

```kql
-- What changed between two phases?
DIFF
  FROM knowledge WHERE phase = '2.0D.0'
  TO   knowledge WHERE phase = '2.0D.0.5'
  RETURN { added, removed, modified }

-- Supersession chain for a document
CHAIN superseded_by
  FROM KA-ARCH-001
  DIRECTION backward    -- Follow to root (oldest)
  RETURN { id, title, version, deprecated_date }

-- Documents deprecated in the last 90 days
SELECT knowledge_id, title, deprecated_date, superseded_by
FROM knowledge
WHERE status = 'deprecated'
  AND deprecated_date > TODAY() - DAYS(90)
ORDER BY deprecated_date DESC
```

---

## 7. KQL Functions Reference

| Function | Returns | Description |
|----------|---------|-------------|
| `TODAY()` | Date | Current date |
| `DAYS(n)` | Duration | Duration of n days |
| `DAYS_SINCE(date)` | Integer | Days elapsed since date |
| `MISSING_REQUIRED_FILES()` | String[] | Required files not present |
| `INBOUND_COUNT()` | Integer | Number of inbound relationship edges |
| `OUTBOUND_COUNT()` | Integer | Number of outbound relationship edges |
| `HEALTH_SCORE()` | Integer | Computed health score (0–100) |
| `COLLECT(field)` | Array | Aggregate field values into array |
| `RESOLVE(id)` | KnowledgeObject | Dereference a Knowledge ID |
| `URI(id)` | String | Get full URI for a Knowledge ID |

---

## 8. KQL Execution Model

### 8.1 Query Processing Pipeline

```
KQL Text
  ↓ Lexer
Token Stream
  ↓ Parser
Abstract Syntax Tree (AST)
  ↓ Semantic Analyzer
  |   - Type checking
  |   - ID resolution
  |   - Schema validation
Validated AST
  ↓ Query Optimizer
  |   - Index selection
  |   - Join ordering
  |   - Pushdown predicates
Execution Plan
  ↓ Executor
  |   - Index lookups
  |   - Graph traversals
  |   - Aggregations
Result Set
  ↓ Formatter
  |   - JSON / YAML / Markdown / Table
Output
```

### 8.2 Query Safety

KQL is **read-only**. There are no INSERT, UPDATE, or DELETE operations. All mutations to the knowledge system go through the document authoring workflow with full audit trail.

### 8.3 Result Format

```json
{
  "query": "SELECT knowledge_id, title FROM knowledge WHERE type = 'architecture'",
  "executed_at": "2026-06-29T14:00:00Z",
  "duration_ms": 12,
  "total_results": 12,
  "results": [
    {
      "knowledge_id": "KA-ARCH-001",
      "title": "Workflow Runtime Architecture"
    }
  ]
}
```

---

## References

- [08-KNOWLEDGE-GRAPH-MODEL.md](08-KNOWLEDGE-GRAPH-MODEL.md) — Graph that KQL traverses
- [10-KNOWLEDGE-INDEX-ENGINE.md](10-KNOWLEDGE-INDEX-ENGINE.md) — Indexes that KQL queries
- [04-KNOWLEDGE-URI-SPECIFICATION.md](04-KNOWLEDGE-URI-SPECIFICATION.md) — URIs used in KQL operands
