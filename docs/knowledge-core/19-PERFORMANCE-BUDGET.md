---
knowledge_id: KA-SPEC-019
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0.5
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-core
type: specification
implements:
  - KA-SPEC-015
depends_on:
  - id: KA-SPEC-018
    reason: "Performance budget is constrained by complexity analysis"
---

# Performance Budget

## Performance Targets for All Knowledge System Operations

---

## 1. Purpose

The Performance Budget translates complexity analysis into concrete, measurable targets that the implementation must meet. These targets are:

- **Testable** — each target can be verified with a benchmark
- **Realistic** — derived from complexity analysis at projected scale
- **Meaningful** — tied to user experience (a build that takes 5 minutes is not useful)
- **Phased** — targets tighten as the platform matures

---

## 2. Reference Hardware

All performance targets are specified for:

```
CPU:    4-core, 3GHz (equivalent to a mid-range developer workstation)
RAM:    8GB available to the process
Disk:   SSD, sequential read ~500MB/s
Scale:  D=5,000 documents, E=25,000 edges (Phase 3.0 scale)
```

Performance at Phase 2.0D.1 scale (D=786) will be proportionally faster.

---

## 3. Tool Performance Targets

### 3.1 Index Build (Full Rebuild)

| Metric | Phase 2.0D.1 | Phase 2.0E | Phase 3.0 | Critical |
|--------|-------------|-----------|-----------|---------|
| Full build time | < 5s | < 10s | < 30s | < 60s |
| Memory peak | < 200MB | < 500MB | < 1GB | < 2GB |
| Output file write | < 1s | < 2s | < 5s | < 10s |

**Rationale:** Developers run this on pre-commit hooks. Builds over 10s break the commit flow. Builds over 60s will be bypassed.

### 3.2 Incremental Update

| Metric | All Phases | Critical |
|--------|-----------|---------|
| Per-file incremental update | < 200ms | < 500ms |
| Health recomputation (affected docs) | < 100ms | < 300ms |
| Index freshness after change | < 500ms | < 1s |

**Rationale:** Incremental updates run during editing. They must feel instant.

### 3.3 Health Report Generation

| Metric | Phase 2.0D.1 | Phase 3.0 | Critical |
|--------|-------------|-----------|---------|
| Full health report | < 10s | < 30s | < 60s |
| Repository health score | < 1s | < 5s | < 10s |
| Per-capability health | < 100ms | < 500ms | < 1s |

### 3.4 Audit Tool

| Metric | Phase 2.0D.1 | Phase 3.0 | Critical |
|--------|-------------|-----------|---------|
| Full audit run (all rules) | < 15s | < 45s | < 90s |
| Per-document validation | < 5ms | < 10ms | < 50ms |
| Broken link detection | < 10s | < 30s | < 60s |

### 3.5 Compiler

| Target | Phase 2.0D.1 | Phase 3.0 | Critical |
|--------|-------------|-----------|---------|
| PromptPack (single capability) | < 1s | < 2s | < 5s |
| PromptPack (full repository) | < 30s | < 90s | < 180s |
| HTML (full repository) | < 60s | < 180s | < 300s |
| JSONL dataset (full repository) | < 60s | < 180s | < 300s |
| LLMMemory (single capability) | < 500ms | < 1s | < 2s |

---

## 4. Query Performance Targets

### 4.1 KQL Query Targets

| Query Type | Target Latency | Critical |
|-----------|---------------|---------|
| Registry lookup (`WHERE id = X`) | < 1ms | < 5ms |
| Type index query (`WHERE type = T`) | < 5ms | < 20ms |
| Capability index query | < 5ms | < 20ms |
| Health score lookup | < 1ms | < 5ms |
| Stale document list (top 50) | < 10ms | < 50ms |
| Graph traversal, depth 3 | < 50ms | < 200ms |
| Graph traversal, depth 6 | < 200ms | < 1000ms |
| Full graph traversal (unlimited) | < 500ms | < 2000ms |
| Duplicate detection | < 200ms | < 1000ms |
| Orphan detection | < 500ms | < 2000ms |
| Full text search (without inverted index) | < 2000ms | < 5000ms |
| Full text search (with inverted index) | < 50ms | < 200ms |

### 4.2 Compound Query Targets

