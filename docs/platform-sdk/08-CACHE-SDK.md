---
knowledge_id: KA-SDK-009
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
    reason: "Key/value types"
  - id: KA-SDK-006
    reason: "BloomPrefilter for absent-key guard"
---

# Cache SDK

## LRU · LFU · TTL · Write-Through · Write-Back · Multi-Tier · Bloom Guard

---

## 1. Purpose

The Cache SDK provides a unified, pluggable caching abstraction for all
AI Studio components. It separates eviction policy from cache mechanics,
enables multi-tier caching (L1 memory → L2 disk → L3 remote), and supports
write-through and write-back consistency modes. Every Runtime depends on
the Cache SDK for performance isolation from underlying stores.

---

## 2. Cache Architecture

```
Cache[K, V]
├── EvictionPolicy (pluggable)
│   ├── LRU     — Least Recently Used
│   ├── LFU     — Least Frequently Used
│   ├── TTL     — Time-To-Live expiry
│   ├── FIFO    — First-In First-Out
│   └── ARC     — Adaptive Replacement Cache
│
├── WritePolicy (pluggable)
│   ├── WriteThrough   — write to cache + backing store simultaneously
│   ├── WriteBack      — write to cache; flush to store on eviction
│   └── WriteAround    — bypass cache for writes; only cache reads
│
└── CacheTier
    ├── L1: MemoryCache (fastest, smallest)
    ├── L2: DiskCache (fast, larger)
    └── L3: RemoteCache (slowest, distributed)
```

---

## 3. Interfaces

### 3.1 Cache[K, V] — base

```python
class Cache(Protocol[K, V]):
    def get(self, key: K) -> V | None                    # O(1) — cache hit
    def put(self, key: K, value: V) -> None              # O(1) amortized
    def remove(self, key: K) -> bool
    def contains(self, key: K) -> bool                   # O(1)
    def size(self) -> int                                # O(1)
    def capacity(self) -> int                            # O(1)
    def clear(self) -> None
    def stats(self) -> CacheStats
```

### 3.2 EvictionPolicy[K, V]

```python
class EvictionPolicy(Protocol[K, V]):
    def on_access(self, key: K) -> None
    def on_insert(self, key: K, value: V) -> None
    def on_remove(self, key: K) -> None
    def evict_candidate(self) -> K | None    # Key to evict when full
    def reset(self) -> None

# LRU: OrderedDict — O(1) get, O(1) put, O(1) evict
# LFU: min-heap + freq map — O(log N) put, O(log N) evict, O(1) get
# TTL: min-heap on expiry time — O(log N) insert, O(1) expiry check
# FIFO: deque — O(1) all operations
# ARC: 4 internal lists (T1, T2, B1, B2) — O(1) amortized
```

### 3.3 WritePolicy

```python
class WritePolicy(Protocol[K, V]):
    def on_write(self, key: K, value: V,
                 cache: Cache[K, V],
                 store: "BackingStore[K, V]") -> None
    def on_evict(self, key: K, value: V,
                 store: "BackingStore[K, V]") -> None
    def flush(self, cache: Cache[K, V],
              store: "BackingStore[K, V]") -> int   # Returns items flushed

class BackingStore(Protocol[K, V]):
    def read(self, key: K) -> V | None
    def write(self, key: K, value: V) -> None
    def delete(self, key: K) -> None
    def batch_write(self, entries: Sequence[tuple[K, V]]) -> None
```

### 3.4 TTLCache

```python
class TTLCache(Cache[K, V]):
    """Cache with time-based expiry per entry."""
    def get_with_ttl(self, key: K) -> tuple[V, float] | None  # (value, remaining_seconds)
    def put_with_ttl(self, key: K, value: V, ttl_seconds: float) -> None
    def expire(self, key: K) -> None                           # Force expiry
    def cleanup_expired(self) -> int                           # Returns count removed

class ClockProvider(Protocol):
    """Injectable clock for deterministic TTL testing."""
    def now(self) -> float: ...     # Unix timestamp
```

### 3.5 MultiTierCache

```python
class MultiTierCache(Cache[K, V]):
    """Hierarchical cache: L1 miss → L2 → L3 → BackingStore."""
    def add_tier(self, tier: Cache[K, V], priority: int) -> None
    def remove_tier(self, priority: int) -> None
    def tier_count(self) -> int
    def promote(self, key: K, to_tier: int) -> None  # Move entry up
    def demote(self, key: K, to_tier: int) -> None   # Move entry down
    def tier_stats(self) -> Sequence[tuple[int, CacheStats]]
```

### 3.6 CacheStats

```python
@dataclass(frozen=True)
class CacheStats:
    hits:       int
    misses:     int
    evictions:  int
    size:       int
    capacity:   int
    hit_rate:   float    # hits / (hits + misses)
    write_count: int
    flush_count: int     # Write-back only
```

