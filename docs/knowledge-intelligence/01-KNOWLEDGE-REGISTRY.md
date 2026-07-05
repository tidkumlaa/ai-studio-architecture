---
knowledge_id: KA-KIP-002
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
depends_on:
  - id: KA-SPEC-003
    reason: "ID format and registry schema"
  - id: KA-SPEC-010
    reason: "Knowledge Index Engine that feeds the registry"
---

# Knowledge Registry

## KnowledgeRegistry, Catalog, Resolver, Repository, Session, Workspace

---

## 1. Component Hierarchy

```
KnowledgeRepository         ← Loads from file system; the entry point
    │
    ├── KnowledgeRegistry   ← Primary store: id → KnowledgeObject
    │       • HashMap<id, KnowledgeObject>        O(1) get/put
    │       • Bloom filter for absent-id fast reject
    │
    ├── KnowledgeCatalog    ← Typed views over Registry
    │       • by_domain:      HashMap<domain, List<id>>
    │       • by_capability:  HashMap<cap, List<id>>
    │       • by_type:        HashMap<type, List<id>>
    │       • by_status:      HashMap<status, List<id>>
    │
    ├── KnowledgeResolver   ← Resolves URIs and aliases to Registry entries
    │       • ka:// URI resolution
    │       • Path → id resolution
    │       • Title → id resolution (via search index)
    │
    ├── KnowledgeSession    ← Scoped, read-only view for a single query or task
    │       • Pinned to a set of capabilities
    │       • Token-budget-aware: loads only what fits
    │
    └── KnowledgeWorkspace  ← Multi-session state accumulator for AI agents
            • Merges sessions
            • Deduplicates objects
            • Tracks access patterns
```

---

## 2. KnowledgeObject

The atomic unit stored in the Registry:

```python
@dataclass(frozen=True)
class KnowledgeObject:
    id: str                    # KA-ARCH-001
    uri: str                   # ka://dom-workflow/capability/architecture/KA-ARCH-001
    type: str                  # architecture, specification, ...
    capability: str | None
    domain: str | None
    title: str
    path: str                  # Relative to repository root
    status: str                # draft, approved, deprecated, archived
    version: str
    owner: str
    created: str
    review_date: str
    content: str               # Full document body (stripped of frontmatter)
    metadata: dict             # All other frontmatter fields
    health_score: float        # Last computed (0-100); 0 if not yet computed
```

---

## 3. KnowledgeRegistry Interface

```python
class KnowledgeRegistry:
    # Core access — O(1)
    def get(self, id: str) -> KnowledgeObject | None
    def contains(self, id: str) -> bool       # Bloom filter pre-check
    def put(self, obj: KnowledgeObject) -> None
    def all(self) -> list[KnowledgeObject]     # All active (non-archived)
    def all_including_archived(self) -> list[KnowledgeObject]
    def count(self) -> int

    # Typed access — O(1) via Catalog indexes
    def by_id(self, id: str) -> KnowledgeObject | None
    def by_path(self, path: str) -> KnowledgeObject | None
    def by_capability(self, capability: str) -> list[KnowledgeObject]
    def by_type(self, type: str) -> list[KnowledgeObject]
    def by_domain(self, domain: str) -> list[KnowledgeObject]
    def by_status(self, status: str) -> list[KnowledgeObject]

    # Health
    def below_health(self, threshold: float) -> list[KnowledgeObject]
    def stale(self, days: int = 180) -> list[KnowledgeObject]
    def next_id(self, type_code: str) -> str

    # I/O
    def load_from_yaml(self, registry_path: Path) -> None
    def load_from_directory(self, root: Path) -> None
    def save_to_yaml(self, path: Path) -> None
    def stats(self) -> RegistryStats
```

---

## 4. KnowledgeResolver Interface

```python
class KnowledgeResolver:
    def resolve_id(self, id: str) -> KnowledgeObject | None
    def resolve_uri(self, uri: str) -> KnowledgeObject | None
        # Parses ka://domain/capability/type/id  → registry.get(id)
    def resolve_path(self, path: str) -> KnowledgeObject | None
    def resolve_title(self, title: str) -> list[KnowledgeObject]
        # Uses inverted index on titles
    def resolve_alias(self, alias: str) -> KnowledgeObject | None
```

---

## 5. KnowledgeSession

A session is a lightweight, scoped view for a specific query or agent task:

```python
class KnowledgeSession:
    session_id: str
    capabilities: list[str]     # Which capabilities this session covers
    token_budget: int           # Maximum tokens to load
    objects: list[KnowledgeObject]  # Objects loaded in this session

    def load(self, capabilities: list[str], budget: TokenBudget) -> None
    def add(self, obj: KnowledgeObject) -> bool    # Returns False if budget exceeded
    def token_estimate(self) -> int
    def to_prompt_pack(self) -> PromptPack
    def close(self) -> SessionSummary
```

---

## 6. KnowledgeWorkspace

An accumulator across multiple sessions:

```python
class KnowledgeWorkspace:
    workspace_id: str
    sessions: list[KnowledgeSession]

    def open_session(self, capabilities: list[str], budget: TokenBudget) -> KnowledgeSession
    def merge_sessions(self) -> list[KnowledgeObject]  # Deduplicated
    def access_frequency(self) -> dict[str, int]       # id → access count
    def hot_objects(self, top_n: int = 20) -> list[KnowledgeObject]
    def compress(self, target_tokens: int) -> CompressedWorkspace
```

---

## 7. KnowledgeRepository (Entry Point)

```python
class KnowledgeRepository:
    def __init__(self, root_path: Path): ...

    def load(self) -> None
        # Scans root, parses frontmatter, populates registry, catalog, search index

    @property
    def registry(self) -> KnowledgeRegistry
    @property
    def catalog(self) -> KnowledgeCatalog
    @property
    def resolver(self) -> KnowledgeResolver
    @property
    def graph(self) -> KnowledgeGraph
    @property
    def search(self) -> KnowledgeSearch

    def open_session(self, capabilities: list[str]) -> KnowledgeSession
    def open_workspace(self) -> KnowledgeWorkspace
    def health(self) -> float  # Repository health score
```

---

## References

- [KA-SPEC-003](../knowledge-core/03-KNOWLEDGE-IDENTITY-SPECIFICATION.md) — ID format
- [KA-SPEC-010](../knowledge-core/10-KNOWLEDGE-INDEX-ENGINE.md) — KIE feeds registry
- [02-KNOWLEDGE-GRAPH-RUNTIME.md](02-KNOWLEDGE-GRAPH-RUNTIME.md) — Graph built from registry
