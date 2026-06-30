# KNW-CERT-ARCH-019 — Recovery Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the KOS platform can recover from failures — process crashes, partial writes, corrupted state, and snapshots — with defined Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO).

---

## Recovery Targets

| Level | RTO | RPO |
|-------|-----|-----|
| Bronze | ≤ 120s | ≤ 1 hour |
| Silver | ≤ 60s | ≤ 15 minutes |
| Gold | ≤ 30s | ≤ 5 minutes |
| Enterprise | ≤ 10s | ≤ 1 minute |
| Research | ≤ 5s | ≤ 30 seconds |

**RTO** (Recovery Time Objective): time from failure detection to full service restoration.  
**RPO** (Recovery Point Objective): maximum data loss measured in time — i.e., the age of the last recoverable snapshot.

---

## Failure Scenarios

### F1: Clean Restart

Stop the process, restart, verify state.

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Registry state preserved after restart | RC-001 | CRITICAL | All objects retrievable |
| Index state preserved | RC-002 | CRITICAL | All search results correct |
| Graph state preserved | RC-003 | MAJOR | All traversals correct |
| RTO within level target | RC-004 | MAJOR | |

### F2: Crash During Write

Simulate crash mid-write (kill process during register operation).

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| No partial write visible after restart | RC-005 | CRITICAL | Object either fully written or not present |
| Registry consistent after restart | RC-006 | CRITICAL | No orphaned entries |
| RTO after crash ≤ level target | RC-007 | MAJOR | |

### F3: Corrupted State File

Corrupt one registry file, verify graceful error.

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Corruption detected on startup | RC-008 | CRITICAL | StartupError with file path |
| Restore from last snapshot | RC-009 | CRITICAL | State recoverable |
| Non-corrupted objects still accessible | RC-010 | MAJOR | Partial availability |

### F4: Snapshot + Restore

Take snapshot, modify state, restore, verify.

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Snapshot created successfully | RC-011 | CRITICAL | |
| Snapshot is complete (all objects) | RC-012 | CRITICAL | Count matches |
| Restore from snapshot: exact state | RC-013 | CRITICAL | Hash-equal to snapshot |
| Snapshot time ≤ level target | RC-014 | MINOR | Within build time budget |
| Restore time ≤ RTO | RC-015 | MAJOR | |

### F5: Rollback

Perform 100 operations, then roll back to pre-operation state.

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Rollback restores exact previous state | RC-016 | CRITICAL | State hash matches |
| Rollback time ≤ 2× RTO | RC-017 | MAJOR | |
| Rolled-back changes are not visible | RC-018 | CRITICAL | |

---

## Snapshot Protocol

```
1. Pause writes (drain in-flight writes)
2. Capture state to snapshot file
   - snapshot = {registry_index, graph_adjacency, catalog}
   - checksum all files
3. Write snapshot manifest with checksum
4. Resume writes
5. Verify snapshot on next certification run
```

---

## Metrics

| Metric | Bronze | Silver | Gold | Enterprise |
|--------|--------|--------|------|------------|
| RTO (seconds) | ≤ 120 | ≤ 60 | ≤ 30 | ≤ 10 |
| RPO (minutes) | ≤ 60 | ≤ 15 | ≤ 5 | ≤ 1 |
| State consistency after recovery | = 1.0 | = 1.0 | = 1.0 | = 1.0 |
| Partial write prevention | = 1.0 | = 1.0 | = 1.0 | = 1.0 |

---

## Report Format

```json
{
  "domain": "recovery",
  "scenarios": {
    "F1_clean_restart": {
      "rto_seconds": 18.4,
      "state_consistent": true
    },
    "F2_crash_during_write": {
      "partial_writes_visible": 0,
      "rto_seconds": 22.1
    },
    "F4_snapshot_restore": {
      "snapshot_time_s": 4.2,
      "restore_time_s": 11.8,
      "state_hash_match": true
    },
    "F5_rollback": {
      "rollback_time_s": 8.3,
      "state_hash_match": true
    }
  },
  "checks": {
    "RC-005": "PASS", "RC-013": "PASS", "RC-016": "PASS"
  },
  "rto_achieved_s": 22.1,
  "domain_score": 0.923,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert recovery                        # all failure scenarios
kos-cert recovery --scenario F4          # snapshot+restore only
kos-cert recovery --scenario F2          # crash simulation
kos-cert recovery --output reports/recovery.json
```

---

## Cross-References

- Backup → `20-BACKUP-CERTIFICATION`
- Stress (random failures) → `16-STRESS-CERTIFICATION`
- Reliability weight → `27-OVERALL-SCORING`