### 3.7 CacheLoader

```python
class CacheLoader(Protocol[K, V]):
    """Load-through strategy: on miss, load from backing store."""
    def load(self, key: K) -> V | None: ...
    def load_all(self, keys: Sequence[K]) -> ImmutableMap[K, V]: ...

class LoadingCache(Cache[K, V]):
    """Cache with automatic load-on-miss from CacheLoader."""
    def __init__(self, backing: Cache[K, V], loader: CacheLoader[K, V])
    def get_or_load(self, key: K) -> V | None   # Loads from backing if miss
    def refresh(self, key: K) -> None           # Force reload from backing
    def refresh_all(self) -> None
```

### 3.8 CacheBuilder

```python
class CacheBuilder(Protocol[K, V]):
    def capacity(self, n: int) -> "CacheBuilder[K, V]": ...
    def eviction(self, policy: EvictionPolicy[K, V]) -> "CacheBuilder[K, V]": ...
    def write_policy(self, policy: WritePolicy[K, V]) -> "CacheBuilder[K, V]": ...
    def ttl(self, seconds: float) -> "CacheBuilder[K, V]": ...
    def bloom_guard(self, capacity: int) -> "CacheBuilder[K, V]": ...
    def loader(self, loader: CacheLoader[K, V]) -> "CacheBuilder[K, V]": ...
    def build(self) -> Cache[K, V]: ...
```

---

## 4. Contracts

| ID | Contract |
|----|----------|
| C-CAC-001 | `get` after `put` returns the same value (before eviction). |
| C-CAC-002 | `get` after TTL expiry returns `None`. |
| C-CAC-003 | When capacity is full, exactly one entry is evicted before inserting. |
| C-CAC-004 | `CacheStats.hit_rate` = `hits / (hits + misses)`; 0 if no accesses. |
| C-CAC-005 | `WriteThrough`: backing store is updated synchronously on every `put`. |
| C-CAC-006 | `WriteBack`: backing store is updated no later than eviction time. |
| C-CAC-007 | `MultiTierCache.get`: on L1 miss, promotes value from lower tier to L1. |
| C-CAC-008 | `BloomPrefilter` guard: `get` on absent key skips cache lookup if Bloom says absent. |
| C-CAC-009 | `clear()` resets all stats to zero. |
| C-CAC-010 | `LoadingCache`: a loaded value is stored in cache before returning. |

---

## 5. Complexity Table

| Cache Type | get | put | evict |
|-----------|-----|-----|-------|
| LRU | O(1) | O(1) | O(1) |
| LFU | O(1) | O(log N) | O(log N) |
| TTL | O(1) | O(log N) | O(log N) |
| FIFO | O(1) | O(1) | O(1) |
| ARC | O(1) | O(1) amort | O(1) |
| MultiTier | O(T) tiers | O(1) | O(1) |
| Loading | O(1)+load | O(1) | O(1) |

---

## 6. Dependencies

- Collections SDK (KA-SDK-002) — internal storage structures
- Index SDK (KA-SDK-006) — BloomPrefilter for absent-key guard

---

## 7. Extension Points

```python
class CacheSerializer(Protocol[K, V]):
    """Persist cache to disk / remote for L2/L3 tiers."""
    def serialize_entry(self, key: K, value: V) -> bytes: ...
    def deserialize_entry(self, data: bytes) -> tuple[K, V]: ...

class CacheEventListener(Protocol[K, V]):
    """Observe cache events for metrics and debugging."""
    def on_hit(self, key: K) -> None: ...
    def on_miss(self, key: K) -> None: ...
    def on_evict(self, key: K, value: V) -> None: ...
    def on_expire(self, key: K) -> None: ...

class EvictionPolicyFactory(Protocol):
    """Produce custom eviction policies from config."""
    def create(self, name: str, config: dict) -> EvictionPolicy: ...
```

---

## 8. Verification

| Check | Criterion |
|-------|-----------|
| V-CAC-001 | LRU: access order determines eviction — MRU item survives |
| V-CAC-002 | LFU: least-accessed item evicted when capacity exceeded |
| V-CAC-003 | TTL: `get` after expiry returns `None` |
| V-CAC-004 | `CacheStats.hit_rate` = 0.5 after 1 hit and 1 miss |
| V-CAC-005 | WriteThrough: `BackingStore.read` returns value immediately after `put` |
| V-CAC-006 | WriteBack: `BackingStore.read` returns value only after eviction |
| V-CAC-007 | MultiTier: hit on L2 promotes to L1 |
| V-CAC-008 | Bloom guard: no lookup needed for keys that were never inserted |
| V-CAC-009 | `clear()` makes `size() == 0` and `stats().hits == 0` |
| V-CAC-010 | `LoadingCache.get_or_load`: missing key triggers `loader.load` exactly once |
