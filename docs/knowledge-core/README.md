# Knowledge Core Architecture

> Phase 3.0C — Complete architecture for the KOS Knowledge Core. Frozen 2026-06-30.

**Phase:** 3.0C | **Owner:** architecture-board | **Status:** CANONICAL | **Version:** 1.0.0

---

## What This Is

43 immutable architecture documents defining every concept, rule, algorithm, data structure, and contract needed to build the Knowledge Operating System (KOS) Knowledge Core. No implementation code — architecture only.

---

## Document Map

### Tier 1 — Foundation (read first)

| # | Document | One-Line Summary |
|---|----------|-----------------|
| 00 | [README](00-README.md) | Navigation guide and reading order |
| 01 | [Identity Engine](01-IDENTITY-ENGINE.md) | 14 identity fields; KNW ID format; fingerprint; duplicate detection |
| 02 | [Universal Schema](02-UNIVERSAL-SCHEMA.md) | 16 mandatory sections every object must carry |
| 03 | [Namespace System](03-NAMESPACE-SYSTEM.md) | 14 domains; 32 type codes; canonical name formula |
| 04 | [URI Specification](04-URI-SPECIFICATION.md) | `knw://` scheme; 4 version channels; resolution protocol |
| 05 | [Object Inheritance](05-OBJECT-INHERITANCE.md) | 8 abstract + 33 concrete types; hierarchy rules |

### Tier 2 — Core Engines

| # | Document | One-Line Summary |
|---|----------|-----------------|
| 06 | [Relationship Engine](06-RELATIONSHIP-ENGINE.md) | First-class relationships; O(N+E) traversal; propagation |
| 07 | [Relationship Types](07-RELATIONSHIP-TYPES.md) | 24 types; source/target constraints; cycle policies |
| 08 | [Version Engine](08-VERSION-ENGINE.md) | SemVer; version records; fork/merge; compatibility matrix |
| 09 | [Snapshot Engine](09-SNAPSHOT-ENGINE.md) | 6 snapshot types; restore; diff; tombstone model |

### Tier 3 — Quality Layer

| # | Document | One-Line Summary |
|---|----------|-----------------|
| 10 | [Evidence Engine](10-EVIDENCE-ENGINE.md) | 10 evidence types; weights; freshness decay model |
| 11 | [Quality Engine](11-QUALITY-ENGINE.md) | 9 dimensions QD-1–QD-9; weighted composite; health penalty |
| 12 | [Confidence Model](12-CONFIDENCE-MODEL.md) | declared×0.30 + evidence×0.70; 5 levels; domain half-lives |
| 13 | [Traceability](13-TRACEABILITY.md) | Req→Arch→Module→Test chain; coverage formula; gap detection |

### Tier 4 — Registries

| # | Document | One-Line Summary |
|---|----------|-----------------|
| 14 | [Registry Architecture](14-REGISTRY-ARCHITECTURE.md) | Master → 9 domain registries; write protocol O(log N) |
| 15 | [Object Registry](15-OBJECT-REGISTRY.md) | All 33 types; file format; 10 indexes |
| 16 | [Service Registry](16-SERVICE-REGISTRY.md) | 6 core services; discovery protocol |
| 17 | [API Registry](17-API-REGISTRY.md) | 8 API families; standard error envelope |
| 18 | [Event Registry](18-EVENT-REGISTRY.md) | Topic convention; 10 domains; delivery guarantees |
| 19 | [Runtime Registry](19-RUNTIME-REGISTRY.md) | 9 runtimes; boot order; health schema |
| 20 | [Product Registry](20-PRODUCT-REGISTRY.md) | 5 products; entry format |
| 21 | [Algorithm Registry](21-ALGORITHM-REGISTRY.md) | 24 algorithms with IDs, complexity, pseudocode |
| 22 | [Pattern Registry](22-PATTERN-REGISTRY.md) | 16 patterns in 3 groups |
| 23 | [Dataset Registry](23-DATASET-REGISTRY.md) | 8 categories; provenance; quality |

### Tier 5 — Graph & Query

