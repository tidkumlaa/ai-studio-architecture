# KIP Architecture — Frozen Specification

Status: **FROZEN**. This document is the permanent architectural contract for
the Knowledge Ingestion Pipeline (KIP, `studio.kip`). It describes the system
as it exists today — every class, boundary, and rule named below is real,
already implemented, and already enforced by tests. It does not propose
changes and introduces no new APIs.

Every future subsystem that touches KIP — Runtime wiring, the CLI, VS Code,
MCP, Cloud, Search, or AI features — must build *on top of* this contract,
not around it.

---

## 1. Vision

**Purpose.** KIP turns arbitrary source documents (`.txt`, `.md`, `.csv`,
`.docx`, `.pdf`, `.xlsx`) into validated, permanently-identified Knowledge
Objects inside the Knowledge Intelligence Graph (KIL). It is the *only*
supported path by which external, unstructured content becomes part of the
knowledge corpus.

**Why KIP exists.** Before KIP, every document entering KIL had to be
hand-authored or hand-mapped into a `KnowledgeObject`. KIP mechanizes that:
deterministic, rule-based extraction and classification, with no AI, no NLP,
and no semantic inference anywhere in the pipeline. The same input file
produces the same output every time it is run — a hard requirement, not an
implementation detail (see [§7 Determinism Rules](#7-determinism-rules)).

**KIP vs. KIL.**

| | KIP (`studio.kip`) | KIL (`tools.knowledge`) |
|---|---|---|
| Role | *Producer* — turns documents into candidate Knowledge Objects | *Store* — owns the permanent Knowledge Registry and Knowledge Graph |
| State | Mostly stateless/pure; the only cross-call state is `DocumentCatalog` (fingerprint history) | Long-lived, stateful (`KIGRuntime`) |
| Write access to the graph | None, except through one designated file | Owns `add_node()` / `update_node()` / `remove_node()` directly |
| Lifecycle | A pipeline invoked per ingestion run | A runtime instance built once per process/session and reused |
| Dependency direction | KIP depends on KIL (one narrow import surface) | KIL has zero knowledge of KIP |

KIP is upstream of KIL. KIL never imports, calls, or knows about `studio.kip`.

---

## 2. Layer Diagram

```
KnowledgeDocument                  (studio.kip.document)
      ↓  Loader                    (studio.kip.loaders — one per format)
KnowledgeDocument (loaded)
      ↓  Normalizer                (studio.kip.normalizer)
KnowledgeDocument (normalized)  ──────────────┐
      ↓  Analyzer                             │  fingerprint (SHA-256) ──→ DocumentCatalog
DocumentStructure                             │                           (NEW/MODIFIED/UNCHANGED)
      ↓  Extractor                            │
CandidateFragment (×5 sub-extractors)         │
      ↓  CandidateBuilder → CandidateFactory  │
KnowledgeCandidate (KIL-independent)          │
      ↓  KnowledgeMapper  ←── KnowledgeTypeResolver
KnowledgeObject draft (id = "")               │
      ↓  KnowledgeValidator                   │
ValidationResult (valid / issues)             │
      ↓  ── all of the above orchestrated by Importer ──
ImportOutcome  (+ ImportSummary aggregation, ImportPolicy, ImportHooks)
      ↓  KIL Adapter        ←── IdAssigner (permanent id)
AdaptOutcome
      ↓  KIGRuntime.add_node()
Knowledge Registry  (self._nodes: id → KnowledgeNode)
      ↓
Knowledge Graph     (self._graph: vertices + edges)
      ↓
KIG Runtime (queryable, the surface Runtime/Integration consume)
```

### Public Application Service Layer

External callers (Runtime today; MCP/Cloud in future) do not orchestrate
Importer and KilAdapter themselves. `KipImportService`
(`studio.kip.import_service`) is the one sanctioned public entry point that
composes them:

```
External caller (e.g. Runtime, via KipIngestor)
      ↓
KipImportService.ingest(path, kig_runtime)   (Public Application Service Layer)
      ↓ (internally, unchanged)
Importer  →  KilAdapter.adapt_many(outcomes, kig_runtime, catalog)
      ↓
IngestSummary
```

Built once per session by `KipLoaderBuilder.build_import_service()` —
construction only, no business logic in the Composition Root itself (see
[§4](#4-dependency-rules), [§13 ADR-007](#13-architecture-decision-records-adr)).

### Layer responsibilities

| Layer | Module | Responsibility | Never does |
|---|---|---|---|
| Loader | `studio.kip.loaders.*` | Parse one source format into a `KnowledgeDocument` | Normalize, classify, validate |
| Normalizer | `studio.kip.normalizer.normalizer` | Clean/canonicalize document content | Structural analysis |
| Analyzer | `studio.kip.analyzer.analyzer` (+4 sub-analyzers) | Discover deterministic structure → `DocumentStructure` | Construct a `KnowledgeObject` |
| Extractor | `studio.kip.extractor.extractor` (+5 sub-extractors) | Turn structure facts into `CandidateFragment`s | Construct a `KnowledgeCandidate` |
| Candidate | `studio.kip.candidate.*` | `CandidateFactory` (sole constructor) + `CandidateBuilder` turn fragments into immutable `KnowledgeCandidate`s; `CandidateRegistry` holds them for one session | Touch KIL types |
| KnowledgeTypeResolver | `studio.kip.mapper.knowledge_type_resolver` | Rule-based `KnowledgeType` code resolution (filename → extension → directory → metadata → default) | Import `tools.knowledge` |
| KnowledgeMapper | `studio.kip.mapper.knowledge_mapper` | Map one `KnowledgeDocument` + its candidates → one draft `KnowledgeObject` (`id=""`) | Assign a permanent id, write to KIL |
| KnowledgeValidator | `studio.kip.validator.knowledge_validator` | Structural correctness gate → `ValidationResult` | Mutate, write, raise |
| Fingerprint | `studio.kip.catalog.fingerprint` | Pure SHA-256 content fingerprint of a `KnowledgeDocument` | Persist anything |
| DocumentCatalog | `studio.kip.catalog.catalog` (+`catalog_store` for persistence) | Classify NEW/MODIFIED/UNCHANGED per source path; track version + `target_object_id` | Assign ids, write to KIL |
| Importer | `studio.kip.importer.importer` | Orchestrate Loader→…→Validator for one file/directory → `ImportOutcome` | Write to KIL |
| KIL Adapter | `studio.kip.adapter.kil_adapter` (+`id_assigner`) | Assign the permanent id; perform the one real KIL write (`kig_runtime.add_node()`) | Parse, normalize, map, validate |
| Knowledge Registry | `tools.knowledge.kig.KIGRuntime._nodes` | Id → `KnowledgeNode` lookup | — (owned entirely by KIL) |
| Knowledge Graph | `tools.knowledge.kig.KIGRuntime._graph` | Vertices/edges, queries, traversal | — (owned entirely by KIL) |
| KipImportService | `studio.kip.import_service` | Public Application Service Layer — composes Importer + KilAdapter into one call for external callers (e.g. Runtime) | Parse, normalize, map, validate, or write directly — delegates every one of those to the stages above |

---

## 3. Public APIs

Stability legend: **STABLE** — frozen public surface, callers may depend on
signature and behavior. **INTERNAL** — implementation detail of a stable
class; may change without notice, never imported directly by callers outside
its own module.

| Class / function | Module | Responsibility | Inputs | Outputs | Owner | Stability |
|---|---|---|---|---|---|---|
| `KnowledgeDocument`, `DocumentSection`, `DocumentTable`, `DocumentFormat` | `document.py` | Canonical intermediate document model | — | — | Loader output shape | STABLE |
| `LoaderRegistry`, `DocumentLoader` (+6 format loaders) | `loaders/*` | Format-specific parsing | file path | `KnowledgeDocument` | Loader stage | STABLE |
| `Normalizer` | `normalizer/normalizer.py` | Content cleaning | `KnowledgeDocument` | `KnowledgeDocument` | Normalizer stage | STABLE |
| `Analyzer` | `analyzer/analyzer.py` | Structure discovery | `KnowledgeDocument` | `DocumentStructure` | Analyzer stage | STABLE |
| `DocumentStructure` and its node types | `structure.py` | Structural fact model | — | — | Analyzer output shape | STABLE |
| `Extractor` | `extractor/extractor.py` | Fragment discovery | `DocumentStructure` | `tuple[CandidateFragment, ...]` | Extractor stage | STABLE |
| `CandidateFragment`, `CandidateLocation`, `KnowledgeCandidate` | `candidate/candidate.py` | Candidate data model | — | — | Candidate stage | STABLE |
| `CandidateFactory` | `candidate/candidate_factory.py` | **Sole** constructor of `KnowledgeCandidate` | fragment + source | `KnowledgeCandidate` | Candidate stage | STABLE |
| `CandidateBuilder` | `candidate/candidate_builder.py` | Fragments → candidates, batch form | fragments + source | `tuple[KnowledgeCandidate, ...]` | Candidate stage | STABLE |
| `CandidateRegistry` | `candidate/candidate_registry.py` | Session-scoped candidate holding | — | — | Candidate stage | STABLE |
| `KnowledgeTypeResolver` | `mapper/knowledge_type_resolver.py` | Type-code resolution | `KnowledgeDocument` | `str` (type code) | Mapper stage | STABLE |
| `KnowledgeMapper` | `mapper/knowledge_mapper.py` | Draft `KnowledgeObject` construction | document + candidates | `tools.knowledge.models.KnowledgeObject` (`id=""`) | Mapper stage; **1 of 6 KIL-bridge files** | STABLE |
| `KnowledgeValidator`, `ValidationResult`, `ValidationIssue` | `validator/knowledge_validator.py` | Structural validation | `KnowledgeObject` | `ValidationResult` | Validator stage; **1 of 6 KIL-bridge files** | STABLE |
| `compute_fingerprint` | `catalog/fingerprint.py` | Deterministic content hash | `KnowledgeDocument` | `str` (hex digest) | Catalog stage | STABLE |
| `DocumentCatalog`, `CatalogEntry`, `ChangeType` | `catalog/catalog.py` | Fingerprint history, change classification | source + fingerprint | `ChangeType` | Catalog stage | STABLE |
| `CatalogStore`, `CatalogCorruptedError` | `catalog/catalog_store.py` | JSON persistence of a `DocumentCatalog` | path | `DocumentCatalog` | Catalog persistence | STABLE |
| `Clock`, `SystemClock` | `importer/clock.py` | Injectable time source | — | ISO timestamp string | Importer collaborator | STABLE |
| `ImportOutcome` | `importer/import_outcome.py` | Per-file pipeline result | — | — | Importer output shape; **1 of 6 KIL-bridge files** | STABLE |
| `ImportPolicy` | `importer/import_policy.py` | Batch continue/stop control | — | — | Importer stage | STABLE |
| `ImportHooks` (Protocol) | `importer/import_hooks.py` | Optional observation seam | — | — | Importer stage | STABLE |
| `ImportSummary`, `summarize()` | `importer/import_summary.py` | Pure aggregation over outcomes | `tuple[ImportOutcome, ...]` | `ImportSummary` | Importer stage | STABLE |
| `Importer` | `importer/importer.py` | Full-pipeline orchestration | file/directory path | `ImportOutcome` / `tuple[ImportOutcome, ...]` | Importer stage | STABLE |
| `IdAssigner` | `adapter/id_assigner.py` | Deterministic permanent id generation | source path | `str` (`"KIP-{SLUG}"`) | KIL Adapter stage | STABLE |
| `KilAdapter`, `AdaptOutcome`, `AdaptStatus` | `adapter/kil_adapter.py` | The one real KIL write | `ImportOutcome` + `KIGRuntime` + `DocumentCatalog` | `AdaptOutcome` | KIL Adapter stage; **1 of 6 KIL-bridge files** | STABLE |
| `KipLoaderBuilder` | `composition_root.py` | **Sole** composition root — constructs every collaborator above, including `KipImportService` (via `build_import_service()`); construction only, no business logic | — | fully-wired collaborators | Whole package | STABLE (name frozen since Phase 1) |
| `KipImportService`, `IngestSummary` | `import_service.py` | Public Application Service Layer — the sanctioned entry point for external callers; composes `Importer` + `KilAdapter` for one path | file/directory path + `KIGRuntime` | `IngestSummary` (imported/skipped/failed counts) | Public Application Service Layer; **1 of 6 KIL-bridge files** | STABLE |
| CLI `main()` | `cli.py` | `kip ingest <path>` entry point | argv | process exit code | CLI stage; **1 of 6 KIL-bridge files** | STABLE |
| Individual sub-loaders, sub-analyzers, sub-extractors (e.g. `HeadingExtractor`, `TableAnalyzer`) | respective `*/*.py` | Single-responsibility collaborators composed by their parent stage | — | — | Their parent stage | INTERNAL — construct only via `KipLoaderBuilder` or the parent stage, never call directly from outside the package |

---

## 4. Dependency Rules

### Allowed

- Any `studio.kip` file may depend on: the Python standard library, `python-docx`, `pypdf`, `openpyxl` (the Phase 1 third-party allowlist), and sibling `studio.kip` modules.
- Exactly 6 files may import `tools.knowledge` (the "KIL bridge set"): `mapper/knowledge_mapper.py`, `validator/knowledge_validator.py`, `importer/import_outcome.py`, `adapter/kil_adapter.py`, `cli.py`, `import_service.py`. Each carries or constructs a real `KnowledgeObject`/`KIGRuntime` and would lose type safety with a stand-in type. `import_service.py` (Phase 13, the Public Application Service Layer) is the newest member — `composition_root.py` deliberately does **not** join this set; it only constructs a `KipImportService`, never calls into `KIGRuntime` itself (Composition Root discipline, [§13 ADR-007](#13-architecture-decision-records-adr)).
- Only `adapter/kil_adapter.py` may call `KIGRuntime.add_node()`.
- Only `candidate/candidate_factory.py` may construct `KnowledgeCandidate`.
- Only `mapper/knowledge_mapper.py` and `validator/knowledge_validator.py` construct or accept a `tools.knowledge.models.KnowledgeObject` as a first-class value (Mapper constructs it; Validator inspects it — neither hands construction to any other file).

### Forbidden

| Rule | Enforced by |
|---|---|
| No `studio.kip` file imports `platform` or `platform_kernel` | `test_no_forbidden_top_level_imports` |
| No `studio.kip` file outside the 6-file bridge set imports `tools` (`tools.knowledge`) | `test_only_designated_files_import_tools_knowledge` |
| `composition_root.py` (the Composition Root) imports zero `tools.knowledge` — construction only, no business logic | `test_composition_root_has_no_kil_or_platform_dependencies` |
| `studio.runtime` may depend on `studio.kip` only through `studio.kip.composition_root` (construction) and `studio.kip.import_service` (`KipImportService`, business logic) — never `mapper`/`validator`/`catalog`/`adapter`/`importer` internals | `test_kip_dependency_limited_to_composition_root` (Runtime's own dependency-footprint test) |
| No `studio.kip` file imports `studio.runtime` or `studio.integration`, in any form | `test_never_imports_studio_runtime_or_studio_integration` (AST-based — a docstring mentioning the name does not trip it) |
| No third-party import outside the Phase 1 allowlist (`docx`, `pypdf`, `openpyxl`) | `test_third_party_imports_limited_to_phase_1_allowlist` |
| No file other than `adapter/kil_adapter.py` calls `.add_node()` | `test_only_kil_adapter_calls_add_node` |
| No file other than `candidate/candidate_factory.py` constructs `KnowledgeCandidate` | `test_only_candidate_factory_constructs_knowledge_candidate` |
| `Loader` must never import Runtime | covered by the blanket `studio.runtime` prohibition above |
| `Importer` must never import `tools.knowledge` | `test_importer_and_clock_have_no_kil_or_platform_dependencies` |
| `KnowledgeTypeResolver`, `DocumentCatalog`, `CatalogStore`, `IdAssigner`, `ImportPolicy`/`ImportHooks`/`ImportSummary` must never import `tools.knowledge` | individually asserted, e.g. `test_id_assigner_has_no_kil_or_platform_dependencies`, `test_catalog_has_no_kil_or_platform_dependencies`, `test_import_policy_hooks_and_summary_have_no_kil_or_platform_dependencies` |

All of the above are enforced continuously by `tests/test_kip_dependency_footprint.py` — an AST-walking test suite, not a style guideline. A change that violates any rule fails CI, it does not merely violate a convention.

---

## 5. Frozen Boundaries

The following types and classes are frozen: their names, module locations,
and public signatures do not change. Future work **extends** them (new
fields via additive dataclass changes, new methods, new optional
parameters with safe defaults) — it never renames, relocates, or replaces
them.

| Boundary | Why frozen |
|---|---|
| `KnowledgeDocument` (+ `DocumentSection`, `DocumentTable`, `DocumentFormat`) | Every downstream stage, all 6 loaders, and all tests are written once against this shape |
| `DocumentStructure` (+ `HeadingNode`, `SectionNode`, `TableLocation`, `HyperlinkLocation`, `ListItem`, `MetadataEntry`) | Sole analyzer output contract consumed by Extractor |
| `KnowledgeCandidate` / `CandidateFragment` | KIL-independent candidate representation; changing it ripples through 5 extractors + Mapper |
| `ImportOutcome` | Consumed by `KilAdapter`, `ImportSummary.summarize()`, the CLI, and ~64 existing tests across Phases 8–11 |
| `ImportSummary` / `summarize()` | Pure aggregation contract; additive-only by design (a "closes a gap" utility, not a pipeline stage) |
| `ImportPolicy` | Two-member enum controlling batch behavior; the additive Phase 7 revision that must not become a breaking one |
| `ImportHooks` | `Protocol`; purely observational, must never gain the power to alter `Importer`'s return values |
| `KilAdapter` | The single write boundary into KIL (Gate 1) — its name and `adapt()`/`adapt_many()` signatures are the permanent contract every future caller (CLI, Runtime, MCP, …) integrates against |
| `IdAssigner` | The sole owner of permanent id generation (Gate 2); its determinism (`same path → same id, forever`) is load-bearing for idempotent re-ingestion |
| `KipLoaderBuilder` | Composition root; name frozen since Phase 1 even though scope grew far past "loader" — new phases add a `build_*()` method here, never a second composition root |
| `KipImportService` | The Public Application Service Layer's permanent entry point (Phase 13) — external callers (Runtime, future MCP/Cloud) integrate against `KipImportService.ingest()`, never against `Importer`/`KilAdapter` individually |

Any change to one of these that is not additive (rename, signature-breaking
change, relocation, removal) requires a new architecture decision, not a
silent edit under this document's authority.

---

## 6. Extension Points

KIP was built with specific, named seams for future growth. Extension always
happens by **adding**, never by modifying a frozen boundary.

| Extension point | Mechanism | Example future use |
|---|---|---|
| New loader | `KipLoaderBuilder.with_loader(loader)` (fluent, before `.build()`) or a new default in `_default_loaders()` | OCR loader, PPTX loader |
| New analyzer | New sub-analyzer composed into `Analyzer`, following the existing `heading_analyzer.py` / `table_analyzer.py` / `hyperlink_analyzer.py` / `metadata_analyzer.py` pattern | Diagram/image structure analyzer |
| New extractor | New sub-extractor composed into `Extractor`, following the existing 5-extractor pattern; output is still a `CandidateFragment`, routed through `CandidateFactory` unchanged | Code-block extractor |
| New validator rule | New `_check_*` method inside `KnowledgeValidator.validate()` | Duplicate-id check (deferred, becomes real once ids are populated pre-validation) |
| New fingerprint algorithm | `compute_fingerprint`'s signature (`KnowledgeDocument -> str`) is the contract; internal hashing strategy may evolve (e.g. multi-dimensional fingerprint, precedented by `platform/knowledge_runtime`'s `KnowledgeFingerprint`) without callers changing | Structure-aware or metadata-aware fingerprinting |
| Importer hooks | Implement the `ImportHooks` protocol and pass it to `KipLoaderBuilder.build_import_service(hooks=...)` | Progress reporting, structured logging, metrics emission |
| CLI | `studio/kip/cli.py` is its own entry point (`kip` console script); new subcommands are added there, never merged into `studio.runtime`'s CLI | `kip validate`, `kip diff` |
| Runtime | Consumes KIP exclusively through `KipLoaderBuilder.build_import_service()` → `KipImportService` (the Public Application Service Layer), sharing a caller-owned `KIGRuntime` (see [§9](#9-runtime-integration)) — implemented, not hypothetical, as of Phase 13 | Runtime-triggered ingestion of user-uploaded documents (`aistudio ingest <path>`, live today) |
| VS Code | Never talks to KIP directly — only through Runtime (see [§10](#10-vs-code-integration)) | "Import this file into knowledge base" command |
| MCP / Cloud | Same rule as Runtime: integrate through `KipImportService`, never through internal modules or by calling `KIGRuntime.add_node()` directly | Remote ingestion endpoint |

---

## 7. Determinism Rules

KIP guarantees that **the same input, run at any time, on any machine,
produces byte-identical output** (modulo the one deliberate real-time seam
noted below). This is not a style preference — every phase's completion
report was verified by running its test suite twice and diffing output.

| Rule | Where it holds | Why |
|---|---|---|
| No wall-clock time | Every stage except `Importer`'s injected `Clock` | `datetime.now()` scattered through the pipeline would make fingerprinting, id assignment, and re-ingestion outcomes non-reproducible between runs |
| No random values | Everywhere | Randomness anywhere upstream of `IdAssigner` would break the "same source → same id" idempotency guarantee |
| No UUID generation | Everywhere, especially `IdAssigner` | `IdAssigner` uses a pure path-slug transform (`KIP-{SLUG}`), not `uuid4()` — a UUID would make re-ingesting an unchanged file create a duplicate node instead of overwriting |
| No hidden mutable state | Everywhere except `DocumentCatalog` (explicitly, intentionally stateful — that is its entire purpose) and `CandidateRegistry` (explicitly session-scoped) | Hidden state breaks the isolation that lets `Importer.import_directory()` process files independently and lets `KilAdapter.adapt_many()` isolate batch failures |
| No implicit singleton | Everywhere; `KipLoaderBuilder` constructs a fresh object graph on every call | A hidden singleton would leak state between unrelated ingestion runs (e.g. two CLI invocations, two Runtime requests) |
| No hidden I/O | Every stage except `Loader` (reads the source file — its entire job), `CatalogStore` (explicit, opt-in JSON persistence), and `KilAdapter` (the one real KIL write) | Every other stage is a pure function of its inputs; this is what makes unit-testing every stage in isolation possible without mocks |

The one deliberate exception — `Clock` — exists precisely because a real
system needs *some* real timestamp to enter the pipeline (Catalog's
`imported_at`). It is injected, not hard-coded (`SystemClock` is the
production default; tests inject a fake clock), which is what keeps this
exception from contaminating everything downstream.

---

## 8. Ownership Matrix

| Layer | Owns |
|---|---|
| Loader | Reading files, format-specific parsing |
| Normalizer | Content cleaning |
| Analyzer | Structural analysis (headings, sections, tables, hyperlinks, lists, metadata) |
| Extractor | Candidate fragment generation |
| Candidate (Factory/Builder/Registry) | `KnowledgeCandidate` construction and session-scoped holding |
| KnowledgeTypeResolver | Deterministic type-code classification |
| Mapper | `KnowledgeObject` draft creation (id left unassigned), URI construction |
| Validator | Structural validation of a `KnowledgeObject` |
| Fingerprint | Deterministic content hashing |
| Catalog | Fingerprint history, NEW/MODIFIED/UNCHANGED classification, version counting |
| Importer | Pipeline orchestration, per-file failure isolation, batch policy |
| KIL Adapter | Permanent id assignment, the one real KIL write, catalog `target_object_id` patching |
| KIL (Knowledge Registry + Knowledge Graph) | Persistence-in-runtime, node/edge storage, everything downstream of `add_node()` (confidence scoring, evidence linkage, graph queries) |

No layer owns more than one row. In particular: **duplicate detection**
belongs to Catalog (via fingerprint comparison), not the KIL Adapter — by
the time an `ImportOutcome` reaches `KilAdapter`, Catalog has already
classified it as NEW/MODIFIED/UNCHANGED; the adapter's own idempotency
(same source → same id → overwrite, not duplicate) is a consequence of
`IdAssigner`'s determinism, not a separate duplicate-detection mechanism.

---

## 9. Runtime Integration

**Current state: wired (Phase 13).** `aistudio ingest <path>` is a real,
shipped command. Runtime consumes KIP exclusively through `KipImportService`
(the Public Application Service Layer, [§3](#3-public-apis)) — it never
imports `Importer`, `KnowledgeMapper`, `KnowledgeValidator`,
`DocumentCatalog`, `KilAdapter`, or `IdAssigner` directly, enforced by
Runtime's own dependency-footprint test
(`test_kip_dependency_limited_to_composition_root`).

- Runtime must never call `KIGRuntime.add_node()` directly. The only sanctioned write path is `KilAdapter.adapt()` / `adapt_many()`, reached exclusively through `KipImportService`.
- Runtime must never bypass `Importer` — i.e. never construct a `KnowledgeObject` itself and hand it straight to `KilAdapter`. Documents enter the graph only by going through the full Loader→…→Validator pipeline, which `KipImportService` runs unchanged.
- Runtime's own Composition Root (`StudioRuntimeBuilder`) imports `KipLoaderBuilder` (KIP's Composition Root) purely to call `build_import_service()` — composition-root-to-composition-root wiring, the same pattern already used for `AgentBridgeBuilder`. The actual business-logic collaborator Runtime holds and calls per request (`KipIngestor`) imports only `KipImportService` — never `KipLoaderBuilder`, never any KIP internal.
- The `KIGRuntime` instance KIP writes into is the same caller-owned, long-lived instance the rest of AI Studio uses — matching the existing pattern in `studio.integration.agent_bridge.AgentBridgeBuilder.with_kig(kig)` ("build once per process/session, reuse across requests"). `StudioRuntimeBuilder` builds exactly one `KIGRuntime` (gated on `CapabilityFlags.document_parsing_enabled`, default `False`) and hands the *same instance* to both `AgentBridgeBuilder.with_kig()` and `KipImportService` — proven, not assumed: ingesting a document and then calling `AgentBridge.search()` finds it, using the identical object.

Implemented integration shape:

```
User: aistudio ingest <path>
      ↓
StudioRuntime.execute_ingest(path)
      ↓
KipIngestor.ingest(path)                              (Runtime-owned, DI-injected)
      ↓
KipImportService.ingest(path, kig_runtime)             (KIP's Public Application Service Layer)
      ↓ (internally, unchanged)
Importer.import_file()/import_directory()  →  KilAdapter.adapt_many(outcomes, kig_runtime, catalog)
      ↓
IngestSummary → ImportRuntimeResult
      ↓
same <shared KIGRuntime instance> ── also passed to ── AgentBridgeBuilder.with_kig(...)
      ↓
AgentBridge.ask()/search() sees the newly ingested node immediately
```

`KipLoaderBuilder.build_import_service()` is construction only — it builds
a `DocumentCatalog`, an `Importer` bound to it, and a `KilAdapter`, then
hands back a `KipImportService`. No pipeline logic lives in the Composition
Root ([§13 ADR-007](#13-architecture-decision-records-adr)).

---

## 10. VS Code Integration

**Current state: no VS Code ↔ KIP integration exists.** This section is
architecture-only, establishing the permanent contract for whenever a VS
Code extension is built or wired to this pipeline.

- The VS Code extension must never call `studio.kip` directly.
- The VS Code extension must never call `tools.knowledge` (KIL) directly.
- There is exactly one sanctioned execution path:

```
VS Code
   ↓
Runtime
   ↓
KIP  (KipImportService, the Public Application Service Layer — per §9)
   ↓
KIL
```

Any design that has VS Code hold a direct reference to a KIP or KIL type is
out of contract, regardless of convenience, because it would create a second,
unenforced write path into KIL — violating Gate 1 ("only the KIL Adapter
writes into KIL") at the system level even though the rule technically still
holds inside `studio.kip` itself.

---

## 11. Future Roadmap

The following are additive future extensions. None of them require a
breaking change to any frozen boundary in [§5](#5-frozen-boundaries); each
is either a new stage plugged in via an existing extension point
([§6](#6-extension-points)) or a new consumer built on top of the existing
public API ([§3](#3-public-apis)).

- **Semantic Search** — consumes the Knowledge Graph read-side; no KIP change required.
- **Embeddings** — a new, optional enrichment step, most naturally attached after `KilAdapter` writes a node (KIL-side), not inside KIP's deterministic pipeline.
- **Vector Index** — KIL-side persistence concern; orthogonal to KIP.
- **Relationship Extraction** — explicitly out of scope for the current Extractor/Candidate layer (Phase 3 decision); would be a new extractor + a new Mapper-side (or KIL-side) relationship-construction step, additive.
- **Auto Classification** — would extend `KnowledgeTypeResolver`'s rule tables or add a new resolution strategy behind the same `resolve() -> str` contract; does not change any caller.
- **Incremental Watcher** — a new caller of the existing `Importer` + `DocumentCatalog` + `CatalogStore` APIs (a filesystem-watching wrapper), no pipeline change.
- **Cloud Sync** — a new `CatalogStore`-like persistence backend behind the same load/save contract, or a new consumer of `KipLoaderBuilder`.
- **Multi-user Collaboration** — a KIL/Runtime-side concern (identity, locking, conflict resolution); KIP's pipeline remains single-writer, deterministic, and stateless per run.

No item on this roadmap requires renaming, relocating, or changing the
signature of anything listed in [§5](#5-frozen-boundaries).

---

## 12. Architecture Principles

These are the permanent engineering rules for KIP. Every future phase,
extension, or integration is expected to uphold all of them.

- **Single Responsibility** — every stage does exactly one thing (parse, normalize, analyze, extract, map, validate, catalog, orchestrate, adapt); no stage absorbs a neighboring stage's job.
- **Dependency Injection** — every collaborator receives its dependencies through its constructor (`Importer`, `KnowledgeMapper`, `KilAdapter`, …); nothing reaches out and constructs its own dependency.
- **Deterministic Execution** — see [§7](#7-determinism-rules) in full; the same input always produces the same output.
- **Immutable Data** — every model in the pipeline (`KnowledgeDocument`, `DocumentStructure`, `KnowledgeCandidate`, `ImportOutcome`, `AdaptOutcome`, …) is a frozen dataclass; transformations produce new values, never mutate in place.
- **Single Write Boundary** — exactly one file (`adapter/kil_adapter.py`) may call `KIGRuntime.add_node()`; enforced by AST-based test, not convention (Gate 1).
- **Composition Root** — exactly one class (`KipLoaderBuilder`) constructs every collaborator in the package; nothing outside it directly instantiates a stage.
- **Stateless Services** — every stage collaborator is stateless except the two explicitly, deliberately stateful exceptions (`DocumentCatalog`, `CandidateRegistry`), both scoped and documented as such.
- **Explicit Dependencies** — no hidden imports, no implicit reach-across-package access; the dependency footprint is enumerable and tested (`test_kip_dependency_footprint.py`).
- **Additive Evolution** — every phase since Phase 7 has extended the pipeline without breaking an existing caller (the Phase 7 `ImportPolicy`/`ImportHooks`/`ImportSummary` revision is the reference example); this document exists to keep that true going forward.
- **Public API Stability** — the classes marked STABLE in [§3](#3-public-apis) are the permanent integration surface for Runtime, CLI, MCP, Cloud, and VS Code; INTERNAL collaborators may change freely behind them.

---

## 13. Architecture Decision Records (ADR)

These records capture the historical decisions, already made and already
implemented, that produced the architecture described in §1–§12. They are
reference material, not new rulings — every decision below is inferred
directly from shipped code, module docstrings, and commit history (all KIP
phases landed 2026-07-02).

### ADR-001 — Importer never writes directly into KIL

**Context.** Phase 7 built `Importer` to orchestrate every stage from Loader
through `KnowledgeValidator`. At that point in the pipeline, a document has
been fully parsed, structured, extracted, mapped into a draft
`KnowledgeObject` (`id=""`), and validated — everything needed to write it
already exists.

**Decision.** `Importer` stops at producing an `ImportOutcome` carrying the
draft object. It never calls `KIGRuntime.add_node()` or constructs a
`KIGRuntime`. The actual write is deferred entirely to a later, dedicated
stage (`KilAdapter`, Phase 8).

**Consequences.** `Importer` (and its ~64 dependent tests across Phases
8–11) can be exercised, including `import_directory()` over a whole tree,
with zero risk of graph side effects — useful for preview/dry-run style
usage. It also makes Gate 1 (single write boundary) achievable: because
Importer structurally cannot write, the "only one file writes" rule only
has to be enforced at one later boundary, not audited across the whole
pipeline.

### ADR-002 — KnowledgeMapper never assigns permanent IDs

**Context.** Phase 4's first draft of `KnowledgeMapper` proposed generating
a permanent id (a deterministic path-slug scheme) at mapping time. Review
rejected it — along with the alternative of reusing KIL's own stateful
`KA-{TYPE}-{NNN}` counter — because id assignment is a write-adjacent
concern and `KnowledgeMapper` runs long before anything is actually written.

**Decision.** `KnowledgeMapper.map()` always sets `id=""`. The identical
path-slug scheme that Phase 4 rejected as the mapper's responsibility was
reintroduced, unchanged, in Phase 8's `IdAssigner` — same algorithm,
relocated owner.

**Consequences.** `KnowledgeMapper` stays a pure function of
`(KnowledgeDocument, candidates) → KnowledgeObject`, safely re-runnable any
number of times with no id-collision risk during iteration or testing.
Permanent id generation happens exactly once per write, immediately before
`add_node()`, which is also what makes ADR-006's idempotent-rewrite
guarantee possible.

### ADR-003 — KnowledgeCandidate is KIL-independent

**Context.** Phase 3 needed an intermediate representation between
`DocumentStructure` (Analyzer output) and whatever eventually becomes a
`KnowledgeObject`. The alternative — extractors producing
`tools.knowledge` types directly — would have pulled a KIL dependency into
every extractor.

**Decision.** `KnowledgeCandidate`/`CandidateFragment` were defined as a
wholly separate model that never imports `tools.knowledge` and never
becomes a `KnowledgeObject`/`KnowledgeNode`/`KnowledgeEdge`/`KnowledgeGraph`.
Whether and how a candidate becomes a `KnowledgeObject` was left entirely to
a later phase (`KnowledgeMapper`).

**Consequences.** All five sub-extractors, `CandidateBuilder`,
`CandidateFactory`, and `CandidateRegistry` — the majority of the pipeline's
file count — have zero legitimate reason to import `tools.knowledge`, which
is what makes the 5-file KIL-bridge allowlist ([§4](#4-dependency-rules))
this narrow. It also means extraction logic can be tested, and extended
with new extractors, without any KIL fixture or mock.

### ADR-004 — Runtime and KIP are intentionally separated

**Context.** After KIP reached feature-completeness (Phases 1–11), a
Runtime-wiring decision was evaluated as a genuine three-option fork: stay
fully separate, add thin CLI delegation, or build deep shared-`KIGRuntime`
integration. `studio.runtime.CapabilityFlags.document_parsing_enabled` /
`workspace_indexing_enabled` (both `False` by default) existed as
pre-built hooks for the deep-integration option.

**Decision.** Stay fully separate. `studio.runtime` does not import
`studio.kip` in either direction (confirmed: no reference to `studio.kip`
anywhere under `studio/runtime` or `studio/integration`). `kip` remains its
own standalone console script rather than a subcommand of
`aistudio`/`claude-studio`.

**Consequences.** Runtime's dependency footprint stays exactly what its own
docstring claims — "depends only on `studio.integration` and
`tools.knowledge`" — undisturbed by KIP's existence or evolution. KIP can
add loaders, phases, or extensions with zero risk to Runtime's certified
surface. If deep integration is wanted later, that is new scope requiring a
fresh decision, not a reopening of this one (see [§20](#20-future-governance)).

**Update (Phase 13).** That fresh decision was made: Runtime now depends on
`studio.kip`, narrowly, through `KipLoaderBuilder.build_import_service()`
(construction) and `KipImportService` (business logic) only — never on any
KIP internal. This does not reopen ADR-004; it exercises exactly the "new
scope, fresh decision" escape hatch this ADR always anticipated. See
[§9](#9-runtime-integration), [§15](#15-package-dependency-diagram).

### ADR-005 — KilAdapter is the only write boundary

**Context.** Once a validated draft `KnowledgeObject` exists, something has
to call `KIGRuntime.add_node()`. Left unconstrained, any future caller
(CLI, Runtime, a test, a script) could call it directly.

**Decision.** Centralize the one real write in a single class
(`KilAdapter`) and enforce it structurally: `test_only_kil_adapter_calls_add_node`
AST-walks every file in `studio/kip` other than `kil_adapter.py` and fails
the build if any of them calls `.add_node`.

**Consequences.** Auditing "what can write to the knowledge graph" reduces
to reading one file. Every current and future caller — the `kip` CLI today;
Runtime, MCP, or Cloud tomorrow — is forced through
`KilAdapter.adapt()`/`adapt_many()`, which also guarantees the
invalid/failed-outcome skip logic and permanent-id assignment happen
consistently, with no code path able to accidentally bypass them.

### ADR-006 — Deterministic execution is mandatory

**Context.** Every KIP phase was verified by running its test suite twice
and diffing output for byte-identical results — a discipline that only
works if nothing in the pipeline can vary between runs. `IdAssigner`'s
idempotent-rewrite guarantee (same source path → same id → the second
`add_node()` call overwrites rather than duplicates) specifically depends
on this.

**Decision.** Forbid wall-clock time, randomness, UUID generation, hidden
mutable state, implicit singletons, and hidden I/O everywhere in the
pipeline, with three explicitly named, narrow exceptions: `Clock` (injected,
`SystemClock` default), `Loader` (reads the source file — its entire job),
and `KilAdapter`/`CatalogStore` (the sanctioned write/persistence points).

**Consequences.** Every stage is unit-testable without mocking time or
identity generation. Re-ingesting an unchanged file is provably idempotent
(verified live against a real `KIGRuntime` in Phase 8, not just asserted).
CI can assert reproducibility rather than trust it.

### ADR-007 — Business logic separated from the Composition Root into KipImportService

**Context.** Phase 13's first Runtime-integration pass added an `ingest()`
method directly to `KipLoaderBuilder` to compose `Importer` + `KilAdapter`
for external callers. Architectural review flagged this as a Composition
Root purity violation: `KipLoaderBuilder`'s entire contract (since Phase 1)
is construction and wiring — it must never execute business behavior.

**Decision.** Extract `ingest()` and its `IngestSummary` return type into a
new class, `KipImportService` (`studio.kip.import_service`) — KIP's Public
Application Service Layer, the sanctioned entry point external callers
integrate against. `KipLoaderBuilder.build_import_service()` replaces
`ingest()`: it only constructs a `DocumentCatalog`, an `Importer` bound to
it, and a `KilAdapter`, and returns a `KipImportService` wired with all
three — no branching, no loops, no calls into either collaborator.

**Consequences.** `composition_root.py` reverts to needing zero
`tools.knowledge` imports, restoring the Composition Root's original,
pre-Phase-13 purity. The KIL-bridge file count could not return to 5 —
something still has to type-hint a real `KIGRuntime` to call
`KilAdapter.adapt_many()` — but that necessity relocated from the
Composition Root (the wrong place) to `import_service.py` (a purpose-built
business-logic module, the right place). Runtime's own dependency
footprint narrows further as a result: its business-logic collaborator
(`KipIngestor`) now imports only `KipImportService`, never
`KipLoaderBuilder` at all.

---

## 14. Boundary Diagram

As of Phase 13, Runtime and KIP are genuinely connected — the diagram below
is the real, implemented shape, confirmed by the actual import graph
([§15](#15-package-dependency-diagram)), not a hypothetical.

### Runtime ↔ KIP (implemented)

```
studio.runtime (CLI: aistudio / claude-studio)
      │
      ├── StudioRuntimeBuilder calls KipLoaderBuilder.build_import_service()
      │        [composition-root-to-composition-root wiring only] → KipImportService
      │
      ├── AgentBridgeBuilder.with_kig(shared_kig) → studio.integration
      │        (AgentBridge, *Service classes)
      │
      └── same shared KIGRuntime instance passed to both branches above
                      ↓                                    ↓
              KipImportService                     studio.integration
        (Public Application Service Layer)          → tools.knowledge
        Importer → KilAdapter (unchanged)
                      ↓                                    ↓
                      └──────────────┬─────────────────────┘
                                      ↓
                        Single Write Boundary
              (only KilAdapter.adapt()/adapt_many() calls
                     add_node() — Gate 1, enforced)
                                      ↓
                            tools.knowledge (KIL)
                         ┌────────────┴────────────┐
                         ↓                          ↓
                 Knowledge Registry           Knowledge Graph
                (KIGRuntime._nodes)          (KIGRuntime._graph)
```

`studio.vscode` (a TypeScript extension under `studio/vscode/extension`) is
not part of this diagram — it contains no reference to `studio.kip`,
`studio.runtime`, or any Python execution path. It is a standalone, unwired
artifact from the diagram's perspective.

### VS Code ↔ Runtime (contract only — not implemented, per §10)

```
VS Code  →  Runtime  →  KIP (KipImportService)  →  Single Write Boundary (KilAdapter)  →  KIL  →  Knowledge Graph
```

Everything left of "Runtime" above is still a future contract, not
today's reality — [§10](#10-vs-code-integration) commits future work to
this shape *if and when* a VS Code extension is wired up; this document
does not schedule or propose building it.

### What crosses each boundary

| Boundary | APIs that cross it |
|---|---|
| CLI → `studio.runtime` | `StudioRuntimeBuilder`, `StudioRuntime`, `Task` |
| `studio.runtime` → `studio.integration` | `AgentBridgeBuilder`, `AgentBridge`, its `*Service` classes |
| `studio.integration` → KIL | `KIGRuntime` (via `with_kig()` / `build_kig()`), `tools.knowledge.models.*` |
| `studio.runtime`'s Composition Root → `studio.kip`'s Composition Root | `KipLoaderBuilder.build_import_service()` — construction only |
| `studio.runtime`'s business logic (`KipIngestor`) → `studio.kip` | `KipImportService` only — never `Importer`/`KilAdapter`/`DocumentCatalog`/`IdAssigner` directly |
| CLI (`kip`) → `studio.kip` | `KipLoaderBuilder` (composition root) |
| `studio.kip` internal pipeline | `KnowledgeDocument` → … → `ImportOutcome` (all STABLE types in [§3](#3-public-apis)) |
| `studio.kip` → KIL (Single Write Boundary) | `KilAdapter.adapt()`/`adapt_many()` only — the sole crossing point, per Gate 1 |
| KIL internal | `KIGRuntime.add_node()` → Knowledge Registry (`_nodes`) → Knowledge Graph (`_graph`), one atomic call |

---

## 15. Package Dependency Diagram

Verified directly against the import graph (not inferred): `tools.knowledge`
imports nothing from `studio.*`; `studio.integration` never imports
`studio.kip`; `studio.kip` never imports `studio.runtime` or
`studio.integration` (enforced by
`test_never_imports_studio_runtime_or_studio_integration`). **As of Phase
13, `studio.runtime` *does* import `studio.kip`** — narrowly and by design,
not the "no dependency either direction" this document previously
described. That statement is now corrected below.

```
studio.vscode  (TypeScript, no Python import graph — isolated today)

studio.runtime  ──depends on──▶  studio.integration  ──depends on──▶  tools.knowledge
      │
      ├──depends on (construction only)──▶  studio.kip.composition_root  (KipLoaderBuilder)
      └──depends on (business logic)─────▶  studio.kip.import_service    (KipImportService)
                                                       │
studio.kip  ─────────────────────depends on (6 bridge files only)────────▶  tools.knowledge

CLI (aistudio / claude-studio)  ──depends on──▶  studio.runtime
CLI (kip)                       ──depends on──▶  studio.kip
```

| From | To | Allowed? | Why |
|---|---|---|---|
| `studio.kip` | `tools.knowledge` | Allowed, restricted to 6 named files | Those files carry real `KnowledgeObject`/`KIGRuntime` values; a stand-in type would lose type safety |
| `studio.integration` | `tools.knowledge` | Allowed | Integration's entire purpose is bridging Runtime to KIL |
| `studio.runtime` | `studio.integration` | Allowed | Documented dependency in Runtime's own `__init__.py` |
| `studio.runtime` | `studio.kip.composition_root` | **Allowed, restricted** | Composition-root-to-composition-root wiring — `StudioRuntimeBuilder` calls `KipLoaderBuilder.build_import_service()` only, mirroring the pre-existing `AgentBridgeBuilder` pattern |
| `studio.runtime` | `studio.kip.import_service` | **Allowed, restricted** | The one business-logic symbol (`KipImportService`) Runtime's `KipIngestor` may hold; enforced by `test_kip_dependency_limited_to_composition_root` |
| `studio.runtime` | `studio.kip.mapper` / `.validator` / `.catalog` / `.adapter` / `.importer` | **Forbidden** | Those remain KIP-internal implementation details; enforced by `test_kip_dependency_limited_to_composition_root` |
| `studio.kip` | `studio.runtime` | **Forbidden** | ADR-004's separation still holds in this direction; enforced by `test_never_imports_studio_runtime_or_studio_integration` |
| `studio.kip` | `studio.integration` | **Forbidden** | Same rule, same enforcement |
| `studio.integration` | `studio.kip` | **Forbidden** | Only `studio.runtime` itself integrates with KIP — Integration's own dependency footprint is unchanged by Phase 13 |
| `tools.knowledge` | anything under `studio.*` | **Forbidden** | KIL is the foundation layer; it must never depend upward on anything that depends on it |
| `studio.kip` | `platform` / `platform_kernel` | **Forbidden** | [§4](#4-dependency-rules), enforced by `test_no_forbidden_top_level_imports` |
| `studio.vscode` | `studio.kip` (directly) | **Forbidden** by contract ([§10](#10-vs-code-integration)) | Only sanctioned path is VS Code → Runtime → KIP |

---

## 16. API Version Matrix

Every STABLE public API ([§3](#3-public-apis)) is at **API Version 1.0** as
of this document. There is only one version in existence — nothing has
reached a 2.0 boundary yet.

| API | Version |
|---|---|
| `KnowledgeDocument` / `DocumentSection` / `DocumentTable` / `DocumentFormat` | 1.0 |
| `DocumentStructure` (+ node types) | 1.0 |
| `Analyzer` | 1.0 |
| `Extractor` | 1.0 |
| `KnowledgeCandidate` / `CandidateFragment` | 1.0 |
| `CandidateFactory` / `CandidateBuilder` / `CandidateRegistry` | 1.0 |
| `KnowledgeTypeResolver` | 1.0 |
| `KnowledgeMapper` | 1.0 |
| `KnowledgeValidator` / `ValidationResult` / `ValidationIssue` | 1.0 |
| `compute_fingerprint` | 1.0 |
| `DocumentCatalog` / `CatalogEntry` / `ChangeType` | 1.0 |
| `CatalogStore` / `CatalogCorruptedError` | 1.0 |
| `Clock` / `SystemClock` | 1.0 |
| `ImportOutcome` | 1.0 |
| `ImportPolicy` | 1.0 |
| `ImportHooks` | 1.0 |
| `ImportSummary` / `summarize()` | 1.0 |
| `Importer` | 1.0 |
| `IdAssigner` | 1.0 |
| `KilAdapter` / `AdaptOutcome` / `AdaptStatus` | 1.0 |
| `KipLoaderBuilder` | 1.0 |
| `KipImportService` / `IngestSummary` | 1.0 |
| CLI (`kip ingest`) | 1.0 |

**Versioning rule.** An **additive** extension — a new optional constructor
parameter with a safe default, a new method, a new dataclass field with a
default value, a new loader/analyzer/extractor plugged in through an
existing extension point ([§6](#6-extension-points)) — keeps every API
above at **Version 1.0**. This is precedented: the Phase 7 revision
(`ImportPolicy`, `ImportHooks`, `ImportSummary`) shipped additively and
`ImportOutcome` stayed 1.0 unchanged.

A **breaking** change — renaming a class or method, removing or
repurposing a field, changing a return type, changing what a frozen
boundary ([§5](#5-frozen-boundaries)) means — requires the API in question
to become **Version 2.0**, requires a new ADR under [§13](#13-architecture-decision-records-adr)
justifying the break, and requires the governance process in
[§20](#20-future-governance).

---

## 17. Stability Levels

Extends the two-level model introduced in [§3](#3-public-apis) with a third
level for completeness. All three are defined below; only the first two
have any members today.

| Level | Meaning | Who may depend on it |
|---|---|---|
| **STABLE** | Frozen public surface; signature and behavior are the permanent contract | Any caller, including future Runtime/VS Code/MCP/Cloud integrations |
| **INTERNAL** | Implementation detail of a stable class; may change without notice | Only its own parent module/stage — never imported directly by outside callers |
| **EXPERIMENTAL** | Not yet part of the frozen contract; may be renamed, changed, or removed without an ADR | Opt-in callers only, with the understanding that it is not covered by [§5](#5-frozen-boundaries) or [§20](#20-future-governance) governance until promoted |

**Classification of every major component today:**

| Component | Level |
|---|---|
| All classes/functions listed in [§3](#3-public-apis)'s table as STABLE (every stage's primary class, every frozen data model, `KipLoaderBuilder`, `KipImportService`, the CLI) | STABLE |
| Individual sub-loaders, sub-analyzers (`heading_analyzer.py`, `table_analyzer.py`, `hyperlink_analyzer.py`, `metadata_analyzer.py`), sub-extractors (`heading_extractor.py`, `table_extractor.py`, `metadata_extractor.py`, `hyperlink_extractor.py`, `list_extractor.py`) | INTERNAL |
| — | EXPERIMENTAL: **none exist in `studio.kip` today.** Every shipped class is either STABLE or INTERNAL; nothing is behind a flag or partially built. (Contrast with `studio.runtime`'s `CapabilityFlags.document_parsing_enabled`/`workspace_indexing_enabled`, which are the closest thing to an EXPERIMENTAL marker in the wider codebase, and both are `False` by default, outside KIP itself.) |

If a future extension is added behind an opt-in mechanism before it is
ready to be frozen, it belongs in this table as EXPERIMENTAL until promoted
to STABLE via the governance process in [§20](#20-future-governance) — it
must not be added directly as STABLE.

---

## 18. AI Agent Engineering Rules

Explicit, concise rules for any autonomous AI coding agent operating on this
codebase. Each rule restates an existing, already-enforced boundary — none
of these are new constraints.

- Never bypass `Importer`. All document ingestion goes through `Importer.import_file()`/`import_directory()`; never call Loader/Normalizer/Analyzer/Extractor/Mapper/Validator directly and assemble the result by hand.
- Never bypass `KilAdapter`. Never call `KIGRuntime.add_node()` from anywhere other than `studio/kip/adapter/kil_adapter.py`.
- Never construct a `KnowledgeObject` outside `KnowledgeMapper`. `KnowledgeMapper.map()` is the sole constructor of a draft `KnowledgeObject` within `studio.kip`.
- Never assign a permanent id outside `IdAssigner`. Never hand-write a `"KIP-…"` id string or replicate its slug algorithm elsewhere.
- Never call `KIGRuntime.add_node()` outside `KilAdapter`.
- Always construct KIP collaborators through `KipLoaderBuilder`. Never `import` and directly instantiate an internal stage class from outside `studio.kip`.
- Always integrate with KIP from outside the package through `KipImportService` (the Public Application Service Layer), never by holding an `Importer`/`KilAdapter`/`DocumentCatalog` reference directly. `KipLoaderBuilder` itself is only for construction (e.g. Runtime's own Composition Root calling `build_import_service()`) — never for business logic.
- Never construct a `KnowledgeCandidate` outside `CandidateFactory`.
- Never violate a dependency rule in [§4](#4-dependency-rules) — in particular, never add a `tools.knowledge` import to any file outside the 6-file bridge set, and never add a `studio.runtime`/`studio.integration` import anywhere in `studio.kip`.
- Never bypass or weaken an AST-based dependency-enforcement test in `tests/test_kip_dependency_footprint.py` to make a change pass — a failing footprint test means the change is architecturally wrong, not that the test is outdated.
- Never rename, relocate, or change the signature of anything listed in [§5](#5-frozen-boundaries) — extend additively instead.
- Never introduce wall-clock time, randomness, or UUID generation into any stage other than the named `Clock` exception ([§7](#7-determinism-rules)).
- When integrating a new consumer (Runtime, MCP, Cloud, VS Code), route it through the public STABLE API only — never reach into an INTERNAL collaborator to "save a step."

---

## 19. Architecture Validation Checklist

Run before every commit that touches `studio/kip` or its boundary with KIL.
Generic by design — applies to any change, not just a specific phase.

```
□ No forbidden imports (platform, platform_kernel) anywhere in studio/kip
□ No studio.runtime dependency anywhere in studio/kip
□ No studio.integration dependency anywhere in studio/kip
□ No direct KIL write outside adapter/kil_adapter.py
□ No call to KIGRuntime.add_node() outside adapter/kil_adapter.py
□ Deterministic behavior preserved (same input -> same output, re-run twice to confirm)
□ No UUID generation introduced anywhere in the pipeline
□ No wall-clock dependency introduced outside the injected Clock
□ Public (STABLE) APIs unchanged in name, signature, and meaning
□ Any new behavior is additive, not a breaking change to a frozen boundary (§5)
□ tests/test_kip_dependency_footprint.py passes in full
□ Composition Root (KipLoaderBuilder) remains the only place collaborators are constructed
□ tools.knowledge import surface still limited to exactly the 6 designated bridge files
□ Composition Root (KipLoaderBuilder) contains construction/wiring only -- no business logic
□ Runtime (if touched) imports only KipLoaderBuilder (construction) and KipImportService (business logic) from studio.kip
□ No new hidden I/O, hidden mutable state, or implicit singleton introduced
```

A change that fails any item above is not ready to merge, regardless of
whether its own unit tests pass — this checklist protects the architecture,
not just the change's local correctness.

---

## 20. Future Governance

This document, together with `tests/test_kip_dependency_footprint.py` as its
executable enforcement mechanism, is the permanent governance authority for
KIP's architecture. Future architectural change follows these rules:

- **Public APIs are frozen.** Everything at API Version 1.0 ([§16](#16-api-version-matrix)) stays at 1.0 through additive change only.
- **Breaking changes require a new Architecture Decision Record.** Any rename, signature change, removal, or redefinition of a frozen boundary ([§5](#5-frozen-boundaries)) must be preceded by a new ADR (continuing the numbering in [§13](#13-architecture-decision-records-adr)) documenting the context, the decision, and the consequences — the same standard ADR-001 through ADR-006 were held to.
- **Dependency rules cannot be bypassed.** The allowed/forbidden dependency directions in [§4](#4-dependency-rules) and [§15](#15-package-dependency-diagram) hold regardless of short-term convenience; a change that requires violating one requires revisiting the rule itself, explicitly, via a new ADR — not a one-off exception.
- **New functionality must be additive.** New loaders, analyzers, extractors, validators, fingerprint strategies, or Importer hooks are added through the extension points in [§6](#6-extension-points), never by modifying a frozen class's existing behavior.
- **Runtime, VS Code, MCP, Cloud, Search, and AI features must integrate through existing public APIs.** Per [§9](#9-runtime-integration) and [§10](#10-vs-code-integration): through `KipImportService` (the Public Application Service Layer), sharing a caller-owned `KIGRuntime` — never by importing an INTERNAL collaborator, never by writing to KIL through any path other than `KilAdapter`.

This section describes governance only. It does not propose, schedule, or
scope any specific future implementation — those decisions are made when
that work is actually undertaken, against this document as the standing
contract.
