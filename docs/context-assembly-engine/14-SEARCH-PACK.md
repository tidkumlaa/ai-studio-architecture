# CAE-DOC-014 — Search Pack

**Phase:** 3.0D.1
**Status:** CANONICAL
**Owner:** architecture-board
**Version:** 1.0.0

---

## Purpose

The Search Pack is the output format for search UI consumers — dashboards,
IDE plugins, documentation portals, and discovery tools. It returns a ranked
list of compact result cards, each containing just enough information for a
human to decide whether to click through.

SearchPack is the default output when `consumer_profile.type == SEARCH_UI`
or `intent_record.intent_type == INT-06 SEARCH`.

---

## Search Pack Schema

```yaml
search_pack:
  pack_id: string
  pack_type: SEARCH
  query_id: string
  query_text: string
  generated_at: string

  result_count: integer
  total_candidates: integer            # before ranking
  elapsed_ms: integer                  # time to produce this pack

  facets:
    types: {string: integer}           # e.g. {"SERVICE": 12, "MODULE": 4}
    namespaces: {string: integer}
    quality_bands: {string: integer}   # "EXCELLENT": 5, "GOOD": 10, ...

  results:
    - rank: integer
      knowledge_id: string
      canonical_name: string
      object_type: string
      namespace: string
      status: string
      purpose: string                  # one sentence, from ai_context.short_summary
      quality_score: float
      ai_readiness_score: float
      confidence_level: string
      relevance_score: float
      match_reason: string             # human-readable explanation of why matched

  spelling_correction: string | null   # corrected query if applied
  empty_reason: string | null          # why results is empty (if empty)
  next_query_suggestions: [string]     # KIL ai_context.questions relevant to top result
```

---

## SearchPack Build Algorithm

```
BUILD_SEARCH_PACK(ranked_objects, intent_record, query_context):

  pack = SearchPack(
    pack_id = uuid4(),
    pack_type = SEARCH,
    query_id = intent_record.query_id,
    query_text = intent_record.raw_query,
    generated_at = now()
  )

  pack.total_candidates = len(ranked_objects.all_candidates)
  pack.elapsed_ms = query_context.elapsed_ms

  # Build result cards (top 10 max)
  results = []
  for rank, obj in enumerate(ranked_objects.selected[:10], start=1):
    entry = INDEX.get(obj.knowledge_id)
    results.append(SearchResult(
      rank = rank,
      knowledge_id = obj.knowledge_id,
      canonical_name = entry.canonical_name,
      object_type = entry.object_type,
      namespace = entry.namespace,
      status = entry.metadata.state,
      purpose = entry.micro,
      quality_score = entry.quality_score,
      ai_readiness_score = entry.ai_readiness_score,
      confidence_level = entry.confidence_level,
      relevance_score = obj.relevance_score,
      match_reason = EXPLAIN_MATCH(obj, intent_record)
    ))
  pack.results = results
  pack.result_count = len(results)

  # Build facets from all candidates (not just selected)
  pack.facets = {
    types: count_by(ranked_objects.all_candidates, "object_type"),
    namespaces: count_by(ranked_objects.all_candidates, "namespace"),
    quality_bands: count_by_quality(ranked_objects.all_candidates)
  }

  # Next query suggestions from top result
  IF results:
    top = INDEX.get(results[0].knowledge_id)
    pack.next_query_suggestions = top.ai_context.questions[:3]

  IF not results:
    pack.empty_reason = derive_empty_reason(intent_record, ranked_objects)

  RETURN pack
```

---

## Match Reason Generation

```
EXPLAIN_MATCH(candidate, intent_record):

  reasons = []
  tokens = intent_record.parsed_tokens

  IF candidate.semantic_similarity >= 0.80:
    reasons.append("semantically similar to query")

  IF candidate.keyword_score >= 0.70:
    matched = [t for t in tokens WHERE t in candidate.primary_terms]
    reasons.append(f"keyword match: {', '.join(matched)}")

  IF candidate.graph_proximity >= 0.50:
    reasons.append("directly related to a matched object")

  IF candidate.co_occurrence_rate >= 0.40:
    reasons.append("frequently co-accessed with search target")

  RETURN "; ".join(reasons) or "partial relevance"
```

---

## Quality Band Mapping

```
count_by_quality(candidates):
  EXCELLENT:  quality_score >= 0.90
  GOOD:       quality_score >= 0.80
  ACCEPTABLE: quality_score >= 0.70
  POOR:       quality_score < 0.70
```

---

## Empty Reason Derivation

```
derive_empty_reason(intent_record, ranked_objects):

  IF ranked_objects.total_count == 0:
    return "No objects matched the query terms."

  IF all dropped due to AIRS_TOO_LOW:
    return "Objects found but all have AIRS < 0.60 (excluded from AI context)."

  IF all dropped due to quality < threshold:
    return "Objects found but quality below acceptable threshold."

  return "Query matched no objects in the knowledge index."
```

---

## Rendered Output Format

```
SEARCH: "{query_text}"
{result_count} results (from {total_candidates} candidates, {elapsed_ms}ms)

Filters: Type [{types}] | Namespace [{namespaces}] | Quality [{quality_bands}]

──────────────────────────────
#1 [{relevance_score:.2f}] {canonical_name} ({object_type})
   Namespace: {namespace} | Status: {status} | Q: {quality_score:.2f}
   {purpose}
   Why matched: {match_reason}

#2 ...
──────────────────────────────

Next queries you might try:
  - {next_query_suggestions}

{empty_reason if no results}
```

---

## Rules

| Rule | Statement |
|------|-----------|
| CAE-061 | SearchPack must return at most 10 results — not paginated at this layer |
| CAE-062 | SearchPack must not include a guard block — it is not a reasoning context |
| CAE-063 | `match_reason` must be human-readable — never internal field names |
| CAE-064 | Facets are built from all candidates, not just selected results |
| CAE-065 | If results are empty, `empty_reason` must be set — a blank result without reason is forbidden |

---

## Cross-References

- Relevance model → `06-RELEVANCE-MODEL`
- Object selector → `05-OBJECT-SELECTOR`
- Intent model → `03-INTENT-MODEL` (INT-06 SEARCH)
- Context validator → `16-CONTEXT-VALIDATOR`
- KIL search optimization → Phase 3.0D.0.6 `28-KNOWLEDGE-SEARCH-OPTIMIZATION`
