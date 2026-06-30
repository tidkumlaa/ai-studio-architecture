# CAE-DOC-010 — Context Assembler

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Assembler executes the assembly plan produced by the Context Planner.
It fetches compressed representations from the KIL index, renders them to strings
or structured objects, and builds the raw ContextPack before the AH Guard and
Validator run.

---

## Assembly Algorithm

```
ASSEMBLE(assembly_plan) → RawContextPack:

  pack = RawContextPack(
    query_id=plan.query_id,
    pack_type=plan.pack_type,
    assembled_at=now()
  )

  token_count = 0

  FOR section in assembly_plan.sections:
    section_content = []

    FOR obj_spec in section.objects:

      # Fetch from KIL index
      index_entry = KIL_INDEX.get(obj_spec.knowledge_id)
      IF index_entry is null:
        LOG warning: "Object {obj_spec.knowledge_id} not in index"
        CONTINUE

      # Render to target compression level
      rendered = RENDER(index_entry, obj_spec.compression_level,
                        plan.query_type, plan.pack_type)

      # Token count check
      tokens = COUNT_TOKENS(rendered)
      IF token_count + tokens > assembly_plan.total_estimated_tokens × 1.05:
        # 5% tolerance before hard failure
        LOG warning: "Token overage: skipping {obj_spec.knowledge_id}"
        CONTINUE

      token_count += tokens
      section_content.append(RenderedObject(
        knowledge_id=obj_spec.knowledge_id,
        role=obj_spec.role,
        compression_level=obj_spec.compression_level,
        content=rendered,
        token_count=tokens
      ))

    pack.sections.append(AssembledSection(
      section_type=section.section_type,
      objects=section_content
    ))

  pack.raw_token_count = token_count
  RETURN pack
```

---

## RENDER Function

```
RENDER(index_entry, level, query_type, pack_type):

  # For AI agent packs (PromptPack)
  IF pack_type == PROMPT:
    CASE L1: return index_entry.nano
    CASE L2: return index_entry.micro
    CASE L3: return index_entry.mini    # string from KIL compression.mini
    CASE L4: return RENDER_STANDARD(index_entry, query_type)
    CASE L5: return RENDER_FULL(index_entry, query_type)

  # For human packs
  IF pack_type == HUMAN:
    audience = plan.audience    # DEVELOPER | ARCHITECT | EXECUTIVE | OPERATOR
    CASE AUDIENCE:
      return index_entry.object.intelligence.explanation[audience]
    CASE L4: return RENDER_STANDARD(index_entry, query_type)
    CASE L3: return index_entry.mini

  # For operator packs
  IF pack_type == OPERATOR:
    return index_entry.object.intelligence.explanation.operator

  # For search packs
  IF pack_type == SEARCH:
    return RENDER_SEARCH_RESULT(index_entry)


RENDER_STANDARD(index_entry, query_type):
  # Start with standard compression block
  base = index_entry.standard           # from KIL compression.standard
  augments = QUERY_AUGMENTATIONS[query_type]

  extra_blocks = []
  FOR field_path in augments:
    value = GET_FIELD(index_entry.object, field_path)
    IF value is not null:
      extra_blocks.append(format(field_path, value))

  return merge(base, extra_blocks)


RENDER_SEARCH_RESULT(index_entry):
  return {
    "knowledge_id": index_entry.knowledge_id,
    "name": index_entry.canonical_name.split(".")[-1],
    "type": index_entry.object_type,
    "namespace": index_entry.namespace,
    "summary": index_entry.micro,
    "quality_score": index_entry.quality_score,
    "ai_readiness_score": index_entry.ai_readiness_score
  }
```

---

## Preamble Generation

The preamble is generated, not fetched from objects:

```
GENERATE_PREAMBLE(intent_record, primary_object):

  CASE FACTUAL_LOOKUP:
    return f"About {primary_object.nano}:"

  CASE REASONING_QUERY:
    return (
      f"Reasoning context for: '{intent_record.raw_query}'\n"
      f"Primary object: {primary_object.nano}\n"
      f"Intent: Explain WHY this object exists."
    )

  CASE IMPACT_ANALYSIS:
    return (
      f"Impact analysis for: {primary_object.nano}\n"
      f"Question: What are the consequences of changing or removing this object?"
    )

  CASE COMPARISON:
    p1, p2 = entities
    return f"Comparison: {p1.nano} vs {p2.nano}"

  CASE CONTEXT_PRODUCTION:
    return (
      primary_object.intelligence.thinking
        .chain_of_thought_template.preamble
    )

  DEFAULT:
    return f"Context for: '{intent_record.raw_query}' [{intent_record.intent_type}]"
```

---

## Raw Context Pack Schema

```yaml
raw_context_pack:
  query_id: string
  pack_type: string
  assembled_at: string
  assembly_version: "1.0"

  preamble: string

  sections:
    - section_type: string
      objects:
        - knowledge_id: string
          role: string
          compression_level: string
          content: string | object
          token_count: integer

  raw_token_count: integer
  assembly_latency_ms: integer        # time taken by assembler stage
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-041 | Objects missing from KIL index are skipped with warning — assembly continues |
| CAE-042 | Token overage of > 5% over estimated triggers object removal, not truncation |
| CAE-043 | Preamble is always generated — it is never read from an object's fields |
| CAE-044 | RENDER must produce valid UTF-8 string or structured dict — never bytes |
| CAE-045 | Raw context pack is immutable after assembly — AH Guard creates a new pack |

---

## Cross-References

- Context planner → `09-CONTEXT-PLANNER`
- Anti-hallucination guard → `15-ANTI-HALLUCINATION-GUARD`
- Context validator → `16-CONTEXT-VALIDATOR`
- Output packs → docs 11–14
