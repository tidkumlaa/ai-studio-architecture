---
knowledge_id: KNW-PLAT-ARCH-021
title: "Platform Algorithms"
status: AUTHORITATIVE
phase: "2.1D.0"
version: "1.0.0"
created: "2026-06-29"
purpose: "Specify all non-trivial algorithms used in the platform: routing, prediction, scoring, learning"
canonical_source: "architecture/docs/platform-consolidation/21-ALGORITHMS.md"
dependencies:
  - "20-DATA-STRUCTURES.md"
  - "05-MODULE-CATALOG.md"
related_documents:
  - "27-PERFORMANCE-BUDGET.md"
  - "22-MIGRATION-ENGINE.md"
acceptance_criteria:
  - "Every algorithm has a pseudocode or Python specification"
  - "Time and space complexity stated for each"
  - "All numeric constants named and explained"
verification_checklist:
  - "[ ] Algorithms are deterministic (no random without seeding policy)"
  - "[ ] All constants defined as named module-level values"
  - "[ ] Edge cases (empty input, zero values) handled"
future_extensions:
  - "Algorithm performance benchmarks in 27-PERFORMANCE-BUDGET.md"
  - "ML-based replacement candidates identified"
---

# Platform Algorithms

## A1 — Workload Type Detection

**Module:** `workload_profiler`  
**Complexity:** O(W × S) where W = WorkloadType count (18), S = max signals per type (~10)

```
ALGORITHM: detect_workload_type(prompt: str) -> WorkloadType

INPUT:   prompt — raw user text
OUTPUT:  WorkloadType with highest signal score

1. Normalize prompt: lowercase, strip punctuation
2. For each workload_type T in WorkloadType (excluding UNKNOWN):
   a. score[T] = 0.0
   b. For each (keyword, weight) in SIGNALS[T]:
      - If keyword in normalized_prompt:
          score[T] += weight
3. best_type = argmax(score)
4. If score[best_type] == 0.0: return UNKNOWN
5. Return best_type

CONSTANTS:
  SIGNALS: dict[WorkloadType, list[tuple[str, float]]]
  Weights range: 0.05 – 0.30 per keyword
```

**Edge cases:**
- Empty prompt → UNKNOWN
- Tie → first match in enum declaration order

---

## A2 — Task Complexity Scoring

**Module:** `task_complexity`  
**Complexity:** O(P) where P = prompt word count

```
ALGORITHM: score_complexity(prompt: str, profile: WorkloadProfile) -> float

INPUT:   prompt, workload profile
OUTPUT:  score in [0.0, 1.0]

1. word_count = len(prompt.split())
2. word_score:
   - word_count < 50  → 0.10
   - word_count < 200 → 0.20
   - word_count >= 200 → 0.30
3. complexity_words = count of COMPLEXITY_KEYWORDS in prompt
   complexity_score = min(0.30, complexity_words × 0.05)
4. step_words = count of STEP_KEYWORDS in prompt
   step_score = min(0.15, step_words × 0.05)
5. code_words = count of CODE_KEYWORDS in prompt
   code_score = min(0.15, code_words × 0.05)
6. profile_bonus:
   - workload_type in {ARCHITECTURE, SPECIFICATION, REVERSE_ENGINEERING} → 0.20
   - otherwise → 0.0
7. total = word_score + complexity_score + step_score + code_score + profile_bonus
8. return min(1.0, total)

THRESHOLDS:
  score < 0.15  → TRIVIAL
  score < 0.35  → SIMPLE
  score < 0.50  → MODERATE
  score < 0.75  → COMPLEX
  score >= 0.75 → EXPERT
```

---

## A3 — Token Prediction (EMA Learning)

**Module:** `token_predictor`  
**Complexity:** O(1) predict, O(1) update

```
ALGORITHM: predict_tokens(prompt: str, workload_type: WorkloadType, model_id: str)
           -> (input_tokens, output_tokens)

INPUT_TOKEN_ESTIMATE:
  input_tokens = ceil(len(prompt.split()) × 1.33)  # words to tokens ratio

OUTPUT_TOKEN_ESTIMATE:
  ratio = OUTPUT_RATIOS[workload_type]  # default if not learned
  output_tokens = ceil(input_tokens × ratio)

ALGORITHM: record_outcome(workload_type, actual_output_ratio)
  # EMA update
  α = 0.3   # learning rate for new data
  β = 0.7   # weight for existing knowledge
  OUTPUT_RATIOS[workload_type] = β × old_ratio + α × actual_output_ratio

CONSTANTS:
  DEFAULT_OUTPUT_RATIOS: dict[WorkloadType, float]
    CODING → 2.5
    ARCHITECTURE → 3.0
    DOCUMENTATION → 2.0
    CONVERSATION → 1.2
    ... (one per WorkloadType)
  α = 0.3
  β = 0.7
```

---

## A4 — Cost Prediction

**Module:** `cost_predictor`  
**Complexity:** O(1)

