---
knowledge_id: KA-SDK-005
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
    reason: "SearchResult collections"
  - id: KA-SDK-006
    reason: "Index SDK for inverted index backend"
  - id: KA-SDK-014
    reason: "Similarity SDK for scoring"
---

# Search SDK

## Pluggable Multi-Strategy Search — Lexical · Semantic · Fuzzy · Hybrid

---

## 1. Purpose

The Search SDK provides a provider-agnostic, composable search abstraction
covering tokenization, indexing, scoring, ranking, and result assembly. It
unifies lexical (BM25), fuzzy (edit-distance), prefix (trie), and semantic
(vector) strategies behind a single interface. Consumers choose strategies;
the SDK handles composition.

---

## 2. Search Architecture

```
Query (string)
    │
    ▼ Tokenizer
Tokens
    │
    ▼ QueryExpander (optional: synonyms, stemming)
Expanded Tokens
    │
    ├──▶ LexicalEngine (BM25, TF-IDF)
    ├──▶ PrefixEngine (Trie)
    ├──▶ FuzzyEngine (BK-Tree, Edit Distance)
    └──▶ SemanticEngine (Vector Similarity) [pluggable]
         │
    ▼ ScoreFusion (weighted combination)
Fused Scores
    │
    ▼ Ranker
Ranked Results
    │
    ▼ ResultAssembler (snippet extraction, facets)
SearchResponse
```

---

## 3. Interfaces

### 3.1 Tokenizer

```python
class Tokenizer(Protocol):
    def tokenize(self, text: str) -> Sequence[str]  # O(|text|)
    def tokenize_query(self, query: str) -> Sequence[str]

# Built-in tokenizers:
# WhitespaceTokenizer    — split on whitespace
# AlphanumericTokenizer  — regex [a-z0-9]+
# NgramTokenizer         — character n-grams
# SubwordTokenizer       — BPE-style subword units (pluggable)
```

### 3.2 Indexable

```python
class Indexable(Protocol):
    """Any document type indexable by the Search SDK."""
    def doc_id(self) -> str
    def doc_text(self) -> str           # Primary text field for indexing
    def doc_fields(self) -> dict[str, str]  # Named fields for field-scoped search
```

### 3.3 SearchEngine

```python
class SearchEngine(Protocol):
    def index(self, document: Indexable) -> None        # O(|text| · T) T=tokens
    def remove(self, doc_id: str) -> None               # O(T)
    def update(self, document: Indexable) -> None       # remove + index
    def search(self, query: str, limit: int = 20,
               filters: SearchFilters | None = None) -> SearchResponse
    def doc_count(self) -> int                          # O(1)
    def clear(self) -> None
```

### 3.4 SearchFilters

```python
@dataclass(frozen=True)
class SearchFilters:
    fields:       dict[str, str] | None     # field=value constraints
    min_score:    float | None              # score threshold
    doc_ids:      frozenset[str] | None     # restrict to these IDs
    max_results:  int = 20
    offset:       int = 0                  # pagination
```

### 3.5 SearchResponse

```python
@dataclass(frozen=True)
class SearchResult:
    doc_id:        str
    score:         float
    snippet:       str
    matched_terms: Sequence[str]
    field_scores:  ImmutableMap[str, float]  # per-field breakdown

@dataclass(frozen=True)
class SearchResponse:
    query:         str
    results:       Sequence[SearchResult]
    total_hits:    int
    elapsed_ms:    float
    strategies_used: Sequence[str]
    facets:        ImmutableMap[str, int]    # term → document count
```

### 3.6 SearchStrategy (pluggable per-engine)

```python
class SearchStrategy(Protocol):
    name: str
    weight: float  # 0.0 – 1.0 in hybrid composition

    def score(self, doc_id: str, query: str) -> float
    def candidates(self, query: str, limit: int) -> Sequence[str]  # doc IDs
    def index(self, document: Indexable) -> None
    def remove(self, doc_id: str) -> None
```

### 3.7 HybridSearchEngine

