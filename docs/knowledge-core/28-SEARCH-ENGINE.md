# KNW-KC-ARCH-028 — Search Engine

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Search Engine provides full-text, semantic, and structured search over all Knowledge Objects. It extends the KQL query engine with ranking, relevance scoring, and faceted filtering.

---

## Search Modes

| Mode | Description | Backend |
|------|-------------|---------|
| `STRUCTURED` | Exact match on indexed fields | KQL + Indexes |
| `FULL_TEXT` | Text search across name, description, metadata | Inverted index |
| `SEMANTIC` | Embedding-based similarity search | Vector index |
| `HYBRID` | Weighted combination of full_text + semantic | Both |
| `GRAPH` | Search constrained to a graph neighbourhood | KQL + Traversal |

---

## Search Request Schema

```python
@dataclass
class SearchRequest:
    query: str                              # text query or KQL
    mode: SearchMode = SearchMode.HYBRID
    namespaces: list[str] = field(default_factory=list)
    object_types: list[KnowledgeObjectType] = field(default_factory=list)
    status_filter: list[LifecycleStatus] = field(default_factory=list)
    owner_filter: list[str] = field(default_factory=list)
    tag_filter: list[str] = field(default_factory=list)
    min_quality_score: float = 0.0
    min_confidence: float = 0.0
    graph_root: str | None = None           # for GRAPH mode
    graph_depth: int = 2
    limit: int = 20
    offset: int = 0
    sort_by: SearchSortField = SearchSortField.RELEVANCE
    include_deprecated: bool = False
```

---

## Search Result Schema

```python
@dataclass
class SearchResult:
    knowledge_id: str
    canonical_name: str
    knowledge_uri: str
    object_type: KnowledgeObjectType
    namespace: str
    name: str
    description: str
    status: LifecycleStatus
    version: str
    quality_score: float | None
    confidence: float
    relevance_score: float                  # 0.0..1.0 — ranking signal
    text_score: float                       # full-text match score
    semantic_score: float                   # embedding similarity
    match_highlights: dict[str, str]        # field → highlighted excerpt
    tags: list[str]

@dataclass
class SearchResponse:
    results: list[SearchResult]
    total_count: int
    returned_count: int
    execution_ms: float
    query_parsed: str
    facets: dict[str, list[FacetBucket]]
    suggestions: list[str]
```

---

## Full-Text Index

```
INVERTED_INDEX:
  Documents: all KnowledgeObject text fields
    - canonical_name (weight: 3.0)
    - name (weight: 2.5)
    - aliases (weight: 2.0)
    - description (weight: 1.5)
    - tags (weight: 2.0)
    - metadata.purpose (weight: 1.0)
    - metadata.constraints (weight: 0.5)

  Tokenization:
    - Lowercase
    - Whitespace + punctuation split
    - Stop words removed (english)
    - Stemming: Porter stemmer
    - Knowledge IDs indexed as single tokens (no split)

  Scoring: BM25(k1=1.2, b=0.75)
```

---

## Semantic Index (Vector Search)

```
EMBEDDING_MODEL:
  Source: platform embedded model OR external embedding API
  Dimensions: 768
  Fields embedded: name + description (concatenated)
  Similarity: cosine

INDEX_TYPE: HNSW (Hierarchical Navigable Small World)
  ef_construction: 200
  M: 16
  ef_search: 100

QUERY FLOW:
  1. Embed query text → vector q (768 dims)
  2. HNSW search → top-K nearest nodes (K = limit * 3)
  3. Filter by namespace/type/status
  4. Score: semantic_score = cosine_similarity(q, node_embedding)
```

---

## Hybrid Scoring

```
relevance_score =
  text_weight    * text_score
  + semantic_weight * semantic_score
  + quality_boost

Default weights:
  text_weight:     0.40
  semantic_weight: 0.40
  quality_boost:   quality_score * 0.20

quality_boost nudges high-quality objects up — but cannot overcome
a very low text or semantic score.
```

---

## Faceted Search

Search returns facet counts for:
```yaml
facets:
  object_type:
    ARCHITECTURE: 12
    MODULE: 45
    SERVICE: 6
  namespace:
    kos: 18
    platform: 32
    ai: 13
  status:
    CANONICAL: 8
    APPROVED: 15
    VERIFIED: 30
  owner:
    "team:architecture-board": 18
    "team:ai-runtime": 22
```

---

## Search Engine Protocol

```python
class KnowledgeSearchEngine(Protocol):
    def search(self, request: SearchRequest) -> SearchResponse: ...
    def suggest(self, partial: str, limit: int) -> list[str]: ...
    def similar(self, knowledge_id: str, limit: int) -> list[SearchResult]: ...
    def index_object(self, obj: KnowledgeObject) -> None: ...
    def remove_from_index(self, knowledge_id: str) -> None: ...
    def rebuild_index(self) -> None: ...
    def get_index_stats(self) -> SearchIndexStats: ...
```

---

## Index Rebuild Triggers

| Trigger | Action |
|---------|--------|
| `knowledge.object.registered` | Add to full-text + semantic index |
| `knowledge.object.updated` | Re-index updated fields |
| `knowledge.object.deprecated` | Remove from default search; keep in INCLUDE_DEPRECATED index |
| `knowledge.object.tombstoned` | Remove from all indexes |

---

## Performance Requirements

| Operation | P50 | P99 |
|-----------|-----|-----|
| Structured search | < 5ms | < 20ms |
| Full-text search (10K docs) | < 30ms | < 100ms |
| Semantic search (1M docs) | < 50ms | < 200ms |
| Hybrid search | < 100ms | < 500ms |
| Index single object | < 10ms | < 50ms |
| Full index rebuild (1M objects) | < 60min | — |

---

## Cross-References

- KQL used for structured mode → `27-QUERY-LANGUAGE`
- Semantic embedding → `29-SEMANTIC-LAYER`
- Search indexes built on graph indexes → `26-GRAPH-INDEXES`
- Registry events trigger re-indexing → `18-EVENT-REGISTRY`
