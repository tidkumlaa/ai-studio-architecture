---
knowledge_id: KA-SDK-006
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
    reason: "Key/value types from Collections"
---

# Index SDK

## B+Tree · Hash · Inverted · Composite · Bitmap · Spatial

---

## 1. Purpose

The Index SDK provides platform-level index structures that are reusable across
the Knowledge Runtime, Graph Store, Search Engine, and Kernel. All indexes are
generic over key and value types. All support range queries where applicable.
All guarantee documented complexity for point lookup, range scan, and mutation.

---

## 2. Index Taxonomy

```
Index[K, V]
├── HashIndex[K, V]          — O(1) point lookup, no ordering
├── BPlusTreeIndex[K, V]     — O(log N) point + O(log N + R) range
├── InvertedIndex[K, V]      — term → posting list, BM25 scoring
├── CompositeIndex[K, V]     — multi-column compound key
├── BitmapIndex[K]           — set membership, O(1) contains
├── SpatialIndex[K, V]       — R-Tree, bounding-box queries
└── BloomPrefilter[K]        — probabilistic absent-key guard
```

---

## 3. Interfaces

### 3.1 Index (base)

```python
class Index(Protocol[K, V]):
    def get(self, key: K) -> V | None          # O(depends on subtype)
    def put(self, key: K, value: V) -> None
    def remove(self, key: K) -> bool           # True if key existed
    def contains(self, key: K) -> bool
    def size(self) -> int                      # O(1)
    def clear(self) -> None
    def entries(self) -> Sequence[tuple[K, V]] # O(N)
```

### 3.2 RangeIndex (extends Index)

```python
class RangeIndex(Index[K, V]):
    """Ordered index supporting range and prefix scans."""
    def range(self, low: K, high: K, inclusive: bool = True) -> Sequence[tuple[K, V]]  # O(log N + R)
    def prefix(self, key_prefix: K) -> Sequence[tuple[K, V]]  # O(log N + R)
    def min_key(self) -> K | None              # O(1) or O(log N)
    def max_key(self) -> K | None              # O(1) or O(log N)
    def count_range(self, low: K, high: K) -> int  # O(log N + R)
    def keys(self) -> Sequence[K]             # O(N), sorted ascending
```

### 3.3 HashIndex[K, V]

```python
class HashIndex(Index[K, V]):
    """Open-addressing or chaining hash table."""
    # get: O(1) avg, O(N) worst
    # put: O(1) amortized
    # remove: O(1) avg
    def load_factor(self) -> float   # resizes when > threshold (default 0.75)
    def capacity(self) -> int
```

### 3.4 BPlusTreeIndex[K, V]

```python
class BPlusTreeIndex(RangeIndex[K, V]):
    """B+Tree with configurable branching factor."""
    # All leaves are linked — sequential scan is O(N) without extra seeks
    # get: O(log N)
    # put: O(log N)
    # remove: O(log N) with rebalancing
    # range(low, high): O(log N + R) where R = results
    def branching_factor(self) -> int  # typically 64-256 for cache alignment
    def height(self) -> int
    def leaf_count(self) -> int
```

### 3.5 InvertedIndex[K, V]

```python
class InvertedIndex(Index[str, Sequence[V]]):
    """Maps terms to posting lists; supports BM25 scoring."""
    def index_document(self, doc_id: str, tokens: Sequence[str]) -> None
    def remove_document(self, doc_id: str) -> None
    def postings(self, term: str) -> Sequence[tuple[str, int]]  # (doc_id, term_freq)
    def doc_frequency(self, term: str) -> int                   # number of docs containing term
    def doc_count(self) -> int
    def avg_doc_length(self) -> float
    def bm25_score(self, doc_id: str, query_terms: Sequence[str],
                   k1: float = 1.5, b: float = 0.75) -> float

    # BM25 formula:
    # IDF(t) = log((N - df(t) + 0.5) / (df(t) + 0.5) + 1)
    # TF_norm(t,d) = tf(t,d) * (k1+1) / (tf(t,d) + k1*(1-b+b*|d|/avgdl))
    # score(d,q) = Σ IDF(t) * TF_norm(t,d)
```

### 3.6 CompositeIndex[K, V]

```python
class CompositeIndex(RangeIndex[tuple, V]):
    """Multi-column compound key index. Keys are tuples."""
    # Supports prefix queries on leading columns:
    # query(col1=X) — prefix scan O(log N + R)
    # query(col1=X, col2=Y) — more selective, still O(log N + R)
    def query(self, **column_values) -> Sequence[tuple[tuple, V]]
    def column_names(self) -> Sequence[str]
    def add_column(self, name: str, key_fn: Callable[[V], Any]) -> None
```

### 3.7 BitmapIndex[K]

```python
class BitmapIndex(Protocol[K]):
    """Per-value bitmap for set operations. Ideal for low-cardinality fields."""
    def add(self, key: K, doc_id: int) -> None   # O(1)
    def remove(self, key: K, doc_id: int) -> None
    def get_bitmap(self, key: K) -> Bitmap        # O(1)
    def and_(self, a: K, b: K) -> Bitmap         # O(N/64) — SIMD bitwise
    def or_(self, a: K, b: K) -> Bitmap
    def not_(self, key: K) -> Bitmap

class Bitmap:
    def contains(self, doc_id: int) -> bool       # O(1)
    def cardinality(self) -> int                  # O(N/64)
    def to_list(self) -> Sequence[int]            # O(N)
```

