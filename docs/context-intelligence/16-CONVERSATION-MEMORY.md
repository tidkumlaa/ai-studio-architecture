# UICE-DOC-016 — Conversation Memory

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Conversation Memory gives UICE **history-awareness**: within a session, it tracks
which KnowledgeObjects have already been sent, what fields were included, and
at what version. This enables Delta mode (module 06) and OMIT mode (module 12),
which together achieve the "Repeated Conversation Savings > 90%" target.

Without Conversation Memory, every query re-sends the full context for every
object. With Memory active, a 10-query session about the same 3 objects sends
full context once and deltas thereafter — 90%+ token savings on queries 2–10.

---

## MemoryEntry Schema

```yaml
MemoryEntry:
  knowledge_id: string
  session_id: string

  # What was sent
  version_sent: string                    # object version at time of send
  fields_sent: dict[str, Any]             # exact fields included in last pack
  field_hash: string                      # sha256(json_serialize(fields_sent))
  intent_type_sent: IntentType            # intent that triggered the send
  query_type_sent: QueryType

  # Tracking state
  send_count: int                         # how many times sent in this session
  first_sent_at: datetime
  last_sent_at: datetime
  last_pack_id: string                    # link to the pack that sent it

  # Status flags
  already_explained: bool                 # full context has been sent ≥ 1 time
  already_verified: bool                  # consumer confirmed understanding
  version_current: bool                   # True if stored version == current version

  # Delta readiness
  delta_type: VERSION_CHANGE | FIELD_UPDATE | REMINDER | NO_CHANGE
```

---

## MemorySession Schema

```yaml
MemorySession:
  session_id: string
  created_at: datetime
  last_activity: datetime
  consumer_type: AI_AGENT | HUMAN_DEV | OPERATOR | SEARCH_UI
  consumer_id: string | null

  entries: dict[str, MemoryEntry]         # keyed by knowledge_id
  entry_count: int                        # current count
  max_capacity: int                       # = 10,000 (UI-07)

  # Aggregate tracking
  objects_explained: int                  # count of already_explained == True
  total_tokens_sent: int                  # cumulative across session
  token_savings_delta: int                # tokens saved via delta/omit mode
```

---

## Memory Operations

```
MEMORY_RECORD(session_id, pack, intent_profile) → void:
  session = GET_OR_CREATE_SESSION(session_id)

  for object_id in EXTRACT_OBJECT_IDS(pack):
    fields = EXTRACT_FIELDS_SENT(pack, object_id)
    field_hash = sha256(json_serialize(fields, sort_keys=True))
    version = GET_OBJECT_VERSION(object_id)

    existing = session.entries.get(object_id)
    if existing:
      // Update existing entry
      existing.version_sent     = version
      existing.fields_sent      = fields
      existing.field_hash       = field_hash
      existing.intent_type_sent = intent_profile.intent_type
      existing.query_type_sent  = query_profile.query_type
      existing.send_count      += 1
      existing.last_sent_at     = NOW()
      existing.already_explained = True
      existing.version_current  = True
    else:
      // Create new entry
      if session.entry_count >= session.max_capacity:
        EVICT_OLDEST_ENTRY(session)
      session.entries[object_id] = MemoryEntry(
        knowledge_id      = object_id,
        session_id        = session_id,
        version_sent      = version,
        fields_sent       = fields,
        field_hash        = field_hash,
        intent_type_sent  = intent_profile.intent_type,
        send_count        = 1,
        first_sent_at     = NOW(),
        last_sent_at      = NOW(),
        already_explained = True,
        version_current   = True,
      )
    session.entry_count = len(session.entries)
    session.objects_explained = sum(1 for e in session.entries.values() if e.already_explained)


MEMORY_FILTER(session_id, object_ids) → MemoryFilter:
  session = GET_SESSION(session_id)
  if session is None:
    return MemoryFilter(active=False)  // memory not available

  filter = MemoryFilter(active=True, session_id=session_id)
  for oid in object_ids:
    entry = session.entries.get(oid)
    if entry:
      // Check if version is still current
      current_version = GET_OBJECT_VERSION(oid)
      entry.version_current = (current_version == entry.version_sent)
      entry.delta_type = COMPUTE_DELTA_TYPE(entry, current_version)
      filter.entries[oid] = entry
  return filter


COMPUTE_DELTA_TYPE(entry, current_version) → DeltaType:
  if current_version != entry.version_sent:
    return VERSION_CHANGE
  // Check field hash against live object
  current_fields = EXTRACT_FIELDS_SENT_FOR_LAST_INTENT(entry.knowledge_id,
                                                         entry.intent_type_sent,
                                                         entry.query_type_sent)
  current_hash = sha256(json_serialize(current_fields, sort_keys=True))
  if current_hash != entry.field_hash:
    return FIELD_UPDATE
  return NO_CHANGE  // nothing changed
```

---

## Session Lifecycle

```
CREATION:
  Session is created on first query with a session_id header.
  Sessions without session_id: memory is DISABLED (stateless mode).

EXPIRY:
  Sessions expire after 30 minutes of inactivity (last_activity + 30min).
  Expired sessions: entries purged from memory.

CAPACITY (UI-07):
  Maximum 10,000 entries per session.
  When capacity reached: evict oldest entry (by last_sent_at).
  Eviction is logged for observability.

PERSISTENCE:
  In-memory (non-persistent between server restarts).
  Runtime may implement persistent storage — not specified here.
```

---

## "Already Sent" Tracking Categories

```
already_explained:
  True after the object's full context (FLA or FULL mode) was sent at least once.
  Triggers: DELTA or OMIT mode on subsequent queries.

already_verified:
  True when consumer sends explicit "understood" signal.
  Triggers: OMIT even in PRODUCTION urgency (no REMINDER needed).
  Set via: consumer feedback API (not auto-detected).
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-076 | Conversation Memory is DISABLED (stateless mode) when no session_id is provided |
| UICE-077 | Delta mode requires already_explained == True; it is never applied on first send |
| UICE-078 | Maximum 10,000 entries per session (UI-07); oldest entry evicted on overflow |
| UICE-079 | field_hash uses canonical JSON serialization (sorted keys, no whitespace) — same as Delta Generator |
| UICE-080 | Session expiry is 30 minutes of inactivity; expired sessions must be cleared from memory |

---

## Cross-References

- Context Delta Generator → `06-CONTEXT-DELTA-GENERATOR`
- Context Compression Engine → `12-CONTEXT-COMPRESSION-ENGINE`
- Dynamic Context Planner → `04-DYNAMIC-CONTEXT-PLANNER`
- Context Reuse Engine → `17-CONTEXT-REUSE-ENGINE`
- UICE invariant UI-07 → `README.md`
