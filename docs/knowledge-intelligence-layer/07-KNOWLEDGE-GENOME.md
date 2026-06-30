# KNW-KIL-DOC-007 — Knowledge Genome

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object receives a Knowledge Genome — a structured, 8-field
profile that captures its complete identity, capabilities, relationships,
evidence, history, and role in the ecosystem.

The Genome is the highest-level summary of an object across all dimensions.
It enables instant object classification, comparison, and routing without
reading the full object.

This document defines `intelligence.genome`.

---

## Genome Fields

| Field | # | Description |
|-------|---|-------------|
| identity | 1 | Who/what is this object fundamentally |
| category | 2 | Object class and ecosystem role |
| domain | 3 | Knowledge domain and namespace position |
| capabilities | 4 | Functional capabilities with maturity levels |
| relationships | 5 | Key structural connections (compressed) |
| evidence | 6 | Evidence profile and confidence summary |
| algorithms | 7 | Algorithms used by or relevant to this object |
| history | 8 | Creation, major versions, lineage |

---

## Schema

```yaml
intelligence:
  genome:
    version: "1.0"
    generated_at: string               # ISO 8601

    identity:                          # Field 1: Core identity
      knowledge_id: string
      canonical_name: string
      object_uri: string
      object_type: string
      version: string
      checksum: string                 # identity checksum (excl. knowledge_id)
      fingerprint: string              # genome-level fingerprint (hash of all 8 fields)

    category:                          # Field 2: Classification
      primary_class: string            # e.g. "enforcement", "routing", "storage"
      secondary_class: string | null
      pattern_type: string | null      # e.g. "Guard", "Facade", "Registry"
      abstraction_level: PRIMITIVE | COMPONENT | SERVICE | PLATFORM | ECOSYSTEM
      object_family: string            # grouping of related object types

    domain:                            # Field 3: Domain position
      namespace: string
      package: string
      knowledge_domain: string
      subdomain: string | null
      cross_domain_references: [string] # namespaces this object references

    capabilities:                      # Field 4: Capability fingerprint
      - name: string
        category: string
        maturity: STABLE | BETA | EXPERIMENTAL
        since_version: string

    relationships:                     # Field 5: Key relationships (nano compressed)
      strong_dependencies: [string]    # HARD DEPENDS_ON
      implements: [string]             # IMPLEMENTS targets
      provides_to: [string]            # objects that depend on this
      tests: [string]                  # TEST objects
      relationship_count:
        total: integer
        by_type: {}                    # map: relationship_type → count

    evidence:                          # Field 6: Evidence profile
      item_count: integer
      level_distribution: {}           # map: level → count
      highest_level: integer           # 1–6
      quality_score: float
      confidence_score: float
      freshness_rating: FRESH | AGING | STALE | CRITICAL
      last_evidence_date: string

    algorithms:                        # Field 7: Algorithms
      uses:                            # algorithms this object uses
        - knowledge_id: string
          algorithm_name: string
          role: PRIMARY | SECONDARY | FALLBACK
      relevant:                        # algorithms relevant to understanding this object
        - knowledge_id: string
          relevance: string

    history:                           # Field 8: Lineage
      created_at: string
      created_by: string
      version_count: integer
      major_versions: [string]
      origin:
        phase: string
        rationale: string
      lineage:                         # predecessor objects
        - knowledge_id: string
          relationship: EVOLVED_FROM | REPLACED | MERGED | SPLIT
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  genome:
    version: "1.0"
    generated_at: "2026-06-30T00:00:00Z"

    identity:
      knowledge_id: KNW-PLT-MOD-001
      canonical_name: plt.module.quota-manager
      object_uri: "knw://plt/module/quota-manager@2.1.0"
      object_type: MODULE
      version: "2.1.0"
      checksum: "sha256:a1b2c3d4e5f6..."
      fingerprint: "genome:7f8e9d0c..."

    category:
      primary_class: "enforcement"
      secondary_class: "resource-management"
      pattern_type: "Guard"
      abstraction_level: COMPONENT
      object_family: "platform-governance"

    domain:
      namespace: "plt"
      package: "kos.platform.package"
      knowledge_domain: "platform"
      subdomain: "resource-governance"
      cross_domain_references: ["rt", "fin", "meta"]

    capabilities:
      - name: "quota-enforcement"
        category: "GUARD"
        maturity: STABLE
        since_version: "1.0.0"
      - name: "quota-reporting"
        category: "MONITOR"
        maturity: STABLE
        since_version: "1.5.0"

    relationships:
      strong_dependencies: [KNW-RT-RT-001, KNW-PLT-CFG-001]
      implements: [KNW-PLT-REQ-001, KNW-META-STD-007]
      provides_to: [KNW-PLT-SVC-002, KNW-API-API-001]
      tests: [KNW-TEST-TST-001, KNW-TEST-BENCH-001]
      relationship_count:
        total: 12
        by_type:
          DEPENDS_ON: 2
          IMPLEMENTS: 2
          DEPENDENCY_OF: 3
          TESTED_BY: 2
          PROVIDES: 3

    evidence:
      item_count: 5
      level_distribution:
        1: 1
        2: 1
        3: 2
        4: 1
      highest_level: 1
      quality_score: 0.87
      confidence_score: 0.91
      freshness_rating: FRESH
      last_evidence_date: "2026-06-15"

    algorithms:
      uses:
        - knowledge_id: KNW-ALG-ALG-012
          algorithm_name: "Token Bucket"
          role: PRIMARY
      relevant:
        - knowledge_id: KNW-ALG-ALG-009
          relevance: "Rate limiting algorithm — often confused with quota enforcement"

    history:
      created_at: "2025-01-15T00:00:00Z"
      created_by: "platform-team"
      version_count: 4
      major_versions: ["1.0.0", "1.5.0", "2.0.0", "2.1.0"]
      origin:
        phase: "3.0C"
        rationale: "Initial quota enforcement module for multi-tenant platform"
      lineage:
        - knowledge_id: KNW-PLT-MOD-LEGACY-001
          relationship: EVOLVED_FROM
```

---

## Genome Fingerprint

The genome fingerprint is a hash of all 8 genome fields (excluding `generated_at`).
Two objects with the same fingerprint are semantically identical.

```
genome_fingerprint = sha256(
  identity.checksum +
  category.primary_class + category.abstraction_level +
  domain.namespace + domain.package +
  sorted(capabilities[*].name) +
  sorted(relationships.strong_dependencies) +
  evidence.quality_score + evidence.confidence_score +
  sorted(algorithms.uses[*].knowledge_id) +
  history.created_at
)
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-042 | Every CANONICAL object MUST have a complete genome with all 8 fields |
| KIL-043 | `genome.identity.fingerprint` must be recomputed whenever any genome field changes |
| KIL-044 | `genome.history.lineage` must trace back to the originating phase |
| KIL-045 | `genome.relationships.relationship_count` must exactly match the object's relationship block |
| KIL-046 | `genome.evidence.quality_score` must equal `metadata.quality_score` — they are the same value |
| KIL-047 | `genome.category.abstraction_level` must be consistent with `object_type` |

---

## Cross-References

- DNA (immutable core) → `08-KNOWLEDGE-DNA`
- Memory (runtime updates) → `09-KNOWLEDGE-MEMORY`
- Evolution (history) → `11-KNOWLEDGE-EVOLUTION`
- Confidence → `13-KNOWLEDGE-CONFIDENCE`
- Search index → `29-KNOWLEDGE-CANONICAL-INDEX`