### 3.8 SpatialIndex[K, V]

```python
class SpatialIndex(Protocol[K, V]):
    """R-Tree for 2D bounding box and point queries."""
    def insert(self, key: K, bbox: BoundingBox, value: V) -> None  # O(log N)
    def delete(self, key: K) -> None                                 # O(log N)
    def search(self, bbox: BoundingBox) -> Sequence[tuple[K, V]]    # O(log N + R)
    def nearest(self, point: Point, k: int = 1) -> Sequence[tuple[K, V]]  # O(k log N)

@dataclass(frozen=True)
class BoundingBox:
    min_x: float; min_y: float
    max_x: float; max_y: float

@dataclass(frozen=True)
class Point:
    x: float; y: float
```

### 3.9 BloomPrefilter[K]

```python
class BloomPrefilter(Protocol[K]):
    """Probabilistic absent-key guard before expensive index lookup."""
    def add(self, key: K) -> None
    def might_contain(self, key: K) -> bool  # O(hash_count)
    def false_positive_rate(self) -> float
    def item_count(self) -> int
    def reset(self) -> None
```

### 3.10 IndexManager

```python
class IndexManager(Protocol):
    """Registry and lifecycle manager for multiple indexes."""
    def register(self, name: str, index: Index) -> None
    def get(self, name: str) -> Index | None
    def drop(self, name: str) -> None
    def rebuild(self, name: str, source: Iterable[tuple[Any, Any]]) -> None
    def list(self) -> Sequence[str]
    def stats(self, name: str) -> IndexStats

@dataclass(frozen=True)
class IndexStats:
    name: str
    type: str
    size: int
    height: int | None      # Tree-based only
    load_factor: float | None  # Hash-based only
    memory_bytes: int
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-IDX-001 | `get` returns `None` for absent keys (not raise). |
| C-IDX-002 | `put` on existing key replaces the value. |
| C-IDX-003 | `range(low, high)` with `low > high` returns empty. |
| C-IDX-004 | BitmapIndex cardinality ≤ total documents indexed. |
| C-IDX-005 | `BloomPrefilter.might_contain(k)` is true for all added keys (no false negatives). |
| C-IDX-006 | `InvertedIndex.bm25_score` is 0.0 if no query terms appear in doc. |
| C-IDX-007 | `CompositeIndex.query()` with no args returns all entries. |
| C-IDX-008 | `BPlusTreeIndex.keys()` returns keys in ascending sorted order. |
| C-IDX-009 | `remove` on absent key returns `False` and is a no-op. |
| C-IDX-010 | `IndexManager.rebuild` replaces all entries — does not append. |

---

## 5. Complexity Table

| Index | get | put | remove | range | contains |
|-------|-----|-----|--------|-------|----------|
| HashIndex | O(1) avg | O(1) amort | O(1) avg | — | O(1) avg |
| BPlusTree | O(log N) | O(log N) | O(log N) | O(log N+R) | O(log N) |
| InvertedIndex | O(P) | O(T) | O(T) | — | O(1) |
| BitmapIndex | O(1) | O(1) | O(1) | O(N/64) | O(1) |
| SpatialIndex | O(log N+R) | O(log N) | O(log N) | O(log N+R) | — |
| BloomPrefilter | O(h) | O(h) | — | — | O(h) |

P = postings count, T = tokens, R = result count, h = hash function count

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — key/value sequence types

---

## 7. Extension Points

```python
class IndexSerializer(Protocol):
    """Serialize/deserialize index state to bytes."""
    def serialize(self, index: Index) -> bytes: ...
    def deserialize(self, data: bytes) -> Index: ...

class IndexMigration(Protocol):
    """Schema migration for index structure changes."""
    def migrate(self, old_index: Index, new_schema: dict) -> Index: ...

class CompressionStrategy(Protocol):
    """Compress posting lists or B+Tree nodes."""
    def compress(self, data: bytes) -> bytes: ...
    def decompress(self, data: bytes) -> bytes: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-IDX-001 | `get` after `put` returns same value |
| V-IDX-002 | `get` after `remove` returns `None` |
| V-IDX-003 | `BPlusTree.range(1, 5)` returns all keys 1 ≤ k ≤ 5 |
| V-IDX-004 | `BPlusTree.keys()` is sorted ascending |
| V-IDX-005 | Bloom: `might_contain` true for all added keys |
| V-IDX-006 | Bloom: false positive rate ≤ declared `error_rate` over large random sample |
| V-IDX-007 | `InvertedIndex.bm25_score` is 0 for query with no terms in doc |
| V-IDX-008 | BitmapIndex `and_(A, B)` returns IDs in both A and B |
| V-IDX-009 | `CompositeIndex.query(col1=X)` returns all entries with col1=X |
| V-IDX-010 | `IndexManager.rebuild` produces same result as fresh build from same source |
