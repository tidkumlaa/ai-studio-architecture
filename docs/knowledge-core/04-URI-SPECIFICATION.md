# KNW-KC-ARCH-004 — URI Specification

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Knowledge URI provides a globally addressable, human-readable, machine-parseable locator for every Knowledge Object. URIs are stable across versions and must remain resolvable even after deprecation.

---

## URI Format

```
knw://{namespace}/{type}/{name}[@{version}][#{fragment}]

Components:
  scheme     = "knw"                    (always)
  namespace  = registered namespace     (lowercase, hyphens allowed)
  type       = object type code         (lowercase)
  name       = canonical slug           (lowercase, hyphens)
  version    = optional semver pin      (@1.2.3 or @latest or @stable)
  fragment   = optional section anchor  (#section-name)
```

---

## URI Examples

```
knw://kos/arch/vision                        # latest version
knw://kos/arch/vision@1.0.0                  # pinned version
knw://kos/arch/vision@latest                 # explicit latest
knw://platform/service/auth@stable           # stable channel
knw://ai/module/quota-manager                # module
knw://kos/arch/vision#golden-rules           # section fragment
knw://financial/market/eurusd                # financial object
knw://platform/api/intelligence@2.0.0        # versioned API
```

---

## Version Channels

| Channel | Resolves To |
|---------|-------------|
| `@latest` | Highest version number in any lifecycle state |
| `@stable` | Highest version in APPROVED or CANONICAL state |
| `@canonical` | Highest version in CANONICAL state only |
| `@{semver}` | Exact version pin |

---

## URI Resolution Protocol

```
RESOLVE(uri):
  1. Parse scheme, namespace, type, name, version, fragment
  2. Look up namespace in Namespace Registry → domain
  3. Look up canonical_name = "{namespace}.{type}.{name}"
  4. Look up in Object Registry by canonical_name
  5. Apply version channel resolution
  6. If fragment: resolve to section anchor within document
  7. Return KnowledgeObject reference
  8. If not found → URI_NOT_FOUND (404)
  9. If ambiguous → URI_AMBIGUOUS (409)
```

---

## URI Stability Rules

### UR-001 — Permanent Scheme
The `knw://` scheme is permanent. No other scheme is valid.

### UR-002 — Namespace Stability
The namespace component of a URI never changes after registration. Renames require a redirect alias for 2 years.

### UR-003 — Name Stability
The name component is stable across all versions of an object. A change to the canonical name is a breaking change and requires a new object with a supersede relationship.

### UR-004 — Tombstone Redirect
Deleted objects retain their URI but resolve to a tombstone record pointing to the successor object or deprecation reason.

### UR-005 — Version Immutability
A pinned URI (`@1.2.3`) must always resolve to the same object content. Once a version is published, it is immutable.

### UR-006 — Fragment Stability
Fragment anchors are considered best-effort — a section rename does not break the URI but may return a `FRAGMENT_NOT_FOUND` warning.

---

## URI Comparison Rules

```
Two URIs are EQUIVALENT if:
  - namespace, type, name are equal (case-insensitive)
  - version resolves to the same object version

Two URIs are ALIASES if:
  - namespace differs but registered as alias
  - canonical_name resolves to the same global_id

Two URIs CONFLICT if:
  - same namespace + name but different type
  - (this is a schema error — prevented at registration time)
```

---

## URI Index Format

```yaml
# uri-index.yaml (machine-generated)
uris:
  "knw://kos/arch/vision":
    global_id: "a3f2c1d4-0000-0000-0000-000000000001"
    knowledge_id: "KNW-KOS-ARCH-001"
    canonical_name: "kos.arch.vision"
    current_version: "1.3.0"
    channels:
      latest: "1.3.0"
      stable: "1.3.0"
      canonical: "1.2.0"
    aliases:
      - "knw://kos/architecture/vision"
    tombstoned: false
    redirects_to: null
```

---

## HTTP Gateway (Future)

When a KOS HTTP gateway is implemented, URIs map to HTTP endpoints:

```
knw://kos/arch/vision
  → https://kos.internal/api/v1/objects/kos/arch/vision
  → returns: KnowledgeObject (JSON)

knw://kos/arch/vision@1.0.0
  → https://kos.internal/api/v1/objects/kos/arch/vision?version=1.0.0

knw://kos/arch/vision#golden-rules
  → https://kos.internal/api/v1/objects/kos/arch/vision#golden-rules
```

---

## URI Error Codes

| Code | Condition |
|------|-----------|
| `URI-001` | Malformed URI (fails parse) |
| `URI-002` | Unknown namespace |
| `URI-003` | Unknown type code |
| `URI-004` | Object not found |
| `URI-005` | Version not found |
| `URI-006` | Tombstoned — redirect available |
| `URI-007` | Ambiguous alias — multiple matches |

---

## Cross-References

- Namespace → `03-NAMESPACE-SYSTEM`
- Versioning → `08-VERSION-ENGINE`
- Object Registry lookup → `15-OBJECT-REGISTRY`
- Tombstone model → `09-SNAPSHOT-ENGINE`
