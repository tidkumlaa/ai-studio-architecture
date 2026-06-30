# KNW-KE-ARCH-009 — Knowledge Catalog

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Knowledge Catalog is the discovery layer for the repository. It enables humans and machines to find, browse, and filter Knowledge Objects without knowing their IDs.

---

## Catalog File Format

```yaml
# knowledge/catalog/catalog.yaml
version: "1.0.0"
generated_at: "2026-06-30T00:00:00Z"
total_objects: 0
total_packages: 9
total_namespaces: 9

statistics:
  by_status:
    CANONICAL: 0
    VERIFIED: 0
    APPROVED: 0
    DRAFT: 0
  by_type: {}
  by_namespace: {}
  avg_quality_score: 0.0

facets:
  - name: object_type
    values: [module, service, algorithm, ...]
  - name: namespace
    values: [plt, rt, prov, alg, pat, api, test, fin, meta]
  - name: status
    values: [DRAFT, REVIEW, VERIFIED, APPROVED, CANONICAL, DEPRECATED, ARCHIVED]
  - name: tag
    values: []   # populated by generator
  - name: package
    values: [kos.platform.package, ...]

entries: []  # populated by kos catalog build
```

---

## Catalog Entry Format

```yaml
# One entry per object in catalog.yaml
- knowledge_id: KNW-PLT-MOD-001
  canonical_name: plt.module.quota-manager
  knowledge_uri: knw://plt/module/quota-manager@1.0.0
  name: Quota Manager
  object_type: module
  namespace: plt
  package: kos.platform.package
  version: 1.0.0
  status: CANONICAL
  owner: team:platform
  description_excerpt: "Manages per-session and per-day resource quotas..."
  tags: [platform, quota, resource-management]
  quality_score: 0.92
  evidence_score: 0.85
  confidence: 0.88
  last_updated: "2026-06-30"
```

---

## Canonical Tag Registry

The following tags are canonical. Custom tags not in this list trigger KL-032 warning:

### Domain Tags (first tag in list)
```
platform, runtime, provider, algorithm, pattern, api, test, financial, meta
```

### Function Tags
```
quota, routing, scheduling, execution, conversation, memory, caching, context,
audit, analytics, budget, benchmark, policy, registry, provisioning, vault,
billing, health, configuration, deployment, monitoring, security
```

### Technology Tags
```
anthropic, openai, llm, embedding, vector, graph, search, sql, grpc, rest,
websocket, mcp, cli, python, async, streaming, http
```

### Quality Tags
```
canonical, stable, experimental, beta, deprecated, performance, coverage,
evidence-required, high-confidence, low-confidence
```

### Domain-Specific Tags
```
forex, market, equity, crypto, commodity, tick-data, ohlcv, news,
backtesting, strategy, trading, risk
```

---

## Discovery Features

### Faceted Search

Filter by any combination of facets:

```
GET /catalog/search?type=module&namespace=plt&status=CANONICAL&tag=quota
```

Returns matching entries sorted by `quality_score DESC`.

### Full-Text Search

Search across `name`, `description_excerpt`, `tags`:

```
GET /catalog/search?q=quota+resource
```

Uses BM25 ranking over the catalog entries.

### Dependency Graph Discovery

Starting from any object, discover what it depends on or what depends on it:

```
GET /catalog/graph/{knowledge_id}/dependencies?depth=3
GET /catalog/graph/{knowledge_id}/dependents?depth=2
```

### Similarity Search

Find semantically similar objects (powered by `29-SEMANTIC-LAYER`):

```
GET /catalog/similar/{knowledge_id}?limit=5
```

---

## Catalog CLI

```bash
kos catalog build                    # rebuild catalog from packages
kos catalog search "quota"           # full-text search
kos catalog list --type module       # list by type
kos catalog list --namespace plt     # list by namespace
kos catalog list --tag performance   # list by tag
kos catalog show KNW-PLT-MOD-001     # show catalog entry
kos catalog stats                    # show statistics
kos catalog deps KNW-PLT-MOD-001     # show dependencies
kos catalog graph                    # output dependency graph (JSON)
```

---

## Catalog Generation

The catalog is generated automatically by `kos catalog build`:

```
Input:   knowledge/packages/*/objects/*.yaml
         knowledge/packages/*/relationships/*.yaml
Process: 1. Load all objects
         2. Compute quality scores
         3. Build facet indexes
         4. Generate entries
         5. Sort by quality_score DESC
Output:  knowledge/catalog/catalog.yaml
         knowledge/catalog/facets/*.yaml
         knowledge/registry/search-index.yaml
```

CI regenerates the catalog on every merge to `main`.

---

## Package-Level Catalog

Each package has its own mini-catalog:

```yaml
# knowledge/packages/{name}/index.yaml
package: kos.platform.package
version: 1.0.0
object_count: 0
objects:
  - knowledge_id: KNW-PLT-MOD-001
    name: Quota Manager
    status: CANONICAL
    quality_score: 0.92
```

---

## Cross-References

- Tag list enforced by linter → `06-KNOWLEDGE-LINTER` KL-032
- Search backed by search engine → Phase 3.0C `28-SEARCH-ENGINE`
- Catalog drives semantic queries → Phase 3.0C `29-SEMANTIC-LAYER`
- Object discovery via KQL → Phase 3.0C `27-QUERY-LANGUAGE`
