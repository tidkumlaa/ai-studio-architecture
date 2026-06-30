# CAE-DOC-009 — Context Planner

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Planner decides the assembly ORDER and STRUCTURE of selected objects.
Order matters: an AI agent reading context processes it sequentially — the most
important information must arrive first, prerequisites before dependents.

---

## Assembly Order Principles

```
1. PREREQUISITES come before objects that depend on them
2. PRIMARY comes before CONTEXT objects
3. CONTEXT objects come before EVIDENCE objects
4. ANTI_CONFUSION objects come after primary (they clarify by contrast)
5. Guard block comes last (injected by AH Guard, not planner)
```

---

## Standard Section Order

```
Section 1: Preamble (generated, not from objects)
  - Query restatement
  - Intent type
  - Primary object identified as: {id} {name} {type}

Section 2: Prerequisites (if any, at L2–L3)
  - Objects that must be understood first
  - Ordered by reasoning.depends[*].order

Section 3: Primary Object
  - At L3–L5 depending on budget
  - With query-type field augmentations

Section 4: Context Objects (CONTEXT role)
  - Ordered by: HARD dependencies first, then by relevance_score DESC
  - Each at L2–L4

Section 5: Evidence / Corroboration
  - EVIDENCE-role objects at L2
  - Support claims made in primary

Section 6: Comparative (COMPARATIVE intent only)
  - Second entity (for INT-05)
  - Semantic diff block

Section 7: Anti-Confusion (ANTI_CONFUSION role)
  - At L2
  - Introduced with: "NOTE: Do not confuse with:"

Section 8: Guard Block (added by AH Guard, not planner)
  - scope_boundaries
  - hedges
  - typical_mistakes
```

---

## Intent-Specific Structure Plans

### INT-01 FACTUAL_LOOKUP

```
Structure:
  preamble → primary(L4) → [1-2 context objects(L2)]
  → guard_block

Rationale: Single object, deep understanding, minimal context.
```

### INT-03 REASONING_QUERY (Why)

```
Structure:
  preamble
  → prerequisites(L2)          # what must be understood before "why"
  → primary.cortex.why(full)   # the answer
  → evolution.origin           # historical context
  → decisions[:2]              # decisions that shaped it
  → [tradeoffs reference]
  → guard_block

Rationale: "Why" questions need historical and decision context.
```

### INT-04 IMPACT_ANALYSIS

```
Structure:
  preamble
  → primary(L3 + risk block)
  → direct_impacts:            # reasoning.impacts WHERE severity=CRITICAL|HIGH
      - each at L2 with impact description
  → transitive_impacts:        # one hop further from critical impacts
      - each at L1 with severity label
  → mitigation_note            # from risk.failure_modes[*].mitigation
  → guard_block

Rationale: Impact analysis needs breadth over depth.
```

### INT-05 COMPARISON

```
Structure:
  preamble: "Comparing {X} and {Y}"
  → entity_A(L3):
      id, name, type, purpose, primary capabilities, key constraints
  → entity_B(L3):
      same fields
  → diff_section:
      similarity_score
      key differences by dimension
      guidance: when to use X vs Y
  → anti_confusion note (if similarity > 0.70)
  → guard_block

Rationale: Side-by-side structure for comparison.
```

### INT-06 SEARCH

```
Structure:
  preamble: "Results for: {query}"
  → result_list:
      - rank: 1
        object: L2
        relevance_score: float
        match_reason: string
      - rank: 2 ...
      (up to 10 results at L2)
  No guard block for SEARCH pack (not needed for list display)

Rationale: Search is compact — 10 × L2 objects.
```

### INT-08 CONTEXT_PRODUCTION

```
Structure:
  preamble: chain_of_thought_template.preamble from primary.thinking
  → prerequisites(L2):
      objects in reasoning.depends, ordered by reasoning.depends[*].order
  → primary(L4 + cortex + thinking):
      id + full L4 + cortex.why_not + dna.fundamental_constraints
      + thinking.guard_conditions
  → always_include_objects(L3):
      ai_context.related_objects WHERE fetch_priority=ALWAYS
  → likely_include_objects(L2):
      ai_context.related_objects WHERE fetch_priority=LIKELY
  → reasoning_scaffold:
      from primary.thinking.chain_of_thought_template
  → guard_block:
      full scope boundaries + hedges + typical_mistakes

Rationale: Context production is the richest pack — it prepares
an AI agent to answer any question about the primary object.
```

### INT-09 TRACEABILITY

```
Structure:
  preamble: "Traceability chain for {entity}"
  → chain:
      - level: 1
        object: L3
        link_type: satisfies | implements | ...
      - level: 2 ...
      (up to 9 levels at L3 each)
  → completeness_note:
      "Chain is complete / missing: [levels]"
  → guard_block

Rationale: Traceability is linear — chain order is critical.
```

---

## Assembly Plan Schema

```yaml
assembly_plan:
  query_id: string
  intent_type: string
  query_type: string
  pack_type: string

  sections:
    - section_id: integer
      section_type: PREAMBLE | PREREQUISITE | PRIMARY | CONTEXT |
                    EVIDENCE | COMPARATIVE | ANTI_CONFUSION | GUARD
      objects:
        - knowledge_id: string
          compression_level: string
          role: string
          position: integer          # order within section
          estimated_tokens: integer

  total_estimated_tokens: integer
  guard_position: integer            # section index of guard block
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-036 | Prerequisites must appear before any object that depends on them |
| CAE-037 | ANTI_CONFUSION objects must be labeled explicitly — never silently included |
| CAE-038 | Guard block is always the last section — never interleaved |
| CAE-039 | SEARCH pack has no guard block — it is not a reasoning context |
| CAE-040 | INT-05 COMPARISON must produce side-by-side structure — not sequential A then B |

---

## Cross-References

- Budget manager → `07-BUDGET-MANAGER`
- Compression selector → `08-COMPRESSION-SELECTOR`
- Context assembler → `10-CONTEXT-ASSEMBLER`
- Anti-hallucination guard → `15-ANTI-HALLUCINATION-GUARD`
