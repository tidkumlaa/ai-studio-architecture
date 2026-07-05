---
knowledge_id: KA-SDK-012
version: "1.0.0"
status: approved
owner: Chief Platform Architect
phase: 2.0D.2.5A
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-PLATFORM
capability: platform-sdk
type: specification
depends_on:
  - id: KA-SDK-002
    reason: "Fact and rule collections"
  - id: KA-SDK-004
    reason: "Graph SDK for belief network representation"
  - id: KA-SDK-016
    reason: "Traversal for rule dependency ordering"
---

# Reasoning SDK

## Rule Engine · Forward/Backward Chaining · Unification · Constraint Propagation

---

## 1. Purpose

The Reasoning SDK provides a general-purpose, data-driven rule engine and
logical inference substrate. It generalizes the Knowledge Reasoning component
(KA-KIP-006) into a platform primitive usable by the Knowledge Operating
System, Architecture Constitution Engine, Impact Analyzer, and any component
requiring formal inference. Rules are data — no reasoning logic is hardcoded.

---

## 2. Reasoning Architecture

```
Knowledge Base
├── FactStore         — ground facts (asserted truths)
├── RuleBase          — conditional rules (IF condition THEN action)
└── BeliefState       — inferred beliefs with confidence

Inference Engines
├── ForwardChainer    — data-driven: facts → rules → new facts
├── BackwardChainer   — goal-driven: prove goal by finding supporting facts
├── UnificationEngine — matches patterns against facts
└── ConstraintPropagator — reduces domains based on inference constraints
```

---

## 3. Interfaces

### 3.1 Fact

```python
@dataclass(frozen=True)
class Fact:
    predicate:  str             # e.g. "depends_on", "implements", "is_stale"
    subject:    str             # e.g. "KA-ARCH-001"
    object_:    str | None      # e.g. "KA-SPEC-001"
    value:      Any | None      # Scalar value for property facts
    confidence: float = 1.0     # 0.0 – 1.0
    source:     str | None = None  # Where this fact came from

@dataclass
class FactStore:
    def assert_fact(self, fact: Fact) -> None
    def retract_fact(self, fact: Fact) -> None
    def query(self, predicate: str | None = None,
              subject: str | None = None,
              object_: str | None = None) -> Sequence[Fact]
    def all(self) -> Sequence[Fact]
    def size(self) -> int
```

### 3.2 Rule

```python
@dataclass(frozen=True)
class Rule:
    rule_id:     str
    name:        str
    conditions:  Sequence["Condition"]
    actions:     Sequence["Action"]
    confidence:  float = 1.0
    priority:    int = 0          # Higher runs first in conflict resolution
    enabled:     bool = True
    category:    str | None = None

@dataclass(frozen=True)
class Condition:
    predicate:  str
    subject:    str | Variable
    object_:    str | Variable | None
    value:      Any | Variable | None
    negated:    bool = False

@dataclass(frozen=True)
class Variable:
    name: str   # e.g. "?X", "?Y" — variables in rule conditions/actions

@dataclass(frozen=True)
class Action:
    kind:       str   # "assert" | "retract" | "derive" | "emit_event"
    predicate:  str
    subject:    str | Variable
    object_:    str | Variable | None
    value:      Any | Variable | None
    confidence: float = 1.0

class RuleBase:
    def add(self, rule: Rule) -> None
    def remove(self, rule_id: str) -> None
    def get(self, rule_id: str) -> Rule | None
    def all(self) -> Sequence[Rule]
    def by_category(self, category: str) -> Sequence[Rule]
    def enabled(self) -> Sequence[Rule]
```

### 3.3 ForwardChainer

```python
class ForwardChainer(Protocol):
    """Data-driven inference: matches rules against current facts."""
    def run(self, fact_store: FactStore,
            rule_base: RuleBase,
            max_iterations: int = 100) -> ForwardChainingResult
    # Complexity: O(R × F × I) — R=rules, F=facts, I=iterations until fixpoint

@dataclass
class ForwardChainingResult:
    derived_facts:   Sequence[Fact]
    fired_rules:     Sequence[tuple[str, Binding]]  # (rule_id, variable_bindings)
    iterations:      int
    fixpoint_reached: bool

@dataclass(frozen=True)
class Binding:
    """Variable bindings from unification."""
    vars: ImmutableMap[str, str]  # "?X" → "KA-ARCH-001"
```

### 3.4 BackwardChainer

```python
class BackwardChainer(Protocol):
    """Goal-driven inference: proves a goal by searching for supporting facts."""
    def prove(self, goal: Fact,
              fact_store: FactStore,
              rule_base: RuleBase,
              max_depth: int = 10) -> BackwardChainingResult
    # Complexity: O(R^D × F) — R=rules, D=depth, F=facts

@dataclass
class BackwardChainingResult:
    goal:         Fact
    proved:       bool
    proof_tree:   "ProofNode | None"
    bindings:     Binding

@dataclass
class ProofNode:
    fact:       Fact
    rule_used:  str | None   # Rule that derived this fact
    children:   Sequence["ProofNode"]
    confidence: float
```

