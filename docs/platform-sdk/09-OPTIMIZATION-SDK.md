---
knowledge_id: KA-SDK-010
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
    reason: "Solution space collections"
  - id: KA-SDK-013
    reason: "Statistics SDK for convergence analysis"
---

# Optimization SDK

## Combinatorial · Constraint · Greedy · Pareto · Cost Modeling

---

## 1. Purpose

The Optimization SDK provides reusable optimization primitives for planning,
scheduling, resource allocation, and cost minimization. It is used by the
Platform Scheduler, Execution Planner (Phase 2.1A), and the Universal Product
Factory (Phase 2.3). The SDK is solver-agnostic: consumers define objectives
and constraints; the SDK selects or delegates to optimization strategies.

---

## 2. Optimization Taxonomy

```
Optimization Problem
├── Combinatorial
│   ├── Knapsack
│   ├── Set Cover
│   ├── Assignment Problem
│   └── Traveling Salesman (approximation)
├── Constraint Satisfaction (CSP)
│   ├── Backtracking Search
│   ├── Arc Consistency (AC-3)
│   └── Constraint Propagation
├── Greedy Heuristics
│   ├── Greedy Best-First
│   ├── Local Search
│   └── Hill Climbing
├── Pareto Multi-Objective
│   ├── Pareto Front Computation
│   └── NSGA-II Style Dominance
└── Cost Modeling
    ├── Cost Function Registry
    └── Cost Estimation Pipeline
```

---

## 3. Interfaces

### 3.1 Problem Definition

```python
@dataclass
class OptimizationProblem(Generic[S]):
    """Generic optimization problem over solution space S."""
    name:         str
    variables:    Sequence[Variable]
    constraints:  Sequence[Constraint]
    objectives:   Sequence[Objective]
    sense:        str       # "minimize" | "maximize"

@dataclass(frozen=True)
class Variable:
    name:    str
    domain:  Domain       # Set of allowed values
    kind:    str          # "integer" | "real" | "boolean" | "enum"

@dataclass(frozen=True)
class Domain:
    values:   Sequence[Any] | None    # Explicit enumeration
    low:      Any | None              # Range lower bound
    high:     Any | None              # Range upper bound

@dataclass(frozen=True)
class Constraint:
    name:       str
    expression: str        # Human-readable constraint expression
    evaluator:  Callable[["Solution"], bool]
    kind:       str        # "hard" | "soft"
    weight:     float      # For soft constraints

@dataclass(frozen=True)
class Objective:
    name:        str
    fn:          Callable[["Solution"], float]
    sense:       str    # "minimize" | "maximize"
    weight:      float  # For weighted-sum multi-objective
```

### 3.2 Solution

```python
@dataclass
class Solution(Generic[S]):
    values:          ImmutableMap[str, Any]  # variable_name → assigned_value
    objective_values: ImmutableMap[str, float]
    constraint_violations: Sequence[str]     # Names of violated constraints
    feasible:        bool                    # All hard constraints satisfied
    total_cost:      float
    metadata:        ImmutableMap[str, Any]

    def value_of(self, variable: str) -> Any
    def is_better_than(self, other: "Solution", sense: str) -> bool
```

### 3.3 Optimizer

```python
class Optimizer(Protocol[S]):
    """Solves an OptimizationProblem."""
    def solve(self, problem: OptimizationProblem[S],
              time_limit_ms: int | None = None) -> OptimizationResult[S]
    def complexity(self) -> str
    def is_exact(self) -> bool      # True = proven optimal; False = heuristic

@dataclass
class OptimizationResult(Generic[S]):
    problem_name: str
    best:         Solution[S]
    all_solutions: Sequence[Solution[S]]  # For multi-objective: Pareto front
    status:       str    # "optimal" | "feasible" | "infeasible" | "timeout"
    elapsed_ms:   float
    iterations:   int
    bound:        float | None  # Proven lower/upper bound
```

### 3.4 Constraint Solver (CSP)

```python
class ConstraintSolver(Optimizer[S]):
    """Backtracking CSP solver with AC-3 propagation."""
    def add_propagator(self, propagator: "ConstraintPropagator") -> None
    def solve_all(self, problem: OptimizationProblem[S],
                  limit: int = 100) -> Sequence[Solution[S]]

class ConstraintPropagator(Protocol):
    """Reduces variable domains based on constraint graph."""
    def propagate(self, var: Variable, assignment: dict,
                  domains: MutableMap[str, Domain]) -> bool  # False = inconsistency

# AC-3 complexity: O(E · D³) where E = constraint arcs, D = max domain size
```

### 3.5 Greedy Optimizer

