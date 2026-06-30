# CAE-DOC-001 — Context Assembly Engine Overview

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## What the CAE Is

The Context Assembly Engine is a stateless query processor. Given a query and
consumer profile, it returns an optimally assembled context packet.

```
CAE: (Query, ConsumerProfile, Budget) → ContextPack
```

It is NOT:
- A search engine (search is one selection strategy among many)
- A document retrieval system (it selects + compresses + assembles)
- An AI model (it prepares context FOR AI models)
- A cache (caching is optional infrastructure layered on top)

---

## Architecture Layers

```
┌─────────────────────────────────────────────────────┐
│                   CAE Public API                     │
│  assemble(query, consumer, budget) → ContextPack     │
├─────────────────────────────────────────────────────┤
│               Pipeline Orchestrator                  │
│  Coordinates 9 pipeline stages in order             │
├──────────────┬──────────────────────────────────────┤
│  Intent &    │  Object        │  Assembly   │ Output │
│  Selection   │  Scoring       │  & Safety   │ Packs  │
│              │                │             │        │
│  IntentParser│  RelevanceRank │  Assembler  │  PP    │
│  ObjectSelect│  BudgetMgr     │  AHGuard    │  HP    │
│  QueryType   │  CompressSelect│  Validator  │  OP    │
│              │  ContextPlanner│             │  SP    │
└──────────────┴──────────────────────────────────────┘
         ▲                                    ▲
         │                                    │
    KIL Index                           Output Pack
  (IDX-001–010)                      Schema Definitions
```

---

## Pipeline Stages

| Stage | Component | Input | Output | Max Latency |
|-------|-----------|-------|--------|-------------|
| S1 | Intent Parser | raw query | IntentRecord | 5ms |
| S2 | Object Selector | IntentRecord | candidate set | 20ms |
| S3 | Relevance Ranker | candidate set | ranked list | 15ms |
| S4 | Budget Manager | ranked list + budget | allocation plan | 5ms |
| S5 | Compression Selector | allocation plan | compression map | 2ms |
| S6 | Context Planner | compression map | assembly plan | 5ms |
| S7 | Context Assembler | assembly plan | raw context | 30ms |
| S8 | AH Guard | raw context | guarded context | 10ms |
| S9 | Context Validator | guarded context | validated context | 10ms |
| — | Pack Builder | validated context | ContextPack | 5ms |

**Total P99 target: < 120ms** (sum of stage P99 targets)

---

## CAE Data Structures

### QueryRecord

```yaml
query_record:
  query_id: string                     # UUID
  raw_query: string                    # original query text
  consumer_type: AI_AGENT | DEVELOPER | ARCHITECT | OPERATOR | EXECUTIVE | SEARCH
  pack_type: PROMPT | HUMAN | OPERATOR | SEARCH | REASONING
  budget_tokens: integer               # total token budget for this assembly
  context:
    session_id: string | null          # ongoing conversation context
    prior_objects: [string]            # knowledge_ids from prior turns
    language: string                   # response language (default: same as query)
  timestamp: string
```

### IntentRecord

```yaml
intent_record:
  query_id: string
  intent_type: string                  # one of 12 intent types (doc 03)
  query_type: string                   # one of 10 query types (doc 04)
  primary_entity:
    knowledge_id: string | null        # if directly referenced
    search_term: string                # for semantic/keyword lookup
    entity_type: string | null         # expected object type
  secondary_entities: [string]         # additional referenced entities
  constraints:
    namespace: string | null           # restrict to namespace
    object_type: string | null         # restrict to type
    state: string | null               # restrict by lifecycle state
  audience_depth: SURFACE | STANDARD | DEEP | EXPERT
  confidence: float                    # intent classification confidence
```

### ContextPack

```yaml
context_pack:
  pack_id: string
  query_id: string
  pack_type: string
  assembled_at: string
  assembly_version: "1.0"

  intent:
    type: string
    confidence: float

  primary:
    knowledge_id: string
    compression_level: string          # L1–L5
    content: string | object           # compressed representation

  context_objects:
    - knowledge_id: string
      role: CONTEXT | EVIDENCE | PREREQUISITE | ANTI_CONFUSION
      compression_level: string
      content: string | object
      relevance_score: float

  guard_block:
    scope_boundaries: [string]
    hedges: [string]
    typical_mistakes: [string]
    confidence_hedge: string | null    # if confidence < 0.70

  reasoning_scaffold:
    chain_of_thought: [string]
    conclusion_format: string | null

  pack_metadata:
    total_tokens: integer
    budget_tokens: integer
    budget_utilization: float          # 0.0–1.0
    object_count: integer
    context_confidence: float          # propagated from objects
    assembly_latency_ms: integer
    cache_hit: boolean
```

---

## CAE Design Invariants

| # | Invariant |
|---|-----------|
| CI-01 | The primary object is ALWAYS included at minimum L3 compression |
| CI-02 | Context must never exceed `budget_tokens` — hard limit, not target |
| CI-03 | Anti-Hallucination Guard runs on EVERY assembly — never skipped |
| CI-04 | Context Validator must pass all 20 checks before pack is emitted |
| CI-05 | CAE is stateless — same inputs always produce the same outputs |
| CI-06 | `context_confidence` = min(confidence score of all included objects) |
| CI-07 | Guard block always uses L2 compression — never inlined at L1 |
| CI-08 | Objects with `ai_readiness_score < 0.60` may not appear in PromptPack |
| CI-09 | Every ContextPack has a unique `pack_id` for feedback correlation |
| CI-10 | Pipeline stages execute in order S1→S9 — no stage may be skipped |

---

## Cross-References

- Design principles → `02-CONTEXT-DESIGN-PRINCIPLES`
- Full pipeline stages → docs 03–16
- Output pack specs → docs 11–14
- Performance targets → `21-PERFORMANCE-MODEL`
- Public API → `25-CAE-API`
