---
knowledge_id: KA-SDK-003
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
    reason: "Operates on Collection interfaces"
---

# Algorithms SDK

## Composable Algorithm Pipeline — Sort · Search · Transform · Reduce · Partition

---

## 1. Purpose

The Algorithms SDK provides a composable, generic, complexity-guaranteed set of
algorithmic primitives that operate on Collections SDK types. All algorithms are
stateless functions or composable pipeline stages. No algorithm modifies its
input. Every algorithm declares its time and space complexity.

---

## 2. Algorithm Categories

```
Algorithms SDK
├── Sort             — comparison, radix, topological
├── Search           — linear, binary, interpolation
├── Transform        — map, flatMap, filter, zip, enumerate
├── Reduce           — fold, scan, aggregate, groupBy
├── Partition        — split, chunk, window, sliding
├── Set Operations   — union, intersection, difference, cartesian product
├── Order            — min, max, median, percentile, rank
└── Pipeline         — compose, chain, parallel, lazy
```

---

## 3. Interfaces

### 3.1 Algorithm (base)

```python
class Algorithm(Protocol[I, O]):
    """Stateless transform: I → O."""
    def apply(self, input: I) -> O
    def complexity(self) -> ComplexitySpec
    def verify_preconditions(self, input: I) -> None  # raises PreconditionError

@dataclass(frozen=True)
class ComplexitySpec:
    time_best:    str   # e.g. "O(N log N)"
    time_avg:     str
    time_worst:   str
    space:        str
    stable:       bool  # for sort algorithms
    in_place:     bool
```

### 3.2 Sort

```python
class SortAlgorithm(Algorithm[Sequence[T], ImmutableList[T]]):
    """Produces a sorted copy. Never mutates input."""
    def sort(self, seq: Sequence[T], key: Callable[[T], Any] | None = None,
             reverse: bool = False) -> ImmutableList[T]

# Built-in implementations:
# TimSort         — O(N log N) worst, O(N) best (pre-sorted)
# MergeSort       — O(N log N) stable
# HeapSort        — O(N log N) in-place, not stable
# RadixSort       — O(N·k) for integer keys, k=digits
# CountingSort    — O(N+K) for bounded integer keys
# TopologicalSort — O(V+E) for DAGs (from Traversal SDK)
```

### 3.3 Search

```python
class SearchAlgorithm(Algorithm[tuple[Sequence[T], T], int | None]):
    """Returns index of item or None."""
    def search(self, seq: Sequence[T], target: T,
               key: Callable[[T], Any] | None = None) -> int | None

# Built-in implementations:
# LinearSearch    — O(N), works on unsorted
# BinarySearch    — O(log N), requires sorted input
# InterpolationSearch — O(log log N) avg for uniform distribution
# ExponentialSearch  — O(log N), good for unbounded sequences
```

### 3.4 Transform

```python
class Transform(Protocol[T, U]):
    def map(self, seq: Sequence[T], fn: Callable[[T], U]) -> ImmutableList[U]  # O(N)
    def flat_map(self, seq: Sequence[T], fn: Callable[[T], Sequence[U]]) -> ImmutableList[U]  # O(N·M)
    def filter(self, seq: Sequence[T], pred: Callable[[T], bool]) -> ImmutableList[T]  # O(N)
    def zip(self, a: Sequence[A], b: Sequence[B]) -> ImmutableList[tuple[A, B]]  # O(min(N,M))
    def enumerate(self, seq: Sequence[T]) -> ImmutableList[tuple[int, T]]  # O(N)
    def distinct(self, seq: Sequence[T]) -> ImmutableList[T]  # O(N) with hash
    def flatten(self, seq: Sequence[Sequence[T]]) -> ImmutableList[T]  # O(N·M)
```

### 3.5 Reduce

```python
class Reduce(Protocol):
    def fold(self, seq: Sequence[T], initial: U,
             fn: Callable[[U, T], U]) -> U  # O(N)
    def scan(self, seq: Sequence[T], initial: U,
             fn: Callable[[U, T], U]) -> ImmutableList[U]  # O(N) — prefix sums
    def group_by(self, seq: Sequence[T],
                 key_fn: Callable[[T], K]) -> ImmutableMap[K, ImmutableList[T]]  # O(N)
    def partition(self, seq: Sequence[T],
                  pred: Callable[[T], bool]) -> tuple[ImmutableList[T], ImmutableList[T]]  # O(N)
    def count_by(self, seq: Sequence[T],
                 key_fn: Callable[[T], K]) -> ImmutableMap[K, int]  # O(N)
```

### 3.6 Partition

```python
class Partition(Protocol):
    def chunk(self, seq: Sequence[T], size: int) -> ImmutableList[ImmutableList[T]]  # O(N)
    def window(self, seq: Sequence[T], size: int,
               step: int = 1) -> ImmutableList[ImmutableList[T]]  # O(N·W)
    def split_at(self, seq: Sequence[T], index: int) -> tuple[ImmutableList[T], ImmutableList[T]]
    def split_by(self, seq: Sequence[T],
                 pred: Callable[[T], bool]) -> ImmutableList[ImmutableList[T]]  # O(N)
```

