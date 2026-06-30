# CAE-DOC-008 — Compression Selector

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Compression Selector chooses which compression level (L1–L5) to use for
each object, given the budget allocated to it and the object's role in the context.

It also decides WHICH FIELDS from the intelligence block to include at each level,
since different query types and pack types need different fields even at the same
compression level.

---

## Level Selection Rules

```
SELECT_LEVEL(budget_tokens, role, query_type, pack_type):

  # Hard rules by role
  IF role == PRIMARY:
    min_level = L3
    IF budget_tokens >= 500:  L4
    ELIF budget_tokens >= 200: L3  (minimum enforced)
    ELSE: L3  (never below, even if budget doesn't cover it — drop other objects)

  IF role == ANTI_CONFUSION:
    level = L2  (always — they exist only to provide name + scope boundary)

  IF role == CONTEXT:
    IF budget_tokens >= 500:  L4
    IF budget_tokens >= 200:  L3
    IF budget_tokens >= 50:   L2
    IF budget_tokens >= 15:   L1
    ELSE: DROP

  IF role == PREREQUISITE:
    level = max(L2, CONTEXT_level)  # at least L2 for prerequisites

  IF role == EVIDENCE:
    level = L2  (evidence objects only need ID + short summary)

  # Pack-type overrides
  IF pack_type == SEARCH:
    ALL objects → L2 max (search results are compact)

  IF pack_type == HUMAN:
    primary role → L5 machine not used; instead audience-specific explanation
    (explanation.developer or .architect or .executive block)
```

---

## Field Inclusion by Level

### L1 — Nano

```
Source: intelligence.compression.nano
Fields:
  - knowledge_id
  - name
Tokens: ≤ 15
Use: graph labels, list items, inline references
```

### L2 — Micro

```
Source: intelligence.compression.micro
Fields:
  - knowledge_id
  - name
  - object_type
  - namespace
  - purpose (one sentence from ai_context.short_summary)
Tokens: ≤ 50
Use: search results, anti-confusion objects, evidence citations
```

### L3 — Mini

```
Source: intelligence.compression.mini
Fields:
  - knowledge_id, canonical_name, version
  - purpose
  - primary capabilities (name + category)
  - critical constraints (top 2)
  - HARD dependencies (names only)
  - primary output
  - scope boundary if has why_not confusion
  - state + quality_score
Tokens: ≤ 200
Use: context objects, standard assembly
```

### L4 — Standard

```
Source: intelligence.compression.standard (augmented by query_type)
Fields:
  - All L3 fields
  - all capabilities
  - all constraints
  - key decisions (top 3 from decisions[])
  - reasoning.requires (list)
  - confidence.overall_score + level
  - evidence summary (count + highest level)
  - relationship_count by type
Additional by query_type:
  BEHAVIORAL: + dna.core_behavior + self_describing.inputs + outputs
  CAUSAL:     + cortex.why + evolution.origin.genesis
  IMPACT:     + risk.failure_modes[:3]
  STRUCTURAL: + semantic.capabilities (full) + genome.category
Tokens: ≤ 500
Use: primary object in most assemblies
```

### L5 — Full (PromptPack) / Audience (HumanPack)

```
For PromptPack (AI):
  Source: intelligence.compression.machine + KIL blocks per query_type
  All L4 fields +
    + cortex.why + cortex.why_not + cortex.common_errors
    + thinking.reasoning_protocol
    + confidence.hedging_rules
    + full alternatives list (L2 each)
  Tokens: ≤ 2000

For HumanPack (developer):
  Source: intelligence.explanation.[audience]
  Full audience-targeted explanation block
  No compression level concept — full prose
  Tokens: uncapped (human reader)

For OperatorPack:
  Source: intelligence.explanation.operator
  Tokens: uncapped
```

---

## Query-Type Field Augmentations

| Query Type | Extra Fields Added to Primary |
|------------|------------------------------|
| QT-02 BEHAVIORAL | `dna.core_behavior`, `self_describing.inputs`, `self_describing.outputs` |
| QT-03 RELATIONAL | `genome.relationships` (full), `semantic.provides`, `semantic.consumes` |
| QT-04 CAUSAL | `cortex.why` (full), `evolution.origin`, `decisions[:2]` |
| QT-05 COMPARATIVE | `intelligence.diff.object_comparison`, `alternatives[:3]` |
| QT-06 IMPACT | `risk.failure_modes` (full), `reasoning.impacts` (full) |
| QT-07 STRUCTURAL | `genome.category`, `semantic.capabilities` (full), `semantic.extends` |
| QT-08 TEMPORAL | `evolution` (full), `genome.history`, `decisions` (full) |
| QT-09 OPERATIONAL | `explanation.operator` (full) |
| QT-10 MULTI_HOP | Each hop object at L3; relationship between hops explicit |

---

## Compression Selection Record

```yaml
compression_selection:
  knowledge_id: string
  role: string
  allocated_budget: integer
  selected_level: string             # L1 | L2 | L3 | L4 | L5 | AUDIENCE
  estimated_tokens: integer
  field_augmentations: [string]      # extra KIL fields added for query_type
  override_reason: string | null     # if level forced (e.g. "primary minimum L3")
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-031 | Primary object compression may never be below L3 — this is enforced before budget degradation |
| CAE-032 | ANTI_CONFUSION objects are always L2 — they must not be elevated or reduced |
| CAE-033 | AUDIENCE-level compression is only valid for HumanPack and OperatorPack |
| CAE-034 | Field augmentations must only use fields present in the object's intelligence: block |
| CAE-035 | L5 machine format must be valid JSON — failure is an AssemblyError |

---

## Cross-References

- Budget manager → `07-BUDGET-MANAGER`
- Context planner → `09-CONTEXT-PLANNER`
- KIL compression spec → Phase 3.0D.0.6 `05-KNOWLEDGE-COMPRESSION`
- Pack builders → docs 11–14