```python
class GreedyOptimizer(Optimizer[S]):
    """Builds solution incrementally via greedy choice at each step."""
    def choice_function(self, problem: OptimizationProblem[S],
                        partial: Solution[S]) -> tuple[str, Any]
    # Set custom choice_fn for domain-specific greedy strategies

class LocalSearchOptimizer(Optimizer[S]):
    """Hill-climbing local search with neighborhood definition."""
    def neighborhood(self, solution: Solution[S]) -> Sequence[Solution[S]]
    def acceptance_criterion(self, current: float, neighbor: float) -> bool
    # Default: accept only improving moves
    # Override for simulated annealing or tabu search
```

### 3.6 Pareto Optimizer

```python
class ParetoOptimizer(Optimizer[S]):
    """Multi-objective optimizer returning full Pareto front."""
    def dominates(self, a: Solution[S], b: Solution[S]) -> bool
    # a dominates b if a is at least as good on all objectives and strictly
    # better on at least one.

    def pareto_front(self, solutions: Sequence[Solution[S]]) -> Sequence[Solution[S]]
    # Complexity: O(N² · M) for N solutions, M objectives
```

### 3.7 Cost Model

```python
class CostModel(Protocol[I]):
    """Estimates cost of an action or configuration."""
    def estimate(self, item: I) -> CostEstimate
    def breakdown(self, item: I) -> ImmutableMap[str, float]

@dataclass(frozen=True)
class CostEstimate:
    time_ms:    float
    tokens:     int | None
    memory_mb:  float
    confidence: float       # 0-1, how reliable the estimate is
    unit:       str         # "tokens" | "ms" | "requests" | custom

class CostModelRegistry(Protocol):
    def register(self, name: str, model: CostModel) -> None
    def get(self, name: str) -> CostModel | None
    def estimate_pipeline(self, steps: Sequence[tuple[str, Any]]) -> CostEstimate
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-OPT-001 | A `Solution` is feasible only if all hard constraints are satisfied. |
| C-OPT-002 | `OptimizationResult.status == "optimal"` only if provably optimal solution is returned. |
| C-OPT-003 | Greedy optimizer never revisits an assigned variable. |
| C-OPT-004 | `ParetoOptimizer.pareto_front` returns no pair (a, b) where a dominates b. |
| C-OPT-005 | `ConstraintSolver` with no feasible solutions returns `status="infeasible"`. |
| C-OPT-006 | `time_limit_ms` is a hard deadline: optimizer returns best-found within limit. |
| C-OPT-007 | `CostModel.estimate` is deterministic for same input. |
| C-OPT-008 | `OptimizationProblem` with zero variables has trivially optimal empty solution. |
| C-OPT-009 | Soft constraint violations reduce fitness score but do not make solution infeasible. |
| C-OPT-010 | `LocalSearchOptimizer` terminates when no improving neighbor exists or limit reached. |

---

## 5. Complexity Table

| Algorithm | Complexity | Notes |
|-----------|-----------|-------|
| AC-3 constraint propagation | O(E · D³) | E = arcs, D = domain size |
| Backtracking CSP | O(D^N) worst | N = variables; pruning reduces actual |
| Greedy assignment | O(N · C) | N = items, C = constraint checks |
| Pareto front | O(N² · M) | N = solutions, M = objectives |
| Local search | O(I · K) | I = iterations, K = neighborhood size |
| Knapsack (DP) | O(N · W) | N = items, W = capacity |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — variable domains, solution collections
- Statistics SDK (KA-SDK-013) — convergence analysis, confidence intervals

---

## 7. Extension Points

```python
class SolverPlugin(Protocol[S]):
    """Register an external solver (e.g. OR-Tools, Gurobi wrapper)."""
    def solver_name(self) -> str: ...
    def create_optimizer(self, config: dict) -> Optimizer[S]: ...

class HeuristicFunction(Protocol[S]):
    """Custom heuristic for greedy or search-based optimizers."""
    def evaluate(self, partial: Solution[S],
                 remaining: Sequence[Variable]) -> float: ...

class ObjectiveFunction(Protocol[S]):
    """Pluggable objective for multi-objective scenarios."""
    def name(self) -> str: ...
    def evaluate(self, solution: Solution[S]) -> float: ...
    def sense(self) -> str: ...  # "minimize" | "maximize"
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-OPT-001 | Feasible solution satisfies all hard constraints |
| V-OPT-002 | `OptimizationResult.all_solutions` non-empty for Pareto optimizer |
| V-OPT-003 | Pareto front: no pair (A, B) where A dominates B |
| V-OPT-004 | Greedy: each variable assigned exactly once |
| V-OPT-005 | CSP with conflicting constraints: `status == "infeasible"` |
| V-OPT-006 | Time-limited solve: elapsed_ms ≤ time_limit_ms + tolerance |
| V-OPT-007 | `CostModel.estimate` same input → same output |
| V-OPT-008 | Knapsack: total weight of solution ≤ capacity |
| V-OPT-009 | Local search terminates |
| V-OPT-010 | `Solution.is_better_than` transitive: if A > B and B > C then A > C |
