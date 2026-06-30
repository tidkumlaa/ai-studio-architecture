# KNW-KC-ARCH-003 — Namespace System

**Phase:** 3.0C  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

The Namespace System establishes the hierarchical naming structure for all Knowledge Objects. Namespaces prevent collision, enable domain-scoped queries, and encode organizational structure.

---

## Namespace Hierarchy

```
knw://                              # root — the KOS universe
  └── {domain}/                     # top-level domain
        └── {subdomain}/            # optional refinement
              └── {type}/           # object type
                    └── {name}      # canonical object name
```

---

## Top-Level Domains (Registered)

| Domain Code | Namespace | Description |
|-------------|-----------|-------------|
| `KOS` | `kos` | Knowledge Operating System core |
| `PLAT` | `platform` | Platform kernel and SDK |
| `AI` | `ai` | AI runtime and models |
| `FIN` | `financial` | Financial runtime |
| `FOREX` | `forex` | Forex runtime |
| `RESEARCH` | `research` | Research runtime |
| `AGENT` | `agent` | Agent runtime |
| `CONTENT` | `content` | Content factory |
| `DESKTOP` | `desktop` | Desktop application |
| `API` | `api` | API layer |
| `PRODUCT` | `product` | Products |
| `DEPLOY` | `deploy` | Deployment |
| `TEST` | `test` | Testing infrastructure |
| `DATA` | `data` | Datasets and data pipelines |

---

## Knowledge ID Format

```
KNW-{DOMAIN_CODE}-{TYPE_CODE}-{SEQUENCE}

Rules:
  DOMAIN_CODE  = 2–6 uppercase letters from registered domain list
  TYPE_CODE    = 2–8 uppercase letters from registered type list
  SEQUENCE     = 3+ digits (zero-padded) OR descriptive_slug

Examples:
  KNW-KOS-ARCH-001          # Architecture document #001 in KOS domain
  KNW-AI-MOD-quota_manager  # Module named quota_manager in AI domain
  KNW-FIN-MKT-001           # Market object #001 in Financial domain
  KNW-PLAT-SVC-001          # Service #001 in Platform domain
```

---

## Type Codes (Registered)

| Type Code | Object Type |
|-----------|-------------|
| `ARCH` | Architecture |
| `SPEC` | Specification |
| `DEC` | Decision (ADR) |
| `REQ` | Requirement |
| `PAT` | Pattern |
| `ALG` | Algorithm |
| `MOD` | Module |
| `SVC` | Service |
| `API` | API |
| `CAP` | Capability |
| `PROV` | Provider |
| `KERN` | Kernel |
| `SDK` | SDK |
| `RUN` | Runtime |
| `PKG` | Package |
| `TEST` | Test |
| `BENCH` | Benchmark |
| `DEPLOY` | Deployment |
| `CFG` | Configuration |
| `PROD` | Product |
| `AGENT` | Agent |
| `PROMPT` | Prompt |
| `CONV` | Conversation |
| `TASK` | Task |
| `STRAT` | Strategy |
| `PLAN` | Execution Plan |
| `DS` | Dataset |
| `MKT` | Market |
| `FX` | Forex |
| `NEWS` | News |
| `DOC` | Document |
| `KB` | Knowledge Base |

---

## Namespace Resolution Rules

### NR-001 — Uniqueness within Namespace
`canonical_name` must be unique within its namespace. The registry enforces this globally.

### NR-002 — Namespace Inheritance
A subdomain inherits all parent namespace policies unless explicitly overridden.

### NR-003 — Cross-Namespace References
An object in namespace `kos` may reference an object in namespace `platform` only via its fully qualified `knowledge_id`. Relative references are forbidden.

### NR-004 — Reserved Namespaces
Namespaces `system`, `internal`, `core`, `root` are reserved. User-defined objects may not be registered in these namespaces.

### NR-005 — Namespace Registration
New top-level domains require Architecture Board approval. Subdomains may be created by domain owners without approval.

### NR-006 — Alias Scoping
Aliases are scoped to their namespace. The same alias string may appear in different namespaces if they resolve to different objects.

---

## Canonical Name Computation

```
canonical_name = namespace + "." + type_lower + "." + slug

Where:
  slug = knowledge_id[last segment].lower().replace("-", "_")

Examples:
  KNW-KOS-ARCH-001    → "kos.arch.001"
  KNW-AI-MOD-quota_manager → "ai.module.quota_manager"
  KNW-PLAT-SVC-001    → "platform.service.001"
```

---

## Namespace Registry Format

```yaml
# namespace-registry.yaml
namespaces:
  kos:
    code: KOS
    full_name: "Knowledge Operating System"
    owner: "team:architecture-board"
    status: ACTIVE
    subdomains:
      arch: {code: ARCH, owner: "team:architecture-board"}
      spec: {code: SPEC, owner: "team:architecture-board"}
  platform:
    code: PLAT
    full_name: "AI Studio Platform"
    owner: "team:platform-core"
    status: ACTIVE
  ai:
    code: AI
    full_name: "AI Runtime"
    owner: "team:ai-runtime"
    status: ACTIVE
```

---

## Namespace Governance

| Action | Authority | Review |
|--------|-----------|--------|
| Register new top-level domain | Architecture Board | Required |
| Register new subdomain | Domain Owner | Not required |
| Register new type code | Architecture Board | Required |
| Rename namespace | Architecture Board | Required + migration plan |
| Deprecate namespace | Architecture Board | Required + sunset notice |

---

## Cross-References

- Identity fields → `01-IDENTITY-ENGINE`
- URI format → `04-URI-SPECIFICATION`
- Registry storage → `15-OBJECT-REGISTRY`
- Governance → Phase 2.1D.0 `33-PLATFORM-GOVERNANCE`