| # | Document | One-Line Summary |
|---|----------|-----------------|
| 24 | [Knowledge Graph Model](24-KNOWLEDGE-GRAPH-MODEL.md) | Property graph G=(V,E); 5 invariants; edge weight formula |
| 25 | [Graph Algorithms](25-GRAPH-ALGORITHMS.md) | DFS, BFS, Tarjan SCC, TopoSort, Dijkstra, DepClosure, Diff |
| 26 | [Graph Indexes](26-GRAPH-INDEXES.md) | 10 node + 7 edge + 5 derived indexes; O(log N) maintenance |
| 27 | [Query Language](27-QUERY-LANGUAGE.md) | 8 KQL types; SELECT/WHERE/TRAVERSE/LIMIT; parse→plan→execute |

### Tier 6 — Intelligence

| # | Document | One-Line Summary |
|---|----------|-----------------|
| 28 | [Search Engine](28-SEARCH-ENGINE.md) | BM25(k1=1.2, b=0.75) + HNSW(768 dims); hybrid 0.40+0.40+0.20 |
| 29 | [Semantic Layer](29-SEMANTIC-LAYER.md) | NL→KQL 5-step pipeline; cosine similarity; dedup nightly |
| 30 | [Reasoning Model](30-REASONING-MODEL.md) | R-1 through R-10; impact severity thresholds |
| 31 | [Execution Context](31-EXECUTION-CONTEXT.md) | 6 plan types; required knowledge; lifecycle |

### Tier 7 — Governance

| # | Document | One-Line Summary |
|---|----------|-----------------|
| 32 | [Metadata Standard](32-METADATA-STANDARD.md) | 5 categories M1-M5; 8 MV rules; SLA tiers |
| 33 | [Lifecycle Extensions](33-LIFECYCLE-EXTENSIONS.md) | Domain substatus/gates/hooks; bypass policy |
| 34 | [State Machines](34-STATE-MACHINES.md) | SM-1 through SM-7 with all transitions/guards/effects |

### Tier 8 — Specification

| # | Document | One-Line Summary |
|---|----------|-----------------|
| 35 | [Data Structures](35-DATA-STRUCTURES.md) | DS-01 through DS-15 canonical dataclasses |
| 36 | [Algorithms](36-ALGORITHMS.md) | A-01 through A-12 non-graph algorithms with pseudocode |
| 37 | [Performance Budget](37-PERFORMANCE-BUDGET.md) | P50/P99 for every operation at 1M objects, 10M relations |

### Tier 9 — Freeze

| # | Document | One-Line Summary |
|---|----------|-----------------|
| 38 | [Verification](38-VERIFICATION.md) | 50 checks across 8 areas — all PASS |
| 39 | [Test Strategy](39-TEST-STRATEGY.md) | T1–T5 tiers; coverage targets; scenario descriptions |
| 40 | [Architecture Freeze](40-ARCHITECTURE-FREEZE.md) | Freeze declaration; change process; phase boundary |

---

## Reading Order

**Architect (start here):** 01 → 02 → 05 → 06 → 07 → 24 → 27 → 40

**Quality engineer:** 10 → 11 → 12 → 13 → 34 → 38

**Registry implementer:** 14 → 15 → 08 → 35 → 36 → 37

**Full study (11-tier):** 00 → 01–05 → 06–09 → 10–13 → 14–23 → 24–27 → 28–31 → 32–34 → 35–37 → 38–40

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Object types | 33 |
| Relationship types | 24 |
| Evidence types | 10 |
| Quality dimensions | 9 |
| State machines | 7 |
| Reasoning types | 10 |
| Algorithms | 19 (12 non-graph + 7 graph) |
| Data structures | 15 |
| Registries | 10 (1 master + 9 domain) |
| Verification checks | 50 (all PASS) |
| Scale target | 1M objects, 10M relations |

---

## Phase Boundary

- **Before this (3.0B):** 33-type object model, Pydantic v2, 184 unit tests — `platform/knowledge_runtime/`
- **This (3.0C):** Architecture only — these 43 documents — FROZEN
- **Next (3.0D):** Implementation of the engines defined here
