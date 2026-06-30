# KNW-CERT-ARCH-014 — Performance Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the KOS platform meets latency, throughput, and resource consumption targets across 4 dataset sizes. All benchmarks are reproducible with fixed seeds and declared environment.

---

## Dataset Sizes

| Size | Objects | Relationships | Index | Registry | Graph |
|------|---------|---------------|-------|----------|-------|
| 1K | 1,000 | 5,000 | 1K | 1K | 5K edges |
| 10K | 10,000 | 50,000 | 10K | 10K | 50K edges |
| 100K | 100,000 | 500,000 | 100K | 100K | 500K edges |
| 1M | 1,000,000 | 10,000,000 | 1M | 1M | 10M edges |

---

## Build Time Targets

| Operation | 1K | 10K | 100K | 1M |
|-----------|----|----|------|----|
| Index build | ≤ 1s | ≤ 10s | ≤ 60s | ≤ 600s |
| Registry build | ≤ 2s | ≤ 20s | ≤ 120s | ≤ 1200s |
| Graph build | ≤ 1s | ≤ 10s | ≤ 120s | ≤ 1200s |
| Catalog build | ≤ 1s | ≤ 10s | ≤ 60s | ≤ 600s |

---

## Operation Latency Targets (P50 / P95 / P99)

### Search

| Dataset | P50 | P95 | P99 |
|---------|-----|-----|-----|
| 1K | ≤ 5ms | ≤ 15ms | ≤ 30ms |
| 10K | ≤ 10ms | ≤ 30ms | ≤ 60ms |
| 100K | ≤ 20ms | ≤ 50ms | ≤ 100ms |
| 1M | ≤ 50ms | ≤ 150ms | ≤ 300ms |

### Registry Lookup

| Dataset | P50 | P95 | P99 |
|---------|-----|-----|-----|
| 1K | ≤ 0.5ms | ≤ 1ms | ≤ 2ms |
| 10K | ≤ 0.5ms | ≤ 1ms | ≤ 3ms |
| 100K | ≤ 1ms | ≤ 2ms | ≤ 5ms |
| 1M | ≤ 2ms | ≤ 5ms | ≤ 10ms |

### Graph Traversal (BFS from random node)

| Dataset | P50 | P95 | P99 |
|---------|-----|-----|-----|
| 1K (5K edges) | ≤ 2ms | ≤ 5ms | ≤ 10ms |
| 10K (50K edges) | ≤ 5ms | ≤ 15ms | ≤ 30ms |
| 100K (500K edges) | ≤ 20ms | ≤ 50ms | ≤ 100ms |
| 1M (10M edges) | ≤ 100ms | ≤ 300ms | ≤ 500ms |

### Compilation (if applicable)

| Dataset | P50 | P95 | P99 |
|---------|-----|-----|-----|
| 1K | ≤ 500ms | ≤ 1s | ≤ 2s |
| 10K | ≤ 5s | ≤ 10s | ≤ 20s |
| 100K | ≤ 60s | ≤ 120s | ≤ 300s |

---

## Resource Targets

### Memory

| Dataset | Index | Registry | Graph | Total |
|---------|-------|----------|-------|-------|
| 1K | ≤ 10 MB | ≤ 10 MB | ≤ 5 MB | ≤ 50 MB |
| 10K | ≤ 80 MB | ≤ 80 MB | ≤ 40 MB | ≤ 300 MB |
| 100K | ≤ 600 MB | ≤ 600 MB | ≤ 300 MB | ≤ 2 GB |
| 1M | ≤ 5 GB | ≤ 5 GB | ≤ 2.5 GB | ≤ 15 GB |

### CPU

| Operation | Max CPU% (single thread) |
|-----------|--------------------------|
| Search | ≤ 100% (1 core) |
| Registry lookup | ≤ 50% (1 core) |
| Graph traversal | ≤ 100% (1 core) |
| Build (parallel OK) | any |

---

## Benchmark Checks

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Index build within time target | PC-001 | MAJOR | At all 4 sizes |
| Registry build within time target | PC-002 | MAJOR | At all 4 sizes |
| Graph build within time target | PC-003 | MAJOR | At all 4 sizes |
| Search P99 within latency target | PC-004 | MAJOR | At 100K (Gold) |
| Registry lookup P99 within target | PC-005 | CRITICAL | At 100K (Gold) |
| Graph P99 within target | PC-006 | MAJOR | At 100K (Gold) |
| Memory within target at 1M | PC-007 | MAJOR (Enterprise) | |
| No memory leak (10K ops, flat RSS) | PC-008 | MAJOR | |

---

## Environment Capture

Every benchmark run captures and reports:

```json
{
  "environment": {
    "os": "Windows 11 26200",
    "python": "3.13.0",
    "cpu": "Intel Core i9-13900K",
    "memory_gb": 64,
    "disk": "NVMe SSD",
    "dataset_version": "sha256:abc123"
  }
}
```

---

## Report Format

```json
{
  "domain": "performance",
  "datasets": {
    "1K": {
      "index_build_s": 0.3,
      "search_p99_ms": 18.2,
      "registry_lookup_p99_ms": 1.4,
      "graph_bfs_p99_ms": 7.8,
      "memory_total_mb": 38
    },
    "100K": {
      "index_build_s": 42.1,
      "search_p99_ms": 89.3,
      "registry_lookup_p99_ms": 4.1,
      "graph_bfs_p99_ms": 87.6,
      "memory_total_mb": 1820
    }
  },
  "checks": {
    "PC-004": "PASS", "PC-005": "PASS", "PC-007": "FAIL"
  },
  "domain_score": 0.851,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert benchmark                       # all sizes
kos-cert benchmark --size 1K,10K         # specific sizes only
kos-cert benchmark --operation search    # search only
kos-cert benchmark --seed 42
kos-cert benchmark --output reports/performance.json
```

---

## Cross-References

- Phase 3.0C performance budget → Phase 3.0C `37-PERFORMANCE-BUDGET`
- Scalability → `15-SCALABILITY-CERTIFICATION`
- Stress testing → `16-STRESS-CERTIFICATION`
- Overall score weight (10%) → `27-OVERALL-SCORING`
