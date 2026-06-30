# KNW-FINAL-003 — Golden Dataset

**Phase:** 3.0D.0.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Defines the complete specification for the KOS v1.0 golden dataset — the authoritative data corpus used for all certification, regression, and AI evaluation.

---

## Dataset Composition

| Component | Count | Description |
|-----------|-------|-------------|
| Knowledge Objects | 100,000 | Across all 9 packages and 33 types |
| Relationships | 1,000,000 | All 10 relationship types |
| Search Queries | 10,000 | With ground-truth answer sets |
| Reasoning Questions | 5,000 | With ground-truth answers |
| Dependency Cases | 2,000 | Package resolution scenarios |
| Traceability Chains | 2,000 | Complete 9-level chains |
| Impact Analyses | 1,000 | "What breaks if X is removed?" |
| Hallucination Cases | 500 | Questions with no correct answer in KB |

---

## Object Distribution

### By Package

| Package | Objects | % |
|---------|---------|---|
| Platform (PLT) | 20,000 | 20% |
| Runtime (RT) | 18,000 | 18% |
| Algorithm (ALG) | 15,000 | 15% |
| Provider (PROV) | 12,000 | 12% |
| Pattern (PAT) | 10,000 | 10% |
| API (API) | 8,000 | 8% |
| Test (TEST) | 8,000 | 8% |
| Financial (FIN) | 5,000 | 5% |
| Meta (META) | 4,000 | 4% |
| **Total** | **100,000** | **100%** |

### By Lifecycle State

| State | Count | % |
|-------|-------|---|
| CANONICAL | 60,000 | 60% |
| VERIFIED | 15,000 | 15% |
| PROPOSED | 10,000 | 10% |
| DRAFT | 8,000 | 8% |
| DEPRECATED | 5,000 | 5% |
| ARCHIVED | 2,000 | 2% |

### By Object Type (top 10)

| Type | Count |
|------|-------|
| MODULE | 12,000 |
| REQUIREMENT | 10,000 |
| TEST | 9,000 |
| SERVICE | 8,000 |
| ALGORITHM | 8,000 |
| PATTERN | 7,000 |
| DECISION | 6,000 |
| CONFIGURATION | 5,000 |
| API | 5,000 |
| BENCHMARK | 4,000 |
| Other 23 types | 26,000 |

---

## Relationship Distribution

| Type | Count | % |
|------|-------|---|
| DEPENDS_ON | 250,000 | 25% |
| IMPLEMENTS | 180,000 | 18% |
| TESTS | 150,000 | 15% |
| USES | 140,000 | 14% |
| PROVIDES | 100,000 | 10% |
| REFERENCES | 80,000 | 8% |
| EXTENDS | 50,000 | 5% |
| RELATED_TO | 30,000 | 3% |
| SUPERSEDES | 20,000 | 2% |
| OWNED_BY | 0 | — |
| **Total** | **1,000,000** | **100%** |

---

## Search Query Dataset

10,000 queries with pre-computed ground-truth top-10 results:

| Category | Count | Examples |
|----------|-------|---------|
| Identity lookup | 2,000 | "Quota Manager", "BM25 ranker" |
| Concept search | 2,000 | "rate limiting", "embedding" |
| Namespace search | 1,500 | "plt:module", "alg:algorithm" |
| Tag search | 1,500 | tag:quota, tag:ml |
| Relationship search | 1,000 | "implements routing" |
| Typo variants | 1,000 | "quota managar", "algorithim" |
| Alias search | 500 | canonical_name lookup |
| Unicode / multi-lang | 500 | Thai, CJK, Arabic terms |

---

## Reasoning Question Dataset

5,000 questions with ground-truth structured answers:

| Category | Count |
|----------|-------|
| Identity | 500 |
| Relationship | 750 |
| Dependency | 750 |
| Architecture | 500 |
| Implementation | 500 |
| API | 500 |
| Graph | 500 |
| Capability | 250 |
| Lifecycle | 250 |
| Evidence | 250 |
| Impact | 250 |

---

## Hallucination Cases

500 questions designed to elicit hallucination:

| Type | Count | Strategy |
|------|-------|---------|
| Non-existent entity | 150 | Ask about KNW-PLT-MOD-999 (does not exist) |
| Plausible but wrong relationship | 100 | A DEPENDS_ON B, but B DEPENDS_ON A in reality |
| Near-miss name | 100 | "Quota Engine" (correct: "Quota Manager") |
| Outdated information | 100 | Ask about deprecated/replaced object |
| Fabricated evidence | 50 | Ask for specific benchmark that does not exist |

---

## Dataset Generation Rules

| Rule | Description |
|------|-------------|
| GD-001 | All objects generated with seed 42 for reproducibility |
| GD-002 | Object IDs follow KNW ID format exactly |
| GD-003 | Relationships are bidirectional in storage |
| GD-004 | Quality scores are computed, not random |
| GD-005 | Every CANONICAL object has ≥ 1 evidence item |
| GD-006 | Ground-truth answers stored alongside each question |
| GD-007 | Dataset manifest checksums all files before certification runs |
| GD-008 | Hallucination cases are explicitly tagged to prevent false positives |

---

## Dataset Files

```
knowledge/datasets/
├── manifest.yaml               # checksums, counts, seed, version
├── golden/
│   ├── objects/                # 100K YAML files
│   ├── relationships.yaml      # 1M relationship entries
│   └── ground-truth/
│       ├── search-queries.yaml    # 10K queries + top10
│       ├── reasoning-qa.yaml      # 5K Q&A
│       ├── dependency-cases.yaml  # 2K dep scenarios
│       ├── traceability-chains.yaml # 2K chains
│       ├── impact-analyses.yaml   # 1K impact analyses
│       └── hallucination-cases.yaml # 500 cases
├── regression/                 # Pinned 50-object subset
├── performance/                # Synthetic 1K/10K/100K/1M
├── adversarial/                # 500 adversarial cases
└── traceability/               # 2K traceability dataset
```

---

## Cross-References

- Dataset certification → Phase 3.0D.0 `21-DATASET-CERTIFICATION`
- Golden dataset rules → Phase 3.0C.5 `12-KNOWLEDGE-GOLDEN-DATASET`
- Benchmark objects → Phase 3.0C.5 `13-KNOWLEDGE-BENCHMARKS`
