---
knowledge_id: KA-SPEC-009
version: "1.0.0"
status: approved
owner: Chief Knowledge Architect
phase: 2.0D.0.5
created: 2026-06-29
review_date: 2026-12-29
canonical: true
domain: DOM-KNOWLEDGE
capability: knowledge-core
type: specification
implements:
  - KA-SPEC-001
depends_on:
  - id: KA-SPEC-001
    reason: "Compiles KnowledgeObjects"
  - id: KA-SPEC-008
    reason: "Resolves relationships during compilation"
---

# Knowledge Compiler Specification

## Transforming Knowledge Objects into Multiple Output Formats

---

## 1. Purpose

The Knowledge Compiler takes KnowledgeObjects as input and produces multiple output artifacts. It is a multi-target compiler where the source language is the Knowledge Object Model and the output targets range from human-readable documents to AI-optimized prompt packs to training datasets.

This enables write-once, publish-everywhere: architecture knowledge is authored once and compiled into every format that downstream consumers require.

---

## 2. Compiler Architecture

```
Input Sources
  ├── Markdown + YAML frontmatter (primary)
  ├── KDSL declarations
  └── Knowledge Registry + Index

              ↓
        ┌──────────────┐
        │   FRONTEND   │
        │  (Parser)    │
        └──────┬───────┘
               │  Internal Representation (IR)
               │  = KnowledgeObject[] + KnowledgeGraph
               ↓
        ┌──────────────┐
        │  MIDDLE END  │
        │  (Resolver)  │  ← Resolves relationships, validates schema
        └──────┬───────┘
               │  Enriched IR
               ↓
        ┌──────────────┐
        │   BACKENDS   │  (one per output target)
        ├──────────────┤
        │  Markdown    │ → Cleaned, cross-linked Markdown
        │  HTML        │ → Rendered HTML with navigation
        │  Wiki        │ → Confluence/Notion markup
        │  PromptPack  │ → Optimized LLM context packs
        │  LLMMemory   │ → Compressed AI memory format
        │  Dataset     │ → JSONL training dataset
        │  Website     │ → Static architecture website
        │  Graph       │ → graph.json for visualization
        └──────────────┘
```

---

## 3. Internal Representation (IR)

The IR is the compiler's canonical in-memory form, produced by the frontend and consumed by all backends.

```
IR = {
  objects:    KnowledgeObject[],   # All parsed knowledge objects
  graph:      KnowledgeGraph,      # Relationship graph
  registry:   KnowledgeRegistry,   # ID → KnowledgeObject map
  indexes:    {
    by_id:          Map<ID, KnowledgeObject>
    by_capability:  Map<Capability, KnowledgeObject[]>
    by_type:        Map<Type, KnowledgeObject[]>
    by_domain:      Map<Domain, KnowledgeObject[]>
  },
  diagnostics: Diagnostic[],       # Errors and warnings
  metadata: {
    compiled_at:  Timestamp
    source_count: Integer
    error_count:  Integer
  }
}
```

---

## 4. Output Targets

### 4.1 Markdown Target

**Purpose:** Clean, cross-linked, validated Markdown for the architecture repository.

**What it does:**
- Validates all internal links and replaces broken ones with `[BROKEN LINK]` markers
- Inserts automatic back-links ("Referenced by: ...")
- Adds deprecation banners to deprecated documents
- Inserts health score badges
- Normalizes frontmatter to the canonical schema

**Output format:** `.md` files (same location as input, overwritten)

**Compiler flags:**
```
--target markdown
--validate-links         # Check all internal links
--insert-backlinks       # Add reverse reference sections
--insert-health-badge    # Add health score to frontmatter display
--normalize-frontmatter  # Reorder fields to canonical order
```

### 4.2 HTML Target

**Purpose:** Rendered HTML documentation for the Architecture Website.

**What it does:**
- Renders Markdown to HTML with syntax highlighting
- Generates navigation sidebar from capability indexes
- Creates cross-capability hyperlinks from Knowledge IDs
- Generates search index (Lunr.js)
- Applies design system styles

**Output:** `dist/html/` — static site

### 4.3 Wiki Target

**Purpose:** Confluence or Notion-compatible markup.

**What it does:**
- Converts Markdown to Confluence Storage Format or Notion API blocks
- Maps heading hierarchy to Confluence heading macros
- Maps code blocks to Confluence code macros
- Preserves internal links as wiki page links

**Output:** `.confluence.xml` or `.notion.json`

### 4.4 PromptPack Target

**Purpose:** AI-optimized packs for LLM consumption. Minimizes tokens while preserving semantic completeness.

**Format:**
```
[KNOWLEDGE PACK: workflow-runtime | DOM-WORKFLOW | generated:2026-06-29]
[OBJECTS: 11 | RELATIONSHIPS: 47 | HEALTH: 82/100]

──── KA-OVW-001 | overview | approved ────────────────────────────
Workflow Runtime: The capability responsible for executing multi-step AI 
workflows. Manages step execution, retry, error handling, and result 
aggregation. Dependencies: provider-framework, prompt-os.

──── KA-ARCH-001 | architecture | approved | confidence:high ─────
[Architecture content — compressed to key decisions and patterns]
Depends on: KA-ARCH-005 (provider delegation), KA-ARCH-003 (prompt execution)
Implemented by: platform/workflow-runtime/src/executor.py [verified]
Key decisions: KA-ADR-012 (async executor), KA-ADR-017 (retry strategy)

──── KA-API-001 | api | approved ─────────────────────────────────
[API summary with endpoints and contracts]

[RELATIONSHIPS]
KA-ARCH-001 → depends_on → KA-ARCH-005 (required)
KA-ARCH-001 → implemented_by → platform/workflow-runtime/src/executor.py
...
```