| Query | Target | Description |
|-------|--------|-------------|
| Impact analysis (`TRAVERSE FROM X, depth=unlimited`) | < 500ms | Used for change impact |
| Full capability traceability | < 200ms | Requirement → evidence |
| Coverage gap report | < 1000ms | All capabilities |
| Repository health dashboard | < 500ms | Aggregate scores |

---

## 5. Memory Budgets

### 5.1 In-Memory Index Sizes

| Structure | Phase 2.0D.1 | Phase 3.0 |
|-----------|-------------|-----------|
| KnowledgeRegistry | ~5MB | ~30MB |
| KnowledgeGraph (adjacency) | ~2MB | ~15MB |
| CapabilityIndex | ~1MB | ~5MB |
| HealthIndex | ~2MB | ~10MB |
| All indexes combined | ~15MB | ~80MB |
| Full document body store | ~50MB | ~300MB |

**Total at Phase 3.0:** ~380MB — within the 8GB budget.

### 5.2 PromptPack Memory Budgets

| Budget Level | Target Token Count | Approximate Size |
|-------------|------------------|-----------------|
| `small` | 4,096 tokens | ~16KB |
| `medium` | 16,384 tokens | ~64KB |
| `large` | 65,536 tokens | ~256KB |
| `full` | Unlimited | Full documents |

Token counting uses the tiktoken estimate: 1 token ≈ 4 bytes for English text.

---

## 6. Performance Measurement Protocol

### 6.1 Benchmark Suite

A benchmark suite is maintained at `tools/benchmarks/`:

```
tools/benchmarks/
├── bench_index_build.py       # Full build at various scales
├── bench_incremental.py       # Incremental update latency
├── bench_health.py            # Health computation
├── bench_audit.py             # Audit rule evaluation
├── bench_kql.py               # Query latency by type
├── bench_compiler.py          # Compiler target latency
└── fixtures/
    ├── small/  (100 docs)
    ├── medium/ (1000 docs)
    └── large/  (5000 docs synthetic)
```

### 6.2 Benchmark Execution

```bash
# Run all benchmarks
python tools/benchmarks/run_all.py

# Run with specific scale
python tools/benchmarks/bench_index_build.py --scale large

# Generate performance report
python tools/benchmarks/report.py --output PERFORMANCE-REPORT.md
```

### 6.3 CI Integration

Performance benchmarks run weekly in CI (not on every commit — too slow):

```yaml
# .github/workflows/performance.yml
name: Performance Benchmarks
on:
  schedule:
    - cron: '0 0 * * 0'  # Weekly on Sunday
  workflow_dispatch:       # Manual trigger
jobs:
  benchmark:
    steps:
      - run: python tools/benchmarks/run_all.py --ci --budget medium
      - run: python tools/benchmarks/report.py --compare baseline
      - if: regression detected
        run: notify-team --message "Performance regression detected"
```

---

## 7. Performance Budget Violations

| Severity | Condition | Action |
|---------|-----------|--------|
| **Warning** | Actual < 150% of target | Investigate; note in release |
| **Error** | Actual < 200% of target | Must fix before Phase gate |
| **Critical** | Actual < 300% of target | Blocks merge to main |

---

## 8. Optimization Priorities

When performance targets are missed, optimize in this priority order:

1. **Algorithm selection** — O(n²) → O(n log n) or O(n) (highest ROI)
2. **Caching** — Memoize repeated computations (index lookups, health scores)
3. **Parallelism** — Document parsing and health computation are embarrassingly parallel
4. **Lazy evaluation** — Don't compute what isn't queried
5. **Index specialization** — Add targeted indexes for common slow queries
6. **Infrastructure** — Only after algorithmic optimizations are exhausted

---

## References

- [18-COMPLEXITY-GUIDE.md](18-COMPLEXITY-GUIDE.md) — Complexity analysis that underpins these targets
- [17-ALGORITHMS-CATALOG.md](17-ALGORITHMS-CATALOG.md) — Algorithms being measured
- [10-KNOWLEDGE-INDEX-ENGINE.md](10-KNOWLEDGE-INDEX-ENGINE.md) — Primary performance-critical component
- [20-KNOWLEDGE-RUNTIME-ARCHITECTURE.md](20-KNOWLEDGE-RUNTIME-ARCHITECTURE.md) — Runtime that must meet these budgets
