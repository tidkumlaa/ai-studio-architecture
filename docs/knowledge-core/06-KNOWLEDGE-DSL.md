---
knowledge_id: KA-SPEC-006
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
  - id: KA-SPEC-002
    reason: "DSL constructs map to knowledge types"
  - id: KA-SPEC-005
    reason: "DSL is a human-friendly surface over the schema"
---

# Knowledge DSL

## A Domain-Specific Language for Authoring Knowledge Declarations

---

## 1. Purpose

The Knowledge DSL (KDSL) is a concise, human-readable language for declaring knowledge objects and their relationships. It provides an alternative to YAML frontmatter for:

- Declaring complex relationship graphs
- Expressing capability knowledge structure
- Generating metadata stubs for new capabilities
- Machine-readable summaries for AI assistants

KDSL compiles to valid YAML frontmatter. It is an authoring convenience — not a replacement for the canonical Markdown + YAML format.

---

## 2. Design Goals

| Goal | Description |
|------|-------------|
| **Readable** | A senior architect must be able to read KDSL without a manual |
| **Minimal** | Fewer characters than equivalent YAML for common patterns |
| **Composable** | Capability and module declarations compose naturally |
| **Compilable** | Every valid KDSL file compiles to valid YAML frontmatter |
| **Diffable** | KDSL changes produce meaningful git diffs |
| **Verifiable** | The KDSL compiler validates against the Knowledge Schema |

---

## 3. KDSL Syntax

### 3.1 Document Declaration

```kdsl
knowledge "Workflow Runtime Architecture" {
  id:         KA-ARCH-001
  version:    1.2.0
  type:       architecture
  status:     approved
  canonical:  true
  domain:     DOM-WORKFLOW
  capability: workflow-runtime
  owner:      "Capability Architect"
  created:    2026-06-29
  review:     2026-12-29
  phase:      2.0D.0
  confidence: high
}
```

### 3.2 Relationship Declarations

```kdsl
knowledge "Workflow Runtime Architecture" {
  id: KA-ARCH-001
  ...

  implements {
    KA-STD-001
    KA-STD-002
  }

  depends {
    KA-ARCH-005 as "Provider Framework Architecture"
      reason: "Workflow executor delegates to providers"
      critical: true

    KA-ARCH-003 as "Prompt OS Architecture"
      reason: "Workflow steps execute prompts"
      critical: false
  }

  implemented_by {
    module "platform/workflow-runtime/src/executor.py"
      confidence: high
      verified: true
      verified_on: 2026-06-28

    module "platform/workflow-runtime/src/scheduler.py"
      confidence: high
      verified: true
  }

  evidence {
    source "platform/workflow-runtime/src/executor.py"
      verified: true
    test "platform/workflow-runtime/tests/test_executor.py"
      verified: true
  }

  related {
    api     KA-API-001 "Workflow Execution API"
    db      KA-DB-001  "Workflow State Schema"
    events  KA-EVT-001 "Workflow Events"
    adr     KA-ADR-012 "Executor Selection"
    adr     KA-ADR-017 "Retry Strategy"
  }

  consumers {
    product content-factory as required
    product claude-cli       as required
  }

  coverage {
    architecture:  90
    implementation: 75
    testing:       60
  }

  tags: workflow, execution, orchestration, async
}
```

### 3.3 Capability Declaration

```kdsl
capability "workflow-runtime" {
  domain:  DOM-WORKFLOW
  version: 2.0.0
  status:  active
  owner:   "Capability Architect"

  documents {
    required {
      overview        KA-OVW-001
      architecture    KA-ARCH-001
      implementation  KA-IMPL-001
      testing         KA-TEST-001
      roadmap         KA-ROAD-001
      history         KA-HIST-001
      references      KA-REF-001
    }

    optional {
      api       KA-API-001
      database  KA-DB-001
      events    KA-EVT-001
      security  KA-SEC-001
    }
  }

  depends {
    capability provider-framework as required
    capability prompt-os           as required
    capability context-intelligence as optional
  }

  consumed_by {
    product content-factory as required
    product claude-cli       as required
  }
}
```

### 3.4 ADR Declaration

