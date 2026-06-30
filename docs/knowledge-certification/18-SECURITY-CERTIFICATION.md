# KNW-CERT-ARCH-018 — Security Certification

**Phase:** 3.0D.0  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Certifies that the KOS platform correctly detects and rejects malicious or invalid inputs — duplicate IDs, namespace injection, broken references, invalid metadata, invalid versions, relationship loops, and unauthorised access.

---

## Security Check Categories

| Category | Checks | Severity |
|----------|--------|----------|
| ID Security | NS-001–NS-003 | CRITICAL |
| Namespace Security | NS-004–NS-006 | MAJOR |
| Reference Integrity | NS-007–NS-010 | MAJOR |
| Metadata Tampering | NS-011–NS-015 | MAJOR |
| Version Security | NS-016–NS-018 | MAJOR |
| Relationship Security | NS-019–NS-022 | MAJOR |
| Access Control | NS-023–NS-027 | MAJOR |

---

## ID Security

| Check | ID | Severity | Attack | Pass Criteria |
|-------|----|----------|--------|---------------|
| Duplicate ID injection | NS-001 | CRITICAL | Register object with existing ID | DuplicateIDError |
| ID format injection | NS-002 | CRITICAL | Register object with `../../../etc/passwd` as ID | ValidationError |
| ID with SQL injection | NS-003 | CRITICAL | `KNW-PLT-MOD-001'; DROP TABLE--` | ValidationError |

---

## Namespace Injection

| Check | ID | Severity | Attack | Pass Criteria |
|-------|----|----------|--------|---------------|
| Namespace escape | NS-004 | MAJOR | Object in namespace `plt/../admin` | ValidationError |
| Namespace collision | NS-005 | MAJOR | Two packages declare same namespace | NamespaceConflictError |
| Cross-namespace write | NS-006 | MAJOR | Package `plt` writes object with namespace `rt` | NamespaceMismatchError |

---

## Broken References

| Check | ID | Severity | Attack | Pass Criteria |
|-------|----|----------|--------|---------------|
| Dangling reference in relationship | NS-007 | MAJOR | Relationship to non-existent target | ReferenceNotFoundError |
| Circular traceability chain | NS-008 | MAJOR | A satisfies B; B satisfies A | CircularReferenceError |
| Self-relationship | NS-009 | MAJOR | Object DEPENDS_ON itself | SelfReferenceError |
| Orphaned object after delete | NS-010 | MAJOR | Delete causes dangling reference | ReferenceIntegrityError |

---

## Metadata Tampering

| Check | ID | Severity | Attack | Pass Criteria |
|-------|----|----------|--------|---------------|
| Write to read-only field | NS-011 | MAJOR | Set `quality_score` directly | FieldNotWritableError |
| Checksum mismatch | NS-012 | CRITICAL | Modified content without updating checksum | ChecksumMismatchError |
| Backdated `created_at` | NS-013 | MINOR | Set `created_at` to future date | ValidationError |
| Invalid lifecycle state | NS-014 | CRITICAL | Set state to non-enum value | ValidationError |
| Invalid object_type | NS-015 | CRITICAL | Set `object_type` to unknown value | ValidationError |

---

## Version Security

| Check | ID | Severity | Attack | Pass Criteria |
|-------|----|----------|--------|---------------|
| Downgrade version attack | NS-016 | MAJOR | Register v1.0.0 after v2.0.0 exists | VersionConflictError |
| Invalid semver | NS-017 | MAJOR | `version: "not-a-version"` | ValidationError |
| Version rollback | NS-018 | MAJOR | Update existing object to lower version | VersionDowngradeError |

---

## Relationship Loops

| Check | ID | Severity | Attack | Pass Criteria |
|-------|----|----------|--------|---------------|
| Direct DEPENDS_ON cycle | NS-019 | MAJOR | A→B→A | CircularDependencyError |
| Indirect cycle (10 hops) | NS-020 | MAJOR | A→B→…→A (10 nodes) | CircularDependencyError |
| Self-loop | NS-021 | MAJOR | A DEPENDS_ON A | SelfReferenceError |
| Cross-package cycle | NS-022 | MAJOR | plt→rt→plt | CircularDependencyError |

---

## Access Control

| Check | ID | Severity | Pass Criteria |
|-------|----|----------|---------------|
| Unauthenticated write rejected | NS-023 | MAJOR | Returns AuthenticationError |
| Write by non-owner rejected | NS-024 | MAJOR | Returns AuthorizationError |
| Read of ARCHIVED object by non-admin | NS-025 | MINOR | Returns NotFoundError |
| Force-archive without authority | NS-026 | MAJOR | Returns AuthorizationError |
| MAJOR change without board approval | NS-027 | MAJOR | Returns ApprovalRequiredError |

---

## Security Score

| Check Type | Bronze | Silver | Gold | Enterprise |
|------------|--------|--------|------|------------|
| ID Security (NS-001–003) | 100% pass | 100% | 100% | 100% |
| Namespace injection (NS-004–006) | 100% | 100% | 100% | 100% |
| Broken references (NS-007–010) | ≥ 75% | ≥ 90% | 100% | 100% |
| Metadata tampering (NS-011–015) | ≥ 80% | ≥ 90% | 100% | 100% |
| Relationship loops (NS-019–022) | 100% | 100% | 100% | 100% |
| Access control (NS-023–027) | ≥ 60% | ≥ 80% | ≥ 90% | 100% |

---

## Report Format

```json
{
  "domain": "security",
  "categories": {
    "id_security": {"checks": 3, "pass": 3, "rate": 1.0},
    "namespace_injection": {"checks": 3, "pass": 3, "rate": 1.0},
    "broken_references": {"checks": 4, "pass": 4, "rate": 1.0},
    "metadata_tampering": {"checks": 5, "pass": 5, "rate": 1.0},
    "version_security": {"checks": 3, "pass": 2, "rate": 0.667},
    "relationship_loops": {"checks": 4, "pass": 4, "rate": 1.0},
    "access_control": {"checks": 5, "pass": 4, "rate": 0.800}
  },
  "critical_violations": 0,
  "domain_score": 0.933,
  "level_achieved": "Gold"
}
```

---

## CLI

```bash
kos-cert security                        # all security checks
kos-cert security --category id,namespace
kos-cert security --category access-control
kos-cert security --output reports/security.json
```

---

## Cross-References

- Stress (adversarial inputs) → `16-STRESS-CERTIFICATION`
- Registry correctness → `03-REGISTRY-CERTIFICATION`
- Dependency cycle detection → `09-DEPENDENCY-CERTIFICATION`
- Overall score weight (5%) → `27-OVERALL-SCORING`
