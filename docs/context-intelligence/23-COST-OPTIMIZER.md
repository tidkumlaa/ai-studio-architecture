# UICE-DOC-023 — Cost Optimizer

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Cost Optimizer ensures that context assembly and downstream model calls
stay within configured cost budgets. When a cost constraint is present
(BudgetConstraints.max_cost_usd is set), the optimizer selects the most
cost-efficient strategy that still meets quality requirements.

Cost optimization is a secondary constraint after token correctness:
the optimizer never violates CAE CI-01 (primary ≥ 40%) or UI-06 (quality ≥ 0.80)
to save money.

---

## Cost Components

```
Total cost per UICE assembly:
  assembly_cost = uice_infrastructure_cost (fixed; negligible)
  model_cost    = (context_tokens_in + response_tokens_out)
                  × model.token_cost_per_1k / 1000

UICE's primary cost lever: reducing context_tokens_in.
Every 1% token reduction = 1% model cost reduction.
```

---

## CostOptimizationStrategy Schema

```yaml
CostOptimizationStrategy:
  strategy_id: string
  cost_target: MINIMIZE | BALANCED | QUALITY

  # Model selection (when multiple models available)
  preferred_model_id: string | null
  model_fallback_chain: list[string]  # if preferred exceeds budget

  # Compression aggressiveness
  max_primary_tokens: int | null      # override for cost reduction
  max_context_tokens: int | null
  force_delta_mode: bool              # force delta even on first send
  force_summary_mode: bool            # force SUMMARY for all context slots

  # Cost estimates
  estimated_cost_usd: float
  estimated_savings_vs_naive: float   # vs full L4 context + default model
```

---

## Cost Optimization Algorithm

```
OPTIMIZE_FOR_COST(context_plan, model_profile, constraints) → CostOptimizationStrategy:

  max_cost = constraints.max_cost_usd
  if max_cost is None:
    return DEFAULT_STRATEGY()  // no cost constraint — pass through

  // Estimate current cost
  current_tokens = context_plan.estimated_total_tokens
  current_cost   = current_tokens * model_profile.token_cost_per_1k_input / 1000

  if current_cost <= max_cost:
    return DEFAULT_STRATEGY()  // already within budget

  // Try cheaper model first (least impact on quality)
  for fallback_model in GET_CHEAPER_MODELS(model_profile):
    fallback_cost = current_tokens * fallback_model.token_cost_per_1k_input / 1000
    if fallback_cost <= max_cost:
      return CostOptimizationStrategy(
        preferred_model_id   = fallback_model.model_id,
        estimated_cost_usd   = fallback_cost,
      )

  // Try reducing token count (impacts quality more)
  target_tokens = floor(max_cost / model_profile.token_cost_per_1k_input * 1000)
  reduction_needed = current_tokens - target_tokens

  // Force SUMMARY for lowest-relevance context slots
  savings_from_summary = ESTIMATE_SUMMARY_SAVINGS(context_plan)
  if savings_from_summary >= reduction_needed:
    return CostOptimizationStrategy(
      force_summary_mode     = True,
      max_context_tokens     = target_tokens - context_plan.primary.estimated_tokens,
      estimated_cost_usd     = max_cost,
    )

  // Last resort: reduce context slot count
  trimmed_plan = TRIM_SLOTS_FOR_COST(context_plan, target_tokens)
  estimated_cost = trimmed_plan.estimated_total_tokens * model_profile.token_cost_per_1k_input / 1000
  return CostOptimizationStrategy(
    max_primary_tokens   = trimmed_plan.primary.estimated_tokens,
    max_context_tokens   = target_tokens - trimmed_plan.primary.estimated_tokens,
    estimated_cost_usd   = estimated_cost,
  )
```

---

## Model Cost Reference (Approximate)

```
For relative cost comparison only (actual prices vary):

CLAUDE Haiku:    very low cost, moderate quality
CLAUDE Sonnet:   moderate cost, high quality
CLAUDE Opus:     high cost, highest quality
GPT-4o-mini:     very low cost, good quality
GPT-4o:          moderate cost, high quality
Gemini Flash:    very low cost, moderate quality
Gemini Pro:      moderate cost, high quality
DeepSeek Chat:   very low cost, good for code
LOCAL:           infrastructure cost only

Cost optimization cascade (MINIMIZE strategy):
  1. Switch to cheaper model in same family
  2. Switch to different cheaper model family
  3. Reduce context token count
  4. Reduce context object count
```

---

## COST_TARGET Behavior

```
MINIMIZE:
  Optimize for lowest possible cost.
  May degrade quality toward minimum threshold (quality_score ≥ 0.80).
  Will switch model families if needed.

BALANCED (default):
  Balance cost and quality.
  Prefer token reduction over model downgrade.
  Target: cost within 20% of cheapest option while maintaining quality ≥ 0.85.

QUALITY:
  Do not compromise quality for cost.
  Only optimize if pack is already over quality target.
  Cost is secondary to quality_score ≥ 0.90.
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-111 | Cost optimization never reduces primary token budget below CAE CI-01 minimum (40%) |
| UICE-112 | Quality score floor (UI-06: ≥ 0.80) is never violated for cost savings |
| UICE-113 | Model downgrade for cost requires BudgetConstraints.priority == COST; not applied under QUALITY priority |
| UICE-114 | Estimated cost must be recalculated after every compression step |
| UICE-115 | If max_cost_usd is null, Cost Optimizer is a no-op; it never applies heuristics without explicit constraint |

---

## Cross-References

- Token Budget Manager → `10-TOKEN-BUDGET-MANAGER`
- Model Capability Adapter → `22-MODEL-CAPABILITY-ADAPTER`
- Latency Optimizer → `24-LATENCY-OPTIMIZER`
- Quality Optimizer → `25-QUALITY-OPTIMIZER`
