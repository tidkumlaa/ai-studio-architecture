---
knowledge_id: KA-SDK-002
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
  - id: KA-SDK-001
    reason: "SDK contract and layering rules"
---

# Collections SDK

## Typed Container Hierarchy with O-Complexity Guarantees

---

## 1. Purpose

The Collections SDK provides the foundational typed container hierarchy for all
AI Studio components. It replaces direct use of Python built-ins with a
contract-governed layer that enforces immutability, complexity guarantees, and
structural typing. All higher SDKs operate on Collections SDK types.

---

## 2. Container Hierarchy

```
Container[T]
├── Sequence[T]          — ordered, index-accessible
│   ├── ImmutableList[T]
│   ├── MutableList[T]
│   ├── Deque[T]
│   └── Stack[T]
├── Set[T]               — unordered, unique membership
│   ├── ImmutableSet[T]
│   ├── MutableSet[T]
│   └── SortedSet[T]
├── Map[K, V]            — key-value mapping
│   ├── ImmutableMap[K, V]
│   ├── MutableMap[K, V]
│   ├── SortedMap[K, V]
│   ├── MultiMap[K, V]
│   └── BiMap[K, V]
├── Queue[T]             — FIFO/priority access
│   ├── FIFOQueue[T]
│   ├── PriorityQueue[T]
│   └── BoundedQueue[T]
└── Bag[T]               — multiset (duplicates allowed, unordered)
```

---

## 3. Interfaces

### 3.1 Container (base)

```python
class Container(Protocol[T]):
    def size(self) -> int           # O(1)
    def is_empty(self) -> bool      # O(1)
    def contains(self, item: T) -> bool   # O(depends on subtype)
    def to_list(self) -> list[T]    # O(N)
    def copy(self) -> "Container[T]"  # O(N)
```

### 3.2 Sequence[T]

```python
class Sequence(Container[T]):
    def get(self, index: int) -> T  # O(1) for List, O(N) for Deque
    def head(self) -> T | None      # O(1)
    def tail(self) -> T | None      # O(1)
    def slice(self, start: int, end: int) -> "Sequence[T]"  # O(end-start)
    def reversed(self) -> "Sequence[T]"  # O(N)
    def index_of(self, item: T) -> int | None  # O(N)
```

### 3.3 MutableSequence[T] (extends Sequence)

```python
class MutableSequence(Sequence[T]):
    def append(self, item: T) -> None   # O(1) amortized
    def prepend(self, item: T) -> None  # O(1) for Deque, O(N) for List
    def insert(self, index: int, item: T) -> None  # O(N)
    def remove(self, index: int) -> T   # O(N)
    def set(self, index: int, item: T) -> None  # O(1)
    def clear(self) -> None             # O(1)
```

### 3.4 Map[K, V]

```python
class Map(Protocol[K, V]):
    def get(self, key: K) -> V | None   # O(1) hash, O(log N) sorted
    def get_or_default(self, key: K, default: V) -> V
    def put(self, key: K, value: V) -> None   # O(1) hash, O(log N) sorted
    def remove(self, key: K) -> V | None      # O(1) hash, O(log N) sorted
    def contains_key(self, key: K) -> bool    # O(1) hash, O(log N) sorted
    def keys(self) -> Sequence[K]             # O(N)
    def values(self) -> Sequence[V]           # O(N)
    def entries(self) -> Sequence[tuple[K, V]]  # O(N)
    def size(self) -> int                     # O(1)
```

### 3.5 PriorityQueue[T]

```python
class PriorityQueue(Protocol[T]):
    def push(self, item: T, priority: float) -> None  # O(log N)
    def pop(self) -> T                                # O(log N)
    def peek(self) -> T | None                        # O(1)
    def update_priority(self, item: T, new_priority: float) -> None  # O(log N)
    def size(self) -> int                             # O(1)
```

### 3.6 BiMap[K, V]

