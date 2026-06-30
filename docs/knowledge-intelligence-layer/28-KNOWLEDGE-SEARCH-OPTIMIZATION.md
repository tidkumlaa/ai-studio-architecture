# KNW-KIL-DOC-028 — Knowledge Search Optimization

**Phase:** 3.0D.0.6
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

Every Knowledge Object must be optimized for all 6 search modalities: semantic,
vector, hybrid, graph, keyword, and reasoning search. The intelligence block
carries per-object search optimization metadata that the Knowledge Runtime uses
to build and maintain the search index.

This document defines `intelligence.search`.

---

## Search Modalities

| Modality | How It Works | Best For |
|----------|-------------|---------|
| KEYWORD | Exact/prefix match on field values | Exact ID lookup, canonical_name search |
| SEMANTIC | Meaning-based match using embeddings | "what enforces quotas?" |
| VECTOR | Embedding similarity | Finding semantically similar objects |
| GRAPH | Traversal-based discovery | "what depends on X?" |
| HYBRID | KEYWORD + SEMANTIC + rank fusion | General-purpose search |
| REASONING | Multi-step inference to find answer | "why does Y need Z?" |

---

## Schema

```yaml
intelligence:
  search:
    schema_version: "1.0"

    keyword:
      primary_terms: [string]          # canonical search terms for this object
      aliases: [string]                # other names users might search for
      abbreviations: [string]          # short forms
      negative_keywords: [string]      # terms that should NOT match (disambiguation)
      boost_fields:                    # fields with boosted relevance
        - field: string
          boost: float                 # 1.0 = normal; > 1.0 = boosted

    semantic:
      search_phrases: [string]         # natural language phrases that should find this
      concept_clusters: [string]       # conceptual clusters this belongs to
      domain_vocabulary: [string]      # domain-specific terms that map to this
      embedding_fields: [string]       # which text fields to include in embedding
      embedding_priority_weight: float # relative importance in hybrid search (0.0–1.0)

    vector:
      embedding_id: string             # ID of the vector in the vector store
      embedding_model: string          # model used
      dimensions: integer
      last_embedded: string            # ISO 8601
      similarity_threshold: float      # minimum similarity to return in results (0.0–1.0)
      nearest_neighbors: [string]      # top-5 nearest knowledge_ids by vector distance

    graph:
      traversal_roles:                 # how this node appears in graph searches
        - role: SOURCE | SINK | HUB | TRANSIT
          for_relationship: string     # relationship type
          description: string
      reachability:
        can_reach: [string]            # knowledge_ids reachable from this node
        reached_by: [string]           # knowledge_ids that can reach this node
      hub_score: float                 # 0.0–1.0 (1 = highly connected hub)

    hybrid:
      default_rank_strategy: BM25_PLUS_VECTOR | VECTOR_ONLY | BM25_ONLY | RRF
      keyword_weight: float            # weight for keyword score (0.0–1.0)
      semantic_weight: float           # weight for semantic score (0.0–1.0)
      graph_boost: float               # extra boost for graph-connected results

    reasoning:
      answerable_questions: [string]   # types of reasoning questions this answers
      reasoning_chain_role: PREMISE | CONCLUSION | INTERMEDIATE | BRIDGE
      inference_paths: [string]        # reasoning paths this object enables
```

---

## Canonical Example

