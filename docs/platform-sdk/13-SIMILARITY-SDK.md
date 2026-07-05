---
knowledge_id: KA-SDK-014
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
    reason: "Vector and string collections"
  - id: KA-SDK-013
    reason: "Statistical operations for IDF weighting"
---

# Similarity SDK

## Edit Distance · Token · Vector · Set · Structural · Composite

---

## 1. Purpose

The Similarity SDK provides a unified distance and similarity abstraction for
all AI Studio components that need to compare strings, documents, sets, or
structured objects. It covers the full spectrum from character-level edit
distance to semantic vector similarity, composable into hybrid measures. It
is the foundation of the Search SDK's fuzzy strategy, the Canonical Resolver,
and the Deduplication engine.

---

## 2. Similarity Taxonomy

```
SimilarityMeasure[T]
├── Edit Distance Measures
│   ├── LevenshteinDistance        — character edit distance
│   ├── DamerauLevenshtein         — with transposition
│   ├── JaroSimilarity             — transposition-aware, short strings
│   ├── JaroWinklerSimilarity      — prefix-boosted Jaro
│   └── HammingDistance            — same-length strings
│
├── Token-Based Measures
│   ├── JaccardSimilarity          — |A ∩ B| / |A ∪ B|
│   ├── DiceSimilarity             — 2|A ∩ B| / (|A| + |B|)
│   └── OverlapCoefficient         — |A ∩ B| / min(|A|, |B|)
│
├── Vector Measures
│   ├── CosineSimilarity           — dot product / (|a| × |b|)
│   ├── EuclideanDistance          — L2 norm
│   ├── ManhattanDistance          — L1 norm
│   └── DotProduct                 — for pre-normalized vectors
│
├── Fingerprint Measures
│   ├── SimHash                    — near-duplicate detection via hash
│   └── MinHash                    — Jaccard estimation via sketching
│
└── Composite Measures
    └── WeightedComposite          — weighted average of multiple measures
```

---

## 3. Interfaces

### 3.1 SimilarityMeasure[T]

```python
class SimilarityMeasure(Protocol[T]):
    """Returns a similarity score in [0.0, 1.0]. 1.0 = identical."""
    def similarity(self, a: T, b: T) -> float
    def distance(self, a: T, b: T) -> float   # 1.0 - similarity, normalized
    def is_metric(self) -> bool  # True if satisfies triangle inequality

class DistanceMeasure(Protocol[T]):
    """Returns raw distance. Range is measure-dependent."""
    def distance(self, a: T, b: T) -> float
    def max_distance(self, a: T, b: T) -> float  # Normalization denominator
    def normalized(self) -> "SimilarityMeasure[T]"
```

### 3.2 Edit Distance Measures

```python
class LevenshteinDistance(DistanceMeasure[str]):
    """
    Classic Levenshtein edit distance.
    Complexity: O(|a| × |b|) time, O(min(|a|, |b|)) space.
    """
    def distance(self, a: str, b: str) -> float      # Integer in [0, max(|a|,|b|)]
    def normalized(self) -> "LevenshteinSimilarity"  # distance / max_len

class DamerauLevenshtein(DistanceMeasure[str]):
    """
    Damerau-Levenshtein with transpositions.
    Complexity: O(|a| × |b|), same space as Levenshtein.
    """
    def distance(self, a: str, b: str) -> float

class JaroSimilarity(SimilarityMeasure[str]):
    """
    Jaro similarity for short strings.
    Complexity: O(|a| + |b|).
    """
    def similarity(self, a: str, b: str) -> float    # ∈ [0, 1]

class JaroWinklerSimilarity(SimilarityMeasure[str]):
    """
    Jaro-Winkler with prefix boost (default p=0.1).
    Complexity: O(|a| + |b|).
    """
    def __init__(self, p: float = 0.1, max_prefix: int = 4)
    def similarity(self, a: str, b: str) -> float
```

### 3.3 Token-Based Measures

```python
class JaccardSimilarity(SimilarityMeasure[frozenset]):
    """
    Jaccard: |A ∩ B| / |A ∪ B|.
    Complexity: O(|A| + |B|).
    """
    def similarity(self, a: frozenset, b: frozenset) -> float

class DiceSimilarity(SimilarityMeasure[frozenset]):
    """
    Dice: 2|A ∩ B| / (|A| + |B|).
    Complexity: O(|A| + |B|).
    """
    def similarity(self, a: frozenset, b: frozenset) -> float

class TokenSimilarity(Protocol):
    """Higher-level: tokenizes strings then applies set-based measure."""
    def __init__(self, measure: SimilarityMeasure[frozenset],
                 tokenizer: Callable[[str], frozenset[str]])
    def similarity(self, a: str, b: str) -> float
```

### 3.4 Vector Measures

```python
class CosineSimilarity(SimilarityMeasure[Sequence[float]]):
    """
    Cosine similarity between two vectors.
    Complexity: O(N) where N = vector dimension.
    """
    def similarity(self, a: Sequence[float], b: Sequence[float]) -> float

class EuclideanDistance(DistanceMeasure[Sequence[float]]):
    """
    L2 distance. Complexity: O(N).
    """
    def distance(self, a: Sequence[float], b: Sequence[float]) -> float

class TFIDFCosineSimilarity(SimilarityMeasure[str]):
    """
    TF-IDF weighted cosine similarity between documents.
    Requires fitting on a corpus before computing similarity.
    """
    def fit(self, corpus: Sequence[str]) -> None  # O(N × T) N=docs, T=terms
    def similarity(self, a: str, b: str) -> float
    def most_similar(self, query: str,
                     candidates: Sequence[str], k: int = 5) -> Sequence[tuple[str, float]]
```

