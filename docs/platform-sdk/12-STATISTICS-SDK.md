---
knowledge_id: KA-SDK-013
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
    reason: "Data sample sequences"
  - id: KA-SDK-003
    reason: "Algorithmic transforms for aggregation"
---

# Statistics SDK

## Descriptive · Streaming · Regression · Distribution · Hypothesis · Time Series

---

## 1. Purpose

The Statistics SDK provides statistical computing primitives for the Benchmark
Center (Phase 2.0D.7), Knowledge Health Runtime, Performance Budget, and
Reasoning Confidence modeling. All algorithms support streaming computation
(one pass over data) where possible. No external statistical libraries are
required; the SDK provides self-contained implementations of all necessary
algorithms.

---

## 2. Statistics Taxonomy

```
Statistics SDK
├── Descriptive       — mean, variance, skewness, kurtosis, percentiles
├── Streaming         — online single-pass algorithms for large data
├── Regression        — linear, polynomial, multi-variate
├── Distribution      — normal, uniform, exponential, empirical CDF
├── Hypothesis        — t-test, chi-squared, KS test, Mann-Whitney
└── Time Series       — moving average, trend, seasonality, anomaly
```

---

## 3. Interfaces

### 3.1 Sample

```python
class Sample(Protocol):
    """A statistical sample of numeric observations."""
    def data(self) -> Sequence[float]
    def size(self) -> int
    def add(self, value: float) -> None       # For streaming
    def merge(self, other: "Sample") -> "Sample"

@dataclass(frozen=True)
class SampleStats:
    n:         int
    mean:      float
    variance:  float
    std_dev:   float
    min:       float
    max:       float
    median:    float
    p25:       float
    p75:       float
    p95:       float
    p99:       float
    skewness:  float
    kurtosis:  float
```

### 3.2 DescriptiveStats

```python
class DescriptiveStats(Protocol):
    def summarize(self, data: Sequence[float]) -> SampleStats  # O(N log N)
    def mean(self, data: Sequence[float]) -> float              # O(N)
    def variance(self, data: Sequence[float],
                 ddof: int = 1) -> float                        # O(N)
    def std_dev(self, data: Sequence[float],
                ddof: int = 1) -> float                         # O(N)
    def median(self, data: Sequence[float]) -> float            # O(N log N)
    def percentile(self, data: Sequence[float], p: float) -> float  # O(N log N)
    def iqr(self, data: Sequence[float]) -> float               # P75 - P25
    def skewness(self, data: Sequence[float]) -> float          # O(N)
    def kurtosis(self, data: Sequence[float]) -> float          # O(N)
    def mode(self, data: Sequence[float]) -> float | None       # O(N)
    def coefficient_of_variation(self, data: Sequence[float]) -> float  # std_dev / mean
```

### 3.3 StreamingStats (Welford's algorithm)

```python
class StreamingStats(Protocol):
    """
    Online single-pass statistics.
    Uses Welford's algorithm for numerically stable variance.
    All operations O(1) per observation.
    """
    def update(self, value: float) -> None  # O(1)
    def count(self) -> int
    def mean(self) -> float
    def variance(self) -> float
    def std_dev(self) -> float
    def min(self) -> float
    def max(self) -> float
    def reset(self) -> None
    def snapshot(self) -> SampleStats
    def merge(self, other: "StreamingStats") -> "StreamingStats"

class ExponentialMovingAverage(Protocol):
    """EMA with configurable decay factor α."""
    def update(self, value: float) -> None
    def value(self) -> float
    def reset(self) -> None
    # α ∈ (0, 1]: higher α = more weight to recent observations
```

### 3.4 Regression

```python
class LinearRegression(Protocol):
    def fit(self, x: Sequence[float], y: Sequence[float]) -> "RegressionModel"
    # O(N) — single-pass OLS for simple linear regression

class PolynomialRegression(Protocol):
    def fit(self, x: Sequence[float], y: Sequence[float],
            degree: int) -> "RegressionModel"
    # O(N · degree²) — via normal equations

class MultiVariateRegression(Protocol):
    def fit(self, X: Sequence[Sequence[float]],
            y: Sequence[float]) -> "RegressionModel"
    # O(N · P²) — P = predictor count

@dataclass(frozen=True)
class RegressionModel:
    coefficients:  Sequence[float]   # [intercept, slope, ...]
    r_squared:     float
    mse:           float
    std_errors:    Sequence[float]

    def predict(self, x: float | Sequence[float]) -> float
    def predict_batch(self, xs: Sequence[float]) -> Sequence[float]
    def confidence_interval(self, x: float, alpha: float = 0.05) -> tuple[float, float]
```

### 3.5 Distribution

```python
class Distribution(Protocol):
    def pdf(self, x: float) -> float      # Probability density
    def cdf(self, x: float) -> float      # Cumulative probability
    def quantile(self, p: float) -> float # Inverse CDF
    def sample(self, n: int) -> Sequence[float]
    def mean(self) -> float
    def variance(self) -> float

# Built-in distributions:
# NormalDistribution(mean, std_dev)
# UniformDistribution(low, high)
# ExponentialDistribution(rate)
# PoissonDistribution(lambda_)
# BinomialDistribution(n, p)
# EmpiricalDistribution(data)  — from observed sample

class EmpiricalCDF(Protocol):
    def from_data(self, data: Sequence[float]) -> Distribution: ...
    def ks_distance(self, other: Distribution) -> float:  ...
    # Kolmogorov-Smirnov distance between empirical and theoretical CDF
```

