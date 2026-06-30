# KVF-DOC-020 — Cross-Agent Consistency

**Phase:** 3.0D.1.5
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Cross-Agent Consistency validates that multiple AI agents using the same KOS
knowledge base arrive at consistent conclusions about the same objects. If KOS
is truly a single source of truth, different agents querying it independently
should agree on all declarative facts.

---

## 4-Agent Pipeline

```
Agent A — COMPILE
  Role: Architecture understanding
  Task: Read the KOS context for an object and produce a structured
        JSON summary of its key properties.
  Output: AgentSummary {id, name, type, purpose, capabilities, constraints,
                         dependencies, confidence}

Agent B — EXECUTE
  Role: Implementation planning
  Task: Given the AgentSummary from Agent A, produce an implementation
        plan for the object.
  Output: ImplementationPlan {steps, required_inputs, expected_outputs,
                               risk_level, estimated_effort}

Agent C — REVIEW
  Role: Review / verification
  Task: Given the ImplementationPlan from Agent B, verify that it is
        consistent with the KOS context. Flag any inconsistencies.
  Output: ReviewResult {consistent: boolean, inconsistencies: [string],
                        hallucinations_detected: [string]}

Agent D — VERIFY
  Role: Ground truth comparison
  Task: Given the original KOS context and all outputs from A, B, C,
        produce a final consistency verdict.
  Output: VerificationResult {all_consistent: boolean, disagreements: [string],
                               confidence_range: {min, max}}
```

---

## Consistency Measurement

```
For each object O in the test set:
  1. All 4 agents receive the SAME context pack from KOS (same pack_id)
  2. Each agent produces its output independently
  3. Consistency checks are run:

  FACT_CONSISTENCY:
    Agent A's declared capabilities match Agent C's review of them
    Agent A's declared constraints match Agent D's ground truth

  DEPENDENCY_CONSISTENCY:
    Agent A's dependencies match Agent B's required_inputs
    Agent D finds no undeclared dependencies in Agent B's plan

  RISK_CONSISTENCY:
    Agent B's risk_level matches Agent A's risk declarations from KOS
    Agent C finds no risk omissions

  CONFIDENCE_CONSISTENCY:
    confidence_range across all agents is within ±0.15
    (all agents have similar certainty about the same facts)
```

---

## Cross-Agent Consistency Metrics

```
fact_consistency_rate:
  proportion of facts on which all 4 agents agree
  Target: >= 0.95

dependency_consistency_rate:
  proportion of dependencies named by A that match B's required_inputs
  Target: >= 0.90

risk_consistency_rate:
  proportion of runs where B's risk_level matches A's declared risk
  Target: >= 0.85

confidence_range_width:
  max(agent.confidence) - min(agent.confidence) across the 4 agents
  Target: <= 0.15 (agents should have similar certainty)

hallucination_propagation_rate:
  proportion of Agent A hallucinations that propagate to Agent B or C
  Target: < 0.10 (Agent C should catch most A hallucinations)
```

---

## Known Consistency Failure Modes

```
Failure A: Compression-induced disagreement
  Agent A gets L4 context; Agent B gets Agent A's summary (not KOS directly).
  Information loss in A's summary causes B to make wrong assumptions.
  Prevention: Agent B must query KOS independently for clarification, not rely on A.

Failure B: Confidence inflation propagation
  Agent A presents uncertain fact as certain (Type E hallucination).
  Agent B accepts it without questioning.
  Detection: Agent C must check Agent B's plan against the original context pack's
             confidence_hedge.

Failure C: Scope drift
  Agent A correctly describes object X.
  Agent B assumes X also handles behavior Y (scope hallucination).
  Detection: Agent D checks Agent B's plan against scope_boundaries in guard block.
```

---

## Consistency Test Result Schema

```yaml
cross_agent_result:
  objects_tested: integer
  pipeline_runs: integer

  consistency_rates:
    fact_consistency_rate: float
    dependency_consistency_rate: float
    risk_consistency_rate: float
    confidence_range_mean: float
    confidence_range_max: float

  failure_mode_counts:
    compression_induced: integer
    confidence_inflation: integer
    scope_drift: integer
    other: integer

  hallucination_propagation:
    propagated: integer
    caught_by_agent_c: integer
    propagation_rate: float

  agent_agreement_matrix:
    A_vs_B: float
    A_vs_C: float
    A_vs_D: float
    B_vs_C: float
    B_vs_D: float
    C_vs_D: float
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KVF-096 | All 4 agents must receive the same pack_id — separate context assemblies are NOT equivalent |
| KVF-097 | Agent C is the detection agent — hallucination_propagation_rate > 0.20 means Agent C failed |
| KVF-098 | Confidence range > 0.30 across 4 agents means KOS context is ambiguous — review needed |
| KVF-099 | Objects with fact_consistency_rate < 0.80 must be individually reviewed for KIL ambiguity |
| KVF-100 | Cross-agent test must use 4 DISTINCT LLM instances — not 4 calls to the same stateful model |

---

## Cross-References

- AI understanding → `19-AI-UNDERSTANDING`
- Hallucination measurement → `10-HALLUCINATION-MEASUREMENT`
- AH Guard → Phase 3.0D.1 `15-ANTI-HALLUCINATION-GUARD`
- Confidence propagation → Phase 3.0D.1 `18-CONFIDENCE-PROPAGATION`
