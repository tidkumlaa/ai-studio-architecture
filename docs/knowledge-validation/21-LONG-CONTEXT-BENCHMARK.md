# KVF-DOC-021 — Long-Context Benchmark

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Long-Context Benchmark validates KOS behavior when context packs are large
(>8K tokens) — as occurs for REASONING, IMPACT_ANALYSIS, and multi-hop queries
that require many related objects. Large packs stress test the quality model,
budget enforcement, and AH Guard.

---

## Long-Context Definition

```
Long context = total_tokens in assembled pack > 8,000 tokens

Corresponds to budget tiers:
  FULL:      8,000 tokens
  REASONING: 20,000 tokens
```

---

## Long-Context Scenarios

```
Scenario LC-01: Full Ecosystem Context
  Query: "Give me complete context for the Quota Manager for
          an AI agent that will implement a new rate policy."
  Intent: CONTEXT_PRODUCTION
  Budget: REASONING (20,000 tokens)
  Expected pack: L5 primary + 15–20 related objects at L3–L4
  Expected tokens: 12,000–18,000

Scenario LC-02: Multi-Hop Impact
  Query: "What is the full downstream impact if the Auth Service fails?"
  Intent: IMPACT_ANALYSIS
  Budget: FULL (8,000 tokens)
  Expected pack: primary + all transitive impacts (3 hops)
  Expected tokens: 6,000–8,000

Scenario LC-03: Cross-Package Traceability
  Query: "Trace the complete dependency chain for the Order Service."
  Intent: TRACEABILITY
  Budget: FULL (8,000 tokens)
  Expected pack: chain of 10+ objects at L3
  Expected tokens: 4,000–7,000

Scenario LC-04: Comparison Corpus
  Query: "Compare all implementations of the rate limiting policy."
  Intent: COMPARISON
  Budget: EXTENDED (3,000 tokens)
  Expected pack: 5–8 alternative objects at L3
  Expected tokens: 2,000–3,000
```

---

## Long-Context Validation Checks

```
LC-V-01: Budget compliance
  total_tokens <= declared_budget in 100% of long-context packs

LC-V-02: Primary object depth
  For L5-budget queries: primary at L5 (machine or audience)
  For L4-budget queries: primary at L4 minimum

LC-V-03: Guard block presence
  Guard block present and non-empty in all long-context PromptPacks

LC-V-04: Context quality score
  context_quality_score >= 0.70 for all long-context packs
  (higher budget should generally produce higher quality)

LC-V-05: Object count sanity
  Objects included <= (total_budget / 200) × 2
  (approximately max 2 objects per 200-token minimum)

LC-V-06: Missing context declared
  For required objects not included due to budget:
  missing_required_context is populated in guard block

LC-V-07: Token utilization
  utilization_rate = total_tokens / budget >= 0.70
  (long-context packs should use the budget they receive)

LC-V-08: Assembly latency at scale
  Long-context packs (>8K) must still be assembled in < 500ms P99
  (relaxed from standard 120ms due to larger object count)
```

---

## Information Density at Scale

```
information_density = objects_included / (total_tokens / 1000)
Expected for long context:
  1,000-token pack: 1–3 objects
  8,000-token pack: 8–20 objects
  20,000-token pack: 20–40 objects

density_score:
  >= 2.0 objects / 1K tokens → 1.0
  >= 1.5 → 0.8
  >= 1.0 → 0.6
  < 1.0 → 0.3 (pack is too sparse — objects too large relative to count)
```

---

## Long-Context Result Schema

```yaml
long_context_result:
  scenarios_run: integer

  budget_compliance_rate: float         # LC-V-01
  primary_depth_correct_rate: float     # LC-V-02
  guard_block_present_rate: float       # LC-V-03
  quality_score_above_min: float        # LC-V-04
  missing_context_declared_rate: float  # LC-V-06

  token_utilization:
    mean: float
    below_0_70: integer                  # under-utilized

  assembly_latency_p99_ms: float        # LC-V-08

  information_density:
    mean: float
    by_budget_tier:
      - tier: string
        mean_density: float
        mean_objects: float
        mean_tokens: float
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-101 | LC-V-01 (budget compliance) must be 100% — long context does not relax budget enforcement |
| KVF-102 | LC-V-08 latency target is 500ms P99 for packs > 8K — not 120ms (different from standard) |
| KVF-103 | Long-context packs with utilization < 0.50 are flagged — budget was over-declared |
| KVF-104 | LC-V-06 (missing context declared) must be present when required objects were dropped |
| KVF-105 | Long-context benchmark must run Scenarios LC-01 through LC-04 — partial scenarios insufficient |

---

## Cross-References

- Budget manager → Phase 3.0D.1 `07-BUDGET-MANAGER`
- Context planner (CONTEXT_PRODUCTION intent) → Phase 3.0D.1 `09-CONTEXT-PLANNER`
- Performance model → Phase 3.0D.1 `21-PERFORMANCE-MODEL`
- Cold-start benchmark → `22-COLD-START-BENCHMARK`
