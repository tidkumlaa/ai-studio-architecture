# KNW-KIL-DOC-025 — Knowledge Intelligence Dashboard

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Intelligence Dashboard provides an operational view of the Knowledge
Intelligence Layer across the entire knowledge base — showing which objects
are AI-ready, which have gaps, and where the intelligence quality needs work.

---

## Dashboard Sections

### Section 1: KIL Health Banner

```
┌─────────────────────────────────────────────────────────────────┐
│  KNOWLEDGE INTELLIGENCE LAYER — v1.0.0                          │
│  Objects with KIL: 233 / 233   AI-Ready (≥ 0.80): 198 / 233   │
│  Average AI-Readiness: 0.847   Phase: 3.0D.0.6   FROZEN        │
└─────────────────────────────────────────────────────────────────┘
```

---

### Section 2: Block Completion Matrix

```
Intelligence Block Completion (per object)
═══════════════════════════════════════════════════════════

Block                  Complete   Partial   Missing
───────────────────────────────────────────────────────────
self_describing        228        5         0        ✓
executable             225        8         0        ✓
semantic               231        2         0        ✓
ai_context             233        0         0        ✓
compression            233        0         0        ✓
reasoning              218       12         3        !
genome                 233        0         0        ✓
dna                    233        0         0        ✓
memory                 233        0         0        ✓  (runtime-populated)
usage                  201       32         0        !
evolution              220       13         0        !
confidence             230        3         0        ✓
quality                233        0         0        ✓
risk                   215       18         0        !
tradeoffs              189       44         0        !
decisions              201       32         0        !
alternatives           176       57         0        !
cortex                 221       12         0        ✓
thinking               198       35         0        !
explanation            228        5         0        ✓
learning               215       18         0        !
optimization           233        0         0        ✓
search                 233        0         0        ✓
═══════════════════════════════════════════════════════════
```

---

### Section 3: AI Readiness Distribution

```
AI Readiness Score Distribution (233 objects)

Score       Count  Bar
─────────────────────────────────────────
0.0–0.5     8      ████████
0.5–0.6     12     ████████████
0.6–0.7     15     ███████████████
0.7–0.8     35     ███████████████████████████████████
0.8–0.9     98     [long bar]
0.9–1.0     65     [medium bar]

Mean: 0.847  Median: 0.863  P10: 0.641  P90: 0.941
```

---

### Section 4: Objects Needing Attention

```
AI Readiness < 0.70 (needs improvement)
════════════════════════════════════════
Knowledge ID          Name                   Score  Top Gap
──────────────────────────────────────────────────────────────
KNW-FIN-SVC-003       Cost Calculator        0.52   reasoning (0)
KNW-PROV-MOD-012      Legacy Provider        0.58   dna (unsealed)
KNW-TEST-BENCH-008    Quota Bench v1         0.61   tradeoffs (0)
KNW-META-POL-002      Retention Policy       0.65   alternatives (0)
...
(Total: 35 objects below 0.70)
```

---

### Section 5: Block Completion Gaps by Priority

```
Priority Gaps (most impactful to fix)
═══════════════════════════════════════
1. tradeoffs     — 44 partial, 57 missing alternatives entries
2. decisions     — 32 objects missing decision records
3. thinking      — 35 modules/services need reasoning protocol
4. risk          — 18 objects with incomplete risk model
5. learning      — 18 objects missing prerequisite mapping
```

---

### Section 6: Intelligence Quality by Package

```
Package                  Objects  AI-Ready  Avg Score  Status
──────────────────────────────────────────────────────────────
kos.platform.package     35       34         0.891      ✓
kos.algorithm.package    58       56         0.924      ✓
kos.runtime.package      28       26         0.871      ✓
kos.meta.package         42       41         0.935      ✓
kos.provider.package     15       12         0.743      !
kos.api.package          12       11         0.812      ✓
kos.test.package         21       20         0.856      ✓
kos.financial.package    8        5          0.634      ✗
kos.pattern.package      14       13         0.897      ✓
TOTAL                    233      218        0.847
```

---

### Section 7: DNA Seal Status

```
DNA Status
═══════════════════════════════════════════════
Total objects:      233
DNA sealed:         233   ✓ (all CANONICAL objects sealed)
DNA hash valid:     233   ✓
DNA conflicts:      0     ✓
DNA duplicates:     0     ✓ (no objects share fingerprint)
```

---

### Section 8: Intelligence Trends (Last 30 Days)

```
AI Readiness Trend
═══════════════════════════════════════
30 days ago: avg = 0.793
15 days ago: avg = 0.821
Today:       avg = 0.847  (+0.054)

New objects (last 30d):  12
Improved objects:        45
Declined objects:         2  ← need attention
```

---

## Dashboard Output Formats

| Format | File | Contents |
|--------|------|---------|
| HTML | `reports/kil-dashboard.html` | Interactive with charts |
| Markdown | `reports/kil-dashboard.md` | ASCII art summary |
| JSON | `reports/kil-dashboard.json` | Machine-readable all metrics |

---

## CLI

```bash
kos dashboard --kil                  # show intelligence layer dashboard
kos dashboard --kil --format json    # machine-readable
kos dashboard --kil --package plt    # filter to package
kos dashboard --kil --below 0.70     # show objects below threshold
```

---

## Cross-References

- AI readiness score definition → `27-KNOWLEDGE-AI-READINESS`
- KIL metrics → `26-KNOWLEDGE-METRICS`
- Quality dashboard → Phase 3.0D.0.5 `19-KNOWLEDGE-DASHBOARD`
- Certification dashboard → Phase 3.0D.0 `28-DASHBOARD`