```
ALGORITHM: predict_cost(model_id, input_tokens, output_tokens) -> CostPrediction

1. (price_in, price_out) = PRICING[model_id]  # USD per 1M tokens
2. input_cost = Decimal(input_tokens) / 1_000_000 × price_in
3. output_cost = Decimal(output_tokens) / 1_000_000 × price_out
4. expected = input_cost + output_cost
5. best_case = expected × 0.7   # optimistic: fewer output tokens
6. worst_case = expected × 1.5  # pessimistic: retries + longer output
7. return CostPrediction(best_case, expected, worst_case)

CONSTANTS:
  PRICING: dict[str, tuple[Decimal, Decimal]]
  All prices in USD per 1 million tokens (input, output)
```

---

## A5 — Provider Selection Scoring

**Module:** `provider_selector`  
**Complexity:** O(P) where P = provider count (~5)

```
ALGORITHM: select_provider(context: SelectionContext) -> ProviderSelection

FOR each provider_id in available_providers:
  1. quality_score = QUALITY[provider_id]  # 0.0–1.0
  2. latency_score = 1.0 - (LATENCY_MS[provider_id] / MAX_LATENCY)
  3. cost_estimate = estimate_cost(provider_id, context)
  4. If context.max_cost_usd is not None and cost_estimate > context.max_cost_usd:
       SKIP this provider  # note: Decimal("0") is falsy — use `is not None`
  5. composite = (quality_score × 0.5) + (latency_score × 0.3) + (1.0 - normalized_cost × 0.2)
  6. candidates.append((provider_id, composite))

SELECT: provider with max composite score
FALLBACK: if no candidates pass budget filter → return LOCAL provider
```

---

## A6 — Model Selection

**Module:** `model_selector`  
**Complexity:** O(M) where M = catalog size (~11)

```
ALGORITHM: select_model(workload, complexity, quality_requirement, budget) -> ModelSelection

FOR each model in CATALOG:
  1. If model does not meet quality_requirement: SKIP
  2. If model requires capabilities not in workload: SKIP
  3. score = quality_weight × model.quality - cost_weight × model.cost_per_1m
  4. candidates.append((model, score))

SORT candidates by score descending
SELECT top candidate
FALLBACK if none: lowest cost model that meets minimum quality
```

---

## A7 — Execution Complexity Decision

**Module:** `execution_planner`  
**Complexity:** O(1)

```
ALGORITHM: decide_execution_strategy(complexity: ComplexityProfile) -> ExecutionStrategy

difficulty_score = {
  TRIVIAL: 0.1, SIMPLE: 0.3, MODERATE: 0.5, COMPLEX: 0.7, EXPERT: 0.9
}[complexity.level]

IF complexity.parallelizable AND difficulty_score > 0.5:
    return MULTI_AGENT_PARALLEL
ELIF complexity.requires_coordination AND difficulty_score > 0.5:
    return MULTI_AGENT_SEQUENTIAL
ELSE:
    return SINGLE_AGENT
```

---

## A8 — Learning Accuracy Trend

**Module:** `learning_engine`  
**Complexity:** O(N) where N = recent record count (capped at 100,000)

```
ALGORITHM: compute_accuracy_trend(records: list[PredictionRecord]) -> str

IF len(records) < 10: return "stable"

errors = [abs(r.predicted - r.actual) / max(r.actual, 1) for r in records]
midpoint = len(errors) // 2
first_half_mean = mean(errors[:midpoint])
second_half_mean = mean(errors[midpoint:])

improvement_threshold = 0.10

IF second_half_mean < first_half_mean × (1 - improvement_threshold):
    return "improving"
ELIF second_half_mean > first_half_mean × (1 + improvement_threshold):
    return "degrading"
ELSE:
    return "stable"
```

---

## A9 — Import Rewrite

**Module:** `migration engine (tools)`  
**Complexity:** O(F × L) where F = files, L = lines per file

```
ALGORITHM: rewrite_imports(file_path, rewrite_rules: list[ImportRewrite]) -> bool

changed = False
lines = read_file(file_path)
FOR i, line in enumerate(lines):
    FOR rule in rewrite_rules:
        IF re.match(rule.old_pattern, line):
            lines[i] = re.sub(rule.old_pattern, rule.new_pattern, line)
            changed = True
IF changed:
    write_file(file_path, lines)
return changed

SAFETY: Only operates on files in scope. Skips generated files.
        Always operates on a git-clean working tree with snapshot taken.
```

---

## A10 — Manifest Checksum

**Module:** `platform.tools.verify`  
**Complexity:** O(N log N) where N = total IDs

```
ALGORITHM: compute_checksum(module_ids, capability_ids, service_ids) -> str

1. all_ids = sorted(module_ids + capability_ids + service_ids)
2. content = "\n".join(all_ids).encode("utf-8")
3. return "sha256:" + sha256(content).hexdigest()

PROPERTY: Deterministic — same input always produces same output
DETECTS: module add/remove, capability add/remove, renaming
```