### 3.6 Hypothesis Tests

```python
class HypothesisTest(Protocol):
    def test(self, *samples: Sequence[float]) -> TestResult

@dataclass(frozen=True)
class TestResult:
    test_name:  str
    statistic:  float
    p_value:    float
    reject_h0:  bool   # True if p_value < alpha
    alpha:      float
    conclusion: str

# Built-in tests:
# TTest(alpha=0.05)          — one-sample or two-sample t-test
# WelchTTest(alpha=0.05)     — unequal variance t-test
# ChiSquaredTest(alpha=0.05) — goodness of fit
# KSTest(alpha=0.05)         — Kolmogorov-Smirnov
# MannWhitneyTest(alpha=0.05) — non-parametric two-sample
```

### 3.7 Time Series

```python
class TimeSeries(Protocol):
    def moving_average(self, data: Sequence[float],
                       window: int) -> Sequence[float]  # O(N)
    def exponential_smoothing(self, data: Sequence[float],
                               alpha: float) -> Sequence[float]  # O(N)
    def trend(self, data: Sequence[float]) -> RegressionModel
    def deseasonalize(self, data: Sequence[float],
                      period: int) -> Sequence[float]  # STL decomposition
    def anomaly_score(self, data: Sequence[float],
                      window: int = 20) -> Sequence[float]
    # Anomaly score: z-score of each point relative to local window

@dataclass(frozen=True)
class TimeSeriesDecomposition:
    trend:      Sequence[float]
    seasonal:   Sequence[float]
    residual:   Sequence[float]
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-STA-001 | `mean([])` raises `EmptyDataError`; never returns 0 silently. |
| C-STA-002 | `StreamingStats` mean equals `DescriptiveStats.mean` for the same data. |
| C-STA-003 | `percentile(data, 0)` = min; `percentile(data, 100)` = max. |
| C-STA-004 | `variance(data, ddof=0)` is population variance; `ddof=1` is sample variance. |
| C-STA-005 | `RegressionModel.predict` is deterministic for same input. |
| C-STA-006 | `HypothesisTest.reject_h0 == True` iff `p_value < alpha`. |
| C-STA-007 | `Distribution.cdf(x) ∈ [0, 1]` for all x. |
| C-STA-008 | `EmpiricalDistribution.sample(N)` returns exactly N values from the original data. |
| C-STA-009 | `StreamingStats.merge` produces same result as computing over combined data. |
| C-STA-010 | `anomaly_score` for a constant series returns all zeros. |

---

## 5. Algorithm Complexity

| Algorithm | Complexity | Notes |
|-----------|-----------|-------|
| mean, variance | O(N) | Single pass |
| median, percentile | O(N log N) | Sort-based |
| StreamingStats update | O(1) | Welford's algorithm |
| EMA update | O(1) | Recurrence formula |
| Linear regression | O(N) | Simple OLS |
| Polynomial regression | O(N · d²) | d = degree |
| Moving average | O(N) | Sliding window sum |
| Anomaly score | O(N · W) | W = window size |
| KS test | O(N log N) | Sort-based |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — data sample sequences
- Algorithms SDK (KA-SDK-003) — sort for percentile, partition

---

## 7. Extension Points

```python
class StatisticalModel(Protocol):
    """Custom statistical model beyond built-in regression/distribution."""
    def fit(self, data: Any) -> "FittedModel": ...
    def predict(self, input: Any) -> Any: ...
    def score(self, data: Any) -> float: ...

class AnomalyDetector(Protocol):
    """Pluggable anomaly detection strategy."""
    def fit(self, baseline: Sequence[float]) -> None: ...
    def score(self, value: float) -> float: ...  # Higher = more anomalous
    def threshold(self) -> float: ...

class StreamingAggregator(Protocol):
    """Custom statistic maintainable in O(1) per update."""
    def update(self, value: float) -> None: ...
    def result(self) -> float: ...
    def reset(self) -> None: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-STA-001 | `mean([1,2,3])` = 2.0 |
| V-STA-002 | `variance([1,1,1], ddof=0)` = 0.0 |
| V-STA-003 | `StreamingStats.mean()` = `DescriptiveStats.mean()` for same data |
| V-STA-004 | `percentile(data, 50)` ≈ `median(data)` |
| V-STA-005 | `LinearRegression.fit([1,2,3],[2,4,6])`: slope≈2, intercept≈0 |
| V-STA-006 | `NormalDistribution.cdf(mean)` ≈ 0.5 |
| V-STA-007 | `moving_average([1,2,3,4], 2)` = `[1.5, 2.5, 3.5]` |
| V-STA-008 | t-test on two identical samples: `p_value ≈ 1.0`, `reject_h0 == False` |
| V-STA-009 | `anomaly_score` for constant series = all zeros |
| V-STA-010 | `StreamingStats.merge(A, B).mean()` = `mean(A.data + B.data)` |