### 3.5 UnificationEngine

```python
class UnificationEngine(Protocol):
    """Matches patterns containing Variables against ground facts."""
    def unify(self, pattern: Fact,
              ground: Fact) -> Binding | None  # None = no match
    def match_all(self, pattern: Fact,
                  facts: Sequence[Fact]) -> Sequence[tuple[Fact, Binding]]
    def apply_binding(self, fact: Fact, binding: Binding) -> Fact
    # Complexity: O(|pattern| × |ground|) per unification
```

### 3.6 ConstraintPropagator

```python
class ConstraintPropagator(Protocol):
    """Reduces variable domains during reasoning to prune search space."""
    def propagate(self, variables: Sequence[Variable],
                  constraints: Sequence[Condition],
                  fact_store: FactStore) -> "PropagationResult"

@dataclass
class PropagationResult:
    reduced_domains: ImmutableMap[str, Sequence[Any]]  # variable → remaining values
    consistent:      bool
    reductions:      int   # Number of domain values eliminated
```

### 3.7 BeliefState

```python
class BeliefState(Protocol):
    """Tracks inferred beliefs with associated confidence and support."""
    def assert_belief(self, fact: Fact, support: Sequence[str]) -> None
        # support = fact/rule IDs that justify this belief
    def retract_belief(self, fact: Fact) -> None
    def confidence(self, fact: Fact) -> float
    def support(self, fact: Fact) -> Sequence[str]
    def all_beliefs(self) -> Sequence[tuple[Fact, float]]

class TruthMaintenanceSystem(BeliefState):
    """Retracts derived beliefs when their support is retracted."""
    def retract_with_dependents(self, fact: Fact) -> Sequence[Fact]
        # Removes fact and all beliefs that depended on it
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-RSN-001 | `ForwardChainer` terminates at fixpoint: no new facts derivable. |
| C-RSN-002 | `BackwardChainer` with `max_depth=0` proves only ground facts. |
| C-RSN-003 | `Unification` is commutative: `unify(A, B) == unify(B, A)` (for ground facts). |
| C-RSN-004 | `RuleBase.remove` disables a rule; derived facts are not automatically retracted. |
| C-RSN-005 | `TruthMaintenanceSystem.retract_with_dependents` is recursive: dependents of dependents also retracted. |
| C-RSN-006 | Confidence of derived fact ≤ minimum confidence of supporting facts × rule confidence. |
| C-RSN-007 | `max_iterations` in forward chaining is a hard limit; `fixpoint_reached=False` if hit. |
| C-RSN-008 | `PropagationResult.consistent=False` means no solution exists for current constraints. |
| C-RSN-009 | Negated conditions match if no fact satisfies the pattern. |
| C-RSN-010 | `ForwardChainingResult.fired_rules` contains the same rule_id at most once per binding. |

---

## 5. Complexity Table

| Operation | Complexity | Notes |
|-----------|-----------|-------|
| ForwardChaining (one pass) | O(R × F) | R=rules, F=facts |
| ForwardChaining (to fixpoint) | O(R × F × I) | I=iterations |
| BackwardChaining | O(R^D × F) | D=depth, worst case |
| Unification | O(|terms|) | Per pattern-ground pair |
| match_all | O(P × F) | P=patterns |
| Constraint propagation (AC-3) | O(E · D³) | E=arcs, D=domain |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — fact/rule sequences, binding maps
- Graph SDK (KA-SDK-004) — belief network as property graph
- Traversal SDK (KA-SDK-016) — rule dependency ordering

---

## 7. Extension Points

```python
class RuleLoader(Protocol):
    """Load rules from external sources (YAML, JSON, database)."""
    def load(self, source: str) -> Sequence[Rule]: ...

class InferenceStrategy(Protocol):
    """Custom inference algorithm beyond forward/backward chaining."""
    def name(self) -> str: ...
    def infer(self, fact_store: FactStore,
              rule_base: RuleBase) -> Sequence[Fact]: ...

class ConfidenceCombiner(Protocol):
    """Custom confidence aggregation for derived facts."""
    def combine(self, confidences: Sequence[float]) -> float: ...
    # Default: product of confidences
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-RSN-001 | ForwardChainer reaches fixpoint (no new facts after final iteration) |
| V-RSN-002 | BackwardChaining proves ground fact in `proof_tree` with depth 0 |
| V-RSN-003 | `unify(?X, KA-001)` produces binding `{"?X": "KA-001"}` |
| V-RSN-004 | Negated condition matches when no fact satisfies the pattern |
| V-RSN-005 | TMS: retracting support fact also retracts dependent beliefs |
| V-RSN-006 | Derived fact confidence ≤ product of support confidences |
| V-RSN-007 | `max_iterations` hit: `fixpoint_reached == False` |
| V-RSN-008 | `PropagationResult.consistent == False` when constraints unsatisfiable |
| V-RSN-009 | Disabled rule is never fired |
| V-RSN-010 | `ForwardChainingResult.derived_facts` contains only new facts (not already in FactStore) |
