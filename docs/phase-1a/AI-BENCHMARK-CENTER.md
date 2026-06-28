# AI Benchmark Center — Performance Scoring and Model Recommendations

**Version:** 1.0.0  
**Status:** Approved for Implementation  
**Phase:** 1A  
**Date:** 2026-06-29  
**Classification:** Internal Architecture

---

## 1. Purpose

The AI Benchmark Center tracks and scores the real-world performance of every AI provider and model as experienced by AI Studio applications. Unlike industry benchmarks measured on academic datasets, the Benchmark Center measures performance on actual AI Studio tasks — code generation for AI Software Factory, content creation for Content Factory, game balance for Mythic Realms — making its recommendations actionable and domain-specific.

---

## 2. Responsibilities

- Collect performance data from every AI execution (latency, token usage, success/failure)
- Score model quality by task category using both automated and human-validated signals
- Maintain historical performance data (trend analysis, regression detection)
- Generate model recommendations for the Routing Engine
- Run synthetic benchmark jobs on a schedule (controlled test prompts with known good answers)
- Detect provider/model performance regressions
- Provide cost-efficiency rankings (quality per dollar)
- Write benchmark scores back to the Model Catalog

---

## 3. Architecture

```
Execution Engine
      │ (after every execution, async)
      │ record_execution_outcome(outcome)
      ▼
┌──────────────────────────────────────────────────────────────────────┐
│                       Benchmark Center                                 │
│                                                                        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  Outcome         │  │  Benchmark       │  │  Synthetic       │   │
│  │  Collector       │  │  Scorer          │  │  Job Runner      │   │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘   │
│           │                     │                      │              │
│  ┌────────▼─────────────────────▼──────────────────────▼───────────┐  │
│  │                     Benchmark Core                                 │  │
│  │                                                                    │  │
│  │  record_outcome(outcome: ExecutionOutcome) → void                 │  │
│  │  score_model(model_id, task_category) → BenchmarkScore           │  │
│  │  recommend(task_profile) → List[ModelRecommendation]             │  │
│  │  run_synthetic_suite(suite_id) → BenchmarkRun                    │  │
│  │  detect_regressions() → List[RegressionAlert]                    │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                        │
│  ┌──────────────────┐  ┌──────────────────────────────────────────┐  │
│  │  Regression      │  │  Recommendation Engine                    │  │
│  │  Detector        │  │                                           │  │
│  └──────────────────┘  └──────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
         │
         ▼
  Model Catalog (benchmark_scores updated)
  Routing Engine (recommendations consumed)
```

---

## 4. Task Categories

Every AI execution is tagged with a task category that the Benchmark Center uses for domain-specific scoring:

| Category ID | Description | Products |
|-------------|-------------|---------|
| `code.generation` | Generate new code files or functions | AI Software Factory |
| `code.review` | Review code for bugs, security, style | AI Software Factory |
| `code.test_generation` | Generate test cases | AI Software Factory |
| `code.architecture` | Architectural design and documentation | AI Software Factory |
| `code.debugging` | Identify and fix bugs | AI Software Factory |
| `content.episode_script` | Generate video/audio episode scripts | Content Factory |
| `content.article` | Generate long-form articles | Content Factory |
| `content.social_media` | Social posts, short content | Content Factory |
| `content.translation` | Translate content to another language | Content Factory |
| `content.seo_optimization` | SEO metadata and optimization | Content Factory |
| `game.balance_analysis` | Game economy and balance analysis | Mythic Realms |
| `game.card_generation` | Generate game card descriptions | Mythic Realms |
| `game.lore_writing` | In-universe story and lore | Mythic Realms |
| `game.player_behavior` | Player behavior analysis | Mythic Realms |
| `erp.process_documentation` | Document business processes | ERP Generator |
| `erp.data_mapping` | Map between data schemas | ERP Generator |
| `voice.script_coaching` | Pronunciation and delivery coaching | AI Vocal Coach |
| `voice.transcription` | Audio-to-text transcription | AI Vocal Coach |
| `general.reasoning` | Multi-step reasoning tasks | All |
| `general.summarization` | Summarize long documents | All |
| `general.classification` | Classify text into categories | All |
| `general.extraction` | Extract structured data from text | All |
| `general.embedding` | Generate vector representations | All |

---

## 5. Data Models

### 5.1 ExecutionOutcome

Collected asynchronously after every execution.

