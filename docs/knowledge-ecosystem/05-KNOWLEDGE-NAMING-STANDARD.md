# KNW-KE-ARCH-005 — Knowledge Naming Standard

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Precise naming rules for every identifier in the Knowledge Ecosystem. No ambiguity. Machine-verifiable.

---

## Knowledge ID

Format: `KNW-{DOMAIN}-{TYPE}-{NNN}`

```
Regex: ^KNW-[A-Z][A-Z0-9]*-[A-Z][A-Z0-9]*-[A-Za-z0-9_-]+$

Segment 1: "KNW"       — literal prefix, always
Segment 2: DOMAIN      — uppercase, 2-6 chars, matches package domain_code
Segment 3: TYPE        — uppercase, 2-6 chars, matches object type abbreviation
Segment 4: NNN         — zero-padded 3-digit integer minimum; may be alphanumeric for sub-objects
```

**Domain codes:**

| Domain | Code | Package |
|--------|------|---------|
| Platform | PLT | kos.platform.package |
| Runtime | RT | kos.runtime.package |
| Provider | PROV | kos.provider.package |
| Algorithm | ALG | kos.algorithm.package |
| Pattern | PAT | kos.pattern.package |
| API | API | kos.api.package |
| Test | TEST | kos.test.package |
| Financial | FIN | kos.financial.package |
| Meta | META | kos.meta.package |
| Architecture | ARCH | kos.pattern.package (shared) |
| Quality | QA | kos.test.package (shared) |

**Type abbreviation codes:**

| Object Type | Code |
|-------------|------|
| MODULE | MOD |
| SERVICE | SVC |
| ALGORITHM | ALG |
| CONFIGURATION | CFG |
| API | API |
| CAPABILITY | CAP |
| PROVIDER | PROV |
| TEST | TST |
| BENCHMARK | BENCH |
| REQUIREMENT | REQ |
| DECISION | ADR |
| ARCHITECTURE | ARCH |
| PATTERN | PAT |
| SPECIFICATION | SPEC |
| DEPLOYMENT | DEPL |
| PRODUCT | PROD |
| AGENT | AGNT |
| PROMPT | PRMT |
| CONVERSATION | CONV |
| TASK | TASK |
| STRATEGY | STRAT |
| EXECUTION_PLAN | PLAN |
| MARKET | MKT |
| FOREX | FX |
| NEWS | NEWS |
| DATASET | DS |
| RUNTIME | RT |
| KERNEL | KRN |
| SDK | SDK |
| PACKAGE | PKG |
| DOCUMENT | DOC |
| KNOWLEDGE_BASE | KB |
| PLATFORM | PLT |

**Examples:**
```
KNW-PLT-MOD-001     platform module #1
KNW-ARCH-REQ-042    architecture requirement #42
KNW-FIN-MKT-007     financial market #7
KNW-ALG-ALG-012     algorithm #12
KNW-TEST-BENCH-003  benchmark #3
```

---

## Canonical Name

Format: `{namespace}.{type_lower}.{slug}`

```
namespace  = lower-cased domain code   (e.g., plt, arch, fin)
type_lower = object_type.value.lower() (e.g., module, requirement, market)
slug       = slugify(name)

slugify rules:
  1. lowercase all
  2. spaces → hyphens
  3. underscores → hyphens
  4. remove all chars except [a-z0-9-]
  5. collapse multiple hyphens → single
  6. strip leading/trailing hyphens
  7. empty result → "unnamed"
```

**Examples:**
```
"Quota Manager"    → plt.module.quota-manager
"BFS Algorithm"    → alg.algorithm.bfs-algorithm
"USD/JPY Pair"     → fin.forex.usdjpy-pair
```

---

## Knowledge URI

Format: `knw://{namespace}/{type_lower}/{slug}[@{version}][#{fragment}]`

```
Scheme:    knw
Authority: {namespace}       (e.g., plt, arch, fin)
Path:      /{type_lower}/{slug}
Version:   @{semver}         optional; omit for "latest"
Fragment:  #{section}        optional; for linking to sub-sections
```

**Version channels:**
```
knw://plt/module/quota-manager@latest        → resolves to highest CANONICAL version
knw://plt/module/quota-manager@stable        → highest VERIFIED+ version
knw://plt/module/quota-manager@1.0.0         → exact version pin
knw://plt/module/quota-manager@^1.0.0        → semver range (SemVer compat)
```

**Fragment examples:**
```
knw://plt/module/quota-manager@1.0.0#evidence
knw://alg/algorithm/bm25@1.0.0#pseudocode
```

---

## Package ID

Format: `kos.{namespace}.package`

```
kos.platform.package
kos.runtime.package
kos.provider.package
kos.algorithm.package
kos.pattern.package
kos.api.package
kos.test.package
kos.financial.package
kos.meta.package
```

---

## Tag Names

```
Rules:
  - All lowercase
  - Words separated by hyphens (not underscores)
  - No spaces
  - Minimum 2 chars, maximum 40 chars
  - Alphanumeric and hyphens only

Valid:    platform, quota, resource-management, bm25, ai-provider
Invalid:  Platform, resource_management, ai provider, QUOTA
```

---

## Owner Identifiers

Format: `{scope}:{name}`

```
Scopes:
  team:     team name (kebab-case)      e.g., team:platform, team:financial
  user:     user identifier             e.g., user:jane.doe
  system:   automated system            e.g., system:ci, system:linter

Rules:
  - name is kebab-case (lowercase, hyphens)
  - no special characters except hyphens and dots
```

---

## File Names

```
Object files:   {knowledge-id-lower}.yaml    e.g., knw-plt-mod-001.yaml
                OR {slug}.yaml               e.g., quota-manager.yaml

Template files: {type_lower}-template.yaml   e.g., module-template.yaml

Schema files:   {type_lower}-schema.json     e.g., module-schema.json

Relationship:   {source-kid}_{rel_type}_{target-kid}.yaml
                e.g., KNW-PLT-MOD-001_DEPENDS_ON_KNW-RT-RT-001.yaml
```

---

## Event Names

Format: `knowledge.{domain}.{verb}`

```
knowledge.object.created
knowledge.object.status_changed
knowledge.object.deprecated
knowledge.package.installed
knowledge.package.upgraded
knowledge.registry.rebuilt
knowledge.linter.failed
knowledge.ci.passed
```

---

## Cross-References

- Domain code table → `02-KNOWLEDGE-PACKAGES`
- Slug algorithm → Phase 3.0B `identity/engine.py`:`_slugify()`
- URI resolution protocol → Phase 3.0C `04-URI-SPECIFICATION`
- Tag registry → `09-KNOWLEDGE-CATALOG`
