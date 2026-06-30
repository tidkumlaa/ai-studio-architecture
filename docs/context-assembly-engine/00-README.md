# CAE-DOC-000 — Context Assembly Engine — Navigation

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Mission

The Context Assembly Engine (CAE) is the core runtime of the Knowledge
Operating System. It transforms an incoming query into the optimal context
packet for its consumer — whether AI agent, developer, operator, or search UI.

```
KOS = Single Source of Truth + Context Generator

CAE = the Generator
```

The CAE does not store knowledge. It assembles the right knowledge, in the
right form, at the right size, for the right consumer, on demand.

---

## The Pipeline

```
Query (human / AI / system)
  │
  ├─► Intent Parser (doc 03)
  │     classify intent + extract entities
  │
  ├─► Object Selector (doc 05)
  │     find relevant KnowledgeObjects via KIL index
  │
  ├─► Relevance Ranker (doc 06)
  │     score + rank candidates
  │
  ├─► Budget Manager (doc 07)
  │     allocate tokens: primary 40% / context 30% / related 20% / guard 10%
  │
  ├─► Compression Selector (doc 08)
  │     pick L1–L5 per object
  │
  ├─► Context Planner (doc 09)
  │     decide assembly order + structure
  │
  ├─► Context Assembler (doc 10)
  │     build the packet
  │
  ├─► Anti-Hallucination Guard (doc 15)
  │     inject why_not + hedges + scope boundaries
  │
  ├─► Context Validator (doc 16)
  │     20 checks before output
  │
  └─► Output Pack
        PromptPack (doc 11)   — for AI agents
        HumanPack  (doc 12)   — for developers
        OperatorPack (doc 13) — for SRE/ops
        SearchPack (doc 14)   — for search UI
        ReasoningPack         — for multi-step reasoning (doc 11 extension)
```

---

## Document Map

```
architecture/docs/context-assembly-engine/
│
├── TIER 0 — Foundation
│   ├── 00-README.md                    ← YOU ARE HERE
│   ├── 01-CAE-OVERVIEW.md              ← design invariants, architecture
│   └── 02-CONTEXT-DESIGN-PRINCIPLES.md ← why KOS generates context, not docs
│
├── TIER 1 — Intent & Selection
│   ├── 03-INTENT-MODEL.md              ← 12 intent types + classification rules
│   ├── 04-QUERY-TYPES.md               ← 10 query types with schemas
│   ├── 05-OBJECT-SELECTOR.md           ← selection algorithm
│   └── 06-RELEVANCE-MODEL.md           ← relevance score formula
│
├── TIER 2 — Budget & Compression
│   ├── 07-BUDGET-MANAGER.md            ← token budget allocation
│   ├── 08-COMPRESSION-SELECTOR.md      ← L1–L5 selection algorithm
│   └── 09-CONTEXT-PLANNER.md           ← assembly order + structure
│
├── TIER 3 — Assembly & Output
│   ├── 10-CONTEXT-ASSEMBLER.md         ← builds the context packet
│   ├── 11-PROMPT-PACK.md               ← AI agent output spec
│   ├── 12-HUMAN-PACK.md                ← developer output spec
│   ├── 13-OPERATOR-PACK.md             ← ops output spec
│   └── 14-SEARCH-PACK.md               ← search result output spec
│
├── TIER 4 — Quality & Safety
│   ├── 15-ANTI-HALLUCINATION-GUARD.md  ← inject cortex + hedges
│   ├── 16-CONTEXT-VALIDATOR.md         ← 20 validation checks
│   ├── 17-CONTEXT-QUALITY-MODEL.md     ← completeness/relevance/safety
│   └── 18-CONFIDENCE-PROPAGATION.md    ← object → context confidence
│
├── TIER 5 — Performance & Cache
│   ├── 19-PREFETCH-ENGINE.md           ← co_occurrence-based prefetch
│   ├── 20-CONTEXT-CACHE.md             ← assembled context caching
│   └── 21-PERFORMANCE-MODEL.md         ← latency targets P50/P95/P99
│
├── TIER 6 — Learning & Feedback
│   ├── 22-FEEDBACK-MODEL.md            ← context quality → KIL memory
│   ├── 23-CONTEXT-METRICS.md           ← 30 CAE metrics
│   └── 24-CONTEXT-LEARNING.md          ← CAE improvement over time
│
├── TIER 7 — Contracts & Certification
│   ├── 25-CAE-API.md                   ← public API contract
│   ├── 26-CAE-CERTIFICATION.md         ← 50 certification checks
│   └── 27-CAE-TESTING.md               ← test strategy + canonical cases
│
└── TIER 8 — Freeze
    └── 28-CAE-FREEZE.md                ← architecture freeze + 10 invariants
```

---

## Key Numbers

| Metric | Value |
|--------|-------|
| Documents | 31 |
| Intent types | 12 |
| Query types | 10 |
| Output pack types | 5 |
| Pipeline stages | 9 |
| Validation checks | 20 |
| CAE metrics | 30 |
| CAE rules (CAE-001–CAE-120) | 120 |
| CAE invariants | 10 |
| Certification checks | 50 |

## Dependencies

```
KOS v1.0 Core (Phase 3.0C) ──────────────┐
KIL intelligence: blocks (Phase 3.0D.0.6)─┤
KIL Canonical Index (IDX-001–IDX-010) ────┤
                                           ▼
                    Context Assembly Engine (THIS PHASE)
                                           ▼
                    Knowledge Runtime Implementation (Phase 3.0D.1+)
```
