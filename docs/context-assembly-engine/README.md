# Context Assembly Engine — Phase 3.0D.1

**Phase:** 3.0D.1
**Status:** CANONICAL
**Architecture Version:** CAE-1.0.0
**Frozen:** 2026-06-30

---

## What Is the Context Assembly Engine

The Context Assembly Engine (CAE) is the operational core of KOS. It transforms
a query, a consumer profile, and a token budget into a context pack — the most
relevant, safe, and correctly compressed representation of knowledge that a
human or AI needs to act on that query.

```
(Query, ConsumerProfile, Budget) → ContextPack
```

KOS is not a documentation system. KOS is a Knowledge Operating System that
serves as the Single Source of Truth and generates optimal context for humans
and AI. The CAE is the engine that produces that context.

---

## Architecture at a Glance

```
                        CONTEXT ASSEMBLY ENGINE
    ┌────────────────────────────────────────────────────────────┐
    │                                                            │
    │  Query ──→ [S1 Intent Parser]                              │
    │            [S2 Object Selector]                            │
    │            [S3 Relevance Ranker]                           │
    │            [S4 Budget Manager]                             │
    │            [S5 Compression Selector]                       │
    │            [S6 Context Planner]                            │
    │            [S7 Context Assembler] ←── KIL Index            │
    │            [S8 AH Guard]                                   │
    │            [S9 Context Validator]                          │
    │                   │                                        │
    │                   ▼                                        │
    │           ContextPack (PROMPT | HUMAN | OPERATOR | SEARCH) │
    └────────────────────────────────────────────────────────────┘

    P99 target: < 120ms
```

---

## Document Map

```
Tier 1 — Foundation (read first)
  00-README.md                   This file
  01-CAE-OVERVIEW.md             Architecture, pipeline, 10 invariants

Tier 2 — Design Principles
  02-CONTEXT-DESIGN-PRINCIPLES.md  8 principles of good context

Tier 3 — Input Model (what comes in)
  03-INTENT-MODEL.md             12 intent types; intent classification
  04-QUERY-TYPES.md              10 query types; answer shapes

Tier 4 — Pipeline Stages (how it works)
  05-OBJECT-SELECTOR.md          Stage S2: which objects to consider
  06-RELEVANCE-MODEL.md          Stage S3: how to rank objects
  07-BUDGET-MANAGER.md           Stage S4: token allocation
  08-COMPRESSION-SELECTOR.md     Stage S5: which compression level
  09-CONTEXT-PLANNER.md          Stage S6: assembly order
  10-CONTEXT-ASSEMBLER.md        Stage S7: rendering objects

Tier 5 — Output Packs (what comes out)
  11-PROMPT-PACK.md              For AI agents
  12-HUMAN-PACK.md               For developers, architects, executives
  13-OPERATOR-PACK.md            For SRE and operations
  14-SEARCH-PACK.md              For search UI / discovery

Tier 6 — Safety and Quality
  15-ANTI-HALLUCINATION-GUARD.md Stage S8: guard block injection
  16-CONTEXT-VALIDATOR.md        Stage S9: final validation
  17-CONTEXT-QUALITY-MODEL.md    Quality formula (4 components)
  18-CONFIDENCE-PROPAGATION.md   Confidence = minimum across chain

Tier 7 — Operational
  19-PREFETCH-ENGINE.md          Predictive pre-loading
  20-CONTEXT-CACHE.md            Two-tier render cache
  21-PERFORMANCE-MODEL.md        Latency targets + degradation
  22-FEEDBACK-MODEL.md           Consumer signals → improvement
  23-CONTEXT-METRICS.md          30 metrics (CAE-M-001 to CAE-M-030)
  24-CONTEXT-LEARNING.md         Weekly learning cycle

Tier 8 — Contracts and Governance
  25-CAE-API.md                  External contract (v1)
  26-CAE-CERTIFICATION.md        50 certification checks
  27-CAE-TESTING.md              Test framework and corpus
  28-CAE-FREEZE.md               Architecture freeze declaration
```

---

## Reading Order

| Order | Document | Purpose |
|-------|----------|---------|
| 1 | 01-CAE-OVERVIEW | Understand the full pipeline and 10 invariants first |
| 2 | 02-CONTEXT-DESIGN-PRINCIPLES | Understand why context ≠ retrieval |
| 3 | 03-INTENT-MODEL | Understand what queries look like |
| 4 | 07-BUDGET-MANAGER | Understand the budget partition model |
| 5 | 06-RELEVANCE-MODEL | Understand object scoring |
| 6 | 10-CONTEXT-ASSEMBLER | Understand how objects are rendered |
| 7 | 11-PROMPT-PACK | Understand the primary output format |
| 8 | 15-ANTI-HALLUCINATION-GUARD | Understand the safety layer |
| 9 | 17-CONTEXT-QUALITY-MODEL | Understand quality measurement |
| 10 | 25-CAE-API | Understand the external contract |

---

## Key Numbers

| Dimension | Count |
|-----------|-------|
| Pipeline stages | 9 |
| Design invariants | 10 |
| Intent types | 12 |
| Query types | 10 |
| Pack types | 5 |
| Validation checks | 20 |
| CAE rules | 120 |
| CAE metrics | 30 |
| Certification checks | 50 |
| Architecture documents | 31 |

---

## Dependencies

```
This phase depends on:
  Phase 3.0D.0.6 — Knowledge Intelligence Layer
    intelligence.compression.* (all levels)
    intelligence.ai_context.*
    intelligence.cortex.*
    intelligence.confidence.*
    intelligence.reasoning.*
    intelligence.thinking.*
    intelligence.explanation.*
    All KIL indexes (IDX-001 through IDX-010)

  Phase 3.0D.0.5 — KOS Final
    KnowledgeObject schema v1.0
    All relationship types
    All quality dimensions

This phase does NOT implement:
  - The KIL knowledge indexes (those are Phase 3.0D.0.6 spec)
  - The Knowledge Graph (future Phase 3.0F)
  - LLM inference (in consumer layer, never in CAE)
  - Database or storage (read-only access to KIL index)
```

---

## Total KOS Architecture

```
Phase 3.0A  Core KOS                     43 documents
Phase 3.0B  Quality Model                32 documents
Phase 3.0C  Certification                32 documents
Phase 3.0D.0 KOS Final                   23 documents
Phase 3.0D.0.6 Knowledge Intelligence    33 documents
Phase 3.0D.1 Context Assembly Engine     31 documents
                                    ─────────────────
TOTAL                                   194 documents
```

---

## Stop Condition

```
Architecture for Phase 3.0D.1 is complete.

DO NOT implement the CAE runtime.
DO NOT implement the object selector algorithm.
DO NOT implement the cache.
DO NOT implement graph traversal.
DO NOT implement AI inference.

Architecture + Certification Framework + Verification Tools only.
Stop here. Implementation begins in Phase 3.0E.
```
