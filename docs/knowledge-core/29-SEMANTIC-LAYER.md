# KNW-KC-ARCH-029 — Semantic Layer

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Semantic Layer bridges natural language and structured KQL. It maps free-text questions to formal queries, computes semantic similarity between Knowledge Objects, and enables fuzzy matching that indexes alone cannot provide.

---

## Semantic Layer Functions

| Function | Description |
|----------|-------------|
| `NL → KQL` | Translate natural language question to KQL |
| `Embedding` | Convert text to vector representation |
| `Similarity` | Compute semantic similarity between objects |
| `Clustering` | Group semantically related objects |
| `Disambiguation` | Resolve ambiguous object references |
| `Suggestion` | Auto-complete partial queries |
| `Summarisation` | Generate natural language summary of a result set |

---

## Natural Language → KQL Translation

```
NL_TO_KQL(question: str) → KQLStatement

Pipeline:
  1. INTENT CLASSIFICATION
     Classify question into query type:
       - "What does X depend on?"      → DEPENDENCY_CLOSURE
       - "Show me all canonical APIs"  → SELECT with status filter
       - "What breaks if Y changes?"   → IMPACT query
       - "Find modules related to auth" → HYBRID SEARCH + type filter
       - "Why was ADR-001 made?"       → REASONING query

  2. ENTITY EXTRACTION
     Extract knowledge entities from question text:
       - Pattern match: "KNW-*", "knw://*", ADR-NNN, REQ-NNN
       - Fuzzy match on canonical_name via search index
       - Namespace detection from domain keywords

  3. KQL GENERATION
     Map (intent, entities) → KQL statement
     Examples:
       "What does the AI runtime depend on?"
         → "SELECT * FROM KNW-AI-RUN-001 TRAVERSE depends_on FORWARD DEPTH 10"
       "Show all stale evidence in kos namespace"
         → "EVIDENCE namespace='kos' WHERE freshness_score < 0.30"

  4. KQL VALIDATION
     Parse generated KQL; if invalid, retry with different template

  5. CONFIDENCE SCORE
     Report confidence that translation is correct: 0.0..1.0
     If confidence < 0.5: present alternatives to user
```

---

## Embedding Model

```
EMBEDDING:
  Model:      Platform-local embedding model (first choice)
              External embedding API (fallback)
  Dimensions: 768
  Input max:  2048 tokens
  Fields:     name + description + tags (concatenated, truncated)

EMBEDDING_CACHE:
  TTL:        24 hours after last object update
  Storage:    knowledge/indexes/embeddings/{knowledge_id}.npy
  Invalidate: on any update to name, description, or tags

BATCH_EMBED(objects):
  1. Filter to objects missing or expired embeddings
  2. Batch into groups of 64
  3. Call embedding model
  4. Store vectors
  5. Update HNSW index (28-SEARCH-ENGINE)
  COMPLEXITY: O(N * D) where D=768 dimensions
```

---

## Semantic Similarity

```python
def semantic_similarity(obj_a: KnowledgeObject, obj_b: KnowledgeObject) -> float:
    """Cosine similarity between embedding vectors."""
    vec_a = get_embedding(obj_a.knowledge_id)
    vec_b = get_embedding(obj_b.knowledge_id)
    return dot(vec_a, vec_b) / (norm(vec_a) * norm(vec_b))

# Ranges from -1.0 to 1.0; in practice 0.0 to 1.0 for KOS objects
# > 0.90: near-duplicate
# > 0.75: closely related
# > 0.50: topically related
# < 0.30: different domains
```

---

## Semantic Clustering

Objects are clustered into semantic groups to support browsing and discovery:

```
CLUSTER(objects, n_clusters):
  1. Compute embeddings for all objects
  2. k-means clustering (k=n_clusters)
  3. Label each cluster with top N terms (TF-IDF)
  4. Return: list[Cluster{label, objects, centroid}]

USE CASES:
  - Group related requirements before architecture review
  - Detect semantic duplicates across namespaces
  - Suggest related patterns when registering a new pattern
```

---

## Disambiguation

```
DISAMBIGUATE(query: str, candidates: list[KnowledgeObject]) → KnowledgeObject | list[KnowledgeObject]:
  If candidates has exactly 1 item with similarity > 0.85: return it
  If candidates has multiple items > 0.85: return all (user must choose)
  If candidates has 0 items > 0.70: return empty (no match)
```

---

## Query Suggestion

```
SUGGEST(partial: str, context: SearchRequest) → list[str]:
  1. Expand partial to candidate completions from:
     - canonical_name prefix matches (index)
     - tag prefix matches (index)
     - Recent queries from session
  2. Score by: frequency + semantic relevance to context.namespaces
  3. Return top 5 suggestions
```

---

## Semantic Layer Protocol

```python
class SemanticLayer(Protocol):
    def nl_to_kql(self, question: str) -> NLToKQLResult: ...
    def embed(self, knowledge_id: str) -> list[float]: ...
    def batch_embed(self, knowledge_ids: list[str]) -> dict[str, list[float]]: ...
    def similarity(self, id_a: str, id_b: str) -> float: ...
    def find_similar(self, knowledge_id: str, threshold: float, limit: int) -> list[str]: ...
    def cluster(self, namespace: str, n_clusters: int) -> list[SemanticCluster]: ...
    def disambiguate(self, query: str) -> list[KnowledgeObject]: ...
    def suggest(self, partial: str, limit: int) -> list[str]: ...
    def summarise(self, result_set: list[KnowledgeObject]) -> str: ...

@dataclass
class NLToKQLResult:
    question: str
    kql: str
    intent: str
    entities_found: list[str]
    confidence: float
    alternatives: list[str]    # alternative KQL phrasings
```

---

## Duplicate Detection via Semantic Layer

```
FIND_DUPLICATES(namespace):
  1. Get all objects in namespace
  2. Compute pairwise similarity for objects with same object_type
  3. Flag pairs with similarity > 0.90 as potential duplicates
  4. Emit: registry.duplicate_alert event for each pair
  5. Return: list[DuplicatePair{id_a, id_b, similarity}]

COMPLEXITY: O(N^2 * D) — run offline, not on every write
SCHEDULE: Nightly for each namespace
```

---

## Performance Requirements

| Operation | P50 | P99 |
|-----------|-----|-----|
| NL → KQL translation | < 200ms | < 1s |
| Single embedding | < 50ms | < 200ms |
| Batch embed (64 objects) | < 500ms | < 2s |
| Semantic similarity (pair) | < 1ms | < 5ms |
| Similar objects (top-20) | < 50ms | < 200ms |
| Query suggestion | < 10ms | < 50ms |

---

## Cross-References

- NL queries execute via → `27-QUERY-LANGUAGE`
- Embeddings stored in → `26-GRAPH-INDEXES`
- Semantic search mode → `28-SEARCH-ENGINE`
- Reasoning uses semantic similarity → `30-REASONING-MODEL`
