# CAE-DOC-022 — Feedback Model

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Feedback Model defines how the CAE learns from assembly outcomes to improve
future context quality. Feedback flows from two sources: explicit consumer signals
and implicit usage signals. All feedback is stored in the KIL usage model — the
CAE itself is stateless.

---

## Feedback Sources

```
Source 1: Explicit Consumer Feedback
  AI agents: return {pack_id, useful: bool, missing_info: [string], misleading: [string]}
  Human readers: rate helpfulness (1–5), flag inaccuracies

Source 2: Implicit Signals
  Pack re-query: consumer queried for additional context within 60 seconds
    → signal: original pack was incomplete
  Rapid follow-up query on same object: within 30 seconds
    → signal: question was not answered
  Pack abandoned: pack fetched but no follow-up activity
    → signal: pack may have been unused or unhelpful
```

---

## Feedback Record Schema

```yaml
feedback_record:
  feedback_id: string                  # UUID
  pack_id: string                      # the pack being rated
  query_id: string
  consumer_type: string                # AI_AGENT | DEVELOPER | OPERATOR | SEARCH
  source: EXPLICIT | IMPLICIT
  received_at: string

  # Explicit fields (when source == EXPLICIT)
  explicit:
    rating: integer | null             # 1–5 (null if AI agent)
    useful: boolean | null             # null if human didn't answer
    missing_info: [string]             # what was missing from the pack
    misleading: [string]              # what was incorrect or misleading
    hallucination_flagged: boolean     # AI agent detected guard-block violation

  # Implicit fields (when source == IMPLICIT)
  implicit:
    signal_type: RE_QUERY | RAPID_FOLLOW_UP | ABANDONED | USED
    latency_to_signal_ms: integer
    follow_up_query_id: string | null

  # Derived scores
  derived:
    pack_helpfulness_score: float      # computed from all signals
    primary_object_rated: boolean      # true if feedback is attributable to primary
    context_gap_detected: boolean      # true if re-query or missing_info non-empty
```

---

## Helpfulness Score Computation

```
compute_helpfulness(feedback_record):

  IF source == EXPLICIT:
    IF consumer_type == AI_AGENT:
      score = 1.0 if useful else 0.2
      IF hallucination_flagged: score = max(score - 0.3, 0.0)
    ELSE:
      # human rating 1–5 normalized to 0.0–1.0
      score = (rating - 1) / 4.0

  IF source == IMPLICIT:
    CASE signal_type:
      USED:             score = 0.70   # neutral-positive
      RE_QUERY:         score = 0.40   # incomplete — pack didn't answer
      RAPID_FOLLOW_UP:  score = 0.50   # may have been useful but insufficient
      ABANDONED:        score = 0.30   # likely unhelpful

  RETURN score
```

---

## Feedback → KIL Update Protocol

Feedback does NOT directly modify the KIL. It is written to a feedback log
and batch-applied by the KIL Maintenance Process. Only the maintenance process
(not the CAE) may update KIL fields.

```
Feedback influences these KIL fields (via maintenance process):

  kil.usage.access_log: appended per pack access
  kil.usage.co_occurrence_matrix: updated from pack membership
  kil.memory.helpfulness_scores: appended from explicit feedback
  kil.ai_context.typical_mistakes: updated if hallucination_flagged
  kil.ai_context.questions: updated from missing_info signals
  kil.reasoning.relationships: review triggered if context_gap_detected
```

---

## Feedback-Driven Assembly Improvement

The CAE uses feedback signals to adjust future assemblies:

```
USE_FEEDBACK_SIGNALS(primary_object_id):

  history = FEEDBACK_LOG.get_recent(primary_object_id, days=30)

  # Signal 1: context gap rate
  gap_rate = count(h WHERE context_gap_detected) / len(history)
  IF gap_rate > 0.30:
    # Increase related context for this object
    ASSEMBLY_HINTS.set(primary_object_id, "increase_context_depth", true)

  # Signal 2: hallucination rate
  hallucination_rate = count(h WHERE hallucination_flagged) / len(history)
  IF hallucination_rate > 0.10:
    # Flag for KIL maintenance — cortex.common_errors needs update
    MAINTENANCE_QUEUE.add(
      type = KIL_REVIEW_NEEDED,
      knowledge_id = primary_object_id,
      reason = "Hallucination rate > 10%"
    )

  # Signal 3: abandonment rate
  abandonment_rate = count(h WHERE signal_type == ABANDONED) / len(history)
  IF abandonment_rate > 0.40:
    # Object's KIL blocks may be low quality
    MAINTENANCE_QUEUE.add(
      type = QUALITY_REVIEW_NEEDED,
      knowledge_id = primary_object_id,
      reason = f"Pack abandonment rate {abandonment_rate:.0%}"
    )
```

---

## Feedback Loop Architecture

```
Consumer → Feedback API
    ↓
Feedback Log (append-only)
    ↓
Batch Aggregation (every 6 hours)
    ↓
┌──────────────────────┐
│  Assembly Hints DB   │  ← Influences next assembly (stateful lookup)
└──────────────────────┘
    ↓
KIL Maintenance Queue → KIL Maintenance Process → KIL Object Updates
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-101 | Feedback must NOT directly modify KIL objects — only the maintenance process may |
| CAE-102 | All pack accesses must generate an implicit feedback record (USED signal minimum) |
| CAE-103 | Hallucination flags must be routed to KIL maintenance queue within 1 hour |
| CAE-104 | Feedback log is append-only — no feedback record may be deleted or modified |
| CAE-105 | Assembly hints derived from feedback must be recalculated every 6 hours |

---

## Cross-References

- Context learning → `24-CONTEXT-LEARNING`
- Context metrics → `23-CONTEXT-METRICS`
- CAE API → `25-CAE-API` (feedback endpoint)
- KIL memory → Phase 3.0D.0.6 `09-KNOWLEDGE-MEMORY`
- KIL usage model → Phase 3.0D.0.6 `10-KNOWLEDGE-USAGE-MODEL`
