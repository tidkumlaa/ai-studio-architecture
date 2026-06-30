# KNW-KC-ARCH-034 — State Machines

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

This document specifies the formal state machines for all stateful processes in the Knowledge Core. Each machine defines states, transitions, guards, triggers, and effects.

---

## SM-1: Knowledge Object Lifecycle

The primary lifecycle state machine for all `KnowledgeObject` instances.

```
States: DRAFT | REVIEW | VERIFIED | APPROVED | CANONICAL | DEPRECATED | ARCHIVED

DRAFT
  → REVIEW
    guard:   identity complete (all 14 identity fields present)
    trigger: owner submits for review
    effect:  create LIFECYCLE_SNAPSHOT; assign reviewer; emit lifecycle.object.status_changed

REVIEW
  → VERIFIED
    guard:   quality.overall_score >= 0.65
             evidence.evidence_score >= 0.55
             ≥ 1 EV-CODE or EV-TEST evidence record
             lifecycle_extension gates pass (33-LIFECYCLE-EXTENSIONS)
    trigger: reviewer approves
    effect:  bump version MINOR; emit lifecycle.object.status_changed

  → DRAFT
    guard:   (none)
    trigger: reviewer requests revision
    effect:  emit lifecycle.object.status_changed; notify owner

VERIFIED
  → APPROVED
    guard:   quality.overall_score >= 0.75
             ≥ 1 EV-HAPPROVAL evidence
             approved_by is non-empty
    trigger: approver signs off
    effect:  set approved_at; emit lifecycle.object.status_changed

APPROVED
  → CANONICAL
    guard:   quality.overall_score >= 0.80
             confidence >= 0.80
             traceability coverage >= 0.70
             all MUST requirements linked
             lifecycle_extension gates pass
    trigger: architecture board votes
    effect:  version locked (PATCH only); emit knowledge.architecture.frozen

CANONICAL
  → DEPRECATED
    guard:   successor_id is set (or explicit waiver filed)
    trigger: owner or architecture board triggers deprecation
    effect:  set deprecated_at; emit knowledge.object.deprecated
             auto-transition relationship status → DEPRECATED

DEPRECATED
  → ARCHIVED
    guard:   deprecated_at + 90 days elapsed OR all consumers migrated
    trigger: automated job or manual trigger
    effect:  emit knowledge.object.archived; object is read-only
```

---

## SM-2: Knowledge Identity Lifecycle

```
States: UNREGISTERED | REGISTERED | STABLE | DEPRECATED | TOMBSTONED

UNREGISTERED → REGISTERED
  trigger: first write to registry
  effect:  assign global_id; compute fingerprint; check duplicates

REGISTERED → STABLE
  trigger: lifecycle.status reaches VERIFIED
  effect:  identity locked (global_id, knowledge_id, canonical_name immutable)

STABLE → DEPRECATED
  trigger: object lifecycle deprecated
  effect:  identity preserved; aliases still resolve; redirect to successor

DEPRECATED → TOMBSTONED
  trigger: object lifecycle archived
  effect:  tombstone record created; URI resolves to tombstone forever
```

---

## SM-3: Version State Machine

```
States: DRAFT | PUBLISHED | DEPRECATED | SUPERSEDED

DRAFT → PUBLISHED
  trigger: object lifecycle transitions past DRAFT
  effect:  version record immutable; VERSION_SNAPSHOT created

PUBLISHED → DEPRECATED
  trigger: MAJOR version bump makes this version obsolete
  effect:  version record marked deprecated; migration notes added

PUBLISHED → SUPERSEDED
  trigger: FORK or SUPERSEDES relationship created
  effect:  version record points to superseding version
```

---

## SM-4: Evidence Record Lifecycle

```
States: PROPOSED | ACTIVE (FRESH | DEGRADING | STALE) | EXPIRED | DISPUTED

PROPOSED → ACTIVE
  trigger: creator submits evidence with source reference
  effect:  compute initial freshness_score; set status = ACTIVE

ACTIVE → EXPIRED
  trigger: freshness_score == 0.0 (computed by freshness decay model)
  effect:  emit evidence.record.expired; parent object quality recalculated

ACTIVE → DISPUTED
  trigger: dispute filed
  effect:  emit evidence.record.disputed; parent object status may regress

DISPUTED → ACTIVE
  trigger: dispute dismissed
  effect:  restore original status; notify owner

DISPUTED → EXPIRED
  trigger: dispute upheld
  effect:  evidence removed from consideration
```

---

## SM-5: Snapshot Lifecycle

```
States: CREATED | VERIFIED | AVAILABLE | EXPIRED | RETAINED_FOREVER

CREATED → VERIFIED
  trigger: checksum validation passes
  effect:  manifest sealed

VERIFIED → AVAILABLE
  trigger: all files written successfully
  effect:  snapshot discoverable via API

AVAILABLE → EXPIRED
  trigger: retention period elapsed AND not tagged permanent
  effect:  files eligible for deletion

AVAILABLE → RETAINED_FOREVER
  trigger: tagged as release snapshot or lifecycle snapshot
  effect:  permanently indexed; never deleted
```

---

## SM-6: Registry Index Health

```
States: VALID | STALE | REBUILDING | CORRUPT

VALID → STALE
  trigger: index entry count differs from object count by > 0.1%
  effect:  emit registry.health.alert; schedule incremental rebuild

STALE → REBUILDING
  trigger: scheduled rebuild job runs
  effect:  lock writes; rebuild from primary storage

REBUILDING → VALID
  trigger: rebuild complete, spot-check passes
  effect:  unlock writes; emit registry.index.rebuilt

VALID → CORRUPT
  trigger: checksum mismatch on index read
  effect:  immediate rebuild; alert on-call; block all reads until recovered

CORRUPT → REBUILDING
  trigger: immediate
  effect:  (same as STALE → REBUILDING)
```

---

## SM-7: Relationship Lifecycle

```
States: PROPOSED | ACTIVE | DEPRECATED | REMOVED

PROPOSED → ACTIVE
  guard:   source and target exist; no cycle in non-cyclic types
  trigger: relationship engine validates and accepts
  effect:  add to graph; update indexes

ACTIVE → DEPRECATED
  trigger: source or target object deprecated
  effect:  relation marked deprecated; graph traversal skips by default

DEPRECATED → REMOVED
  trigger: human explicit removal action
  effect:  relation removed from graph; version record preserved

ACTIVE → REMOVED
  trigger: cycle detected post-creation (emergency fix)
  effect:  relation removed immediately; audit log entry created
```

---

## State Machine Summary

| Machine | States | Key Guard |
|---------|--------|-----------|
| SM-1 Object Lifecycle | 7 | quality_score ≥ threshold per state |
| SM-2 Identity Lifecycle | 5 | identity completeness |
| SM-3 Version | 4 | version immutability after publish |
| SM-4 Evidence | 5 | freshness decay |
| SM-5 Snapshot | 5 | checksum validation |
| SM-6 Registry Index | 4 | checksum + count consistency |
| SM-7 Relationship | 4 | cycle detection |

---

## Cross-References

- Quality thresholds for SM-1 → `11-QUALITY-ENGINE`
- Evidence freshness for SM-4 → `10-EVIDENCE-ENGINE`
- Lifecycle extensions (additional guards) → `33-LIFECYCLE-EXTENSIONS`
- Snapshot creation in SM-5 → `09-SNAPSHOT-ENGINE`
- Cycle detection (SM-7) → `25-GRAPH-ALGORITHMS`
