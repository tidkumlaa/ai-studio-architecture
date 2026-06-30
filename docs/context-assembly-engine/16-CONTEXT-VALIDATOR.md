# CAE-DOC-016 — Context Validator

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Context Validator is Stage S9 — the final stage of the CAE pipeline. It
validates the guarded context pack before emission. If the pack does not pass
all checks, it is rejected with a structured AssemblyError. No invalid pack
is emitted.

The validator is the last defense before the context leaves the CAE.

---

## Validator Invariant (CI-04)

```
Validator must pass before any pack is emitted.
A pack that fails validation must raise AssemblyError — not be emitted with warnings.
```

---

## Validation Checks

### V-GROUP-1: Structural Completeness

| Check | Code | Rule |
|-------|------|------|
| Pack has pack_id | V-01 | UUID format, non-null |
| Pack has pack_type | V-02 | One of: PROMPT, HUMAN, OPERATOR, SEARCH |
| Pack has query_id | V-03 | Non-empty string |
| Pack has generated_at | V-04 | Valid ISO 8601 |
| PromptPack has primary section | V-05 | primary.knowledge_id must be present |
| PromptPack has guard_block | V-06 | guard_block.scope_boundaries must be non-empty |
| SearchPack has results list | V-07 | results may be empty but key must exist |

### V-GROUP-2: Budget Compliance

| Check | Code | Rule |
|-------|------|------|
| Total tokens ≤ declared budget | V-08 | pack_metadata.total_tokens ≤ budget_declared |
| Guard block within guard budget | V-09 | guard tokens ≤ guard_budget |
| Primary at minimum L3 | V-10 | primary.compression_level in {L3, L4, L5} |

### V-GROUP-3: Content Safety

| Check | Code | Rule |
|-------|------|------|
| No AIRS < 0.60 in PromptPack | V-11 | All objects must have AIRS ≥ 0.60 |
| No DEPRECATED primary in PromptPack without warning | V-12 | Status warning must be set |
| Guard block present in PromptPack | V-13 | Cannot be null |
| Confidence hedge present | V-14 | guard_block.confidence_hedge non-empty |

### V-GROUP-4: Context Quality Score

| Check | Code | Rule |
|-------|------|------|
| Context quality score ≥ 0.70 | V-15 | CAE minimum to emit |

Context quality score formula (see `17-CONTEXT-QUALITY-MODEL` for full spec):

```
context_quality_score =
  relevance   × 0.35 +
  completeness × 0.30 +
  safety      × 0.25 +
  efficiency  × 0.10
```

### V-GROUP-5: Pack Uniqueness

| Check | Code | Rule |
|-------|------|------|
| pack_id is unique | V-16 | Not present in recent emission log (last 1h) |
| No duplicate object in pack | V-17 | Same knowledge_id may not appear twice in one pack |

### V-GROUP-6: Format Correctness

| Check | Code | Rule |
|-------|------|------|
| All content is valid UTF-8 | V-18 | No binary or invalid sequences |
| L5 machine content is valid JSON | V-19 | If used — parse must succeed |
| relevance_score in [0.0, 1.0] | V-20 | All relevance scores must be in range |

---

## Validator Algorithm