```python
class HybridSearchEngine(SearchEngine):
    """Composes multiple SearchStrategy implementations."""
    def add_strategy(self, strategy: SearchStrategy) -> None
    def remove_strategy(self, name: str) -> None
    def set_weight(self, name: str, weight: float) -> None
    def normalize_weights(self) -> None  # ensures weights sum to 1.0
    def explain(self, doc_id: str, query: str) -> ScoreExplanation

@dataclass(frozen=True)
class ScoreExplanation:
    doc_id: str
    query: str
    total_score: float
    per_strategy: ImmutableMap[str, float]
    per_term: ImmutableMap[str, float]
```

### 3.8 Ranker

```python
class Ranker(Protocol):
    """Re-ranks results after initial scoring."""
    def rank(self, results: Sequence[SearchResult],
             context: RankingContext | None = None) -> Sequence[SearchResult]

@dataclass(frozen=True)
class RankingContext:
    user_history: Sequence[str]     # previously accessed doc IDs
    recency_weight: float           # boost for recently modified docs
    domain_filter: str | None       # restrict to domain
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-SRC-001 | `search("")` returns empty results, not an error. |
| C-SRC-002 | Strategy weights in `HybridSearchEngine` must sum to 1.0 after normalization. |
| C-SRC-003 | Score is always in range [0.0, ∞). Zero means no match; negative scores are not allowed. |
| C-SRC-004 | `remove(doc_id)` on non-existent doc is a no-op. |
| C-SRC-005 | `facets` in `SearchResponse` are computed from the full result set before pagination. |
| C-SRC-006 | `SearchResult.matched_terms` contains only terms that contributed to the score. |
| C-SRC-007 | `explain()` must account for 100% of the total score across per-strategy breakdown. |
| C-SRC-008 | Indexing the same `doc_id` twice replaces the previous document. |
| C-SRC-009 | `limit` and `offset` together implement stateless pagination. |
| C-SRC-010 | `Ranker` is a post-processor: it reorders but does not add or remove results. |

---

## 5. Algorithm Complexity

| Operation | Strategy | Complexity |
|-----------|---------|------------|
| `index` | Any | O(T) — T = token count |
| `search` (BM25) | Lexical | O(T · P) — P = postings per term |
| `search` (Trie) | Prefix | O(k + results) — k = prefix length |
| `search` (BK-Tree) | Fuzzy | O(log N) expected |
| `search` (Hybrid) | Combined | O(max of strategies) |
| `remove` | Any | O(T) |
| `explain` | Hybrid | O(S · T) — S = strategies |
| `rank` | Any Ranker | O(N log N) sort |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — result sequences, filter maps
- Index SDK (KA-SDK-006) — inverted index, B+Tree for prefix search
- Similarity SDK (KA-SDK-014) — edit-distance, cosine for scoring

---

## 7. Extension Points

```python
class SearchStrategyFactory(Protocol):
    """Produces SearchStrategy instances from configuration."""
    def create(self, name: str, config: dict) -> SearchStrategy: ...

class QueryExpander(Protocol):
    """Expands query tokens (synonyms, stemming, spell correction)."""
    def expand(self, tokens: Sequence[str]) -> Sequence[str]: ...

class SnippetExtractor(Protocol):
    """Extracts relevant snippet from document content."""
    def extract(self, content: str, matched_terms: Sequence[str],
                window: int = 150) -> str: ...

class SemanticSearchStrategy(SearchStrategy):
    """Extension point for vector/embedding-based semantic search."""
    def embed(self, text: str) -> Sequence[float]: ...
    def embed_batch(self, texts: Sequence[str]) -> Sequence[Sequence[float]]: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-SRC-001 | `search("")` returns empty `SearchResponse` |
| V-SRC-002 | After `index(doc)`, `search(doc.doc_text())` returns that document |
| V-SRC-003 | After `remove(id)`, `search` no longer returns that document |
| V-SRC-004 | Hybrid weights sum to 1.0 after `normalize_weights()` |
| V-SRC-005 | `explain` per-strategy scores sum to total_score |
| V-SRC-006 | `offset=N` skips first N results |
| V-SRC-007 | Re-indexing same ID replaces (not duplicates) the document |
| V-SRC-008 | `facets` count matches number of results containing that term |
| V-SRC-009 | `min_score` filter removes all results below threshold |
| V-SRC-010 | `SearchResponse.total_hits` reflects pre-pagination count |
