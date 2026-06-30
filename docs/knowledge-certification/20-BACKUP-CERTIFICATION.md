# KNW-CERT-ARCH-020 — Backup Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the KOS platform produces complete, verifiable, and restorable backups. Backup certification is a prerequisite for Enterprise and Research certification levels.

---

## Backup Types

| Type | Trigger | Contents | Frequency |
|------|---------|----------|-----------|
| Full | Manual / nightly | All objects, relationships, index, catalog | Daily |
| Incremental | Per commit | Changed objects since last full | Per operation batch |
| Snapshot | On-demand | Point-in-time state copy | On demand |
| Verification | Post-backup | Checksum of backup contents | Automatic |

---

## Backup File Format

```yaml
# backup/full-{timestamp}/manifest.yaml
backup_type: full
created_at: "2026-06-30T00:00:00Z"
created_by: "kos-cert backup"
kos_version: "1.0.0"
dataset_version: "sha256:abc123"

contents:
  registry:
    files: 200
    checksum: "sha256:def456"
  relationships:
    files: 500
    checksum: "sha256:789abc"
  index:
    files: 1
    checksum: "sha256:cba321"
  catalog:
    files: 1
    checksum: "sha256:fed987"

total_checksum: "sha256:overall123"
compression: gzip
encryption: none    # or aes-256-gcm
```

---

## Backup Checks

### Completeness

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Full backup contains all registry objects | BC-001 | CRITICAL | Object count matches |
| Full backup contains all relationships | BC-002 | CRITICAL | Relationship count matches |
| Full backup contains index | BC-003 | MAJOR | Index file present |
| Full backup contains catalog | BC-004 | MAJOR | Catalog file present |
| Manifest checksum matches contents | BC-005 | CRITICAL | SHA-256 verified |

### Integrity

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Every backup file checksummed | BC-006 | CRITICAL | All files in manifest have checksum |
| Checksum verification passes | BC-007 | CRITICAL | Zero checksum mismatches |
| Backup not truncated | BC-008 | CRITICAL | File size matches manifest |
| Incremental backup covers exactly delta | BC-009 | MAJOR | Only changed objects included |

### Restore

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Restore from full backup: exact state | BC-010 | CRITICAL | State hash match |
| Restore from incremental: correct state | BC-011 | MAJOR | Correct state at incremental point |
| Restore time ≤ RTO | BC-012 | MAJOR | Depends on certification level |
| Partial restore (single object) | BC-013 | MINOR | Single object restorable from full backup |

### Rotation

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Old backups rotated after retention period | BC-014 | MINOR | No backups older than 30 days (default) |
| At least 3 full backups retained | BC-015 | MINOR | |

---

## Backup Performance Targets

| Operation | 1K objects | 10K objects | 100K objects |
|-----------|-----------|-------------|--------------|
| Full backup time | ≤ 5s | ≤ 30s | ≤ 5 min |
| Incremental backup time | ≤ 1s | ≤ 5s | ≤ 30s |
| Restore time | ≤ 10s | ≤ 60s | ≤ 10 min |
| Verification time | ≤ 5s | ≤ 30s | ≤ 5 min |

---

## Report Format

```json
{
  "domain": "backup",
  "backup_types_tested": ["full", "incremental", "snapshot"],
  "checks": {
    "BC-001": "PASS", "BC-005": "PASS",
    "BC-007": "PASS", "BC-010": "PASS",
    "BC-013": "FAIL"
  },
  "full_backup": {
    "time_s": 2.4,
    "size_mb": 12.8,
    "checksum_verified": true,
    "restore_time_s": 8.1,
    "restore_match": true
  },
  "incremental_backup": {
    "time_s": 0.3,
    "objects_backed_up": 15,
    "restore_match": true
  },
  "domain_score": 0.941,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert backup                          # full backup certification
kos-cert backup --type full
kos-cert backup --type incremental
kos-cert backup --verify-only            # checksum verification only
kos-cert backup --restore-test           # backup + restore + verify
kos-cert backup --output reports/backup.json
```

---

## Cross-References

- Recovery (snapshot restore) → `19-RECOVERY-CERTIFICATION`
- Snapshot format → `19-RECOVERY-CERTIFICATION`
- Reliability weight → `27-OVERALL-SCORING`
