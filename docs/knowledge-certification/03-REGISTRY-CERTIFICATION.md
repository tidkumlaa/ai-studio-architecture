# KNW-CERT-ARCH-003 — Registry Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies correctness, consistency, and concurrency safety of the Knowledge Registry. Every CRUD operation must be atomic, consistent, and deterministic.

---

## Functional Checks

### Register

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Register new object | RC-001 | CRITICAL | Object is retrievable by ID immediately after |
| Register with duplicate ID rejected | RC-002 | CRITICAL | Returns error; registry unchanged |
| Register semantic duplicate detected | RC-003 | MAJOR | Checksum match raises DuplicateObjectError |
| Register invalid schema rejected | RC-004 | CRITICAL | Returns validation error; object not stored |
| Batch register 100 objects | RC-005 | MAJOR | All 100 objects registered or zero (atomic) |
| Batch register with one invalid | RC-006 | MAJOR | Zero objects registered on any failure |

### Lookup

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Lookup by knowledge_id | RC-007 | CRITICAL | Returns exact object |
| Lookup by canonical_name | RC-008 | MAJOR | Returns same object as ID lookup |
| Lookup by alias | RC-009 | MAJOR | Resolves alias to canonical object |
| Lookup non-existent ID | RC-010 | CRITICAL | Returns ObjectNotFoundError (not null) |
| Lookup returns current version | RC-011 | MAJOR | Version matches latest registered |

### Resolve

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Resolve version range | RC-012 | MAJOR | Returns highest compatible version |
| Resolve pinned version | RC-013 | MAJOR | Returns exact version or error |
| Resolve with no matching version | RC-014 | MAJOR | Returns VersionNotFoundError |

### Update

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Update existing object | RC-015 | CRITICAL | New version retrievable; old version still queryable |
| Update non-existent ID | RC-016 | CRITICAL | Returns ObjectNotFoundError |
| Update preserves history | RC-017 | MAJOR | All previous versions accessible |
| Update increments version | RC-018 | MAJOR | New version > old version |

### Delete / Archive

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Archive object | RC-019 | MAJOR | Object status = ARCHIVED; not in default results |
| Delete rejected if referenced | RC-020 | MAJOR | Returns ReferenceIntegrityError |
| Force archive overrides | RC-021 | MINOR | With --force, archived even if referenced |

### Namespace / Collision

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Namespace collision detected | RC-022 | MAJOR | Two objects with same canonical_name rejected |
| Cross-namespace lookup | RC-023 | MINOR | `plt:module/quota-manager` resolves correctly |

---

## Concurrency Checks

All concurrency checks use 10 concurrent threads unless stated otherwise.

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| 10 concurrent lookups | CC-001 | MAJOR | All return correct result; no corruption |
| 10 concurrent registers (distinct IDs) | CC-002 | MAJOR | All 10 registered; no data loss |
| 10 concurrent updates to same object | CC-003 | MAJOR | Exactly one update wins; no partial writes |
| 10 concurrent deletes of same object | CC-004 | MAJOR | Exactly one delete succeeds; rest get error |
| Read during write | CC-005 | CRITICAL | Read never returns partial object |
| Stats are consistent under concurrency | CC-006 | MINOR | Stats counts match actual object count |

---

## Latency Targets (at 100K objects)

| Operation | P50 | P95 | P99 |
|-----------|-----|-----|-----|
| Lookup by ID | ≤ 1ms | ≤ 2ms | ≤ 5ms |
| Register | ≤ 5ms | ≤ 10ms | ≤ 20ms |
| Update | ≤ 5ms | ≤ 10ms | ≤ 20ms |
| Batch register (100 objects) | ≤ 100ms | ≤ 200ms | ≤ 500ms |
| Namespace scan | ≤ 10ms | ≤ 20ms | ≤ 50ms |

---

## Registry Check Output Format

```json
{
  "domain": "registry",
  "timestamp": "2026-06-30T00:00:00Z",
  "functional": {
    "RC-001": "PASS",
    "RC-002": "PASS",
    "RC-003": "PASS",
    "RC-004": "PASS",
    "RC-005": "PASS",
    "RC-006": "PASS",
    "RC-022": "FAIL"
  },
  "concurrency": {
    "CC-001": "PASS",
    "CC-002": "PASS",
    "CC-003": "PASS",
    "CC-004": "PASS",
    "CC-005": "PASS",
    "CC-006": "PASS"
  },
  "latency_ms": {
    "lookup_p50": 0.8,
    "lookup_p99": 3.2,
    "register_p50": 3.1,
    "register_p99": 14.5
  },
  "critical_pass": true,
  "major_pass": false,
  "domain_score": 0.921,
  "level_achieved": "Silver"
}
```

---

## CLI

```bash
kos-cert registry                        # full registry certification
kos-cert registry --concurrency 20       # 20 threads for concurrency checks
kos-cert registry --skip-concurrency     # functional only
kos-cert registry --output reports/registry.json
```

---

## Cross-References

- Registry contract → Phase 3.0C.5 `14-KNOWLEDGE-REGISTRY`
- Concurrency → `17-CONCURRENCY-CERTIFICATION`
- Performance at scale → `14-PERFORMANCE-CERTIFICATION`
- Overall score weight (10%) → `27-OVERALL-SCORING`
