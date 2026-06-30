# KNW-KE-ARCH-022 — Knowledge Reference Library

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Reference Library is the curated collection of canonical, stable knowledge objects that all other packages and runtimes depend on. Objects in `knowledge/reference/` are the most authoritative sources in the ecosystem.

---

## Reference Library Contents

```
knowledge/reference/
├── algorithms/       # all 19 algorithm objects (CANONICAL)
├── patterns/         # all 16 pattern objects (CANONICAL)
├── requirements/     # platform-wide requirements (CANONICAL)
├── decisions/        # major architectural ADRs (CANONICAL)
├── benchmarks/       # all 19 benchmark objects (CANONICAL)
├── schemas/          # object type schemas (mirrors knowledge/schemas/)
└── glossary.yaml     # canonical terminology definitions
```

---

## Reference Library Rules

| Rule | Description |
|------|-------------|
| RL-001 | All objects in `reference/` must be CANONICAL status |
| RL-002 | Objects in `reference/` are immutable without ADR + Architecture Board sign-off |
| RL-003 | Objects in `reference/` are read-only in CI (no automated updates) |
| RL-004 | Every object in `reference/` must have at least one cross-package reference |
| RL-005 | `reference/glossary.yaml` is the single source of terminology |

---

## Reference Algorithm Objects

All 19 algorithms from Phase 3.0C `36-ALGORITHMS` and `25-GRAPH-ALGORITHMS` have canonical reference objects:

| Knowledge ID | Algorithm | Status |
|-------------|-----------|--------|
| KNW-ALG-ALG-001 | A-01 Knowledge ID Generation | CANONICAL |
| KNW-ALG-ALG-002 | A-02 Checksum Computation | CANONICAL |
| KNW-ALG-ALG-003 | A-03 Fingerprint Computation | CANONICAL |
| KNW-ALG-ALG-004 | A-04 Quality Score Computation | CANONICAL |
| KNW-ALG-ALG-005 | A-05 Evidence Freshness Decay | CANONICAL |
| KNW-ALG-ALG-006 | A-06 Confidence Decay | CANONICAL |
| KNW-ALG-ALG-007 | A-07 BM25 Text Ranking | CANONICAL |
| KNW-ALG-ALG-008 | A-08 Hybrid Search Scoring | CANONICAL |
| KNW-ALG-ALG-009 | A-09 Traceability Coverage | CANONICAL |
| KNW-ALG-ALG-010 | A-10 Confidence Path Product | CANONICAL |
| KNW-ALG-ALG-011 | A-11 Cosine Similarity | CANONICAL |
| KNW-ALG-ALG-012 | A-12 Registry Bulk Validation | CANONICAL |
| KNW-ALG-ALG-013 | G-01 DFS Graph Traversal | CANONICAL |
| KNW-ALG-ALG-014 | G-02 BFS Graph Traversal | CANONICAL |
| KNW-ALG-ALG-015 | G-03 Tarjan SCC | CANONICAL |
| KNW-ALG-ALG-016 | G-04 Topological Sort | CANONICAL |
| KNW-ALG-ALG-017 | G-05 Dijkstra Shortest Path | CANONICAL |
| KNW-ALG-ALG-018 | G-06 Dependency Closure | CANONICAL |
| KNW-ALG-ALG-019 | G-07 Graph Diff | CANONICAL |

---

## Reference Pattern Objects

All 16 patterns from Phase 3.0C `22-PATTERN-REGISTRY`:

| Knowledge ID | Pattern | Category |
|-------------|---------|---------|
| KNW-PAT-PAT-001 | Registry Pattern | Structural |
| KNW-PAT-PAT-002 | Strategy Pattern | Behavioral |
| KNW-PAT-PAT-003 | Factory Pattern | Creational |
| KNW-PAT-PAT-004 | Observer Pattern | Behavioral |
| KNW-PAT-PAT-005 | Adapter Pattern | Structural |
| KNW-PAT-PAT-006 | Facade Pattern | Structural |
| KNW-PAT-PAT-007 | Command Pattern | Behavioral |
| KNW-PAT-PAT-008 | Singleton Pattern | Creational |
| KNW-PAT-PAT-009 | Decorator Pattern | Structural |
| KNW-PAT-PAT-010 | Chain of Responsibility | Behavioral |
| KNW-PAT-PAT-011 | Event Sourcing | Architectural |
| KNW-PAT-PAT-012 | CQRS | Architectural |
| KNW-PAT-PAT-013 | Circuit Breaker | Architectural |
| KNW-PAT-PAT-014 | Saga Pattern | Architectural |
| KNW-PAT-PAT-015 | Plugin Pattern | Structural |
| KNW-PAT-PAT-016 | Provider Pattern | Structural |

---

## Canonical Glossary

```yaml
# knowledge/reference/glossary.yaml
version: "1.0.0"
terms:
  Knowledge Object:
    definition: >
      An atomic, versioned, traceable unit of knowledge in the KOS ecosystem.
      Every Knowledge Object has a unique knowledge_id, 16 Universal Schema
      sections, and belongs to exactly one package.
    see_also: [KnowledgeObjectType, Universal Schema]

  knowledge_id:
    definition: >
      The permanent, immutable identifier for a Knowledge Object.
      Format: KNW-{DOMAIN}-{TYPE}-{NNN}.
    see_also: [canonical_name, knowledge_uri]

  canonical_name:
    definition: >
      The human-readable, namespace-scoped identifier.
      Format: {namespace}.{type_lower}.{slug}.
    see_also: [knowledge_id, knowledge_uri]

  knowledge_uri:
    definition: >
      The addressable URI for a Knowledge Object.
      Format: knw://{namespace}/{type}/{slug}[@{version}].
    see_also: [canonical_name, knowledge_id]

  Package:
    definition: >
      A domain-scoped collection of Knowledge Objects with a single owner,
      version, and manifest. The unit of distribution.
    see_also: [Package Manager, Dependency Model]

  Evidence:
    definition: >
      A record linking a claim in a Knowledge Object to its source.
      Evidence has a type, weight, source_url, and freshness decay.
    see_also: [Evidence Engine, quality_score]

  Fingerprint:
    definition: >
      A short digest computed from namespace, type_code, and checksum.
      Used to detect semantic duplicates across knowledge_ids.
    see_also: [Checksum, Identity Engine]

  CANONICAL:
    definition: >
      The highest stable lifecycle state. CANONICAL objects are immutable
      (PATCH changes only), fully evidenced, quality-gated, and
      Architecture Board approved.
    see_also: [lifecycle, quality_score]

  Linter:
    definition: "Tool that enforces style, naming, and lifecycle rules. Command: kos lint."
    see_also: [Formatter, CI]

  Formatter:
    definition: "Tool that normalises YAML field order and style. Command: kos format."
    see_also: [Linter, CI]
```

---

## Cross-References

- Algorithms defined in → Phase 3.0C `36-ALGORITHMS`, `25-GRAPH-ALGORITHMS`
- Patterns defined in → Phase 3.0C `22-PATTERN-REGISTRY`
- Reference objects cross-referenced by → `09-KNOWLEDGE-CATALOG`
- Glossary used by → `04-KNOWLEDGE-STYLE-GUIDE` (SG-W-02)
