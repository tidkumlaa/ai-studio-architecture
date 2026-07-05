---
knowledge_id: KA-KIP-005
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.2
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-intelligence
type: specification
---

# Knowledge Search

## Trie · Inverted Index · BM25 · TF-IDF · BK-Tree · Autocomplete · Hybrid Search

---

## 1. Search Architecture

```
Query string
    │
    ├──▶ Trie (prefix match) → exact/prefix results
    │
    ├──▶ InvertedIndex (BM25 + TF-IDF) → ranked term results
    │
    ├──▶ BKTree (edit distance) → fuzzy results
    │
    └──▶ HybridRanker → merged, re-ranked SearchResult list
```

---

## 2. Trie (Prefix Search + Autocomplete)

**Purpose:** Instant prefix completion for IDs, titles, and capabilities.

```
TrieNode:
  children: dict[str, TrieNode]
  is_end:   bool
  values:   list[str]   # Knowledge IDs at this terminal

Trie.insert(key: str, knowledge_id: str):
  node = root
  FOR char IN key.lower():
    IF char NOT IN node.children: node.children[char] = TrieNode()
    node = node.children[char]
  node.is_end = True
  node.values.append(knowledge_id)

Trie.autocomplete(prefix: str, limit: int = 10) → list[str]:
  node = traverse_to(prefix)
  IF node is None: RETURN []
  RETURN collect_all_values(node)[:limit]
```

Keys indexed: knowledge_id, title (normalized), capability name, domain name.

**Complexity:** Insert O(k), Lookup O(k + results), Space O(V×k) where k=avg key length.

---

## 3. Inverted Index (BM25 + TF-IDF)

**Purpose:** Ranked full-text search over document titles and content.

```
Posting:
  doc_id:      str
  term_freq:   int        # TF(term, doc)
  positions:   list[int]  # For phrase matching

InvertedIndex:
  index:     dict[str, list[Posting]]   # term → postings
  doc_lens:  dict[str, int]             # doc_id → token count
  doc_count: int
  avg_len:   float
```

**BM25 Scoring** (k1=1.5, b=0.75):

```
score(doc, query):
  FOR term IN query_terms:
    IF term NOT IN index: CONTINUE
    postings = index[term]
    posting = find_posting(postings, doc)
    IF posting is None: CONTINUE
    
    tf   = posting.term_freq
    df   = len(postings)          # Document frequency
    idf  = log((N - df + 0.5) / (df + 0.5) + 1)
    tf_n = (tf * (k1 + 1)) / (tf + k1 * (1 - b + b * doc_len / avg_len))
    
    score += idf * tf_n
  RETURN score
```

**Indexing pipeline:**
1. Tokenize (split on whitespace + punctuation, lowercase)
2. Remove stopwords
3. Strip YAML frontmatter before indexing content
4. Index title with 3× weight boost

---

## 4. BK-Tree (Fuzzy / Typo-tolerant Search)

**Purpose:** Find documents within edit distance k of the query.

Based on Burkhard-Keller trees — O(log N) expected search vs O(N) linear scan.

```
BKNode:
  word:     str
  doc_ids:  list[str]
  children: dict[int, BKNode]  # distance → child

INSERT(word, doc_id):
  node = root
  WHILE True:
    d = levenshtein(word, node.word)
    IF d == 0: node.doc_ids.append(doc_id); RETURN
    IF d IN node.children: node = node.children[d]; CONTINUE
    node.children[d] = BKNode(word, [doc_id]); RETURN

SEARCH(query, max_dist):
  results = []
  stack = [root]
  WHILE stack:
    node = stack.pop()
    d = levenshtein(query, node.word)
    IF d <= max_dist: results.extend(node.doc_ids)
    FOR dist IN node.children:
      IF abs(dist - d) <= max_dist:
        stack.append(node.children[dist])
  RETURN results
```

**Complexity:** O(log N) expected per query, O(N) worst case.

---

## 5. Hybrid Search

Combines all three signal sources with weighted scoring:

```
hybrid_search(query, limit=20):
  trie_results    = trie.autocomplete(query) → weight 0.40
  bm25_results    = inverted_index.search(query) → weight 0.45
  fuzzy_results   = bktree.search(query, max_dist=2) → weight 0.15

  merged = {}
  FOR (id, score) IN trie_results:
    merged[id] = merged.get(id, 0) + score * 0.40
  FOR (id, score) IN bm25_results:
    merged[id] = merged.get(id, 0) + score * 0.45
  FOR (id) IN fuzzy_results:
    merged[id] = merged.get(id, 0) + 0.15

  ranked = sorted(merged.items(), key=lambda x: x[1], reverse=True)
  RETURN [build_result(id, score) for id, score in ranked[:limit]]
```

---

## 6. SearchResult Structure

```python
@dataclass(frozen=True)
class SearchResult:
    knowledge_id:  str
    title:         str
    path:          str
    score:         float
    snippet:       str     # 150-char excerpt with match highlighted
    capability:    str | None
    domain:        str | None
    match_type:    str     # "exact", "prefix", "fuzzy", "fulltext", "hybrid"
    health_score:  float
```

---

## 7. Complexity Summary

| Operation | Structure | Time |
|-----------|-----------|------|
| Prefix autocomplete | Trie | O(k + results) |
| Full-text search | Inverted Index | O(T × D_T) |
| Fuzzy search (ed ≤ 2) | BK-Tree | O(log N) expected |
| Hybrid search | All three | O(T + log N + k) |
| Index build | All | O(D × W) D=docs, W=words |

---

## References

- [03-KNOWLEDGE-QUERY-ENGINE.md](03-KNOWLEDGE-QUERY-ENGINE.md) — LIKE predicate uses inverted index
- [KA-SPEC-018](../knowledge-core/18-COMPLEXITY-GUIDE.md) — Complexity budget
