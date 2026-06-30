# UICE-DOC-012 — Context Compression Engine

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Compression Engine is the orchestrator of per-object assembly mode
selection. Given a ContextPlan and a set of ranked objects, it decides — for each
object — whether to use FLA, DELTA, SUMMARY, or FULL assembly, and then executes
the selection via the appropriate module.

This is the central throughput module of the UICE pipeline: it coordinates
FLA (module 03), Delta Generator (module 06), Summarizer (module 13), and
Budget Manager (module 10) into a single compression pass.

---

## Compression Modes

```
FLA (Field-Level Assembly):
  Selected fields from FIELD_MATRIX[(intent_type, query_type)]
  Applied to: objects not in memory, within budget, not CONTEXT_PRODUCTION
  Token count: 40–200 per object

DELTA (Conversation Delta):
  Only fields changed since last session send
  Applied to: objects in Conversation Memory (UI-02 verified)
  Token count: 8–80 per object (depends on change size)

SUMMARY (Compressed Summary):
  Condensed representation: top-N most relevant fields only
  Applied to: objects over budget, or low-relevance context slots
  Token count: 20–100 per object

FULL (L5 Full):
  Complete KIL object
  Applied to: CONTEXT_PRODUCTION intent only
  Token count: ≤ 2000 per object

OMIT:
  Object excluded from pack
  Applied to: objects in memory with NO_CHANGE delta and urgency != PRODUCTION
  Token count: 0 (object appears in pack_metadata only)
```

---

## Mode Selection Decision Tree

```
SELECT_COMPRESSION_MODE(object, intent_profile, memory_filter, budget) → Mode:

  // OMIT check (cheapest — do first)
  if memory_filter.contains(object.knowledge_id):
    delta = memory_filter.get_delta_type(object.knowledge_id)
    if delta == NO_CHANGE and intent_profile.urgency != PRODUCTION:
      return OMIT

  // FULL check (most expensive — check condition before applying)
  if intent_profile.intent_type == CONTEXT_PRODUCTION:
    return FULL

  // DELTA check
  if memory_filter.contains(object.knowledge_id):
    if memory_filter.get_delta_type(object.knowledge_id) in {VERSION_CHANGE, FIELD_UPDATE}:
      return DELTA
    if intent_profile.urgency == PRODUCTION:
      return DELTA  // send reminder delta for production-urgency packs

  // Budget check → SUMMARY fallback
  estimated_fla_tokens = ESTIMATE_FLA_TOKENS(object, intent_profile, budget)
  if estimated_fla_tokens > budget.slot_budgets.get(object.knowledge_id, 0):
    return SUMMARY

  // Default: FLA
  return FLA
```

---

## Compression Pipeline

```
COMPRESS(context_plan, ranked_objects, memory_filter, budget) → list[CompressedSlice]:

  compressed = []

  // Primary object
  primary_mode = context_plan.primary.assembly_mode
  primary_slice = APPLY_MODE(
    obj    = primary_kil_object,
    mode   = primary_mode,
    intent = intent_profile,
    query  = query_profile,
    memory = memory_filter,
  )
  compressed.append(primary_slice)

  // Context slots
  for slot in context_plan.context_slots:
    if slot.assembly_mode == OMIT:
      continue  // skip omitted slots
    mode = slot.assembly_mode
    slice = APPLY_MODE(obj=slot.kil_object, mode=mode, ...)
    compressed.append(slice)

  // SCO pass (if enabled)
  if context_plan.use_sco:
    preambles, compressed = SCO.OPTIMIZE_SHARED(compressed, kil_objects_by_id)
  else:
    preambles = []

  // Deduplication pass
  compressed = DEDUPLICATOR.DEDUPLICATE(compressed)

  return CompressedPack(
    primary  = compressed[0],
    context  = compressed[1:],
    preambles = preambles,
  )


APPLY_MODE(obj, mode, intent, query, memory) → Slice:
  match mode:
    FLA:     return FLA_MODULE.select(obj, intent.intent_type, query.query_type)
    DELTA:   return DELTA_GENERATOR.generate_delta(obj, memory.get_entry(obj.knowledge_id))
    SUMMARY: return SUMMARIZER.summarize(obj, intent, query, budget)
    FULL:    return FieldLevelSlice(fields=obj, mode=FULL)
    OMIT:    return OmitPlaceholder(knowledge_id=obj.knowledge_id)
```

---

## CompressedPack Schema

```yaml
CompressedPack:
  primary: FieldLevelSlice | DeltaSlice | SummarySlice
  context: list[FieldLevelSlice | DeltaSlice | SummarySlice | OmitPlaceholder]
  preambles: list[SharedPreamble]    # from SCO
  guard: AdaptiveGuardBlock          # from module 09 (appended after compression)

  compression_stats:
    total_objects_considered: int
    objects_fla: int
    objects_delta: int
    objects_summary: int
    objects_full: int
    objects_omit: int
    total_tokens_before_sco: int
    total_tokens_after_sco: int
    sco_savings: int
    dedup_savings: int
    final_tokens: int
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-056 | Primary object mode is never OMIT — only context slots may be omitted |
| UICE-057 | DELTA mode requires validated Conversation Memory entry (UI-02); absent entry triggers FLA |
| UICE-058 | SUMMARY mode is applied before OMIT; only if SUMMARY also exceeds budget is OMIT applied |
| UICE-059 | compression_stats must be populated for every CompressedPack — required by Context Profiler |
| UICE-060 | SCO and deduplication run after mode selection, not before; mode selection is not SCO-aware |

---

## Cross-References

- Field-Level Assembly → `03-FIELD-LEVEL-ASSEMBLY`
- Context Delta Generator → `06-CONTEXT-DELTA-GENERATOR`
- Context Summarizer → `13-CONTEXT-SUMMARIZER`
- Shared Context Optimizer → `05-SHARED-CONTEXT-OPTIMIZER`
- Context Deduplicator → `07-CONTEXT-DEDUPLICATOR`
- Token Budget Manager → `10-TOKEN-BUDGET-MANAGER`
