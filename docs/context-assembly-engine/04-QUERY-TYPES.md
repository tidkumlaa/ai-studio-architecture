# CAE-DOC-004 — Query Types

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Query types classify the structural form of a query — what shape the answer
should take. Where intent describes WHAT the user wants to know, query type
describes HOW the answer is structured.

10 query types. Each drives a different assembly strategy.

---

## Query Type Definitions

### QT-01: Identity Query

*Who/what is this object at its most basic level?*

```yaml
query_type: IDENTITY
trigger: "what is", "define", "tell me about", direct ID reference
primary_answer_source:
  - intelligence.compression.mini          # first
  - intelligence.dna.identity              # supplement
  - intelligence.genome.identity
expected_output_structure:
  - id + name + type + namespace
  - purpose (one sentence)
  - primary capability
  - lifecycle state + quality score
max_object_count: 1
```

---

### QT-02: Behavioral Query

*What does this object DO — what is its behavior at runtime?*

```yaml
query_type: BEHAVIORAL
trigger: "how does X work", "what does X do", "behavior of X"
primary_answer_source:
  - intelligence.dna.core_behavior
  - intelligence.self_describing.inputs + outputs
  - intelligence.ai_context.full_summary
  - intelligence.explanation.developer.how_to_use
expected_output_structure:
  - primary_action (one sentence)
  - input contract
  - output contract
  - key invariants
  - critical constraints
max_object_count: 1
```

---

### QT-03: Relational Query

*How does X connect to other objects?*

```yaml
query_type: RELATIONAL
trigger: "what depends on", "what uses", "what implements", "connected to"
primary_answer_source:
  - intelligence.genome.relationships
  - intelligence.semantic.provides + consumes
  - intelligence.reasoning.requires + provides
expected_output_structure:
  - strong_dependencies (HARD links)
  - provides_to (dependents)
  - implements (requirements/specs)
  - relationship_count by type
max_object_count: 1 primary + up to 10 related
```

---

### QT-04: Causal Query

*WHY does X exist / WHY was this decision made?*

```yaml
query_type: CAUSAL
trigger: "why does", "reason for", "motivation", "why was X designed"
primary_answer_source:
  - intelligence.cortex.why
  - intelligence.dna.purpose
  - intelligence.evolution.origin.genesis
  - intelligence.tradeoffs (if relevant)
expected_output_structure:
  - why.statement
  - why.created_to_solve
  - why.would_happen_without
  - key decision references
max_object_count: 1 primary + referenced decisions
```

---

### QT-05: Comparative Query

*How does X differ from Y?*

```yaml
query_type: COMPARATIVE
trigger: "vs", "difference", "compare", "same as", "similar to"
primary_answer_source:
  - intelligence.diff.object_comparison
  - intelligence.alternatives (if Y is listed)
  - intelligence.cortex.why_not (if Y is a confusion target)
  - intelligence.semantic.conflicts + equivalents
expected_output_structure:
  - similarity_score
  - key differences (by dimension)
  - use-case when to choose X vs Y
  - confusion warning if applicable
max_object_count: 2 (X and Y required)
```

---

### QT-06: Impact Query

*What happens if X changes / fails / is removed?*

```yaml
query_type: IMPACT
trigger: "what breaks", "impact of", "if X fails", "remove X"
primary_answer_source:
  - intelligence.risk.failure_modes
  - intelligence.reasoning.impacts
  - intelligence.risk.migration_risk
  - intelligence.risk.criticality
expected_output_structure:
  - direct_impacts (immediate dependents)
  - transitive_impacts (downstream)
  - severity per impact
  - recovery / mitigation
max_object_count: 1 primary + full impact graph
```

---

### QT-07: Structural Query

*What is the architecture / structure / composition of X?*

```yaml
query_type: STRUCTURAL
trigger: "architecture of", "structure", "composition", "what is inside"
primary_answer_source:
  - intelligence.genome (full)
  - intelligence.self_describing
  - intelligence.semantic.capabilities + provides
  - intelligence.reasoning.enables
expected_output_structure:
  - category + pattern + abstraction_level
  - capabilities list
  - interfaces
  - key dependencies
  - position in ecosystem
max_object_count: 1 primary + pattern objects
```

---

### QT-08: Temporal Query

*How has X evolved / what is its history?*

```yaml
query_type: TEMPORAL
trigger: "history", "how has X changed", "what was X before", "evolution"
primary_answer_source:
  - intelligence.evolution (full)
  - intelligence.genome.history
  - intelligence.decisions
expected_output_structure:
  - origin (when + why created)
  - version timeline
  - key changes (MAJOR/SEMANTIC)
  - current state + next planned change
max_object_count: 1 primary + predecessors
```

---

### QT-09: Operational Query

*How do I run / monitor / operate / debug X?*

```yaml
query_type: OPERATIONAL
trigger: "how to monitor", "alert", "debug", "operate", "health check"
primary_answer_source:
  - intelligence.explanation.operator
  - intelligence.risk.failure_modes (detection)
  - intelligence.self_describing.constraints
expected_output_structure:
  - health_indicators
  - alerts
  - failure_behavior
  - recovery_steps
  - monitoring queries
max_object_count: 1 primary
```

---

### QT-10: Multi-Hop Query

*Questions requiring traversal across multiple objects.*

```yaml
query_type: MULTI_HOP
trigger: "what is the path from X to Y", "chain from", multi-step reasoning
primary_answer_source:
  - reasoning.depends (ordered traversal)
  - graph traversal via KIL Index
  - multiple objects assembled in chain order
expected_output_structure:
  - ordered chain of objects
  - each object at L3 minimum
  - relationship between each hop
  - total path confidence
max_object_count: up to 10 objects in chain
```

---

## Query Type → Answer Shape

| QT | Shape | Sections | Min Objects |
|----|-------|---------|------------|
| QT-01 Identity | Single card | id, purpose, capability, state | 1 |
| QT-02 Behavioral | Behavior spec | action, inputs, outputs, invariants | 1 |
| QT-03 Relational | Graph summary | deps, provides, implements | 1+N |
| QT-04 Causal | Narrative | why, genesis, problem solved | 1 |
| QT-05 Comparative | Side-by-side | similarity, differences, guidance | 2 |
| QT-06 Impact | Impact tree | direct, transitive, severity | 1+impact graph |
| QT-07 Structural | Architecture card | pattern, caps, interfaces, ecosystem | 1+pattern |
| QT-08 Temporal | Timeline | origin, versions, key changes | 1+history |
| QT-09 Operational | Runbook | health, alerts, recovery | 1 |
| QT-10 Multi-Hop | Chain | ordered objects with relationships | 2–10 |

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-012 | QT-05 COMPARATIVE requires both entities resolved before assembly |
| CAE-013 | QT-06 IMPACT must include `risk.failure_modes` in primary object or emit warning |
| CAE-014 | QT-10 MULTI_HOP chain length must not exceed 10 hops |
| CAE-015 | Query type is derived from intent — it is never set by the caller directly |

---

## Cross-References

- Intent model → `03-INTENT-MODEL`
- Object selector → `05-OBJECT-SELECTOR`
- Context planner → `09-CONTEXT-PLANNER`
- Pack builders → docs 11–14