**Compiler optimization:**
- Drops duplicated content (references instead of copies)
- Trims prose to key facts
- Preserves all structured data (relationships, evidence, coverage)
- Sorts objects by reading priority: overview → architecture → api → others

**Target token budgets:**
```
--budget small   # 4K tokens   — overview only
--budget medium  # 16K tokens  — architecture + api
--budget large   # 64K tokens  — all documents in capability
--budget full    # unlimited   — all documents with full content
```

### 4.5 LLMMemory Target

**Purpose:** Compressed structured memory for persistent AI sessions.

**Format:** JSON with semantic compression — structured facts, not prose.

```json
{
  "capability": "workflow-runtime",
  "domain": "DOM-WORKFLOW",
  "version": "2.0.0",
  "health": 82,
  "core_concepts": [
    "Executes multi-step AI workflows asynchronously",
    "Delegates to provider-framework for AI calls",
    "Executes prompts via prompt-os",
    "Persists state in workflow-state SQLite database"
  ],
  "key_decisions": [
    "ADR-012: Async executor chosen over sync",
    "ADR-017: Exponential backoff retry with 3 max attempts"
  ],
  "interfaces": {
    "api": "KA-API-001",
    "events_produced": ["workflow.started", "workflow.completed", "workflow.failed"],
    "events_consumed": []
  },
  "dependencies": ["provider-framework", "prompt-os", "context-intelligence"],
  "consumers": ["content-factory", "claude-cli"]
}
```

### 4.6 Training Dataset Target

**Purpose:** Generate JSONL training data for fine-tuning AI models on architecture knowledge.

**Format:** JSONL, one example per line, instruction-following format.

```jsonl
{"instruction": "What is the Workflow Runtime?", "output": "The Workflow Runtime (KA-OVW-001) is the AI Studio capability responsible for executing multi-step AI workflows. It manages workflow step execution, retry logic, error handling, and result aggregation. It depends on the Provider Framework for AI calls and Prompt OS for prompt execution."}
{"instruction": "What ADRs affect workflow execution?", "output": "Two ADRs directly affect workflow execution: KA-ADR-012 (Executor Selection) which chose an asynchronous executor, and KA-ADR-017 (Retry Strategy) which defined exponential backoff with 3 maximum attempts."}
```

**Generation strategy:** Cross-reference all relationships to generate question-answer pairs for:
- What is X?
- How does X work?
- What depends on X?
- What does X depend on?
- Why was X designed this way?
- What changed in X?

### 4.7 Architecture Website Target

**Purpose:** A complete static website for browsing AI Studio architecture.

**Structure:**
```
dist/website/
├── index.html              # Master knowledge index
├── domains/
│   ├── index.html
│   └── dom-workflow/
│       └── index.html
├── capabilities/
│   ├── workflow-runtime/
│   │   ├── index.html
│   │   ├── overview.html
│   │   └── architecture.html
└── search/
    └── index.json          # Lunr search index
```

---

## 5. Compiler Pipeline Steps

```
1. SOURCE DISCOVERY
   Glob all .md files in scope

2. PARSING
   Parse YAML frontmatter
   Parse Markdown body
   Extract headings, links, code blocks

3. VALIDATION (Frontend)
   Schema validation (strict or lenient mode)
   Type inference for undeclared types
   ID uniqueness check against registry

4. RELATIONSHIP RESOLUTION
   Resolve all ID references in frontmatter
   Build KnowledgeGraph from resolved relationships
   Detect cycles (SCC algorithm)
   Detect orphans (reachability)

5. ENRICHMENT
   Compute health scores for all objects
   Compute capability health scores
   Compute domain health scores
   Generate reverse relationships

6. BACKEND DISPATCH
   For each selected target, invoke backend
   Each backend receives the enriched IR
   Each backend writes to its output directory

7. DIAGNOSTICS
   Collect all errors and warnings
   Write audit-report.yaml
   Write health-report.yaml
   Write HEALTH-REPORT.md
   Exit with code 1 if any errors (--ci mode)
```

---

## 6. Compiler Invocation

```bash
# Compile all targets
ka-compile --source architecture/ --targets all

# Single target
ka-compile --source architecture/ --target promptpack --capability workflow-runtime

# Budget-constrained prompt pack
ka-compile --target promptpack --capability workflow-runtime --budget medium

# CI mode (fail on errors)
ka-compile --source architecture/ --ci --validate-only
```

---

## References

- [01-KNOWLEDGE-OBJECT-SPECIFICATION.md](01-KNOWLEDGE-OBJECT-SPECIFICATION.md) — Input to the compiler
- [06-KNOWLEDGE-DSL.md](06-KNOWLEDGE-DSL.md) — Alternative input format
- [08-KNOWLEDGE-GRAPH-MODEL.md](08-KNOWLEDGE-GRAPH-MODEL.md) — Graph built during compilation
- [10-KNOWLEDGE-INDEX-ENGINE.md](10-KNOWLEDGE-INDEX-ENGINE.md) — Indexes built during compilation
