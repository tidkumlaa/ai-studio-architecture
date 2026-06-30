# KNW-KC-ARCH-036 — Algorithms

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

This document specifies all non-graph algorithms used in the Knowledge Core — identity computation, quality scoring, evidence freshness, confidence decay, registry operations, and search ranking. Graph algorithms are in `25-GRAPH-ALGORITHMS`.

---

## A-01: Knowledge ID Generation

```
GENERATE_KNOWLEDGE_ID(domain_code, type_code, registry):
  sequence = registry.next_sequence(domain_code, type_code)
  return f"KNW-{domain_code}-{type_code}-{sequence:03d}"

COMPLEXITY: O(1) amortised (sequence counter increment)
```

---

## A-02: Checksum Computation

```
COMPUTE_CHECKSUM(obj: KnowledgeObject) → str:
  canonical = json.dumps(obj.to_content_dict(), sort_keys=True, ensure_ascii=True)
  return "sha256:" + hashlib.sha256(canonical.encode()).hexdigest()

CONTENT_DICT excludes: checksum, fingerprint, operational metadata (read_count, etc.)
COMPLEXITY: O(L) where L = content length in bytes
```

---

## A-03: Fingerprint Computation

```
COMPUTE_FINGERPRINT(domain, type_code, checksum) → str:
  raw = f"{domain}|{type_code}|{checksum}"
  digest = hashlib.sha256(raw.encode()).hexdigest()[:16]
  return f"fp:{domain}:{type_code}:{digest}"

PURPOSE: Detect semantic duplicates (same content, different ID)
COMPLEXITY: O(1) given checksum
```

---

## A-04: Quality Score Computation

From `11-QUALITY-ENGINE`:

```
COMPUTE_QUALITY(obj) → QualityScoreRecord:
  qd1 = completeness(obj)          O(F) where F = field count
  qd2 = consistency(obj)           O(C) where C = check count
  qd3 = coverage(obj)              O(R) where R = requirement count
  qd4 = freshness(obj)             O(1)
  qd5 = evidence_score(obj)        O(E) where E = evidence count
  qd6 = confidence(obj)            O(1)
  qd7 = usage(obj)                 O(1)  (uses pre-computed index)
  qd8 = reasoning(obj) if DECISION else 1.0
  qd9 = health(obj)                O(R + D) where R=relations, D=deps

  weighted = Σ(qdi * wi)           O(1) given computed dimensions
  overall  = weighted * health_penalty

TOTAL COMPLEXITY: O(F + C + R + E)
```

---

## A-05: Evidence Freshness Decay

```
FRESHNESS(evidence_record, now) → float:
  age_days = (now - evidence_record.source_captured_at).days
  threshold = FRESHNESS_THRESHOLD[evidence_record.evidence_type]
  decay_period = DECAY_PERIOD[evidence_record.evidence_type]

  if age_days <= threshold:
    return 1.0
  return max(0.0, 1.0 - (age_days - threshold) / decay_period)

COMPLEXITY: O(1)
```

---

## A-06: Confidence Decay

```
DECAYED_CONFIDENCE(obj, now) → float:
  half_life = HALF_LIFE[obj.classification.domain]
  age_days = (now - obj.identity.updated_at).days

  if age_days <= half_life:
    recency = 1.0
  else:
    recency = exp(-0.693 * (age_days - half_life) / half_life)

  evidence_freshness = avg(freshness(e) for e in obj.evidence.items)
  return obj.composite_confidence * evidence_freshness * recency

COMPLEXITY: O(E) where E = evidence record count
```

---

## A-07: BM25 Text Ranking

```
BM25_SCORE(query_terms, doc, index, k1=1.2, b=0.75) → float:
  score = 0
  avg_doc_len = index.avg_document_length()
  for term in query_terms:
    tf = doc.term_frequency(term)
    df = index.document_frequency(term)
    idf = log((N - df + 0.5) / (df + 0.5) + 1)
    numerator = tf * (k1 + 1)
    denominator = tf + k1 * (1 - b + b * doc.length / avg_doc_len)
    score += idf * (numerator / denominator) * FIELD_WEIGHT[field]
  return score

COMPLEXITY: O(T * log N) where T = query terms, N = total documents
```

