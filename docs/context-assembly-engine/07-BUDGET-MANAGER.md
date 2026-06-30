# CAE-DOC-007 — Budget Manager

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Budget Manager allocates the total token budget across all selected objects
and the guard block. Budget allocation is a hard constraint — no assembly may
exceed its declared budget.

---

## Budget Partitions

```
Total Budget (B)
├── Guard Block:          B × 0.10  (protected — never reduced)
├── Primary Object:       B × 0.40  (protected — minimum L3)
├── Context Objects:      B × 0.30  (shared pool for CONTEXT + PREREQUISITE)
└── Related/Evidence:     B × 0.20  (dropped first when budget is tight)

Adjustable: if no ANTI_CONFUSION objects needed,
  their share (from Related pool) goes to Context.
```

---

## Standard Budget Tiers

| Tier | Total Tokens | Guard | Primary | Context | Related |
|------|-------------|-------|---------|---------|---------|
| NANO | 100 | 10 | 40 | 30 | 20 |
| MICRO | 300 | 30 | 120 | 90 | 60 |
| STANDARD | 1,000 | 100 | 400 | 300 | 200 |
| EXTENDED | 3,000 | 300 | 1,200 | 900 | 600 |
| FULL | 8,000 | 800 | 3,200 | 2,400 | 1,600 |
| REASONING | 20,000 | 2,000 | 8,000 | 6,000 | 4,000 |

Default budget when not specified: STANDARD (1,000 tokens).

---

## Allocation Algorithm

```
ALLOCATE(ranked_objects, budget_tokens, intent):

  # Step 1: Reserve guard block
  guard_budget = max(budget_tokens × 0.10, 50)  # minimum 50 tokens for guard
  available = budget_tokens - guard_budget

  # Step 2: Allocate primary
  primary_budget = available × 0.57  # 40% of total ≈ 57% of available
  primary_compression = COMPRESSION_SELECTOR(primary_budget, role=PRIMARY)

  # Step 3: Allocate context objects
  context_pool = available × 0.43   # remaining after primary
  context_objects = ranked_objects WHERE role in {CONTEXT, PREREQUISITE}
  anti_confusion = ranked_objects WHERE role == ANTI_CONFUSION

  # Step 4: Anti-confusion always gets L2 each (50 tokens)
  anti_confusion_budget = len(anti_confusion) × 50
  IF anti_confusion_budget > context_pool × 0.30:
    anti_confusion = anti_confusion[:floor(context_pool * 0.30 / 50)]
  context_pool -= len(anti_confusion) × 50

  # Step 5: Distribute remaining context pool
  IF len(context_objects) == 0:
    pass
  ELIF len(context_objects) == 1:
    per_context = context_pool
  ELSE:
    # Weighted by relevance score
    total_score = sum(obj.relevance_score for obj in context_objects)
    per_context = {obj: context_pool × obj.relevance_score/total_score
                   for obj in context_objects}

  # Step 6: Apply minimum floor
  per_context_min = 50   # minimum L2 per context object
  IF any(per_context[obj] < per_context_min for obj in context_objects):
    # Drop lowest-relevance context objects until all remaining get >= minimum
    WHILE per_context violations exist:
      drop lowest-relevance object
      redistribute its budget proportionally

  # Step 7: Assign compression levels
  COMPRESSION_MAP = {
    primary: COMPRESSION_SELECTOR(primary_budget, role=PRIMARY),
    **{obj: COMPRESSION_SELECTOR(per_context[obj], role=CONTEXT)
       for obj in context_objects},
    **{obj: L2 for obj in anti_confusion}
  }

  RETURN AllocationPlan(
    guard_budget=guard_budget,
    primary_budget=primary_budget,
    compression_map=COMPRESSION_MAP,
    dropped_objects=[obj for obj not in COMPRESSION_MAP]
  )
```

---

## Per-Object Budget → Compression Level

| Token Budget | Compression Level | Tokens Used |
|-------------|------------------|-------------|
| < 20 | L1 (nano) | ~15 |
| 20–60 | L2 (micro) | ~50 |
| 60–220 | L3 (mini) | ~200 |
| 220–550 | L4 (standard) | ~500 |
| > 550 | L5 (machine) | variable |

Primary object exception: minimum L3 regardless of budget.
If primary budget < 200 tokens: compress other objects first; primary stays L3.

---

## Budget Enforcement Schema

```yaml
allocation_plan:
  total_budget: integer
  guard_budget: integer
  primary_allocation:
    knowledge_id: string
    budget: integer
    compression_level: string
  context_allocations:
    - knowledge_id: string
      role: string
      budget: integer
      compression_level: string
      relevance_score: float
  dropped_objects:
    - knowledge_id: string
      relevance_score: float
      drop_reason: BUDGET_EXHAUSTED | AIRS_TOO_LOW | ROLE_PRUNED
  utilization:
    allocated: integer                 # sum of all allocated tokens
    guard: integer
    reserved: integer                  # budget - allocated (buffer)
    utilization_rate: float            # allocated / total_budget
```

---

## Budget Overrun Handling

```
IF estimated_tokens(assembly) > budget:
  Step 1: Drop all DROPPED-role objects
  Step 2: Downgrade EVIDENCE objects to L1
  Step 3: Downgrade CONTEXT objects from L4→L3, L3→L2
  Step 4: Drop lowest-relevance CONTEXT objects until budget fits
  Step 5: IF still over budget after dropping all context:
    Downgrade primary from L4→L3 (L3 is the minimum)
  Step 6: IF still over budget:
    RAISE AssemblyError(BUDGET_TOO_SMALL)
    message: "Minimum assembly requires {min_tokens} tokens;
              budget is {budget} tokens"
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-026 | Guard block minimum is max(budget × 0.10, 50 tokens) — never less |
| CAE-027 | Primary object minimum compression is L3 — L1/L2 for primary is forbidden |
| CAE-028 | Budget overrun must try graceful degradation before raising AssemblyError |
| CAE-029 | `allocation_plan.utilization_rate` must be ≤ 1.00 — over-allocation is a bug |
| CAE-030 | Dropped objects must be recorded in `dropped_objects` for feedback |

---

## Cross-References

- Compression selector → `08-COMPRESSION-SELECTOR`
- Context planner → `09-CONTEXT-PLANNER`
- Context assembler → `10-CONTEXT-ASSEMBLER`
- Performance targets → `21-PERFORMANCE-MODEL`
