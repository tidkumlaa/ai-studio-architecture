# CAE-DOC-019 — Prefetch Engine

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Prefetch Engine reduces latency by pre-loading objects into the context
cache before they are explicitly queried. It predicts future context needs
based on usage patterns from the KIL usage model and co-occurrence data.

Prefetch is read-only and best-effort — it never blocks assembly.

---

## Prefetch Invariant

```
Prefetch NEVER delays or blocks the current assembly request.
If a prefetch operation would increase S1–S9 pipeline latency,
it must be deferred to a background task.
```

---

## Prefetch Trigger Conditions

```
TRIGGER_PREFETCH(assembled_pack, query_context):

  triggers = []

  FOR obj in assembled_pack.all_objects:
    kil = INDEX.get(obj.knowledge_id)

    # Trigger 1: always_include references not yet in cache
    FOR ref in kil.ai_context.related_objects WHERE priority == ALWAYS:
      IF NOT CACHE.contains(ref.knowledge_id):
        triggers.append(PrefetchTask(
          knowledge_id = ref.knowledge_id,
          reason = ALWAYS_INCLUDE,
          priority = HIGH
        ))

    # Trigger 2: likely references with AIRS >= 0.70
    FOR ref in kil.ai_context.related_objects WHERE priority == LIKELY:
      candidate = INDEX.get(ref.knowledge_id)
      IF candidate and candidate.ai_readiness_score >= 0.70:
        IF NOT CACHE.contains(ref.knowledge_id):
          triggers.append(PrefetchTask(
            knowledge_id = ref.knowledge_id,
            reason = LIKELY_INCLUDE,
            priority = MEDIUM
          ))

    # Trigger 3: top co-occurrence objects
    FOR entry in kil.usage.co_occurrence_matrix[:5]:
      IF entry.co_access_rate >= 0.30:
        IF NOT CACHE.contains(entry.knowledge_id):
          triggers.append(PrefetchTask(
            knowledge_id = entry.knowledge_id,
            reason = CO_OCCURRENCE,
            priority = LOW
          ))

  RETURN deduplicate(triggers)
```

---

## Prefetch Task Schema

```yaml
prefetch_task:
  task_id: string
  knowledge_id: string
  reason: ALWAYS_INCLUDE | LIKELY_INCLUDE | CO_OCCURRENCE | PREEMPTIVE
  priority: HIGH | MEDIUM | LOW
  requested_at: string
  trigger_pack_id: string
  compression_levels_to_warm: [string]   # which levels to pre-render into cache
```

---

## Prefetch Priority Queue

Tasks are queued and executed in priority order:

```
Priority order: HIGH → MEDIUM → LOW

HIGH tasks:
  Execute within 500ms of pack emission
  Load: all compression levels (L1–L4)

MEDIUM tasks:
  Execute within 2s of pack emission
  Load: L1, L2, L3

LOW tasks:
  Execute within 10s of pack emission
  Load: L1, L2
```

---

## What Gets Pre-Rendered

For each prefetched object:

```
PREFETCH_RENDER(knowledge_id, levels):

  entry = INDEX.get(knowledge_id)
  IF entry is null: SKIP  # object may have been deleted

  FOR level in levels:
    rendered = RENDER(entry, level, query_type=GENERIC)
    CACHE.put(
      key = cache_key(knowledge_id, level),
      value = rendered,
      ttl = CACHE_TTL.compute(entry)
    )
```

---

## Preemptive Prefetch (Query Pattern-Based)

In addition to triggered prefetch, the prefetch engine can predict future queries:

```
PREEMPTIVE_PREFETCH(recent_queries):

  # If the same object has been queried N times in M minutes,
  # pre-fetch its entire context neighborhood
  hot_objects = [
    obj for obj in recent_queries
    WHERE count(last 5 minutes) >= 3
  ]

  FOR obj in hot_objects:
    kil = INDEX.get(obj.knowledge_id)
    FOR dep in kil.reasoning.requires:
      IF NOT CACHE.contains(dep.knowledge_id):
        QUEUE_PREFETCH(dep.knowledge_id, priority=MEDIUM, reason=PREEMPTIVE)
```

---

## Prefetch Metrics

| Metric | Target |
|--------|--------|
| Prefetch hit rate | ≥ 40% of assembly lookups served from cache |
| Prefetch latency impact | < 5ms added to emitting pack |
| Stale prefetch eviction | Evict when TTL expires or object updated |
| Prefetch queue depth | Alert if queue depth > 500 tasks |

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-086 | Prefetch must never add latency to the current assembly request |
| CAE-087 | Prefetch tasks must not exceed LOW priority for CO_OCCURRENCE triggers |
| CAE-088 | Prefetch must skip objects that are DEPRECATED or DRAFT |
| CAE-089 | Prefetch render must use GENERIC query type — not the triggering query's type |
| CAE-090 | Prefetch queue must be bounded — if full, drop LOW priority tasks first |

---

## Cross-References

- Context cache → `20-CONTEXT-CACHE`
- Object selector → `05-OBJECT-SELECTOR`
- KIL usage model → Phase 3.0D.0.6 `10-KNOWLEDGE-USAGE-MODEL`
- Performance model → `21-PERFORMANCE-MODEL`
