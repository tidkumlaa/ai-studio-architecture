# CAE-DOC-013 — Operator Pack

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Operator Pack is the output format for SRE and operations engineers responding
to incidents, configuring deployments, or auditing runtime state. It emphasizes
operational parameters, failure modes, SLOs, and runbooks — not design intent.

OperatorPack is the default output when `consumer_profile.type == OPERATOR`.

---

## Operator Pack Schema

```yaml
operator_pack:
  pack_id: string
  pack_type: OPERATOR
  query_id: string
  query_text: string
  intent_type: string
  generated_at: string

  object_header:
    knowledge_id: string
    name: string
    type: string
    namespace: string
    status: string                     # ACTIVE | DEPRECATED | EXPERIMENTAL

  operational_summary:
    content: string                    # from intelligence.explanation.operator
    config_parameters: [string]        # key config knobs
    slo_targets: [string]              # SLO targets if present
    runbook_refs: [string]             # links or references

  failure_modes:
    - name: string
      severity: CRITICAL | HIGH | MEDIUM | LOW
      trigger_condition: string
      detection: string
      mitigation: string
      recovery_time: string | null

  config_view:
    required_params: [string]
    optional_params: [string]
    constraints: [string]              # from dna.fundamental_constraints + executable.constraint_rules

  lifecycle_info:
    current_state: string
    valid_transitions: [string]
    transition_guards: [string]

  related_operators:
    - knowledge_id: string
      name: string
      relationship: string             # depends_on | used_by | configures | configures_me

  quality_note: string | null

  pack_metadata:
    total_tokens: integer
    assembly_latency_ms: integer
```

---

## OperatorPack Build Algorithm

```
BUILD_OPERATOR_PACK(raw_pack, intent_record, consumer_profile):

  primary_kil = INDEX.get(raw_pack.primary.knowledge_id)

  pack = OperatorPack(
    pack_id = uuid4(),
    pack_type = OPERATOR,
    query_id = intent_record.query_id,
    query_text = intent_record.raw_query,
    intent_type = intent_record.intent_type,
    generated_at = now()
  )

  # Header
  pack.object_header = {
    knowledge_id: primary_kil.knowledge_id,
    name: primary_kil.canonical_name,
    type: primary_kil.object_type,
    namespace: primary_kil.namespace,
    status: primary_kil.metadata.state
  }

  # Operational summary — from explanation.operator
  op_explanation = primary_kil.intelligence.explanation.operator
  pack.operational_summary = {
    content: op_explanation.content,
    config_parameters: op_explanation.configuration.parameters,
    slo_targets: op_explanation.slo_targets,
    runbook_refs: op_explanation.runbook_refs
  }

  # Failure modes — from risk.failure_modes
  pack.failure_modes = [
    {
      name: fm.name,
      severity: fm.severity,
      trigger_condition: fm.trigger_condition,
      detection: fm.detection,
      mitigation: fm.mitigation,
      recovery_time: fm.recovery_time
    }
    for fm in primary_kil.risk.failure_modes
    ORDER BY severity_rank(fm.severity) DESC
  ]

  # Configuration view
  pack.config_view = {
    required_params: [p for p in primary_kil.self_describing.inputs
                      WHERE p.required == true],
    optional_params: [p for p in primary_kil.self_describing.inputs
                      WHERE p.required == false],
    constraints: primary_kil.dna.fundamental_constraints
                 + primary_kil.executable.constraint_rules
  }

  # Lifecycle
  pack.lifecycle_info = {
    current_state: primary_kil.metadata.state,
    valid_transitions: primary_kil.executable.lifecycle_rules.transitions,
    transition_guards: primary_kil.executable.transition_guards
  }

  # Related operators (from context objects in raw pack)
  pack.related_operators = [
    {
      knowledge_id: obj.knowledge_id,
      name: INDEX.get(obj.knowledge_id).canonical_name,
      relationship: obj.role
    }
    for obj in raw_pack.context_objects
    LIMIT 5
  ]

  # Quality note
  IF primary_kil.metadata.quality_score < 0.80:
    pack.quality_note = (
      f"Quality score: {primary_kil.metadata.quality_score:.2f}. "
      f"Operational data may be incomplete."
    )

  pack.pack_metadata = compute_metadata(raw_pack, pack)

  RETURN pack
```

---

## Severity Rank

```
severity_rank(severity):
  CRITICAL → 4
  HIGH     → 3
  MEDIUM   → 2
  LOW      → 1
```

---

## Rendered Output Format

```
OPERATOR VIEW: {name} [{knowledge_id}]
Type: {type} | Namespace: {namespace} | Status: {status}

OPERATIONAL SUMMARY
{operational_summary.content}

Config parameters: {config_parameters}
SLO targets: {slo_targets}
Runbooks: {runbook_refs}

FAILURE MODES (by severity)
[CRITICAL] {failure_modes[0].name}
  Trigger: {trigger_condition}
  Detect:  {detection}
  Fix:     {mitigation}

[HIGH] ...

CONFIGURATION
Required: {required_params}
Optional: {optional_params}
Constraints:
  - {constraints}

LIFECYCLE
Current state: {current_state}
Valid transitions: {valid_transitions}
Guards: {transition_guards}

RELATED OBJECTS
{related_operators as list}

{quality_note if present}
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-056 | OperatorPack must always include failure modes section — even if empty list |
| CAE-057 | Failure modes must be ordered by severity (CRITICAL first) |
| CAE-058 | OperatorPack must not include design rationale or cortex.why — operators need what, not why |
| CAE-059 | OperatorPack does not include a guard block — it is operational, not reasoning context |
| CAE-060 | If `explanation.operator` is missing, OperatorPack must include a quality note |

---

## Cross-References

- Context assembler → `10-CONTEXT-ASSEMBLER`
- KIL executable schema → Phase 3.0D.0.6 `02-EXECUTABLE-KNOWLEDGE`
- KIL risk model → Phase 3.0D.0.6 `15-KNOWLEDGE-RISK-MODEL`
- KIL explanation layer → Phase 3.0D.0.6 `21-KNOWLEDGE-EXPLANATION-LAYER`
