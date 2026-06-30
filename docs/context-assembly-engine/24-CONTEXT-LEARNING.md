# CAE-DOC-024 — Context Learning

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Learning defines how the CAE improves over time using signals from the
feedback model and metric trends. Learning is architectural — it defines what
changes and how. No AI model training happens in the CAE itself.

All learning output is a structured change proposal, not an automatic update.
A human reviewer approves changes that modify KIL objects or assembly rules.

---

## Learning Categories

```
Category 1: Selection Improvement
  What changes: which objects get selected for which query patterns
  Signal: context_gap_rate, abandonment_rate
  Output: updated co_occurrence_matrix, updated always_include references

Category 2: Compression Calibration
  What changes: default compression levels for object roles + intent combinations
  Signal: budget_utilization_mean, context_quality_mean
  Output: updated compression recommendation table

Category 3: Guard Block Improvement
  What changes: cortex.common_errors, scope_boundaries text
  Signal: hallucination_flag_rate, explicit.misleading
  Output: proposed KIL field updates (submitted to maintenance queue)

Category 4: Intent Classification Improvement
  What changes: keyword classification rules, semantic pattern rules
  Signal: entity_ambiguity_rate, intent_resolution_rate
  Output: proposed new classification rules

Category 5: Relevance Weight Calibration
  What changes: intent-weighted relevance formula variants
  Signal: helpfulness_score_mean by intent type
  Output: proposed weight adjustments (require architecture review)
```

---

## Learning Cycle

```
Cycle frequency: Every 7 days (weekly batch)

Week N:
  1. COLLECT:   Aggregate all feedback records from past 7 days
  2. ANALYZE:   Compute trend metrics vs. previous week
  3. DETECT:    Identify patterns crossing alert thresholds
  4. PROPOSE:   Generate change proposals per learning category
  5. REVIEW:    Human reviewer approves/rejects proposals
  6. APPLY:     Approved changes written to assembly hints or KIL maintenance queue

No change is applied automatically without human approval.
```

---

## Learning Analysis: Selection Improvement

```
ANALYZE_SELECTION(feedback_records, period_days=7):

  # Group by primary object
  by_primary = group_by(feedback_records, "primary_knowledge_id")

  proposals = []

  FOR primary_id, records in by_primary:
    gap_records = [r for r in records WHERE r.derived.context_gap_detected]
    gap_rate = len(gap_records) / len(records)

    IF gap_rate > 0.25:  # > 25% of queries had context gaps
      # Find what was missing
      missing = flatten([r.explicit.missing_info for r in gap_records
                         WHERE r.explicit is not null])
      # Identify objects named in missing_info that exist in KIL
      for mention in most_common(missing, top=3):
        candidate = KIL_INDEX.search(mention)
        IF candidate:
          proposals.append(SelectionProposal(
            primary_id = primary_id,
            add_to_always_include = candidate.knowledge_id,
            evidence_gap_rate = gap_rate,
            evidence_mention_count = count(mention in missing)
          ))

  RETURN proposals
```

---

## Learning Analysis: Guard Block Improvement

```
ANALYZE_GUARD_QUALITY(feedback_records):

  hallucination_records = [
    r for r in feedback_records
    WHERE r.explicit and r.explicit.hallucination_flagged
  ]

  by_primary = group_by(hallucination_records, "primary_knowledge_id")

  proposals = []

  FOR primary_id, records in by_primary:
    misleading_texts = flatten([r.explicit.misleading for r in records])
    common_mistakes = most_common(misleading_texts, top=5)

    IF common_mistakes:
      proposals.append(GuardImprovement(
        knowledge_id = primary_id,
        proposed_common_errors = common_mistakes,
        evidence_count = len(records),
        evidence_hallucination_rate = len(records) / total_count(primary_id)
      ))

  RETURN proposals
```

---

## Change Proposal Schema

```yaml
change_proposal:
  proposal_id: string
  proposed_at: string
  proposal_type: string               # SELECTION | COMPRESSION | GUARD | INTENT | RELEVANCE
  learning_category: integer          # 1–5

  primary_knowledge_id: string | null # affected object (if applicable)
  rule_target: string | null          # rule code if proposal modifies a rule

  evidence:
    metric_id: string
    current_value: float
    threshold_crossed: float
    sample_size: integer
    period_days: integer

  proposed_change:
    description: string
    field_path: string                # which KIL field or assembly table to change
    current_value: string | null
    proposed_value: string

  expected_impact:
    metric_id: string
    expected_improvement: float

  reviewer: string | null             # filled when reviewed
  decision: PENDING | APPROVED | REJECTED
  reviewed_at: string | null
  rejection_reason: string | null
```

---

## Learning Constraints

```
NEVER auto-apply:
  - Changes to relevance formula weights (require architecture board review)
  - Changes to CAE design invariants
  - Changes to guard block rules (CAE-066 through CAE-070)
  - Changes that affect the minimum quality score (0.70)

May be applied after reviewer approval:
  - KIL co_occurrence_matrix updates
  - KIL cortex.common_errors additions
  - KIL ai_context.always_include additions
  - Assembly hints for specific object-intent combinations

Auto-apply (no review required):
  - Prefetch priority adjustments for hot objects
  - Cache TTL adjustments based on invalidation rate
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-111 | Learning proposals must not be auto-applied without human approval |
| CAE-112 | Relevance formula weight changes require architecture board sign-off |
| CAE-113 | All proposals must include evidence (metric, sample size, period) |
| CAE-114 | Rejected proposals must record rejection reason — not silently dropped |
| CAE-115 | Learning cycle runs every 7 days — it must not block assembly requests |

---

## Cross-References

- Feedback model → `22-FEEDBACK-MODEL`
- Context metrics → `23-CONTEXT-METRICS`
- CAE freeze → `28-CAE-FREEZE` (what is frozen)
- KIL maintenance → Phase 3.0D.0.6 `11-KNOWLEDGE-EVOLUTION`
