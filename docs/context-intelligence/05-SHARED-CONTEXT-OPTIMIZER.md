# UICE-DOC-005 — Shared Context Optimizer

**Phase:** 3.0D.2
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Shared Context Optimizer (SCO) extends Phase 3.0D.2's Shared Context Block
mechanism. When multiple KnowledgeObjects in a pack share common properties,
SCO extracts those properties into a single **SharedPreamble** — sent once —
and each object contributes only a **delta** of what makes it unique.

UICE extends the original SCB (namespace + tags + constraints + patterns) to
support 7 shared dimensions: namespace, package, owner, constraints, patterns,
policies, and examples. This produces greater savings for large, same-domain packs.

---

## SharedPreamble Schema

```yaml
SharedPreamble:
  preamble_id: string
  group_key: string                    # {dimension}:{value} that groups the objects
  dimension: NAMESPACE | PACKAGE | OWNER | CONSTRAINTS | PATTERNS | POLICIES | EXAMPLES

  # Shared content by dimension
  namespace: string | null             # if dimension == NAMESPACE
  package: string | null               # if dimension == PACKAGE
  owner: string | null                 # if dimension == OWNER

  shared_tags: list[str]               # tags present in ALL group objects
  shared_constraints: list[str]        # fundamental_constraints in ALL objects
  shared_patterns: list[str]           # genome.patterns in ALL objects
  shared_policies: list[str]           # governance.policies in ALL objects
  shared_examples: list[str]           # cortex.examples IDs in ALL objects

  object_ids: list[str]               # all objects in this preamble group
  token_count: int
```

---

## 7 Shared Dimensions

```
NAMESPACE    — objects in the same KOS namespace
               Key: namespace value
               Typical saving: 30–40 tokens per object in group

PACKAGE      — objects in the same software package or module
               Key: genome.package value
               Typical saving: 20–30 tokens per object

OWNER        — objects owned by the same team or organization
               Key: owner field value
               Typical saving: 15–25 tokens per object

CONSTRAINTS  — objects sharing identical fundamental_constraints
               Key: sha256(sorted constraints)
               Typical saving: 40–80 tokens per object (constraints are verbose)

PATTERNS     — objects sharing identical genome.patterns
               Key: sha256(sorted patterns)
               Typical saving: 20–40 tokens per object

POLICIES     — objects sharing identical governance policies
               Key: sha256(sorted policies)
               Typical saving: 30–60 tokens per object

EXAMPLES     — objects referencing the same shared examples in cortex
               Key: sha256(sorted example_ids)
               Typical saving: 20–50 tokens per object
```

---

## SCO Algorithm

```
OPTIMIZE_SHARED(slices, kil_objects_by_id) → (list[SharedPreamble], list[DeltaSlice]):

  // Step 1: Group objects by each dimension
  groups_by_dimension = {}
  for dim in [NAMESPACE, PACKAGE, OWNER, CONSTRAINTS, PATTERNS, POLICIES, EXAMPLES]:
    groups = GROUP_BY(slices, lambda s: GET_DIM_KEY(s, dim, kil_objects_by_id))
    groups_by_dimension[dim] = {k: v for k, v in groups.items() if len(v) >= 2}

  // Step 2: Select best grouping strategy
  // Priority: CONSTRAINTS > NAMESPACE > PACKAGE > POLICIES > PATTERNS > OWNER > EXAMPLES
  // (CONSTRAINTS saves most tokens per object)
  selected_groups = []
  used_objects = set()

  for dim in PRIORITY_ORDER:
    for key, group in groups_by_dimension[dim].items():
      eligible = [s for s in group if s.knowledge_id not in used_objects]
      if len(eligible) >= 2:
        selected_groups.append((dim, key, eligible))
        used_objects.update(s.knowledge_id for s in eligible)

  // Step 3: Extract preambles
  preambles = []
  for dim, key, group in selected_groups:
    preamble = BUILD_PREAMBLE(dim, group, kil_objects_by_id)
    preambles.append(preamble)

  // Step 4: Build delta slices
  delta_slices = []
  for s in slices:
    # Find which preamble covers this object (if any)
    covering_preamble = FIND_PREAMBLE(s.knowledge_id, preambles)
    if covering_preamble:
      delta = SUBTRACT_SHARED(s, covering_preamble)
      delta_slices.append(delta)
    else:
      delta_slices.append(s)  # no SCO applied — pass through unchanged

  return preambles, delta_slices


BUILD_PREAMBLE(dim, group_slices, kil_objects_by_id) → SharedPreamble:
  objects = [kil_objects_by_id[s.knowledge_id] for s in group_slices]

  shared_tags        = INTERSECT([o.tags for o in objects])
  shared_constraints = INTERSECT([GET_CONSTRAINTS(o) for o in objects])
  shared_patterns    = INTERSECT([GET_PATTERNS(o) for o in objects])
  shared_policies    = INTERSECT([GET_POLICIES(o) for o in objects])
  shared_examples    = INTERSECT([GET_EXAMPLES(o) for o in objects])

  content = {namespace: ..., shared_tags: ..., shared_constraints: ..., ...}
  return SharedPreamble(
    dimension         = dim,
    group_key         = COMPUTE_KEY(dim, objects[0]),
    shared_tags       = shared_tags,
    shared_constraints = shared_constraints,
    shared_patterns   = shared_patterns,
    shared_policies   = shared_policies,
    shared_examples   = shared_examples,
    object_ids        = [o.knowledge_id for o in objects],
    token_count       = ESTIMATE_TOKENS(content),
  )
```

---

## ContextPack Schema Extension

UICE adds `shared_preambles` to the ContextPack (additive, backward-compatible):

```yaml
UICEContextPack:
  # All existing CAE ContextPack fields preserved
  pack_id: string
  pack_type: PromptPack | HumanPack | OperatorPack | SearchPack
  primary: FieldLevelSlice | DeltaSlice
  guard: AdaptiveGuardBlock
  context: list[FieldLevelSlice | DeltaSlice]
  pack_metadata: PackMetadata

  # UICE additions (backward-compatible)
  shared_preambles: list[SharedPreamble]   # empty list if SCO not triggered
  intent_profile: IntentProfile            # for explainability
  query_profile: QueryProfile              # for traceability
  plan_id: string                          # links to ContextPlan
  session_id: string | null               # for Conversation Memory
```

---

## Token Saving Targets

```
2 objects, same namespace:           ~30–50% saving vs no SCO
3 objects, same namespace:           ~45–60% saving
4 objects, same package + namespace: ~55–70% saving
Objects sharing CONSTRAINTS:         additional 40–80 tokens saved per object
Objects sharing POLICIES:            additional 30–60 tokens saved per object
```

---

## Rules

| Rule | Statement |
|------|-----------|
| UICE-021 | SCO requires ≥ 2 objects to share a dimension key before a preamble is created |
| UICE-022 | CONSTRAINTS dimension has highest priority; it saves the most tokens per object |
| UICE-023 | shared_preambles in UICEContextPack must be empty list (not absent) when SCO is not triggered |
| UICE-024 | SUBTRACT_SHARED must guarantee no data loss — delta + preamble must reconstruct the original |
| UICE-025 | One object belongs to at most one preamble group; the highest-priority group wins |

---

## Cross-References

- Platform implementation → platform/context_engine/scb.py
- Dynamic Context Planner → `04-DYNAMIC-CONTEXT-PLANNER`
- Context Delta Generator → `06-CONTEXT-DELTA-GENERATOR`
- Context Deduplicator → `07-CONTEXT-DEDUPLICATOR`
