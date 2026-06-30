# CAE-DOC-012 — Human Pack

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Human Pack is the output format for human consumers: developers, architects,
and executives reading documentation, exploring the knowledge graph, or debugging
a system. It uses the audience-specific explanation blocks from KIL rather than
machine-compressed text, and is formatted for readability.

HumanPack is the default output when `consumer_profile.type in {DEVELOPER, ARCHITECT, EXECUTIVE}`.

---

## Human Pack Schema

```yaml
human_pack:
  pack_id: string
  pack_type: HUMAN
  audience: DEVELOPER | ARCHITECT | EXECUTIVE
  query_id: string
  query_text: string
  intent_type: string
  generated_at: string

  summary_card:
    knowledge_id: string
    name: string
    type: string
    namespace: string
    status: string
    quality_score: float
    ai_readiness_score: float
    confidence_level: string

  explanation:
    content: string                    # from intelligence.explanation.[audience]
    audience: string
    estimated_reading_time_sec: integer

  related_reading:
    - knowledge_id: string
      name: string
      relationship: string             # why it is related
      micro: string                    # L2 description

  also_see:
    - knowledge_id: string
      name: string
      reason: string                   # why human might want to read this

  quality_note: string | null          # if quality_score < 0.80

  pack_metadata:
    total_tokens: integer
    assembly_latency_ms: integer
```

---

## HumanPack Build Algorithm

```
BUILD_HUMAN_PACK(raw_pack, intent_record, consumer_profile):

  audience = consumer_profile.audience   # DEVELOPER | ARCHITECT | EXECUTIVE

  primary_kil = INDEX.get(raw_pack.primary.knowledge_id)

  pack = HumanPack(
    pack_id = uuid4(),
    pack_type = HUMAN,
    audience = audience,
    query_id = intent_record.query_id,
    query_text = intent_record.raw_query,
    intent_type = intent_record.intent_type,
    generated_at = now()
  )

  # Summary card (always L2 + metadata)
  pack.summary_card = {
    knowledge_id: primary_kil.knowledge_id,
    name: primary_kil.canonical_name,
    type: primary_kil.object_type,
    namespace: primary_kil.namespace,
    status: primary_kil.metadata.state,
    quality_score: primary_kil.metadata.quality_score,
    ai_readiness_score: primary_kil.ai_readiness_score,
    confidence_level: primary_kil.confidence.level
  }

  # Audience-specific explanation
  explanation_block = primary_kil.intelligence.explanation[audience]
  IF explanation_block is null:
    FALLBACK to explanation.developer
  pack.explanation = {
    content: explanation_block,
    audience: audience,
    estimated_reading_time_sec: estimate_reading_time(explanation_block)
  }

  # Related reading (from CONTEXT objects in raw pack)
  pack.related_reading = [
    {
      knowledge_id: obj.knowledge_id,
      name: INDEX.get(obj.knowledge_id).canonical_name,
      relationship: obj.role,
      micro: INDEX.get(obj.knowledge_id).micro
    }
    for obj in raw_pack.context_objects
    WHERE obj.relevance_score >= 0.50
    LIMIT 5
  ]

  # Also-see (from always_include / likely in ai_context)
  pack.also_see = [
    {
      knowledge_id: ref.knowledge_id,
      name: INDEX.get(ref.knowledge_id).canonical_name,
      reason: ref.relationship
    }
    for ref in primary_kil.ai_context.related_objects
    WHERE ref.fetch_priority in {ALWAYS, LIKELY}
    LIMIT 3
  ]

  # Quality note
  IF primary_kil.metadata.quality_score < 0.80:
    pack.quality_note = (
      f"This object has quality score {primary_kil.metadata.quality_score:.2f} "
      f"(threshold: 0.80). Some sections may be incomplete."
    )

  pack.pack_metadata = compute_metadata(raw_pack, pack)

  RETURN pack
```

---

## Audience Variants

### DEVELOPER

```
Explanation block: intelligence.explanation.developer
Focus:
  - Code examples
  - Interface signatures (inputs, outputs, error codes)
  - Integration pattern
  - Common mistakes
  - Local testing guide
Format: Markdown with code blocks
```

### ARCHITECT

```
Explanation block: intelligence.explanation.architect
Focus:
  - Design decisions and rationale
  - Architectural role in the larger system
  - Trade-offs accepted
  - Dependencies and impact surface
  - Alternatives considered
Format: Markdown with diagrams (ASCII)
```

### EXECUTIVE

```
Explanation block: intelligence.explanation.executive
Focus:
  - Business purpose
  - Risk summary (one paragraph)
  - Key metrics
  - No code
Format: Markdown, no code blocks
```

---

## Rendered Output Format

```
Knowledge Object: {name}
Type: {type} | Namespace: {namespace}
Status: {status} | Quality: {quality_score:.2f} | Confidence: {confidence_level}

──────────────────────────────────────
{explanation.content}
──────────────────────────────────────

RELATED READING
{related_reading as bulleted list: name (relationship): micro}

ALSO SEE
{also_see as bulleted list: name — reason}

{quality_note if present}
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-051 | HumanPack must use audience-specific explanation block — not compressed text |
| CAE-052 | If audience explanation block is missing, fall back to `explanation.developer` |
| CAE-053 | Summary card is always included — it provides quick-scan metadata |
| CAE-054 | Quality note must be shown when `quality_score < 0.80` — not hidden |
| CAE-055 | HumanPack does not include a guard block — it is not a reasoning context |

---

## Cross-References

- Context assembler → `10-CONTEXT-ASSEMBLER`
- KIL explanation layer → Phase 3.0D.0.6 `21-KNOWLEDGE-EXPLANATION-LAYER`
- Context validator → `16-CONTEXT-VALIDATOR`
- Operator pack → `13-OPERATOR-PACK`
