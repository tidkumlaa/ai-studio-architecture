# UICE-DOC-004 — Dynamic Context Planner

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Dynamic Context Planner produces a **ContextPlan** — a complete blueprint
for one context pack assembly. It takes IntentProfile, QueryProfile, budget
constraints, and Conversation Memory signals and decides:

- Which objects enter the pack and in what role
- Which assembly mode applies to each (FLA / Delta / Full / Summary)
- How the token budget is partitioned
- Whether SCO, delta, and summarization are warranted

The planner replaces the static slot assignment of CAE S6 (Context Planner)
with a dynamic, constraint-aware allocation algorithm.

---

## ContextPlan Schema

```yaml
ContextPlan:
  plan_id: string                       # unique plan identifier
  session_id: string | null             # conversation session
  query_summary: string                 # 1-line query description

  # Object roles
  primary:
    knowledge_id: string
    assembly_mode: FLA | DELTA | FULL | SUMMARY
    estimated_tokens: int
    in_memory: bool                     # already sent this session

  context_slots: list[ContextSlot]
    ContextSlot:
      knowledge_id: string
      role: DEPENDENCY | RELATED | COMPARISON | BACKGROUND
      assembly_mode: FLA | DELTA | SUMMARY | OMIT
      estimated_tokens: int
      in_memory: bool
      relevance_score: float

  # Optimization decisions
  use_sco: bool                         # Shared Context Optimizer
  use_delta: bool                       # conversation delta mode
  use_summarizer: bool                  # any slots need summarization
  guard_mode: ADAPTIVE | NONE           # NONE only for non-PromptPack

  # Budget allocation
  budget_allocation:
    total_budget: int
    guard_budget: int                   # 10% protected (CAE CI-03)
    primary_budget: int                 # 40% minimum (CAE CI-01)
    context_budget: int                 # 30%
    related_budget: int                 # 20%
    sco_budget: int                     # subtracted from primary (shared preamble)

  # Model and optimization
  model_profile_id: string
  cost_target: MINIMIZE | BALANCED | QUALITY
  latency_target: STRICT | NORMAL | RELAXED

  # Estimates
  estimated_total_tokens: int
  estimated_latency_ms: int
  estimated_cost_usd: float
  plan_version: string
```

---

## Planning Algorithm

```
PLAN_CONTEXT(intent_profile, query_profile, budget, memory_filter,
             ranked_objects) → ContextPlan:

  plan_id = GENERATE_ID()

  // Step 1: Primary object
  primary_id = ranked_objects[0].knowledge_id
  primary_in_memory = memory_filter.contains(primary_id)

  primary_mode = SELECT_MODE(primary_id, primary_in_memory, intent_profile):
    if primary_in_memory:
      version_changed = memory_filter.version_changed(primary_id)
      return DELTA if version_changed else SUMMARY   // send delta or reminder
    if intent_profile.intent_type == CONTEXT_PRODUCTION:
      return FULL
    return FLA  // default: field-level assembly

  primary_tokens = ESTIMATE_TOKENS(primary_id, primary_mode, intent_profile, query_profile)

  // Step 2: Budget allocation (CAE CI-01, CI-02, CI-03 preserved)
  guard_budget   = floor(budget * 0.10)   // protected
  primary_budget = max(floor(budget * 0.40), primary_tokens)
  context_budget = floor(budget * 0.30)
  related_budget = budget - guard_budget - primary_budget - context_budget

  // Step 3: Context slot assignment
  slots = []
  remaining = context_budget + related_budget
  for obj in ranked_objects[1:]:
    if remaining <= 0: break
    in_mem = memory_filter.contains(obj.knowledge_id)
    mode = SELECT_MODE(obj.knowledge_id, in_mem, intent_profile)
    est = ESTIMATE_TOKENS(obj.knowledge_id, mode, intent_profile, query_profile)
    if est <= remaining:
      role = CLASSIFY_ROLE(obj, query_profile)
      slots.append(ContextSlot(knowledge_id=obj.knowledge_id, role=role,
                               assembly_mode=mode, estimated_tokens=est,
                               in_memory=in_mem, relevance_score=obj.relevance))
      remaining -= est

  // Step 4: SCO decision
  namespace_groups = GROUP_BY_NAMESPACE([primary_id] + [s.knowledge_id for s in slots])
  use_sco = any(len(g) >= 2 for g in namespace_groups.values())

  // Step 5: Summarization decision
  use_summarizer = any(s.assembly_mode == SUMMARY for s in slots)

  // Step 6: Guard mode
  guard_mode = ADAPTIVE if pack_type == PROMPT_PACK else NONE

  return ContextPlan(
    plan_id=plan_id,
    primary=PrimarySlot(knowledge_id=primary_id, assembly_mode=primary_mode,
                        estimated_tokens=primary_tokens, in_memory=primary_in_memory),
    context_slots=slots,
    use_sco=use_sco,
    use_delta=(primary_mode == DELTA or any(s.assembly_mode == DELTA for s in slots)),
    use_summarizer=use_summarizer,
    guard_mode=guard_mode,
    budget_allocation=BudgetAllocation(
      total_budget=budget, guard_budget=guard_budget,
      primary_budget=primary_budget, context_budget=context_budget,
      related_budget=related_budget),
    estimated_total_tokens=primary_tokens + sum(s.estimated_tokens for s in slots) + guard_budget,
  )
```

---

## Assembly Mode Selection Logic

```
SELECT_MODE(knowledge_id, in_memory, intent_profile) → AssemblyMode:

  if in_memory:
    if KNOWLEDGE_OBJECT_VERSION_CHANGED(knowledge_id):
      return DELTA         // send only what changed
    elif intent_profile.urgency == PRODUCTION:
      return SUMMARY       // short reminder only
    else:
      return OMIT          // already known, skip entirely
  
  if intent_profile.intent_type == CONTEXT_PRODUCTION:
    return FULL            // AI agent needs everything
  
  if OVER_BUDGET(knowledge_id, available_tokens):
    return SUMMARY         // force summarize if too large

  return FLA               // default: field-level assembly
```

---

## Role Classification

```
CLASSIFY_ROLE(obj, query_profile) → SlotRole:
  if obj.knowledge_id in query_profile.comparison_targets:
    return COMPARISON
  if obj in direct_dependencies(primary):
    return DEPENDENCY
  if obj in secondary_relationships(primary):
    return RELATED
  return BACKGROUND
```

Priority order: DEPENDENCY > COMPARISON > RELATED > BACKGROUND

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-016 | ContextPlan must be produced before any assembly begins — no ad-hoc assembly |
| UICE-017 | Budget allocation must preserve CAE CI-01 (primary ≥ 40%) and CI-03 (guard = 10%) |
| UICE-018 | OMIT mode is only valid for context slots; primary object is never OMIT |
| UICE-019 | use_delta requires Conversation Memory to be active; delta without memory is INVALID |
| UICE-020 | estimated_total_tokens in ContextPlan must not exceed budget; plan must fit before assembly starts |

---

## Cross-References

- Intent Analyzer → `01-INTENT-ANALYZER`
- Query Classifier → `02-QUERY-CLASSIFIER`
- Conversation Memory → `16-CONVERSATION-MEMORY`
- Context Ranking → `11-CONTEXT-RANKING-ENGINE`
- Token Budget Manager → `10-TOKEN-BUDGET-MANAGER`
- Context Verifier → `26-CONTEXT-VERIFIER`