```yaml
# For: KNW-PLT-MOD-001 (Quota Manager)
intelligence:
  search:
    schema_version: "1.0"

    keyword:
      primary_terms:
        - "quota manager"
        - "KNW-PLT-MOD-001"
        - "plt.module.quota-manager"
        - "quota enforcement"
      aliases:
        - "quota enforcer"
        - "resource limiter"
        - "tenant quota"
      abbreviations:
        - "QM"
        - "quota-mgr"
      negative_keywords:
        - "rate limiter"
        - "rate limit"
        - "tokens per second"
      boost_fields:
        - field: "metadata.canonical_name"
          boost: 3.0
        - field: "intelligence.ai_context.short_summary"
          boost: 2.0
        - field: "intelligence.semantic.capabilities[*].name"
          boost: 1.5

    semantic:
      search_phrases:
        - "what enforces resource quotas?"
        - "how does tenant quota work?"
        - "which module prevents resource overconsumption?"
        - "guard against tenant resource starvation"
        - "per-tenant CPU memory request storage limits"
        - "quota check before resource allocation"
      concept_clusters:
        - "multi-tenant governance"
        - "resource enforcement"
        - "platform protection"
        - "guard pattern"
      domain_vocabulary:
        - "fair use enforcement"
        - "tenant isolation"
        - "quota boundary"
        - "resource allocation gate"
      embedding_fields:
        - "intelligence.ai_context.medium_summary"
        - "intelligence.semantic.capabilities[*].description"
        - "intelligence.cortex.why.statement"
        - "intelligence.self_describing.purpose"
      embedding_priority_weight: 0.75

    vector:
      embedding_id: "vec-knw-plt-mod-001-v2.1"
      embedding_model: "text-embedding-3-small"
      dimensions: 1536
      last_embedded: "2026-06-30T00:00:00Z"
      similarity_threshold: 0.72
      nearest_neighbors:
        - KNW-ALG-ALG-009      # Rate Limiter (similar domain, frequently confused)
        - KNW-META-PAT-003     # Guard pattern (structural similarity)
        - KNW-PLT-SVC-002      # Primary consumer
        - KNW-PROV-MOD-012     # Provider-scoped equivalent
        - KNW-PLT-REQ-001      # Requirement it implements

    graph:
      traversal_roles:
        - role: HUB
          for_relationship: DEPENDENCY_OF
          description: "18 objects depend on this — major hub for platform services"
        - role: SINK
          for_relationship: IMPLEMENTS
          description: "Terminal node for KNW-PLT-REQ-001 implementation chain"
        - role: SOURCE
          for_relationship: DEPENDS_ON
          description: "Originates dependencies to Registry and Config"
      reachability:
        can_reach: [KNW-RT-RT-001, KNW-PLT-CFG-001, KNW-PLT-MON-001]
        reached_by: [KNW-PLT-SVC-002, KNW-API-API-001, KNW-PLT-REQ-001]
      hub_score: 0.82

    hybrid:
      default_rank_strategy: BM25_PLUS_VECTOR
      keyword_weight: 0.35
      semantic_weight: 0.65
      graph_boost: 0.10        # small boost when graph-connected to query target

    reasoning:
      answerable_questions:
        - "What enforces quota limits?"
        - "What prevents tenant resource overconsumption?"
        - "What is the Guard module in the plt namespace?"
        - "What does the Quota Manager depend on?"
        - "What happens when Registry is unavailable?"
      reasoning_chain_role: BRIDGE
      inference_paths:
        - "KNW-PLT-REQ-001 → implements → KNW-PLT-MOD-001 → enforces → quota limits"
        - "KNW-RT-RT-001 → supports → KNW-PLT-MOD-001 → checks → quotas"
```

---

## Search Query Routing

```
Incoming query → route to modality:

Contains knowledge_id pattern (KNW-*)  → KEYWORD (exact)
Contains canonical_name (*.*.*)        → KEYWORD (exact)
Natural language question              → HYBRID (default)
"what depends on X?"                   → GRAPH
"how does X relate to Y?"              → GRAPH + REASONING
"is X the same as Y?"                  → VECTOR (similarity)
"why does X exist?"                    → REASONING
```

---

## Rules

| Rule | Statement |
|------|-----------|
| KIL-140 | Every CANONICAL object must have `keyword.primary_terms` with at least its knowledge_id and canonical_name |
| KIL-141 | `keyword.negative_keywords` must list every object in `semantic.conflicts` that shares terminology |
| KIL-142 | `vector.embedding_id` must be updated whenever `embedding_fields` content changes |
| KIL-143 | `hybrid.keyword_weight + hybrid.semantic_weight` must equal 1.0 (graph_boost is additive) |
| KIL-144 | `nearest_neighbors` must include any object in `alternatives` with `similarity_to_this > 0.50` |

---

## Cross-References

- Optimization hints → `24-KNOWLEDGE-OPTIMIZATION`
- Canonical index → `29-KNOWLEDGE-CANONICAL-INDEX`
- Semantic knowledge → `03-SEMANTIC-KNOWLEDGE`
- Learning (best prompt patterns) → `23-KNOWLEDGE-LEARNING-LAYER`
- Search certification → Phase 3.0D.0 `02-SEARCH-CERTIFICATION`