```python
class BiMap(Map[K, V]):
    def inverse(self) -> "BiMap[V, K]"     # O(1) — view, not copy
    def get_key(self, value: V) -> K | None  # O(1)
    def put_enforced(self, key: K, value: V) -> None
        # Raises DuplicateValueError if value already mapped to a different key
```

---

## 4. Contracts

| Contract | Description |
|----------|-------------|
| C-COL-001 | `size()` is always O(1). Implementations must maintain a size counter. |
| C-COL-002 | Immutable containers never expose mutation methods. |
| C-COL-003 | `copy()` returns a structurally equal but independently mutable instance. |
| C-COL-004 | Equality is structural: two containers with same elements in same order are equal. |
| C-COL-005 | Iteration order is deterministic. SortedMap/SortedSet use key comparator. |
| C-COL-006 | `None` is a valid value in Map[K, V] but `get()` returns `None | V` — use `contains_key` to distinguish. |
| C-COL-007 | PriorityQueue is a min-heap by default (lowest priority number = highest precedence). |
| C-COL-008 | BiMap enforces 1:1 key-value bijection. |
| C-COL-009 | Bag counts duplicates; `contains(x)` is true if count(x) > 0. |
| C-COL-010 | All containers are iterable and support `for item in container`. |

---

## 5. Algorithms (supplied by Algorithms SDK)

Collections expose no algorithmic methods beyond access. All transformations
(sort, filter, map, reduce) are supplied by the Algorithms SDK operating on
Collection interfaces.

---

## 6. Complexity Table

| Container | contains | get(index) | put/append | remove |
|-----------|----------|------------|------------|--------|
| ImmutableList | O(N) | O(1) | — | — |
| MutableList | O(N) | O(1) | O(1) amort | O(N) |
| Deque | O(N) | O(N) | O(1) both ends | O(1) ends |
| SortedSet | O(log N) | O(log N) | O(log N) | O(log N) |
| ImmutableMap | O(1) avg | — | — | — |
| SortedMap | O(log N) | O(log N) | O(log N) | O(log N) |
| MultiMap | O(1) avg | O(1) avg | O(1) avg | O(K) where K=values per key |
| PriorityQueue | O(N) | — | O(log N) | O(log N) |
| Bag | O(1) avg | — | O(1) avg | O(1) avg |

---

## 7. Dependencies

- Python 3.11+ standard library (`typing`, `collections.abc`)
- No external dependencies at the Collections layer
- Does NOT depend on any other SDK

---

## 8. Extension Points

```python
class CollectionFactory(Protocol):
    """Produces custom Collection implementations."""
    def list(self, items: Iterable[T]) -> MutableList[T]: ...
    def sorted_map(self, comparator: Comparator[K]) -> SortedMap[K, V]: ...
    def priority_queue(self, key_fn: Callable[[T], float]) -> PriorityQueue[T]: ...

class Comparator(Protocol[T]):
    def compare(self, a: T, b: T) -> int: ...  # negative, 0, positive

class StructuralEquality(Protocol[T]):
    def equals(self, a: T, b: T) -> bool: ...
    def hash(self, item: T) -> int: ...
```

---

## 9. Verification

| Check | Criterion |
|-------|-----------|
| V-COL-001 | `size()` returns 0 for empty container |
| V-COL-002 | `ImmutableList.append` raises `AttributeError` or not defined |
| V-COL-003 | `SortedSet` iteration yields elements in comparator order |
| V-COL-004 | `BiMap.inverse()` returns view: changes to original reflected |
| V-COL-005 | `PriorityQueue.pop()` returns minimum-priority item |
| V-COL-006 | `copy()` is independent: mutating copy does not affect original |
| V-COL-007 | `MultiMap.get(key)` returns all values for key, not just last |
| V-COL-008 | Empty `PriorityQueue.peek()` returns `None` |
| V-COL-009 | `Bag.count(x)` returns number of times x was added |
| V-COL-010 | All containers produce equal results when iterated twice |
