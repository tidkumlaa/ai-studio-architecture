# KNW-CERT-ARCH-028 — Certification Dashboard

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines the structure and content of the KOS Certification Dashboard — the primary visual interface for understanding certification status, trends, and gaps.

---

## Dashboard Sections

### Section 1: Overall Status

```
┌─────────────────────────────────────────────────────────┐
│  KOS CERTIFICATION DASHBOARD                            │
│  Version: 1.1.0 | Run: 2026-06-30 | Seed: 42          │
├─────────────────────────────────────────────────────────┤
│  OVERALL SCORE: 0.847          LEVEL: ████ GOLD        │
│  Previous:      0.831 (+0.016) TREND: ▲                │
└─────────────────────────────────────────────────────────┘
```

---

### Section 2: Domain Scores (gauge per domain)

```
Search        [████████░░]  0.831  ▲ +0.016
Registry      [█████████░]  0.921  ▲ +0.032
Graph         [█████████▌]  0.974  ▲ +0.011
Traceability  [████████░░]  0.847  ▼ -0.003
Reasoning     [████████░░]  0.841  ▲ +0.044
Performance   [████████░░]  0.851  ▲ +0.008
Scalability   [█████████░]  0.912  — 0.000
Reliability   [█████████░]  0.934  ▲ +0.021
Quality       [████████░░]  0.853  ▲ +0.013
Security      [█████████░]  0.933  — 0.000
Evolution     [██████░░░░]  0.623  ▲ +0.123
```

---

### Section 3: Knowledge Health Panel

```
┌─────────────────────────────────────────────────────────┐
│  KNOWLEDGE HEALTH                                        │
│  Health Score:        0.821   ▲                         │
│  Coverage Score:      0.913   ▲                         │
│  Registry Health:     HEALTHY (0 errors)                 │
│  Graph Health:        HEALTHY (0 cycles)                 │
│  Reasoning Accuracy:  84.7%                              │
│  Hallucination Rate:  3.6%    ▼ (lower is better)       │
│  Search Accuracy:     83.1%   ▲                          │
│  P99 Latency (100K): 87.3ms   ▼ (lower is better)       │
└─────────────────────────────────────────────────────────┘
```

---

### Section 4: Trend Chart

7-day rolling trend for each metric. Rendered as ASCII or HTML sparkline:

```
Overall Score (7-day):
0.80 |          ·········
0.78 |     ·····
0.76 | ·····
     └─────────────────→
       -6d  -5d  -4d  now
```

In HTML format: rendered as an interactive SVG line chart.

---

### Section 5: Level Progress

```
Bronze [████████████████] ACHIEVED
Silver [████████████████] ACHIEVED
Gold   [██████████████░░] ACHIEVED (0.847 / 0.80)
                          ↑ current level

Next: Enterprise needs 0.90
Gap:  Search +0.057  Reasoning +0.049  Evolution +0.277
```

---

### Section 6: Failed Checks

```
FAILED CHECKS (5)
Domain         Check   Severity  Issue
─────────────────────────────────────
traceability   TC-006  MAJOR     Chain completeness = 89% < 100% for CANONICAL
coverage       CC-001  MAJOR     Type coverage 84.8% < 90%
coverage       CC-004  MAJOR     Test coverage 87.5% < 90%
search         SC-007  MINOR     Wildcard queries not supported
query          QR-033  MINOR     Paginated query offset incorrect
```

---

### Section 7: Top 3 Improvements Needed

```
1. Evolution score: 0.623  →  Target 0.70  (Gap: 0.077)
   Action: Improve search accuracy or reasoning quality in this version

2. Traceability: 0.847  →  Target 0.90  (Gap: 0.053)
   Action: Complete trace chains for 55 requirements (see traceability-gaps.yaml)

3. Search Top1: 0.831  →  Target 0.88  (Enterprise)
   Action: Improve search ranking for alias queries
```

---

## Output Formats

| Format | File | When used |
|--------|------|-----------|
| HTML | `dashboard.html` | Interactive browser view |
| Markdown | `summary.md` | PR comments, documentation |
| JSON | `summary.json` | Machine consumption |
| CSV | `report.csv` | Spreadsheet analysis |
| Console | stdout | CI log output |

---

## HTML Dashboard Requirements

The HTML dashboard must:
- Load without external network requests (fully self-contained)
- Include interactive domain score gauges
- Include trend sparklines for all metrics
- Include sortable failed checks table
- Include level progress bars
- Render in all major browsers (Chrome, Firefox, Safari)
- Total file size ≤ 2 MB

---

## CLI

```bash
kos-cert dashboard                       # open HTML in default browser
kos-cert dashboard --format markdown     # print markdown summary
kos-cert dashboard --format json         # print JSON
kos-cert dashboard --trend 7d            # show 7-day trend
kos-cert dashboard --output reports/dashboard.html
```

---

## Cross-References

- Score formula → `27-OVERALL-SCORING`
- CI integration → `29-CI-INTEGRATION`
- Evolution trend → `24-EVOLUTION-CERTIFICATION`
