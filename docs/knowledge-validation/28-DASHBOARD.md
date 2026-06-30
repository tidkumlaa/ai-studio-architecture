# KVF-DOC-028 — Validation Dashboard

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Validation Dashboard defines the specification for the KOS validation
reporting interface — a unified view of all validation results, certification
status, and health trends. The dashboard spec defines what must be rendered;
it does not specify the rendering technology.

---

## Dashboard Sections

```
Section 1: CERTIFICATION STATUS
Section 2: KNOWLEDGE HEALTH
Section 3: CONTEXT QUALITY TRENDS
Section 4: SEARCH QUALITY
Section 5: AI CAPABILITY
Section 6: PERFORMANCE METRICS
Section 7: REGRESSION COMPARISON
Section 8: VALIDATION HISTORY
```

---

## Section 1: Certification Status

```
┌─────────────────────────────────────────────────────┐
│  KOS VALIDATION REPORT                               │
│  Run: {run_id}  |  {timestamp}  |  v{impl_version}  │
├─────────────────────────────────────────────────────┤
│  CERTIFICATION: ████ GOLD                           │
│  Score: 0.87 / 1.00                                 │
├─────────────────────────────────────────────────────┤
│  Domain Scores:                                     │
│  Conformance      0.98  ██████████  ✓               │
│  Context Quality  0.82  ████████    ✓               │
│  Coverage         0.81  ████████    ✓               │
│  Search+Reason    0.85  █████████   ✓               │
│  AI Capability    0.87  █████████   ✓               │
│  Performance      0.91  █████████   ✓               │
└─────────────────────────────────────────────────────┘

Fields:
  - Certification level (large, prominent)
  - Overall certification score
  - All 6 domain scores with bar charts
  - Pass/fail status per domain
  - Hard gate status (all gates: ✓ / ✗)
```

---

## Section 2: Knowledge Health

```
┌─────────────────────────────────────────────────────┐
│  KNOWLEDGE HEALTH                                   │
├─────────────────────────────────────────────────────┤
│  Objects:  Total: 10,234  Active: 8,012             │
│            Canonical: 6,891  Deprecated: 412        │
│                                                     │
│  Quality Distribution:                              │
│  EXCELLENT (≥0.90)  ████████  2,891  (42%)          │
│  GOOD      (≥0.80)  ██████    2,100  (30%)          │
│  ACCEPTABLE(≥0.70)  ████      1,400  (20%)          │
│  POOR      (<0.70)  █           500  ( 7%)          │
│                                                     │
│  AIRS Distribution:                                 │
│  AI-NATIVE  (≥0.90) ████████  1,200  (17%)          │
│  AI-READY   (≥0.80) ██████    2,500  (36%)          │
│  AI-CAPABLE (≥0.70) ████      1,800  (26%)          │
│  AI-PARTIAL (≥0.60) ██          900  (13%)          │
│  AI-MANUAL  (<0.60) █           600  ( 9%)          │
│                                                     │
│  Mean Quality Score:      0.83                      │
│  Mean AIRS:               0.79                      │
│  Conformance Rate:        0.989                     │
└─────────────────────────────────────────────────────┘
```

---

## Section 3: Context Quality Trends

```
┌─────────────────────────────────────────────────────┐
│  CONTEXT QUALITY  (last 30 days)                    │
├─────────────────────────────────────────────────────┤
│  Mean Quality Score:  0.82  (+0.03 vs last run)     │
│  Packs below 0.70:    1.2%  (-0.3% vs last run)     │
│                                                     │
│  Token Efficiency:    96.2%  reduction              │
│  Preservation Rate:   0.944                         │
│                                                     │
│  Dimension Scores:                                  │
│  Relevance    0.87  ████████▓                       │
│  Completeness 0.81  ████████                        │
│  Consistency  0.96  █████████▓                      │
│  Safety       0.99  ██████████                      │
│  Confidence   0.88  █████████                       │
│  Compression  0.84  ████████▓                       │
│  Ranking      0.82  ████████                        │
│  Coverage     0.76  ███████▓                        │
└─────────────────────────────────────────────────────┘
```

---

## Section 4: Search Quality

