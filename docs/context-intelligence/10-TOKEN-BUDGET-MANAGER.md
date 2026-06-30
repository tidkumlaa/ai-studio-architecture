# UICE-DOC-010 — Token Budget Manager

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Token Budget Manager enforces multi-constraint optimization on every
context pack. It receives a ContextPlan with estimated token counts and a
set of constraints (max_tokens, max_cost, max_latency, max_objects) and
validates — or rebalances — the plan to fit within all constraints.

UICE extends CAE's single-budget model (tokens only) with four independent
constraint axes. The budget manager optimizes across all four simultaneously.

---

## BudgetConstraints Schema

```yaml
BudgetConstraints:
  max_tokens: int                    # hard limit — never exceeded
  max_cost_usd: float | null         # cost ceiling per pack (optional)
  max_latency_ms: int | null         # compilation latency ceiling (optional)
  max_objects: int | null            # max objects in pack (optional)

  # Priority: which constraint to optimize for when trade-offs arise
  priority: TOKENS | COST | LATENCY | QUALITY
    # TOKENS:  minimize token count (default)
    # COST:    minimize API cost
    # LATENCY: minimize compilation time
    # QUALITY: maximize context quality score

  # Per-section budgets (override automatic allocation)
  guard_tokens_override: int | null
  primary_tokens_override: int | null
```

---

## BudgetAllocation Schema

```yaml
BudgetAllocation:
  total_budget: int
  guard_budget: int          # min 10% of total (CAE CI-03)
  primary_budget: int        # min 40% of total (CAE CI-01)
  context_budget: int        # up to 30%
  related_budget: int        # remaining after above allocations
  sco_budget: int            # tokens saved by SCO (reduces primary actual usage)

  # Per-slot allocations
  slot_budgets: dict[str, int]   # knowledge_id → token budget

  # Utilization
  estimated_total: int
  slack_tokens: int              # total_budget - estimated_total

  # Constraint satisfaction
  tokens_ok: bool
  cost_ok: bool
  latency_ok: bool
  objects_ok: bool
  all_ok: bool
```

---

## Budget Allocation Algorithm

```
ALLOCATE_BUDGET(context_plan, constraints) → BudgetAllocation:

  // Step 1: Validate object count
  n_objects = 1 + len(context_plan.context_slots)
  if constraints.max_objects and n_objects > constraints.max_objects:
    // Drop lowest-relevance slots until within limit
    context_plan = TRIM_SLOTS(context_plan, constraints.max_objects)

  // Step 2: Fixed allocations (CAE CI-01, CI-03 preserved)
  guard_budget   = max(
    floor(constraints.max_tokens * 0.10),
    context_plan.guard_estimated_tokens,
  )
  primary_budget = max(
    floor(constraints.max_tokens * 0.40),
    context_plan.primary.estimated_tokens,
  )

  // Step 3: Check if primary alone exceeds 50% of budget
  if primary_budget > constraints.max_tokens * 0.50:
    // Apply L3 floor instead — primary is too large for this budget
    primary_budget = floor(constraints.max_tokens * 0.50)
    context_plan.primary.assembly_mode = SUMMARY

  // Step 4: Remaining budget for context and related slots
  remaining = constraints.max_tokens - guard_budget - primary_budget

  // Step 5: Prioritized slot allocation
  slot_budgets = {}
  for slot in SORT_BY_RELEVANCE_DESC(context_plan.context_slots):
    if remaining <= 0:
      MARK_OMIT(slot)
      continue
    slot_budget = min(slot.estimated_tokens, remaining)
    slot_budgets[slot.knowledge_id] = slot_budget
    remaining -= slot_budget
    if slot.estimated_tokens > slot_budget:
      slot.assembly_mode = SUMMARY  // force summarize if budget too small

  // Step 6: SCO adjustment
  sco_saving = ESTIMATE_SCO_SAVING(context_plan) if context_plan.use_sco else 0
  primary_budget -= sco_saving  // SCO preamble replaces repeated content

  // Step 7: Cost constraint check
  if constraints.max_cost_usd:
    estimated_cost = ESTIMATE_COST(sum(slot_budgets.values()) + guard_budget + primary_budget,
                                   model_profile.cost_per_1k)
    if estimated_cost > constraints.max_cost_usd:
      context_plan = REDUCE_FOR_COST(context_plan, constraints.max_cost_usd)

  // Step 8: Latency constraint check
  if constraints.max_latency_ms:
    estimated_latency = ESTIMATE_LATENCY(n_objects, context_plan.use_sco,
                                          context_plan.use_delta)
    if estimated_latency > constraints.max_latency_ms:
      context_plan = APPLY_LATENCY_OPTIMIZATIONS(context_plan)

  total = guard_budget + primary_budget + sum(slot_budgets.values())
  return BudgetAllocation(
    total_budget  = constraints.max_tokens,
    guard_budget  = guard_budget,
    primary_budget = primary_budget,
    context_budget = floor(constraints.max_tokens * 0.30),
    related_budget = remaining,
    sco_budget     = sco_saving,
    slot_budgets   = slot_budgets,
    estimated_total = total,
    slack_tokens   = constraints.max_tokens - total,
    tokens_ok      = total <= constraints.max_tokens,
    all_ok         = True,
  )
```

---

## Four Constraint Axes

```
AXIS 1: TOKENS (primary constraint)
  Hard limit — never violated.
  Automatic rebalancing: trim slots, force SUMMARY mode.
  CAE CI-01 (primary ≥ 40%) and CI-03 (guard ≥ 10%) preserved.

AXIS 2: COST (optional)
  Compute: tokens × model.cost_per_1k_tokens
  Action when exceeded: switch to cheaper model, reduce context,
                        or force delta mode.

AXIS 3: LATENCY (optional)
  Estimate: graph traversal + assembly + model call time
  Action when exceeded: use Semantic Cache, enable parallel assembly,
                        skip optional traversal.

AXIS 4: OBJECTS (optional)
  Hard limit on number of context objects.
  Action when exceeded: trim lowest-relevance slots first.
```

---

## Guard Budget Protection

The guard budget is always protected:

```
guard_budget_min = max(
  floor(max_tokens * 0.10),   // 10% floor
  20,                          // absolute minimum 20 tokens
)

If primary + context + related would exhaust budget before guard:
  Trim context/related slots first.
  Primary may be reduced to SUMMARY mode.
  Guard is never reduced below guard_budget_min.
```

This ensures CAE CI-03 is preserved under all constraint scenarios.

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-046 | max_tokens is a hard limit — BudgetAllocation.estimated_total must never exceed it |
| UICE-047 | Guard budget receives priority allocation before context and related slots |
| UICE-048 | Primary budget minimum is 40% of max_tokens; primary may be forced to SUMMARY but never OMIT |
| UICE-049 | Cost constraint check is performed after token allocation; cost reduction never violates token budget |
| UICE-050 | BudgetAllocation.all_ok must be true before ContextPlan is passed to assembly stages |

---

## Cross-References

- Dynamic Context Planner → `04-DYNAMIC-CONTEXT-PLANNER`
- Context Compression Engine → `12-CONTEXT-COMPRESSION-ENGINE`
- Cost Optimizer → `23-COST-OPTIMIZER`
- Latency Optimizer → `24-LATENCY-OPTIMIZER`
- CAE CI-01, CI-02, CI-03 → Phase 3.0D.1 `03-CAE-INVARIANTS`
