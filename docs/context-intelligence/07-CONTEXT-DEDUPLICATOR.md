# UICE-DOC-007 — Context Deduplicator

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Deduplicator eliminates identical and near-identical content
from a context pack before it is planned and assembled. When multiple
objects reference the same pattern, constraint, or relationship, sending
that content multiple times wastes tokens without adding information.

UICE target: **Duplicate Elimination = 100%** (exact duplicates).

---

## Deduplication Types

```
TYPE-1: EXACT FIELD DUPLICATE
  Two slices contain an identical field value (same key, same content).
  Action: Remove from lower-priority slot; keep in higher-priority slot.
  Detection: sha256(json_serialize(field_value))

TYPE-2: EXACT OBJECT DUPLICATE
  Two slots reference the same knowledge_id.
  Action: Keep the higher-priority slot; mark lower-priority as OMIT.
  Detection: knowledge_id match

TYPE-3: NEAR-DUPLICATE CONSTRAINT
  Two slices contain constraints that are semantically equivalent
  (e.g., "Must be stateless" and "Stateless behavior required").
  Action: Keep one; replace the other with a cross-reference marker.
  Detection: Levenshtein similarity ≥ 0.90 between normalized strings

TYPE-4: REDUNDANT RELATIONSHIP
  Object A's requires list includes Object B, and Object B is also
  a named context slot. The relationship fact is then duplicated.
  Action: Remove the relationship entry from A's slice if B is in context.
  Detection: knowledge_id cross-reference
```

---

## Deduplication Priority

```
Priority order (higher priority keeps the content, lower priority yields):
  1. PRIMARY object — always wins
  2. DEPENDENCY role slots
  3. COMPARISON role slots
  4. RELATED role slots
  5. BACKGROUND role slots

Within same priority: slot with higher relevance_score wins.
```

---

## Deduplication Algorithm

```
DEDUPLICATE(slices) → list[DeduplicatedSlice]:

  // Pass 1: Exact object duplicates (TYPE-2)
  seen_ids = {}
  for s in SORTED_BY_PRIORITY(slices):
    if s.knowledge_id in seen_ids:
      MARK_OMIT(s)
    else:
      seen_ids[s.knowledge_id] = s

  // Pass 2: Exact field duplicates (TYPE-1)
  field_registry = {}  // hash → (knowledge_id, field_path)
  for s in SORTED_BY_PRIORITY(slices):
    if s.status == OMIT: continue
    for field_path, value in FLATTEN_FIELDS(s.fields):
      h = sha256(json_serialize(value))
      if h in field_registry:
        // Field already present in a higher-priority slot — remove here
        REMOVE_FIELD(s, field_path)
        s.dedup_log.append(f"Field {field_path} deduplicated from {field_registry[h]}")
      else:
        field_registry[h] = (s.knowledge_id, field_path)

  // Pass 3: Near-duplicate constraints (TYPE-3)
  all_constraints = COLLECT_ALL_CONSTRAINTS(slices)
  for group in CLUSTER_SIMILAR(all_constraints, threshold=0.90):
    canonical = group[0]  // highest priority object's constraint wins
    for duplicate in group[1:]:
      REPLACE_WITH_MARKER(duplicate.slice, duplicate.constraint,
                          f"[ref: {canonical.knowledge_id}.constraints]")

  // Pass 4: Redundant relationship removal (TYPE-4)
  in_context_ids = {s.knowledge_id for s in slices if s.status != OMIT}
  for s in slices:
    if s.status == OMIT: continue
    REMOVE_RELATIONSHIP_ENTRIES(s.fields, lambda r: r.target_id in in_context_ids)

  // Recompute token counts
  for s in slices:
    s.token_count = ESTIMATE_TOKENS(s.fields)

  return [s for s in slices if s.status != OMIT]
```

---

## DeduplicatedSlice Schema

```yaml
DeduplicatedSlice:
  # All fields from FieldLevelSlice or DeltaSlice
  knowledge_id: string
  fields: dict[str, Any]       # post-deduplication content
  token_count: int

  # Deduplication audit
  dedup_type: CLEAN | TYPE1 | TYPE2 | TYPE3 | TYPE4 | MULTIPLE
  tokens_removed: int
  dedup_log: list[string]      # one line per dedup action for explainability
```

---

## Dedup Log (Explainability)

Every deduplication action produces a log entry. This is the mechanism for
UICE's "100% explainability" requirement: every token that is NOT in the pack
must have a log entry explaining why it was removed.

```
Format: "[DEDUP-{TYPE}] {field_path} removed from {object_id} — duplicate of {owner_id}.{field_path}"

Examples:
  "[DEDUP-1] genome.patterns removed from KNW-SVC-002 — duplicate of KNW-SVC-001.genome.patterns"
  "[DEDUP-3] constraint 'stateless' removed from KNW-SVC-003 — near-duplicate of KNW-SVC-001.constraint[0]"
  "[DEDUP-4] requires[KNW-DB-001] removed from KNW-SVC-001 — KNW-DB-001 is a named context slot"
```

---

## Non-Deduplication: What Is Never Removed

```
Never deduplicated regardless of similarity:
  - dna.identity of any object (each identity is unique by definition)
  - knowledge_id references in delta slices (needed for reconstruction)
  - AH Guard content (guard must remain complete per CI-03)
  - Reminder headers in DeltaSlice (needed for continuity)
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-031 | TYPE-2 (exact object duplicate) must be eliminated 100% of the time — zero tolerance |
| UICE-032 | TYPE-1 (exact field duplicate) must be eliminated 100% of the time by content hash |
| UICE-033 | TYPE-3 near-duplicate threshold is Levenshtein ≥ 0.90 after normalization |
| UICE-034 | Every dedup action must produce a dedup_log entry — deduplication without audit is INVALID |
| UICE-035 | Deduplicator must not remove primary object fields; primary slot is never yielded |

---

## Cross-References

- Field-Level Assembly → `03-FIELD-LEVEL-ASSEMBLY`
- Shared Context Optimizer → `05-SHARED-CONTEXT-OPTIMIZER`
- Dynamic Context Planner → `04-DYNAMIC-CONTEXT-PLANNER`
- Context Verifier → `26-CONTEXT-VERIFIER`
- Context Profiler → `21-CONTEXT-PROFILER`
