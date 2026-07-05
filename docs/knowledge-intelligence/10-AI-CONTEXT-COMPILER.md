---
knowledge_id: KA-KIP-011
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
  - id: KA-KIP-010
    reason: "Extends Knowledge Compiler for AI-specific outputs"
  - id: KA-KIP-002
    reason: "Uses KnowledgeSession for scoped loading"
---

# AI Context Compiler

## Repository → ExecutionContext → AgentMemory → PromptPack → RuntimeContext

---

## 1. Purpose

The AI Context Compiler transforms a knowledge repository into AI-consumable
representations optimized for:
- Minimizing tokens while maximizing relevance
- Deduplicating content across loaded contexts
- Structuring knowledge as executable agent memory
- Generating task-specific prompt packs

---

## 2. Compilation Pipeline

```
Task / Query
    │
    ▼ Relevance Scoring
    Relevant Objects (scored by BM25 + graph proximity)
    │
    ▼ Context Planning
    ExecutionContext (prioritized, within token budget)
    │
    ├──▶ AgentMemory (compressed, persistent)
    ├──▶ PromptPack (formatted for LLM injection)
    └──▶ RuntimeContext (relationships + graph subset)
```

---

## 3. ExecutionContext

A task-specific view of the repository, bounded by a token budget:

```python
@dataclass
class ExecutionContext:
    task:              str               # What the AI needs to accomplish
    relevant_objects:  list[ContextObject]
    relationships:     list[dict]        # Graph edges between loaded objects
    capabilities:      list[str]         # Capabilities covered
    token_count:       int               # Estimated token use
    budget:            int               # Maximum tokens
    coverage_percent:  float             # % of relevant knowledge loaded
    generated:         str

@dataclass(frozen=True)
class ContextObject:
    knowledge_id:  str
    title:         str
    capability:    str | None
    content:       str          # Full or compressed based on budget
    relevance:     float        # 0-1 relevance to the task
    tier:          int          # 1=core, 2=supporting, 3=context
```

---

## 4. Relevance Scoring

```
SCORE_RELEVANCE(task, obj, graph, search):
  # Signal 1: BM25 text match
  text_score = search.bm25_score(obj.id, task_terms(task))
  
  # Signal 2: Graph proximity to seed objects
  seed_ids = search.search(task, limit=5).knowledge_ids
  min_distance = min(graph.shortest_path_len(seed, obj.id)
                     for seed in seed_ids) or 10
  graph_score = 1.0 / (1 + min_distance)
  
  # Signal 3: Health and status
  health_boost = obj.health_score / 100.0
  status_boost = 1.2 if obj.status == "approved" else 0.8
  
  RETURN (text_score * 0.50 + graph_score * 0.30) * health_boost * status_boost
```

---

## 5. Token Budget Management

Three tiers of context objects:

| Tier | Role | Token Budget | Content |
|------|------|-------------|---------|
| 1 — Core | Directly addresses the task | 50% of budget | Full content |
| 2 — Supporting | Implementing/depending on core | 30% of budget | Compressed (key facts) |
| 3 — Context | Domain/capability context | 20% of budget | Headers + first paragraph only |

```
PLAN_CONTEXT(objects, budget_tokens):
  sorted_objects = sort_by_relevance(objects)
  
  tier1 = sorted_objects[:5]              # Top 5 always core
  tier2 = sorted_objects[5:15]            # Next 10 supporting
  tier3 = sorted_objects[15:30]           # Next 15 context
  
  context = []
  remaining = budget_tokens
  
  FOR obj, tier IN [(o, 1) for o in tier1] + [(o, 2) for o in tier2]:
    tokens = estimate_tokens(obj.content if tier == 1 else compress(obj))
    IF tokens <= remaining:
      context.append(ContextObject(content=..., tier=tier))
      remaining -= tokens
  
  RETURN context, budget_tokens - remaining
```

---

## 6. AgentMemory

Persistent compressed memory for long-running AI agent sessions:

```python
@dataclass
class AgentMemory:
    session_id:       str
    created:          str
    last_updated:     str
    compressed:       bool
    token_estimate:   int
    objects:          list[dict]      # Compressed KnowledgeObject summaries
    relationships:    list[dict]      # Key relationships
    health_snapshot:  dict            # repo health at load time
    access_log:       list[str]       # IDs accessed this session

class AgentMemory:
    def load(self, capabilities: list[str], budget: int) -> None
    def remember(self, id: str, context: str) -> None
    def recall(self, query: str) -> list[dict]
    def compress(self) -> None        # Reduce to fit within budget
    def to_prompt(self) -> str        # Formatted for LLM injection
    def save(self, path: Path) -> None
    def token_count(self) -> int
```

---

## 7. Compression Algorithm

```
COMPRESS(obj):
  # Remove boilerplate, examples, repeated information
  content = obj.content
  
  # Keep: headers, first sentence of each paragraph, code block signatures
  # Remove: body of examples, repeated definitions, full algorithm details
  
  compressed_lines = []
  FOR section IN parse_sections(content):
    compressed_lines.append(section.heading)
    compressed_lines.append(first_sentence(section.body))
    FOR code_block IN section.code_blocks:
      compressed_lines.append(signature_only(code_block))
  
  RETURN join(compressed_lines)
  # Target: ~20% of original size
```

---

## 8. Interface

```python
class AIContextCompiler:
    def execution_context(
        self,
        task: str,
        registry: KnowledgeRegistry,
        graph: KnowledgeGraph,
        search: KnowledgeSearch,
        budget_tokens: int = 16384,
    ) -> ExecutionContext

    def agent_memory(
        self,
        session_id: str,
        capabilities: list[str],
        registry: KnowledgeRegistry,
        budget_tokens: int = 8192,
    ) -> AgentMemory

    def prompt_pack(
        self,
        context: ExecutionContext,
        format: str = "markdown",
    ) -> str

    def compress(
        self,
        context: ExecutionContext,
        target_tokens: int,
    ) -> ExecutionContext

    def deduplicate(
        self,
        contexts: list[ExecutionContext],
    ) -> ExecutionContext
```

---

## References

- [09-KNOWLEDGE-COMPILER.md](09-KNOWLEDGE-COMPILER.md) — Base compiler (PromptPack, LLMMemory)
- [01-KNOWLEDGE-REGISTRY.md](01-KNOWLEDGE-REGISTRY.md) — KnowledgeSession for scoped loading
- [04-KNOWLEDGE-SEARCH.md](04-KNOWLEDGE-SEARCH.md) — BM25 relevance scoring