---

## A-08: Hybrid Search Scoring

```
HYBRID_SCORE(text_score, semantic_score, quality_score,
             text_w=0.40, semantic_w=0.40, quality_w=0.20) → float:
  return text_w * text_score + semantic_w * semantic_score + quality_w * quality_score

COMPLEXITY: O(1) given component scores
```

---

## A-09: Traceability Coverage

```
TRACEABILITY_COVERAGE(requirements, registry) → float:
  full_chain_count = 0
  for req in requirements:
    chain = FORWARD_TRACE(req.knowledge_id)  # from 13-TRACEABILITY
    if chain.has_min_path():    # req → arch → module → test
      full_chain_count += 1

  return full_chain_count / len(requirements) if requirements else 0.0

COMPLEXITY: O(R * (N + E)) where R = requirement count
```

---

## A-10: Confidence Path Product

```
PATH_CONFIDENCE(path: list[str], edge_confidences: dict) → float:
  node_confidences = [registry.get(n).composite_confidence for n in path]
  edge_conf_list = [edge_confidences.get((path[i], path[i+1]), 1.0)
                    for i in range(len(path)-1)]

  node_factor = geometric_mean(node_confidences)   # nth root of product
  edge_factor = product(edge_conf_list)

  return node_factor * edge_factor

COMPLEXITY: O(P) where P = path length
```

---

## A-11: Semantic Similarity (Cosine)

```
COSINE_SIMILARITY(vec_a: list[float], vec_b: list[float]) → float:
  dot = sum(a * b for a, b in zip(vec_a, vec_b))
  norm_a = sqrt(sum(a**2 for a in vec_a))
  norm_b = sqrt(sum(b**2 for b in vec_b))
  return dot / (norm_a * norm_b) if norm_a and norm_b else 0.0

COMPLEXITY: O(D) where D = embedding dimensions (768)
```

---

## A-12: Registry Bulk Validation

```
BULK_VALIDATE(registry, namespace) → ValidationReport:
  violations = []
  for obj in registry.list_all(filters={namespace: namespace}):
    result = validate_universal_schema(obj)     # O(F)
    if not result.valid:
      violations.append(ValidationViolation(obj.knowledge_id, result.errors))

  return ValidationReport(
    total=count, valid=count-len(violations), violations=violations
  )

COMPLEXITY: O(N * F) where N = object count, F = field count
```

---

## Algorithm Complexity Summary

| Algorithm | Time | Space | Notes |
|-----------|------|-------|-------|
| A-01 ID Generation | O(1) | O(1) | Sequence counter |
| A-02 Checksum | O(L) | O(L) | L = content length |
| A-03 Fingerprint | O(1) | O(1) | Given checksum |
| A-04 Quality Score | O(F+C+R+E) | O(1) | Uses indexes |
| A-05 Evidence Freshness | O(1) | O(1) | Table lookup |
| A-06 Confidence Decay | O(E) | O(1) | |
| A-07 BM25 | O(T log N) | O(N) | T=terms, N=docs |
| A-08 Hybrid Score | O(1) | O(1) | Given components |
| A-09 Traceability Coverage | O(R(N+E)) | O(N) | |
| A-10 Path Confidence | O(P) | O(P) | |
| A-11 Cosine Similarity | O(D) | O(D) | D=768 |
| A-12 Bulk Validation | O(N*F) | O(N) | |

---

## Cross-References

- Graph algorithms (DFS, BFS, etc.) → `25-GRAPH-ALGORITHMS`
- These algorithms registered in → `21-ALGORITHM-REGISTRY`
- Performance budgets → `37-PERFORMANCE-BUDGET`
- Quality dimensions → `11-QUALITY-ENGINE`
