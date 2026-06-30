# UICE-DOC-006 — Context Delta Generator

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Delta Generator produces **DeltaSlice** — a context representation
containing only the fields that have changed since an object was last sent in
this conversation. When an AI agent or developer has already received full context
for an object, resending everything wastes tokens. Delta mode sends only the diff.

This module enables the "Repeated Conversation Savings > 90%" success target:
after the first context pack, subsequent queries about the same objects
send 5–10 tokens (version + changed field list) instead of 500.

---

## DeltaSlice Schema

```yaml
DeltaSlice:
  knowledge_id: string
  delta_type: VERSION_CHANGE | FIELD_UPDATE | REMINDER | NO_CHANGE

  # Delta content (only populated for VERSION_CHANGE and FIELD_UPDATE)
  changed_fields: dict[str, Any]      # only fields that differ from last sent
  removed_fields: list[str]           # fields present before, absent now
  added_fields: dict[str, Any]        # fields not present in last sent version

  # Reference to what was previously sent
  previous_version: string
  previous_sent_at: datetime
  current_version: string

  # Minimal reminder header (always included)
  reminder:
    knowledge_id: string
    name: string
    current_version: string
    change_summary: string            # 1-line: "Updated: core_behavior, constraints"

  token_count: int
  delta_mode: bool                    # always True for DeltaSlice
```

---

## Delta Types

```
VERSION_CHANGE:
  The object's version changed since last send.
  Send: reminder header + full diff of changed/added/removed fields.
  Typical tokens: 30–80 (vs 500 for full FLA)

FIELD_UPDATE:
  Same version but fields were selectively updated (e.g., AIRS score refreshed).
  Send: reminder header + updated fields only.
  Typical tokens: 15–40

REMINDER:
  Object unchanged and in memory, but consumer_urgency == PRODUCTION.
  Send: reminder header only (identity + version + name).
  Typical tokens: 8–15

NO_CHANGE:
  Object unchanged, in memory, urgency != PRODUCTION.
  Send: nothing (OMIT from pack).
  Typical tokens: 0
```

---

## Delta Computation Algorithm

```
GENERATE_DELTA(knowledge_id, current_kil_object, memory_entry) → DeltaSlice:

  // Retrieve last-sent fields from Conversation Memory
  last_sent   = memory_entry.fields_sent
  last_version = memory_entry.version_sent
  current_version = current_kil_object.version

  // Type determination
  if last_version != current_version:
    delta_type = VERSION_CHANGE
  elif memory_entry.field_hashes_changed(current_kil_object):
    delta_type = FIELD_UPDATE
  elif consumer_urgency == PRODUCTION:
    delta_type = REMINDER
  else:
    delta_type = NO_CHANGE

  if delta_type == NO_CHANGE:
    return DeltaSlice(delta_type=NO_CHANGE, token_count=0, ...)

  if delta_type == REMINDER:
    reminder = BUILD_REMINDER(current_kil_object)
    return DeltaSlice(delta_type=REMINDER, reminder=reminder,
                      token_count=ESTIMATE_TOKENS(reminder))

  // Compute field-level diff
  current_fields = EXTRACT_FLA_FIELDS(current_kil_object, intent_type, query_type)
  changed   = {k: v for k, v in current_fields.items() if last_sent.get(k) != v}
  removed   = [k for k in last_sent if k not in current_fields]
  added     = {k: v for k, v in current_fields.items() if k not in last_sent}

  change_summary = BUILD_CHANGE_SUMMARY(changed, removed, added)

  reminder = BUILD_REMINDER(current_kil_object)
  reminder.change_summary = change_summary

  return DeltaSlice(
    knowledge_id    = knowledge_id,
    delta_type      = delta_type,
    changed_fields  = changed,
    removed_fields  = removed,
    added_fields    = added,
    previous_version = last_version,
    previous_sent_at = memory_entry.last_sent_at,
    current_version  = current_version,
    reminder        = reminder,
    token_count     = ESTIMATE_TOKENS({reminder, changed, added}),
    delta_mode      = True,
  )
```

---

## Conversation Savings Model

```
Session query 1 (cold start):
  Primary object → FLA mode → 80 tokens

Session query 2 (same object, unchanged):
  Primary object → NO_CHANGE → 0 tokens
  Saving: 100%

Session query 3 (same object, version changed):
  Primary object → VERSION_CHANGE delta → 35 tokens
  Saving: 56% vs full FLA re-send

Session queries 4–N (same object, no change):
  Primary object → NO_CHANGE or REMINDER → 0–15 tokens
  Saving: 81–100%

Aggregate saving across typical 10-query session: > 90%
```

---

## Field Hash Tracking

Conversation Memory stores a field_hash per sent object:

```
field_hash = sha256(json_serialize(sent_fields, sort_keys=True))
```

On subsequent queries, the same serialization is compared. If the hash differs,
FIELD_UPDATE delta is triggered even if version is unchanged.

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-026 | DeltaSlice requires a valid Conversation Memory entry (UI-02); absent entry → FLA mode |
| UICE-027 | NO_CHANGE delta produces 0 tokens; the object must still appear in pack metadata |
| UICE-028 | Every DeltaSlice must include the reminder header; bare field diffs without context are INVALID |
| UICE-029 | Field hash comparison uses canonical JSON serialization (sorted keys, no whitespace) |
| UICE-030 | Delta type priority: VERSION_CHANGE > FIELD_UPDATE > REMINDER > NO_CHANGE |

---

## Cross-References

- Conversation Memory → `16-CONVERSATION-MEMORY`
- Dynamic Context Planner → `04-DYNAMIC-CONTEXT-PLANNER`
- Context Deduplicator → `07-CONTEXT-DEDUPLICATOR`
- Field-Level Assembly → `03-FIELD-LEVEL-ASSEMBLY`
