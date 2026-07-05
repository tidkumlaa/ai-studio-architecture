---
knowledge_id: KA-KIP-010
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
  - id: KA-SPEC-009
    reason: "Compiler specification with 7 output targets"
---

# Knowledge Compiler

## KnowledgeObjects → Markdown · HTML · Wiki · PromptPack · LLMMemory · Dataset · Website

---

## 1. Compiler Architecture

```
Source (KnowledgeObject list)
    │
    ▼ Frontend
Intermediate Representation (IR)
    │
    ├──▶ Markdown Backend
    ├──▶ HTML Backend
    ├──▶ Wiki Backend
    ├──▶ PromptPack Backend      (token-budget aware)
    ├──▶ LLMMemory Backend       (compressed structured JSON)
    ├──▶ Training Dataset Backend (JSONL instruction pairs)
    └──▶ Architecture Website Backend
```

---

## 2. Intermediate Representation

```python
@dataclass
class KnowledgeIR:
    objects:       list[IRObject]
    graph:         dict              # Serialized graph
    registry:      dict              # ID → path map
    indexes:       dict              # All index types
    diagnostics:   list[IRDiagnostic]
    metadata:      dict              # Build metadata

@dataclass
class IRObject:
    id:          str
    type:        str
    title:       str
    capability:  str | None
    domain:      str | None
    content:     str       # Stripped body
    frontmatter: dict
    relationships: list[dict]
    health_score: float
```

---

## 3. Output Targets

### Markdown

Regenerates frontmatter-enriched Markdown with computed fields added:

```markdown
---
knowledge_id: KA-ARCH-001
health_score: 72.0    # Added by compiler
graph_neighbors: [KA-SPEC-001, KA-SPEC-002]  # Added
---
```

### HTML

Static HTML site with navigation:
- `index.html` — Knowledge index
- `capabilities/` — Per-capability HTML
- `search.js` — Client-side search (JSON index embedded)

### PromptPack

AI-optimized bundle for a specific capability or query:

```
=== KNOWLEDGE PACK ===
Capability: workflow-runtime
Budget: medium (16K tokens)
Generated: 2026-06-29

[KA-ARCH-001] Workflow Runtime Architecture
Status: approved | Health: 72/100
---
[content truncated to fit budget]

[KA-SPEC-005] Workflow Runtime Specification
Status: approved | Health: 68/100
---
[content truncated]
=== END KNOWLEDGE PACK ===
```

Token budgets: small=4K, medium=16K, large=64K, full=unlimited.

### LLMMemory

Compressed structured JSON for persistent AI sessions:

```json
{
  "session": "ws-2026-001",
  "compressed": true,
  "token_estimate": 4200,
  "objects": [
    {
      "id": "KA-ARCH-001",
      "title": "Workflow Runtime Architecture",
      "capability": "workflow-runtime",
      "status": "approved",
      "summary": "50-word compressed summary",
      "key_facts": ["fact1", "fact2"],
      "relationships": ["depends_on:KA-SPEC-001"]
    }
  ]
}
```

### Training Dataset (JSONL)

Instruction-following pairs for fine-tuning:

```jsonl
{"instruction": "What is the architecture of workflow-runtime?", "input": "", "output": "The workflow-runtime capability consists of..."}
{"instruction": "What does KA-ARCH-001 implement?", "input": "", "output": "KA-ARCH-001 implements KA-SPEC-001 and KA-SPEC-008."}
{"instruction": "Which documents depend on KA-SPEC-001?", "input": "", "output": "The following documents depend on KA-SPEC-001: ..."}
```

---

## 4. PromptPack Budget Algorithm

```
BUILD_PROMPT_PACK(capability, registry, budget_tokens):
  objects = registry.by_capability(capability)
  objects = sort_by_health(objects)  # Highest health first
  
  pack_sections = []
  tokens_used = 0
  
  FOR obj IN objects:
    obj_tokens = estimate_tokens(obj.content)
    
    IF tokens_used + obj_tokens <= budget_tokens:
      pack_sections.append(full_section(obj))
      tokens_used += obj_tokens
    ELIF tokens_used + 200 <= budget_tokens:
      # Truncated summary if partial budget remains
      pack_sections.append(summary_section(obj))
      tokens_used += 200
    ELSE:
      BREAK  # Budget exhausted
  
  RETURN PromptPack(sections=pack_sections, token_count=tokens_used)
```

---

## 5. Compiler Interface

```python
class KnowledgeCompiler:
    def compile(
        self,
        registry: KnowledgeRegistry,
        graph: KnowledgeGraph,
        targets: list[CompilerTarget],
        output_path: Path,
    ) -> CompilerResult

    def prompt_pack(
        self,
        capability: str | list[str],
        registry: KnowledgeRegistry,
        budget: str = "medium",  # small|medium|large|full
    ) -> PromptPack

    def llm_memory(
        self,
        objects: list[KnowledgeObject],
        compress: bool = True,
    ) -> LLMMemory

    def training_dataset(
        self,
        registry: KnowledgeRegistry,
        graph: KnowledgeGraph,
        output_path: Path,
    ) -> int  # Number of examples generated
```

---

## 6. Compiler Result

```python
@dataclass
class CompilerResult:
    targets_compiled:  list[str]
    objects_processed: int
    errors:            list[str]
    warnings:          list[str]
    output_path:       Path
    elapsed_seconds:   float
    token_counts:      dict[str, int]  # target → token count
```

---

## References

- [KA-SPEC-009](../knowledge-core/09-KNOWLEDGE-COMPILER-SPECIFICATION.md) — Full compiler spec
- [10-AI-CONTEXT-COMPILER.md](10-AI-CONTEXT-COMPILER.md) — AI-specific compilation
- [KA-SPEC-019](../knowledge-core/19-PERFORMANCE-BUDGET.md) — Compiler performance targets