```
┌─────────────────────────────────────────────────────┐
│  SEARCH QUALITY  (Standard: 1,000 queries)          │
├─────────────────────────────────────────────────────┤
│  Top-1:      0.874  ████████▓   Target: ≥0.85 ✓    │
│  Top-5:      0.962  █████████▓  Target: ≥0.95 ✓    │
│  MRR:        0.891  ████████▓   Target: ≥0.85 ✓    │
│  NDCG@10:    0.831  ████████▓   Target: ≥0.80 ✓    │
│  P@10:       0.742  ███████▓    Target: ≥0.70 ✓    │
│                                                     │
│  Latency:    P50:12ms  P95:41ms  P99:87ms  ✓        │
│                                                     │
│  By Query Type:                                     │
│  IDENTITY:   0.951  ██████████                      │
│  KEYWORD:    0.882  █████████                       │
│  SEMANTIC:   0.841  ████████▓                       │
│  GRAPH:      0.786  ███████▓                        │
│  MULTI-TERM: 0.803  ████████                        │
└─────────────────────────────────────────────────────┘
```

---

## Section 5: AI Capability

```
┌─────────────────────────────────────────────────────┐
│  AI CAPABILITY  (500 tasks)                         │
├─────────────────────────────────────────────────────┤
│  Success Rate:     87.4%   Target: ≥85% ✓           │
│  Context Accuracy: 91.2%   Target: ≥90% ✓           │
│                                                     │
│  Hallucination:                                     │
│  Type A (wrong fact):       0.8%   Target: <2% ✓   │
│  Type B (wrong dep):        0.6%   Target: <2% ✓   │
│  Type C (wrong arch):       1.4%   Target: <5% ✓   │
│  Type D (wrong rel):        1.2%   Target: <5% ✓   │
│  Type E (wrong conf):       3.1%   Target: <8% ✓   │
│  Type F (unsupported):      5.2%   Target: <8% ✓   │
│                                                     │
│  Reasoning Depth (mean):    2.7                     │
│  Cross-Agent Consistency:   95.3%                   │
└─────────────────────────────────────────────────────┘
```

---

## Section 6: Performance Metrics

```
┌─────────────────────────────────────────────────────┐
│  PERFORMANCE  (100 rps, 1 hour)                     │
├─────────────────────────────────────────────────────┤
│  Assembly: P50:22ms  P95:64ms  P99:98ms  ✓          │
│  Cache Hit Rate:  88.4%                  ✓          │
│  Memory RSS:  1.24GB / 2GB max           ✓          │
│  Memory Leak: None detected              ✓          │
│                                                     │
│  Stage Breakdown (P99):                             │
│  S1-IntentParser:    3ms                            │
│  S2-ObjectSelector: 12ms                            │
│  S3-Ranker:          9ms                            │
│  S4-Budget:          1ms                            │
│  S5-Compression:     1ms                            │
│  S6-Planner:         2ms                            │
│  S7-Assembler:      35ms                            │
│  S8-AHGuard:         3ms                            │
│  S9-Validator:       4ms                            │
└─────────────────────────────────────────────────────┘
```

---

## Output Formats

The dashboard must be generated in all 4 formats:

```
MARKDOWN:  dashboard.md
  Plain text with ASCII art boxes
  Suitable for: GitHub, documentation portals, terminal

JSON:      dashboard.json
  Machine-readable: all metrics, all scores, all thresholds, pass/fail
  Suitable for: CI integration, automated monitoring, API consumers

CSV:       dashboard_metrics.csv
  Flat table: metric_id, value, threshold, passed, delta_vs_baseline
  Suitable for: spreadsheet analysis, time-series databases

HTML:      dashboard.html
  Rendered dashboard with interactive charts (specification only)
  Charts: bar charts for distributions, line charts for trends
  Color coding: green (pass), yellow (warning), red (fail)
```

---

## JSON Output Schema

```yaml
dashboard_json:
  run_id: string
  generated_at: string
  implementation_version: string
  kos_schema_version: string

  certification:
    level: string
    score: float
    valid_until: string

  metrics:
    - metric_id: string
      name: string
      value: float | string | integer
      threshold: float | null
      passed: boolean
      delta_vs_baseline: float | null
      category: string                   # domain 1–7
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-136 | All 4 output formats must be generated in every validation run |
| KVF-137 | JSON output must be machine-parseable — no narrative text in metric values |
| KVF-138 | Dashboard must include delta_vs_baseline for all metrics where a baseline exists |
| KVF-139 | Color coding in HTML must follow: green=pass, yellow=warning, red=fail — no other mapping |
| KVF-140 | Dashboard is the deliverable of a validation run — a run without dashboard output is incomplete |

---

## Cross-References

- Certification scoring → `27-CERTIFICATION-SCORING`
- All validation domain docs → docs 02–26
- Architecture freeze → `29-ARCHITECTURE-FREEZE`
