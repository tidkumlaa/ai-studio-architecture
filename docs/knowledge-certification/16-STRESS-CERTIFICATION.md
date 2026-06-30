# KNW-CERT-ARCH-016 — Stress Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the KOS platform remains correct and stable under sustained load, random failures, random deletes, and adversarial query patterns.

---

## Stress Test Scenarios

### S1: Continuous Queries (10 minutes)

Send queries at a sustained rate for 10 minutes with no pauses.

| Rate | Duration | Dataset |
|------|----------|---------|
| 100 QPS | 10 min | 10K objects |
| 500 QPS | 10 min | 100K objects |
| 1,000 QPS | 10 min (Enterprise) | 1M objects |

Pass criteria:
- Zero crashes
- Zero incorrect results
- P99 latency does not degrade more than 2× from start to end
- Memory does not grow unboundedly (RSS stable within 10%)

| Check | ID | Severity |
|-------|----|----------|
| Zero crashes during 100 QPS / 10 min | SS-001 | CRITICAL |
| P99 degradation ≤ 2× | SS-002 | MAJOR |
| RSS stable (≤ 10% growth) | SS-003 | MAJOR |
| Zero incorrect results during stress | SS-004 | CRITICAL |

---

### S2: Random Queries (adversarial)

Send 10,000 random queries including: malformed queries, empty queries, unicode, very long queries (>10K chars), SQL injection patterns, script injection.

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| No crash on malformed query | SS-005 | CRITICAL | Returns error, not crash |
| No crash on empty query | SS-006 | CRITICAL | Returns [] |
| No crash on 10K-char query | SS-007 | MAJOR | Truncates or returns error |
| SQL injection patterns rejected | SS-008 | MAJOR | No SQL evaluation attempted |
| Script injection patterns sanitised | SS-009 | MAJOR | |

---

### S3: Concurrent Reads + Writes

10 threads reading continuously + 5 threads writing continuously for 5 minutes.

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| No data corruption | SS-010 | CRITICAL | All reads return consistent data |
| No lost writes | SS-011 | CRITICAL | Every write eventually readable |
| No deadlock | SS-012 | CRITICAL | No thread blocked > 30s |
| Read latency ≤ 2× single-thread | SS-013 | MAJOR | |
| Write latency ≤ 3× single-thread | SS-014 | MAJOR | |

---

### S4: Random Failures

Inject random failures (simulated process restarts, I/O errors) during operation:

| Failure Type | Frequency | Pass Criteria |
|--------------|-----------|---------------|
| Process restart mid-operation | 10 × random | State fully recoverable |
| I/O error during write | 20 × random | Write rolled back; no partial state |
| Network timeout (simulated) | 30 × random | Operation retried or error returned |

| Check | ID | Severity |
|-------|----|----------|
| Recovery from process restart | SS-015 | CRITICAL |
| No partial writes after I/O error | SS-016 | CRITICAL |
| Retry on timeout within 30s | SS-017 | MAJOR |

---

### S5: Random Deletes / Snapshot Recovery

1. Build registry with 10K objects
2. Delete 5K random objects over 10 minutes
3. Take snapshot
4. Restore from snapshot
5. Verify state matches pre-delete snapshot

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Delete 5K objects: zero errors | SS-018 | MAJOR | |
| Snapshot captures state correctly | SS-019 | CRITICAL | |
| Restore from snapshot: exact match | SS-020 | CRITICAL | |
| Rollback to snapshot: exact match | SS-021 | MAJOR | |

---

## Stress Metrics

| Metric | Bronze | Silver | Gold |
|--------|--------|--------|------|
| Zero crashes | required | required | required |
| P99 degradation factor | ≤ 5× | ≤ 3× | ≤ 2× |
| Memory growth % | ≤ 50% | ≤ 25% | ≤ 10% |
| Data corruption incidents | 0 | 0 | 0 |
| Recovery time from restart | ≤ 120s | ≤ 60s | ≤ 30s |

---

## Report Format

```json
{
  "domain": "stress",
  "scenarios": {
    "S1_continuous": {
      "qps": 100,
      "duration_min": 10,
      "crashes": 0,
      "p99_start_ms": 18.2,
      "p99_end_ms": 21.4,
      "p99_degradation_factor": 1.18,
      "rss_growth_pct": 4.2
    },
    "S3_concurrent_rw": {
      "deadlocks": 0,
      "corruption_incidents": 0
    },
    "S5_snapshot": {
      "snapshot_match": true,
      "restore_match": true
    }
  },
  "checks": {
    "SS-001": "PASS", "SS-010": "PASS", "SS-015": "PASS",
    "SS-019": "PASS", "SS-020": "PASS"
  },
  "domain_score": 0.934,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert stress                          # all stress scenarios
kos-cert stress --scenario S1 --qps 100 --duration 600
kos-cert stress --scenario S3 --readers 10 --writers 5
kos-cert stress --scenario S5
kos-cert stress --output reports/stress.json
```

---

## Cross-References

- Concurrency → `17-CONCURRENCY-CERTIFICATION`
- Recovery → `19-RECOVERY-CERTIFICATION`
- Security (adversarial inputs) → `18-SECURITY-CERTIFICATION`
- Reliability weight → `27-OVERALL-SCORING`
