# UICE-DOC-019 — Context Learning

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Context Learning collects feedback signals — both explicit ratings and implicit
behavioral signals — and proposes adjustments to UICE's field selection weights
and ranking heuristics. Learning makes UICE progressively more efficient:
fields that consistently prove useful gain weight; fields that are ignored
or negatively rated lose weight.

UICE invariant UI-10: Learning proposals require explicit approval before
production application. Learning never auto-applies changes to the live system.

---

## Feedback Sources

```
EXPLICIT:
  Consumer rates the context pack quality (1–5 stars)
  Consumer marks specific fields as "useful" or "irrelevant"
  Consumer signals "context was insufficient" or "too much noise"

IMPLICIT:
  LLM consumption: which fields in the pack did the LLM reference in its response?
  Task success: did the downstream task complete successfully?
  Re-query rate: did the consumer immediately re-query for more context?
    (high re-query rate = context was insufficient)
  Session length: longer productive sessions = higher context quality
```

---

## FeedbackRecord Schema

```yaml
FeedbackRecord:
  feedback_id: string
  session_id: string
  pack_id: string                      # links to the context pack this feedback is about
  knowledge_id: string                 # primary object

  # Consumer rating
  explicit_rating: int | null          # 1–5 scale; null if not provided
  helpful_fields: list[str]            # field paths consumer marked useful
  irrelevant_fields: list[str]         # field paths consumer marked irrelevant

  # Implicit signals
  llm_referenced_fields: list[str]     # fields referenced in LLM's response
  task_success: bool | null            # did the downstream task succeed?
  re_queried: bool                     # did consumer re-query within 30 seconds?
  session_length_s: int | null         # conversation session duration

  # Context metadata
  intent_type: IntentType
  query_type: QueryType
  context_quality_score: float         # recorded at time of assembly
  token_count: int

  recorded_at: datetime
```

---

## Learning Signal Computation

```
COMPUTE_HELPFULNESS(feedback) → float:  // 0.0–1.0

  score = 0.50  // neutral baseline

  // Explicit rating (strongest signal)
  if feedback.explicit_rating:
    score = (feedback.explicit_rating - 1) / 4.0  // 1→0.0, 5→1.0

  else:
    // Implicit signals
    if feedback.task_success == True:  score += 0.20
    if feedback.task_success == False: score -= 0.30
    if feedback.re_queried:            score -= 0.20
    if feedback.llm_referenced_fields:
      coverage = len(feedback.llm_referenced_fields) / max(len(sent_fields), 1)
      score += coverage * 0.30

  return clamp(score, 0.0, 1.0)
```

---

## FieldWeightAdjustment Proposal Schema

```yaml
FieldWeightAdjustment:
  proposal_id: string
  status: PENDING | APPROVED | REJECTED | APPLIED
  created_at: datetime
  approved_by: string | null            # required before APPLIED

  # What to change
  target: FIELD_MATRIX | RELEVANCE_WEIGHTS | HISTORY_WEIGHTS
  intent_type: IntentType
  query_type: QueryType | null

  # FIELD_MATRIX adjustments
  fields_to_promote: list[str]          # add to matrix or increase slice limit
  fields_to_demote: list[str]           # remove from matrix or decrease slice limit

  # RELEVANCE_WEIGHTS adjustments
  weight_changes: dict[str, float]     # dimension → delta (e.g., history_weight += 0.02)

  # Evidence
  feedback_count: int                   # how many records support this proposal
  mean_helpfulness_before: float
  mean_helpfulness_projected: float
  confidence: float                     # 0.0–1.0; low confidence = requires more data
```

---

## Learning Cycle

```
WEEKLY LEARNING CYCLE (async background process):

  // Step 1: Aggregate feedback from the past 7 days
  records = FEEDBACK_STORE.get_records(since=7_days_ago, min_count=50)
  if len(records) < 50:
    return  // insufficient data for learning

  // Step 2: Group by (intent_type, query_type)
  groups = GROUP_BY(records, (intent_type, query_type))

  // Step 3: Identify underperforming combinations
  for (intent, qtype), group_records in groups:
    mean_helpfulness = MEAN(COMPUTE_HELPFULNESS(r) for r in group_records)
    if mean_helpfulness < 0.65:
      // Identify which fields were frequently in irrelevant_fields
      demote_candidates = MOST_COMMON(r.irrelevant_fields for r in group_records)
      promote_candidates = MOST_COMMON(r.llm_referenced_fields
                                        for r in group_records
                                        if r.llm_referenced_fields)

      if demote_candidates or promote_candidates:
        proposal = FieldWeightAdjustment(
          target           = FIELD_MATRIX,
          intent_type      = intent,
          query_type       = qtype,
          fields_to_demote = demote_candidates[:3],
          fields_to_promote = [f for f in promote_candidates[:3]
                               if f not in CURRENT_MATRIX[(intent, qtype)]],
          feedback_count   = len(group_records),
          confidence       = min(1.0, len(group_records) / 200),
        )
        PROPOSAL_STORE.save(proposal)

  // Step 4: Notify human reviewer (UI-10)
  NOTIFY_REVIEWER(pending_proposals=PROPOSAL_STORE.get_pending())
```

---

## Approval and Application (UI-10)

```
Proposals are PENDING until a human reviewer approves or rejects them.

APPROVE(proposal_id, reviewer_id):
  proposal.status    = APPROVED
  proposal.approved_by = reviewer_id
  // Schedule application at next deployment window

APPLY(proposal_id):
  assert proposal.status == APPROVED
  // Apply changes to FIELD_MATRIX (or other targets)
  FIELD_MATRIX[(intent_type, query_type)] = APPLY_CHANGES(current, proposal)
  proposal.status = APPLIED
  LOG_APPLICATION(proposal_id)

REJECT(proposal_id, reason):
  proposal.status = REJECTED
  // Record reason for audit trail
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-091 | Learning proposals require explicit human approval before production application (UI-10) |
| UICE-092 | A minimum of 50 feedback records per (intent, query_type) is required before a proposal is generated |
| UICE-093 | Proposals with confidence < 0.50 must be flagged with a low-confidence warning |
| UICE-094 | Applied proposals are versioned; rollback must be possible within 24 hours |
| UICE-095 | Implicit signals from LLM response are best-effort; they never override explicit consumer ratings |

---

## Cross-References

- Context Ranking Engine (history_weight) → `11-CONTEXT-RANKING-ENGINE`
- Context Metrics (feedback collection) → `20-CONTEXT-METRICS`
- Context Profiler (field usage tracking) → `21-CONTEXT-PROFILER`
- UICE invariant UI-10 → `README.md`