```kdsl
adr KA-ADR-012 "Executor Selection" {
  status:   accepted
  decided:  2026-03-15
  domain:   DOM-WORKFLOW
  owner:    "Chief Enterprise Architect"

  participants {
    "Chief Enterprise Architect"
    "Capability Architect"
  }

  options {
    "Synchronous Executor" {
      pro:  "Simple implementation"
      pro:  "Deterministic behavior"
      con:  "Blocks during long operations"
      con:  "Cannot handle concurrent workflows"
    }

    "Asynchronous Executor" {
      pro:  "Non-blocking"
      pro:  "Supports concurrent workflows"
      con:  "More complex state management"
    }
  }

  decision: "Asynchronous Executor"
  rationale: "Concurrent workflow support is required for Content Factory use case"

  supersedes: KA-ADR-008

  affects {
    KA-ARCH-001 "Changes execution model in sections 3.2, 4.1"
    KA-IMPL-001 "Changes implementation pattern to async/await"
  }
}
```

### 3.5 Domain Declaration

```kdsl
domain DOM-WORKFLOW "Workflow Runtime" {
  description: "Workflow definition, execution, orchestration, and scheduling"
  owner:       "Chief Enterprise Architect"
  status:      active

  capabilities {
    workflow-runtime
  }

  depends {
    domain DOM-AI      as required
    domain DOM-PROMPT  as required
    domain DOM-PROVIDER as required
  }

  products {
    content-factory
    claude-cli
  }
}
```

---

## 4. KDSL Grammar (EBNF)

```ebnf
file              ::= declaration*

declaration       ::= knowledge_decl
                   | capability_decl
                   | domain_decl
                   | adr_decl

knowledge_decl    ::= "knowledge" string "{" knowledge_body "}"
knowledge_body    ::= field* block*

capability_decl   ::= "capability" string "{" capability_body "}"
adr_decl          ::= "adr" knowledge_id string "{" adr_body "}"
domain_decl       ::= "domain" domain_id string "{" domain_body "}"

field             ::= identifier ":" value
block             ::= identifier "{" (field | id_entry | ref_entry)* "}"

id_entry          ::= knowledge_id string? ("as" string)?
ref_entry         ::= ("module"|"source"|"test"|"schema") string field*

knowledge_id      ::= "KA-" [A-Z]{2,5} "-" [0-9]{3,}
domain_id         ::= "DOM-" [A-Z]+
identifier        ::= [a-z_]+
string            ::= '"' [^"]* '"'
value             ::= string | number | boolean | knowledge_id | domain_id | tag_list
tag_list          ::= [a-z]+ ("," [a-z]+)*
number            ::= [0-9]+ ("." [0-9]+)?
boolean           ::= "true" | "false"
```

---

## 5. KDSL → YAML Compilation

The KDSL compiler (`tools/kdsl-compile.py`) transforms KDSL to valid YAML frontmatter:

### 5.1 Input (KDSL)

```kdsl
knowledge "Workflow Runtime Architecture" {
  id:   KA-ARCH-001
  type: architecture
  implements { KA-STD-001 }
  depends { KA-ARCH-005 reason: "Provider delegation" }
}
```

### 5.2 Output (YAML Frontmatter)

```yaml
---
knowledge_id: KA-ARCH-001
version: "0.1.0"
status: draft
canonical: false
type: architecture
implements:
  - KA-STD-001
depends_on:
  - id: KA-ARCH-005
    reason: "Provider delegation"
---
```

Unspecified required fields are set to defaults (draft status, version 0.1.0, canonical false). The author must complete them before status can advance to `approved`.

---

## 6. KDSL Validation

The KDSL compiler validates against the Knowledge Schema before emitting YAML:

| Validation | Error or Warning |
|-----------|-----------------|
| Unknown field name | Error |
| Invalid knowledge_id format | Error |
| Enum value not valid | Error |
| Missing required field (for approved status) | Warning (field added as placeholder) |
| Referenced ID not in registry | Warning |

---

## 7. Use Cases

| Use Case | How KDSL Helps |
|----------|---------------|
| Scaffold new capability | `kdsl-compile.py --capability workflow-runtime --scaffold` generates stubs for all 9 required files |
| Express complex relationships | More readable than equivalent nested YAML |
| AI assistant authoring | AI can generate KDSL more reliably than YAML (less whitespace sensitivity) |
| Batch metadata generation | Compile multiple KDSL files in one pass |
| Knowledge graph visualization input | KDSL is parseable into graph nodes/edges directly |

---

## References

- [05-KNOWLEDGE-SCHEMA.md](05-KNOWLEDGE-SCHEMA.md) — Schema that KDSL compiles against
- [09-KNOWLEDGE-COMPILER-SPECIFICATION.md](09-KNOWLEDGE-COMPILER-SPECIFICATION.md) — The compiler that processes KDSL among other formats
- [01-KNOWLEDGE-OBJECT-SPECIFICATION.md](01-KNOWLEDGE-OBJECT-SPECIFICATION.md) — The objects that KDSL declares
