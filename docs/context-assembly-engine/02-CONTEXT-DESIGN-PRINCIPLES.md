# CAE-DOC-002 — Context Design Principles

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## The Fundamental Shift

Most knowledge systems answer: *"Where is the document about X?"*

KOS answers: *"What is the optimal context for understanding X in this situation?"*

This is not a semantic difference. It changes every design decision:

| Document System | Context Generator (KOS) |
|----------------|------------------------|
| Returns the full document | Returns the right slice at the right size |
| Same output for all consumers | Different output per consumer profile |
| Static retrieval | Dynamic assembly |
| One format | Five pack types |
| Retrieval quality = recall | Context quality = relevance × completeness × safety |
| Fails silently (no doc found) | Fails explicitly (cannot assemble with confidence) |

---

## Principle 1: Context Is Not Retrieval

Retrieval finds the most relevant document.
Context assembly finds the right objects AND decides:
- What to include (selection)
- How much to include (compression)
- In what order (planning)
- What to add for safety (guard)
- What format to use (pack type)

A retrieval system stops after step 1. The CAE runs all five.

---

## Principle 2: The Consumer Defines the Context

The same query about KNW-PLT-MOD-001 produces different context for:

```
Query: "How does Quota Manager work?"

ConsumerType = AI_AGENT:
  → PromptPack with L3 primary + L2 dependencies + AH guard block
  → CoT template from intelligence.thinking.chain_of_thought_template
  → Scope boundaries from intelligence.dna.purpose.not_in_scope

ConsumerType = DEVELOPER:
  → HumanPack with developer explanation (intelligence.explanation.developer)
  → API surface + common pitfalls + debugging hints
  → Full prose, no compression level concept

ConsumerType = EXECUTIVE:
  → HumanPack with executive explanation (intelligence.explanation.executive)
  → Business value + risk + 3 sentences max
  → No technical content

ConsumerType = OPERATOR:
  → OperatorPack with monitoring + alerts + recovery steps
  → intelligence.explanation.operator
  → Focused on: health indicators + failure behavior + recovery

ConsumerType = SEARCH:
  → SearchPack: nano + micro per result, ranked list
  → Score + type + namespace shown per entry
```

The knowledge object is the same. The context is completely different.

---

## Principle 3: Budget Is a Hard Constraint

Token budgets are not suggestions. An AI agent with a 2,000-token context
window that receives a 2,100-token pack will either truncate silently
(corruption) or fail (crash). Both are unacceptable.

The CAE enforces budget as a hard constraint:
- Over-budget assemblies are rejected, not truncated
- Objects are removed or downgraded (L3→L2) to fit
- Guard block budget is protected — it is never sacrificed for content

```
Budget hierarchy:
  1. Guard block (protected — minimum 10% of budget)
  2. Primary object (protected — minimum L3)
  3. Context objects (trimmed first)
  4. Related objects (dropped first)
```

---

## Principle 4: Safety Is Non-Negotiable

Every context pack includes the Anti-Hallucination Guard block.
Even if the primary object is high-confidence (0.95+), the guard block runs.

Reason: High confidence means the knowledge is correct.
It does NOT mean AI agents cannot hallucinate about it.
The guard block injects the information needed to prevent hallucination,
regardless of the object's own quality score.

---

## Principle 5: Context Quality Is Measurable

Context quality is not subjective. It has four measurable dimensions:

```
context_quality_score =
  (relevance    × 0.35) +   # are the right objects included?
  (completeness × 0.30) +   # is the primary object fully covered?
  (safety       × 0.25) +   # is the guard block effective?
  (efficiency   × 0.10)     # is the budget well-utilized?
```

A ContextPack is REJECTED if context_quality_score < 0.70.

---

## Principle 6: Assembly Is Deterministic

Given identical inputs (query, consumer, budget), the CAE produces identical
output. This is a testability requirement.

Random seeding, time-based variation, or LLM-based intermediate steps are
forbidden in the core pipeline. They belong to the AI layer that CONSUMES
the ContextPack, not in its assembly.

---

## Principle 7: Failure Is Explicit

The CAE never returns a degraded or partial context silently.
If assembly fails, it returns a structured error:

```yaml
assembly_error:
  error_type: OBJECT_NOT_FOUND | BUDGET_TOO_SMALL | CONFIDENCE_TOO_LOW |
              VALIDATION_FAILED | SELECTOR_EMPTY | GUARD_FAILED
  message: string
  recoverable: boolean
  suggestion: string          # e.g. "increase budget to 500 tokens"
  partial_result: boolean     # whether a partial pack is attached
```

---

## Principle 8: KIL Is the Source

The CAE derives all context from the KIL intelligence: blocks.
It does not:
- Read raw YAML fields directly (uses KIL-compressed representations)
- Invent context not present in the knowledge base
- Supplement with external sources

If the knowledge base lacks information, the CAE says so explicitly.

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-001 | Every assembly must respect budget as a HARD constraint — never soft |
| CAE-002 | The guard block minimum (10% of budget) may not be reallocated to content |
| CAE-003 | Assembly results must be deterministic for identical inputs |
| CAE-004 | A failed assembly must emit a structured AssemblyError — never return null |
| CAE-005 | Context is assembled from KIL blocks only — no raw YAML field access |
| CAE-006 | Every ContextPack must carry `context_quality_score` ≥ 0.70 |

---

## Cross-References

- Pipeline implementation → `01-CAE-OVERVIEW`
- Guard block → `15-ANTI-HALLUCINATION-GUARD`
- Context quality → `17-CONTEXT-QUALITY-MODEL`
- Failure handling → `25-CAE-API`
