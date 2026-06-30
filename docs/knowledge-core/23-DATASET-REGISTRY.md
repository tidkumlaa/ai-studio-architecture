# KNW-KC-ARCH-023 — Dataset Registry

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Dataset Registry catalogs all structured datasets used for training, benchmarking, analysis, and inference. Every dataset declares its schema, provenance, license, and quality metrics.

---

## Registered Dataset Categories

| Category | Description | Domain |
|----------|-------------|--------|
| `training` | Machine learning training data | AI |
| `benchmark` | Performance benchmark datasets | Quality |
| `forex_historical` | Historical FX price data | Financial |
| `market_data` | OHLCV market time series | Financial |
| `news_corpus` | Market news and events | Financial |
| `knowledge_graph` | KOS graph exports | KOS |
| `test_fixtures` | Test input/output fixtures | Test |
| `evaluation` | Model evaluation benchmarks | AI |

---

## Dataset Entry Format

```yaml
# knowledge/registry/datasets/DS-FOREX-OHLCV-001.yaml
identity:
  knowledge_id: "KNW-FIN-DS-001"
  canonical_name: "financial.dataset.forex_ohlcv_2024"
  knowledge_uri: "knw://financial/dataset/forex-ohlcv-2024"
  namespace: "financial"
  version: "1.0.0"
  owner: "team:financial-runtime"

dataset_spec:
  dataset_id: "DS-FOREX-OHLCV-001"
  dataset_type: time_series
  format: parquet
  storage_path: "data/financial/forex/ohlcv/2024/"
  row_count: 1460000       # 1 year of 1-minute bars for major pairs
  size_bytes: 52428800     # 50 MB

  data_schema:
    timestamp: datetime
    pair: string
    open: decimal
    high: decimal
    low: decimal
    close: decimal
    volume: decimal
    spread: decimal

  features:
    - timestamp
    - pair
    - open
    - high
    - low
    - close
    - volume
    - spread

  target_columns: []       # unsupervised — no target

  splits:
    train: 0.80
    validation: 0.10
    test: 0.10

  quality:
    missing_value_rate: 0.001
    duplicate_row_rate: 0.000
    coverage_gaps: []

  provenance:
    data_source: "market-data-provider"
    collection_start: "2024-01-01"
    collection_end: "2024-12-31"
    collection_method: "API streaming"

  license: "Proprietary — internal use only"

lifecycle:
  status: VERIFIED
```

---

## Dataset Quality Metrics

```yaml
dataset_quality:
  knowledge_id: "KNW-FIN-DS-001"
  computed_at: datetime
  completeness: float        # fraction of non-null values
  consistency: float         # schema conformance rate
  accuracy: float | null     # if ground truth available
  timeliness: float          # freshness relative to data age
  uniqueness: float          # 1.0 - duplicate_rate
  overall_score: float
```

---

## Dataset Registry Rules

### DR-001 — Schema Declaration
Every dataset must declare its full column schema before reaching VERIFIED status.

### DR-002 — Provenance Requirement
`data_source` and collection date range are required for status ≥ APPROVED.

### DR-003 — License Clarity
`license` must be non-empty. Datasets with unclear licensing default to `INTERNAL_ONLY`.

### DR-004 — Size Limits
Datasets > 1 GB must declare a `storage_path` to an external storage system. Registry stores metadata only — not data.

### DR-005 — Split Fractions
`train + validation + test` must sum to 1.0 (± 0.001 floating point tolerance).

---

## Dataset Registry Protocol

```python
class DatasetRegistry(KnowledgeRegistryProtocol):
    def get_by_type(self, dataset_type: str) -> list[DatasetObject]: ...
    def get_by_domain(self, domain: str) -> list[DatasetObject]: ...
    def get_schema(self, dataset_id: str) -> dict: ...
    def get_quality(self, dataset_id: str) -> DatasetQualityMetrics: ...
    def get_by_license(self, license_type: str) -> list[DatasetObject]: ...
    def find_by_feature(self, feature: str) -> list[DatasetObject]: ...
```

---

## Cross-References

- Registry base → `14-REGISTRY-ARCHITECTURE`
- Dataset objects → Phase 3.0B `knowledge_runtime/objects/financial.py`
- Used by Benchmark objects → Phase 3.0B `knowledge_runtime/objects/quality.py`