### 3.7 Order

```python
class Order(Protocol):
    def min(self, seq: Sequence[T], key: Callable[[T], Any] | None = None) -> T | None  # O(N)
    def max(self, seq: Sequence[T], key: Callable[[T], Any] | None = None) -> T | None  # O(N)
    def top_k(self, seq: Sequence[T], k: int, key: Callable[[T], Any] | None = None) -> ImmutableList[T]  # O(N log k)
    def bottom_k(self, seq: Sequence[T], k: int, ...) -> ImmutableList[T]  # O(N log k)
    def rank(self, seq: Sequence[T], item: T) -> int  # O(N log N)
    def percentile(self, seq: Sequence[float], p: float) -> float  # O(N log N)
    def median(self, seq: Sequence[float]) -> float  # O(N) via QuickSelect
```

### 3.8 Pipeline

```python
class Pipeline(Protocol[I, O]):
    """Composable lazy pipeline of Algorithm stages."""
    def then(self, next_alg: Algorithm[O, R]) -> "Pipeline[I, R]"
    def parallel(self, other: "Pipeline[I, R]") -> "Pipeline[I, tuple[O, R]]"
    def lazy(self) -> "LazyPipeline[I, O]"
    def execute(self, input: I) -> O

class LazyPipeline(Pipeline[I, O]):
    """Evaluates only when terminal operation is called."""
    def collect(self) -> O          # terminal — forces evaluation
    def first(self) -> Any | None   # terminal — short-circuits at first
    def count(self) -> int          # terminal
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-ALG-001 | No algorithm mutates its input. Inputs are treated as read-only. |
| C-ALG-002 | All algorithms are referentially transparent (same input → same output). |
| C-ALG-003 | Every algorithm implements `complexity()` returning a valid `ComplexitySpec`. |
| C-ALG-004 | `verify_preconditions` raises `PreconditionError` before the algorithm runs if conditions are not met (e.g. BinarySearch on unsorted sequence). |
| C-ALG-005 | Pipeline is lazy: no computation until a terminal operation is called. |
| C-ALG-006 | `fold` with an empty sequence returns `initial` unchanged. |
| C-ALG-007 | Sort output is a valid permutation of the input (no elements added or dropped). |
| C-ALG-008 | `group_by` result: union of all groups equals the original sequence. |
| C-ALG-009 | `top_k` where k >= N returns all elements sorted. |
| C-ALG-010 | `scan` output length equals input length. |

---

## 5. Algorithms Reference

| Algorithm | Complexity | Notes |
|-----------|-----------|-------|
| TimSort | O(N log N) worst, O(N) best | Stable, adaptive |
| MergeSort | O(N log N) | Stable, O(N) space |
| HeapSort | O(N log N) | In-place, not stable |
| RadixSort | O(N·k) | Integer keys only |
| CountingSort | O(N+K) | Bounded integer keys |
| BinarySearch | O(log N) | Sorted input required |
| QuickSelect | O(N) avg | For median/percentile |
| LinearSearch | O(N) | Any input |
| Fold/Reduce | O(N) | Sequential |
| GroupBy | O(N) | Hash-based |
| top_k | O(N log k) | Min-heap |
| window | O(N·W) | W = window size |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — operates on all Collection types
- No other SDK dependencies

---

## 7. Extension Points

```python
class SortStrategy(Protocol[T]):
    """Custom sort implementation injectable into SortAlgorithm."""
    def sort_impl(self, items: list[T], key: Callable, reverse: bool) -> list[T]: ...

class AlgorithmRegistry(Protocol):
    """Registry for custom algorithm implementations."""
    def register(self, name: str, impl: Algorithm) -> None: ...
    def get(self, name: str) -> Algorithm | None: ...
    def list(self) -> Sequence[str]: ...

class PipelineOptimizer(Protocol):
    """Rewrites pipelines for performance (e.g. filter-before-map fusion)."""
    def optimize(self, pipeline: Pipeline) -> Pipeline: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-ALG-001 | `sort([])` returns `[]` |
| V-ALG-002 | `sort([3,1,2])` returns `[1,2,3]` |
| V-ALG-003 | Stable sort: equal elements maintain relative order |
| V-ALG-004 | `binary_search` on unsorted input raises `PreconditionError` |
| V-ALG-005 | `map(fn, [])` returns `[]` |
| V-ALG-006 | `fold([], 0, add)` returns `0` |
| V-ALG-007 | `group_by` produces groups whose union equals original |
| V-ALG-008 | `scan([1,2,3], 0, add)` = `[1,3,6]` |
| V-ALG-009 | `top_k(seq, 3)` returns 3 largest elements in descending order |
| V-ALG-010 | Pipeline is lazy: applying `map` alone does not execute `fn` |