| Field | Type | Description |
|-------|------|-------------|
| `outcome_id` | `UUID` | Primary key |
| `request_id` | `UUID` | AI ROS request ID |
| `org_id` | `UUID` | |
| `provider_id` | `str` | |
| `canonical_model_id` | `str` | |
| `task_category` | `str` | From task category taxonomy above |
| `latency_ms` | `int` | Time to complete response |
| `ttft_ms` | `int \| None` | Time to first token (streaming) |
| `input_tokens` | `int` | |
| `output_tokens` | `int` | |
| `total_cost_usd` | `Decimal` | |
| `stop_reason` | `str` | `END_TURN` / `MAX_TOKENS` / `ERROR` |
| `error_code` | `str \| None` | Set if request failed |
| `retry_count` | `int` | |
| `is_streaming` | `bool` | |
| `automated_quality_score` | `float \| None` | 0.0–1.0 from automated evaluation |
| `human_quality_score` | `float \| None` | 0.0–1.0 from human review |
| `recorded_at` | `datetime` | |

### 5.2 BenchmarkScore

Aggregated score for a (model, task_category) pair.

| Field | Type | Description |
|-------|------|-------------|
| `score_id` | `UUID` | |
| `canonical_model_id` | `str` | |
| `task_category` | `str` | |
| `sample_size` | `int` | Number of executions in this score |
| `success_rate` | `float` | % of executions with stop_reason=END_TURN |
| `avg_quality_score` | `float \| None` | Average quality (if scored) |
| `p50_latency_ms` | `int` | |
| `p95_latency_ms` | `int` | |
| `p99_latency_ms` | `int` | |
| `avg_input_tokens` | `float` | |
| `avg_output_tokens` | `float` | |
| `avg_cost_usd` | `Decimal` | |
| `cost_per_successful_task_usd` | `Decimal \| None` | |
| `composite_score` | `float` | 0.0–1.0 weighted combination |
| `computed_at` | `datetime` | |
| `window_start` | `datetime` | Data window start |
| `window_end` | `datetime` | Data window end |

### 5.3 Composite Score Formula

```
composite_score = (
  quality_weight  * normalized(avg_quality_score)   +
  latency_weight  * normalized(1 / p95_latency_ms)  +  # lower latency = better
  cost_weight     * normalized(1 / avg_cost_usd)    +  # lower cost = better
  success_weight  * success_rate
)

Default weights (configurable per organization):
  quality_weight:  0.40
  latency_weight:  0.25
  cost_weight:     0.20
  success_weight:  0.15

Normalization: min-max across all models for the same task_category.
```

---

## 6. Automated Quality Scoring

For task categories with verifiable outputs, the Benchmark Center runs automated quality evaluation:

| Task Category | Quality Signal |
|--------------|----------------|
| `code.*` | Syntax validity, test execution pass rate, static analysis score |
| `content.*` | Readability score, keyword coverage, structural completeness |
| `general.extraction` | Schema compliance, field coverage vs. ground truth |
| `general.classification` | Accuracy against labeled test set |
| `voice.transcription` | Word error rate against reference transcript |

For task categories without automatic ground truth:
- Sample-based human review (1% of production executions)
- Comparison scoring (present two responses, human picks better)
- Results fed back to `human_quality_score` field

---

## 7. Synthetic Benchmark Suites

Scheduled benchmark runs using controlled prompts with known good responses.

### 7.1 BenchmarkSuite

| Field | Type | Description |
|-------|------|-------------|
| `suite_id` | `UUID` | |
| `name` | `str` | e.g., "AISF Code Generation Suite v1" |
| `task_category` | `str` | |
| `prompts` | `List[BenchmarkPrompt]` | Controlled test prompts |
| `evaluation_criteria` | `EvaluationCriteria` | How to score responses |
| `schedule` | `str \| None` | Cron expression for auto-run |
| `target_models` | `List[str]` | Canonical model IDs to benchmark |

### 7.2 BenchmarkPrompt

| Field | Type | Description |
|-------|------|-------------|
| `prompt_id` | `UUID` | |
| `input_messages` | `List[Message]` | Fixed test input |
| `expected_output_patterns` | `List[str]` | Must-contain patterns |
| `forbidden_output_patterns` | `List[str]` | Must-not-contain patterns |
| `expected_quality_score` | `float` | Minimum acceptable score |
| `max_acceptable_latency_ms` | `int` | |
| `max_acceptable_cost_usd` | `Decimal` | |

### 7.3 Benchmark Run Results

| Field | Type | Description |
|-------|------|-------------|
| `run_id` | `UUID` | |
| `suite_id` | `UUID` | |
| `started_at` | `datetime` | |
| `completed_at` | `datetime` | |
| `model_results` | `Dict[str, ModelBenchmarkResult]` | Per-model results |
| `regressions_detected` | `List[RegressionAlert]` | |
| `recommendations_updated` | `bool` | Whether routing recommendations were updated |