### 3.5 Fingerprint Measures

```python
class SimHash(Protocol[T]):
    """
    Locality-sensitive hash for near-duplicate detection.
    Two strings with Hamming distance ≤ threshold are near-duplicates.
    """
    def hash(self, document: T) -> int          # 64-bit hash
    def hamming_distance(self, a: int, b: int) -> int  # O(1) bit ops
    def are_near_duplicate(self, a: T, b: T,
                           threshold: int = 3) -> bool  # Hamming distance ≤ threshold

class MinHash(Protocol[T]):
    """
    MinHash sketching for Jaccard similarity estimation.
    Complexity: O(N × k) build, O(k) query — k = number of hash functions.
    Error bound: ~1/√k for k hash functions.
    """
    def __init__(self, num_hashes: int = 128)
    def signature(self, items: frozenset) -> Sequence[int]   # MinHash signature
    def estimated_jaccard(self, a: frozenset, b: frozenset) -> float
    def lsh_bands(self, bands: int, rows: int) -> "LSHIndex"

class LSHIndex(Protocol):
    """Locality Sensitive Hashing for approximate nearest neighbors."""
    def insert(self, id: str, signature: Sequence[int]) -> None
    def query(self, signature: Sequence[int]) -> Sequence[str]  # Candidate IDs
```

### 3.6 Composite Measure

```python
class WeightedCompositeMeasure(SimilarityMeasure[Any]):
    """
    Weighted linear combination of multiple similarity measures.
    Weights must sum to 1.0.
    """
    def add(self, measure: SimilarityMeasure,
            weight: float,
            projection: Callable[[Any], Any]) -> None
        # projection: extracts the relevant aspect for this measure
    def similarity(self, a: Any, b: Any) -> float
    def explain(self, a: Any, b: Any) -> ImmutableMap[str, float]
        # Per-measure contribution to final score
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-SIM-001 | `similarity(a, a) == 1.0` for all measures. |
| C-SIM-002 | `similarity(a, b) ∈ [0.0, 1.0]` for all similarity measures. |
| C-SIM-003 | `similarity(a, b) == similarity(b, a)` — symmetry. |
| C-SIM-004 | For metric measures: `distance(a, c) ≤ distance(a, b) + distance(b, c)` — triangle inequality. |
| C-SIM-005 | `distance(a, b) = 1.0 - similarity(a, b)` for normalized measures. |
| C-SIM-006 | `WeightedCompositeMeasure` weights sum to 1.0 before `similarity` is called. |
| C-SIM-007 | `TFIDFCosineSimilarity.similarity` raises `NotFittedError` if `fit` not called. |
| C-SIM-008 | `SimHash.hamming_distance(h, h) == 0` for any hash h. |
| C-SIM-009 | `MinHash.estimated_jaccard(A, B)` converges to `JaccardSimilarity.similarity(A, B)` as num_hashes → ∞. |
| C-SIM-010 | `LevenshteinDistance.distance("", s) == len(s)` and `distance(s, "") == len(s)`. |

---

## 5. Complexity Table

| Measure | Complexity | Space |
|---------|-----------|-------|
| Levenshtein | O(|a|×|b|) | O(min(|a|,|b|)) |
| Jaro / JaroWinkler | O(|a|+|b|) | O(max(|a|,|b|)) |
| Jaccard (set) | O(|A|+|B|) | O(|A|+|B|) |
| Cosine (vector) | O(N) | O(1) |
| TF-IDF Cosine | O(T) query | O(V·N) fit — V=vocab |
| SimHash | O(T) | O(1) per hash |
| MinHash signature | O(N×k) | O(k) |
| MinHash query | O(k) | — |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — vector sequences, frozenset types
- Statistics SDK (KA-SDK-013) — IDF computation for TF-IDF

---

## 7. Extension Points

```python
class SimilarityMeasureFactory(Protocol):
    """Creates measures from configuration."""
    def create(self, name: str, config: dict) -> SimilarityMeasure: ...

class EmbeddingProvider(Protocol):
    """Provides vector embeddings for semantic similarity."""
    def embed(self, text: str) -> Sequence[float]: ...
    def embed_batch(self, texts: Sequence[str]) -> Sequence[Sequence[float]]: ...
    def dimension(self) -> int: ...

class SimilarityThreshold(Protocol):
    """Determines similarity thresholds from context."""
    def threshold_for(self, measure_name: str,
                      use_case: str) -> float: ...
    # e.g. "levenshtein", "deduplication" → 0.85
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-SIM-001 | `levenshtein("kitten", "sitting")` = 3 |
| V-SIM-002 | `jaro("MARTHA", "MARHTA")` ≈ 0.944 |
| V-SIM-003 | `jaccard({1,2,3}, {2,3,4})` = 0.5 |
| V-SIM-004 | `cosine([1,0], [0,1])` = 0.0 |
| V-SIM-005 | `cosine([1,1], [1,1])` = 1.0 |
| V-SIM-006 | `similarity(a, a) == 1.0` for all measure types |
| V-SIM-007 | `similarity(a, b) == similarity(b, a)` — symmetry |
| V-SIM-008 | `SimHash.hamming_distance(h, h) == 0` |
| V-SIM-009 | `WeightedComposite` with equal weights: result = mean of component scores |
| V-SIM-010 | `levenshtein("", "abc")` = 3 |
