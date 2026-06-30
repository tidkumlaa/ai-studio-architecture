# KNW-KC-ARCH-001 — Identity Engine

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Identity Engine defines the global, unique, permanent identity of every Knowledge Object. No object may exist without a complete identity record. No two objects may share the same identity.

---

## Identity Fields (Required — All Objects)

| Field | Type | Description | Example |
|-------|------|-------------|---------|
| `global_id` | UUID4 | Permanent immutable identifier — never reused | `a3f2c1d4-...` |
| `knowledge_uri` | KNW URI | Canonical addressable URI (see 04-URI-SPECIFICATION) | `knw://kos/arch/001` |
| `knowledge_id` | String | Human-readable stable ID (see 03-NAMESPACE-SYSTEM) | `KNW-KOS-ARCH-001` |
| `namespace` | String | Domain namespace (see 03-NAMESPACE-SYSTEM) | `kos` |
| `canonical_name` | String | Unique canonical name within namespace | `kos.arch.vision` |
| `aliases` | List[String] | Alternative names — all must resolve to same object | `["vision", "kos-vision"]` |
| `version` | SemVer | Semantic version (see 08-VERSION-ENGINE) | `1.3.0` |
| `owner` | OwnerRef | Team or person accountable for this object | `team:architecture-board` |
| `domain` | DomainEnum | Knowledge domain classification | `ARCHITECTURE` |
| `created_at` | ISO 8601 UTC | When this identity was first registered | `2026-01-01T00:00:00Z` |
| `updated_at` | ISO 8601 UTC | Last mutation timestamp | `2026-06-01T12:00:00Z` |
| `checksum` | SHA-256 | Hash of canonical content — detects corruption | `sha256:a1b2c3...` |
| `fingerprint` | String | Structural fingerprint — detects semantic duplication | `fp:kos:arch:001:v1` |
| `lineage` | LineageRef | Reference to parent version or forked object | `KNW-KOS-ARCH-000@1.2.0` |

---

## Identity Rules

### IR-001 — Global Uniqueness
`global_id` is assigned once at creation. It never changes. It is never reassigned after object deletion.

### IR-002 — Knowledge ID Stability
`knowledge_id` format: `KNW-{DOMAIN}-{TYPE}-{NNN}` (uppercase, hyphens only).  
Once assigned, the knowledge_id is stable across all versions of the object.

### IR-003 — Canonical Name Uniqueness
`canonical_name` must be unique within its `namespace`. The registry enforces this at write time.

### IR-004 — Alias Resolution
All aliases resolve to exactly one `canonical_name`. An alias may not be shared between objects, even across namespaces.

### IR-005 — Checksum Integrity
`checksum` is recomputed on every write. A stale checksum triggers a CORRUPTION alert and blocks promotion past REVIEW.

### IR-006 — Fingerprint Deduplication
`fingerprint` is computed from (domain, type, canonical content hash). Duplicate fingerprints trigger a DUPLICATE alert. A human must resolve before the object may reach VERIFIED status.

### IR-007 — Lineage Completeness
Every non-root object must carry a valid `lineage` reference. Root objects use `lineage: null`.

### IR-008 — Owner Accountability
`owner` must reference a registered team or person in the ownership registry. Unowned objects may not leave DRAFT status.

---

## Identity Computation

```
global_id    = UUID4()                          # generated once at creation
knowledge_id = KNW-{DOMAIN_CODE}-{TYPE_CODE}-{SEQUENCE_NNN}
checksum     = SHA-256(canonical_json(content))
fingerprint  = SHA-256(domain + "|" + type + "|" + checksum)[0:16]
canonical_name = namespace + "." + type_lower + "." + short_slug
```

---

## Duplicate Detection

```
DUPLICATE_CHECK:
  1. Look up fingerprint in registry
  2. If match found → DUPLICATE_ALERT
  3. Resolve: merge | supersede | discard
  4. If no match → proceed
```

---

## Identity Lifecycle

```
UNREGISTERED → REGISTERED → STABLE → DEPRECATED → TOMBSTONED
```

- **UNREGISTERED**: local draft, no global_id assigned
- **REGISTERED**: global_id assigned, identity record created in registry
- **STABLE**: all identity fields validated, object promoted past DRAFT
- **DEPRECATED**: identity preserved for backwards compatibility, no writes
- **TOMBSTONED**: permanently retired, global_id preserved in audit log forever

---

## Identity Protocol (Python)

```python
class KnowledgeIdentity(Protocol):
    global_id: UUID
    knowledge_id: str
    knowledge_uri: str
    namespace: str
    canonical_name: str
    aliases: list[str]
    version: str
    owner: str
    domain: str
    created_at: datetime
    updated_at: datetime
    checksum: str
    fingerprint: str
    lineage: str | None

    def recompute_checksum(self) -> str: ...
    def recompute_fingerprint(self) -> str: ...
    def is_duplicate_of(self, other: KnowledgeIdentity) -> bool: ...
    def resolve_alias(self, alias: str) -> str: ...
```

---

## Error Conditions

| Code | Condition | Action |
|------|-----------|--------|
| `IE-001` | `global_id` collision | Hard reject — never proceeds |
| `IE-002` | `canonical_name` conflict in namespace | Hard reject |
| `IE-003` | `alias` conflict | Hard reject |
| `IE-004` | `fingerprint` duplicate | Soft alert — human review required |
| `IE-005` | `checksum` mismatch | CORRUPTION alert — quarantine |
| `IE-006` | Missing `owner` | Soft reject — object stays DRAFT |
| `IE-007` | Malformed `knowledge_id` | Hard reject |

---

## Cross-References

- Namespace assignment → `03-NAMESPACE-SYSTEM`
- URI format → `04-URI-SPECIFICATION`
- Versioning of identity → `08-VERSION-ENGINE`
- Identity in registry → `15-OBJECT-REGISTRY`
- Lineage in graph → `24-KNOWLEDGE-GRAPH-MODEL`