---

## 8. Regression Detection

The Regression Detector compares current benchmark scores against the rolling baseline (30-day average) and alerts when scores degrade significantly.

### 8.1 RegressionAlert

| Field | Type | Description |
|-------|------|-------------|
| `alert_id` | `UUID` | |
| `canonical_model_id` | `str` | |
| `task_category` | `str` | |
| `metric` | `str` | `latency_p95` / `success_rate` / `quality_score` / `cost` |
| `baseline_value` | `float` | 30-day average |
| `current_value` | `float` | Current measurement |
| `degradation_pct` | `float` | % worse than baseline |
| `severity` | `str` | `WARNING` (>10%) / `CRITICAL` (>25%) |
| `detected_at` | `datetime` | |

### 8.2 Regression Thresholds

| Metric | WARNING | CRITICAL |
|--------|---------|----------|
| p95 latency | +10% | +25% |
| Success rate | -5pp | -15pp |
| Quality score | -0.05 | -0.15 |
| Cost per token | +15% | +30% |

On CRITICAL regression: Recommendation Engine automatically de-ranks the affected model. On recovery (metric returns within 5% of baseline): ranking restored.

---

## 9. Recommendation Engine

`recommend(task_profile) → List[ModelRecommendation]`

The Routing Engine calls this before each request (or uses cached recommendations with TTL 5 minutes).

### 9.1 TaskProfile

| Field | Type | Description |
|-------|------|-------------|
| `task_category` | `str` | |
| `quality_priority` | `float` | 0.0–1.0 weight |
| `speed_priority` | `float` | 0.0–1.0 weight |
| `cost_priority` | `float` | 0.0–1.0 weight |
| `required_capabilities` | `List[str]` | |
| `max_cost_per_request_usd` | `Decimal \| None` | |
| `max_latency_ms` | `int \| None` | |
| `excluded_models` | `List[str]` | |

### 9.2 ModelRecommendation

| Field | Type | Description |
|-------|------|-------------|
| `canonical_model_id` | `str` | |
| `recommendation_score` | `float` | 0.0–1.0 |
| `rank` | `int` | 1 = best |
| `strengths` | `List[str]` | Why this model is recommended |
| `concerns` | `List[str]` | Any known issues or limitations |
| `benchmark_score` | `BenchmarkScore` | Underlying data |
| `estimated_cost_usd` | `Decimal` | For a typical request in this category |
| `estimated_latency_ms` | `int` | |

---

## 10. Events Published

| Event | Trigger |
|-------|---------|
| `benchmark.outcome_recorded` | Execution outcome collected |
| `benchmark.score_updated` | Benchmark score recomputed |
| `benchmark.run.started` | Synthetic benchmark run starting |
| `benchmark.run.completed` | Synthetic run complete |
| `benchmark.regression.detected` | Regression alert created |
| `benchmark.regression.cleared` | Provider recovered from regression |
| `benchmark.recommendation.updated` | Routing recommendation changed |

---

## 11. Security

- Benchmark data (outcomes, scores) is internal; not exposed to products
- Synthetic prompts may contain sensitive evaluation content; stored encrypted
- Human review samples are anonymized (org_id masked) before human reviewer sees them

---

## 12. Scalability

- Outcome collection is async and non-blocking (does not delay execution response)
- Score computation runs on a background worker every 15 minutes
- Recommendations cached for 5 minutes; invalidated on regression detection
- Benchmark database is separate from operational database (read-heavy workload; can be on read replica)

---

## 13. Future Evolution

| Feature | Notes |
|---------|-------|
| Cross-customer aggregated benchmarks (anonymized) | Improve score quality with larger sample |
| Task-specific fine-tuned evaluator models | More accurate automated quality scoring |
| A/B benchmark routing | Route % of traffic to new model for live comparison |
| Automatic model recommendation updates in routing config | Zero-touch routing optimization |

---

## 14. Cross References

| Document | Relationship |
|----------|-------------|
| AI-RESOURCE-OS-BLUEPRINT.md | Parent system |
| AI-MODEL-CATALOG.md | Benchmark scores written back here |
| AI-SCHEDULER.md | Routing decisions consume recommendations |
| AI-RESOURCE-EVENTS.md | Event schemas |
| AI-RESOURCE-DATABASE.md | Table schemas |
| AI-RESOURCE-DESKTOP.md | Benchmark dashboard |