```
VALIDATE(guarded_pack, budget_declared) → ValidationResult:

  errors = []
  warnings = []

  # GROUP 1: Structural
  IF not is_valid_uuid(guarded_pack.pack_id):
    errors.append(ValidationError(V-01, "pack_id is not a valid UUID"))
  IF guarded_pack.pack_type not in VALID_PACK_TYPES:
    errors.append(ValidationError(V-02, f"invalid pack_type: {guarded_pack.pack_type}"))
  IF not guarded_pack.query_id:
    errors.append(ValidationError(V-03, "query_id is empty"))
  IF not is_iso8601(guarded_pack.generated_at):
    errors.append(ValidationError(V-04, "generated_at is not ISO 8601"))

  IF guarded_pack.pack_type == PROMPT:
    IF not guarded_pack.primary:
      errors.append(ValidationError(V-05, "PromptPack missing primary object"))
    IF not guarded_pack.guard_block or not guarded_pack.guard_block.scope_boundaries:
      errors.append(ValidationError(V-06, "PromptPack guard_block missing or empty"))

  IF guarded_pack.pack_type == SEARCH:
    IF guarded_pack.results is None:
      errors.append(ValidationError(V-07, "SearchPack results key missing"))

  # GROUP 2: Budget
  IF guarded_pack.pack_metadata.total_tokens > budget_declared:
    errors.append(ValidationError(V-08,
      f"Token overrun: {guarded_pack.pack_metadata.total_tokens} > {budget_declared}"))
  IF guarded_pack.pack_type == PROMPT:
    IF guarded_pack.primary.compression_level not in {L3, L4, L5}:
      errors.append(ValidationError(V-10, "Primary object below minimum compression L3"))

  # GROUP 3: Content safety
  IF guarded_pack.pack_type == PROMPT:
    for obj in guarded_pack.all_objects:
      IF obj.ai_readiness_score < 0.60:
        errors.append(ValidationError(V-11,
          f"{obj.knowledge_id} AIRS={obj.ai_readiness_score:.2f} < 0.60"))
    IF not guarded_pack.guard_block:
      errors.append(ValidationError(V-13, "PromptPack missing guard block"))
    IF not guarded_pack.guard_block.confidence_hedge:
      errors.append(ValidationError(V-14, "PromptPack missing confidence hedge"))

  # GROUP 4: Quality score
  quality_score = compute_context_quality_score(guarded_pack)
  IF quality_score < 0.70:
    errors.append(ValidationError(V-15,
      f"Context quality {quality_score:.2f} below minimum 0.70"))

  # GROUP 5: Uniqueness
  IF guarded_pack.pack_id in RECENT_PACK_LOG:
    errors.append(ValidationError(V-16, f"Duplicate pack_id: {guarded_pack.pack_id}"))
  ids_seen = {}
  for obj in guarded_pack.all_objects:
    IF obj.knowledge_id in ids_seen:
      errors.append(ValidationError(V-17, f"Duplicate object: {obj.knowledge_id}"))
    ids_seen[obj.knowledge_id] = true

  # GROUP 6: Format
  for obj in guarded_pack.all_objects:
    IF not is_valid_utf8(obj.content):
      errors.append(ValidationError(V-18, f"{obj.knowledge_id} content is not UTF-8"))
    IF obj.compression_level == L5 and not is_valid_json(obj.content):
      errors.append(ValidationError(V-19, f"{obj.knowledge_id} L5 content is not valid JSON"))
    IF not (0.0 <= obj.relevance_score <= 1.0):
      errors.append(ValidationError(V-20, f"{obj.knowledge_id} relevance_score out of range"))

  IF errors:
    RAISE AssemblyError(VALIDATION_FAILED, errors=errors)

  RETURN ValidationResult(
    passed=true,
    quality_score=quality_score,
    warnings=warnings
  )
```

---

## Validation Result Schema

```yaml
validation_result:
  passed: boolean
  quality_score: float
  checks_run: integer
  checks_passed: integer
  errors: [ValidationError]
  warnings: [string]

validation_error:
  code: string        # V-01 through V-20
  message: string
  field: string | null
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-071 | Validator must run 20 checks — no check may be skipped |
| CAE-072 | Any error causes AssemblyError — warning-only packs may still emit |
| CAE-073 | V-15 (quality score) must use the formula from `17-CONTEXT-QUALITY-MODEL` |
| CAE-074 | V-16 (pack uniqueness) check window is 1 hour rolling |
| CAE-075 | Validation result must be logged regardless of pass/fail |

---

## Cross-References

- AH Guard → `15-ANTI-HALLUCINATION-GUARD`
- Context quality model → `17-CONTEXT-QUALITY-MODEL`
- Context metrics → `23-CONTEXT-METRICS`
- CAE API → `25-CAE-API` (AssemblyError schema)
