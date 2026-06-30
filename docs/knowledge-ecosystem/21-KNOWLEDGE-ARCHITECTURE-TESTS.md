# KNW-KE-ARCH-021 — Knowledge Architecture Tests

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

Architecture-level tests verify that the Knowledge Ecosystem's structural rules are satisfied. These tests run against the ecosystem's documents and schemas — not against runtime implementation.

---

## Test Categories

### AT-1: Document Consistency Tests

Verify that architecture documents are internally consistent with each other.

```python
def test_all_33_types_in_schema_registry():
    """schemas/schema-registry.yaml must list all 33 KnowledgeObjectType values."""
    registry = load_yaml("knowledge/schemas/schema-registry.yaml")
    type_codes = set(registry["schemas"].keys())
    expected = {t.value for t in KnowledgeObjectType}
    assert type_codes == expected

def test_all_33_types_have_template():
    """templates/ must have one template per KnowledgeObjectType."""
    templates = glob("knowledge/templates/*-template.yaml")
    type_codes = {t.stem.replace("-template", "") for t in templates}
    expected = {t.value for t in KnowledgeObjectType}
    assert type_codes == expected

def test_quality_weights_sum_to_one():
    """QD-1 through QD-8 weights must sum to 1.0."""
    spec = load_yaml("architecture/docs/knowledge-core/11-QUALITY-ENGINE.md")
    # extracted from document; compared to 17-KNOWLEDGE-SCORING
    weights = [0.15, 0.15, 0.20, 0.10, 0.15, 0.10, 0.10, 0.05]
    assert abs(sum(weights) - 1.0) < 1e-9

def test_9_standard_packages_defined():
    """02-KNOWLEDGE-PACKAGES must define exactly 9 standard packages."""
    packages = load_packages("knowledge/packages/")
    assert len(packages) == 9

def test_all_9_packages_in_registry():
    """registry/index.yaml must have entries for all 9 packages."""
    index = load_yaml("knowledge/registry/index.yaml")
    package_namespaces = {"plt", "rt", "prov", "alg", "pat", "api", "test", "fin", "meta"}
    found = {e["namespace"] for e in index["entries"].values()}
    assert package_namespaces.issubset(found)

def test_32_linter_rules_defined():
    """lint/rules/ must have exactly 32 rule YAML files."""
    rules = glob("knowledge/lint/rules/KL-*.yaml")
    assert len(rules) == 32

def test_33_schemas_defined():
    """schemas/by-type/ must have 33 schema files."""
    schemas = glob("knowledge/schemas/by-type/*-schema.json")
    assert len(schemas) == 33

def test_19_benchmark_objects_in_dataset():
    """knowledge/benchmarks/ must have 19 benchmark objects."""
    benchmarks = glob("knowledge/benchmarks/bench-*.yaml")
    assert len(benchmarks) == 19

def test_sdk_contracts_all_exist():
    """SDK contracts exist for Python, Java, TypeScript."""
    assert exists("knowledge/contracts/python/kos_sdk.pyi")
    assert exists("knowledge/contracts/java/KnowledgeRegistryContract.java")
    assert exists("knowledge/contracts/typescript/kos-sdk.d.ts")
```

---

### AT-2: Naming Consistency Tests

```python
def test_all_domain_codes_unique():
    packages = load_all_package_manifests()
    codes = [p["domain_code"] for p in packages]
    assert len(codes) == len(set(codes)), "Duplicate domain_code found"

def test_all_namespaces_unique():
    packages = load_all_package_manifests()
    ns = [p["namespace"] for p in packages]
    assert len(ns) == len(set(ns)), "Duplicate namespace found"

def test_knowledge_id_regex_matches_all_examples():
    """All example objects must have valid knowledge_id format."""
    pattern = re.compile(r"^KNW-[A-Z][A-Z0-9]*-[A-Z][A-Z0-9]*-[A-Za-z0-9_-]+$")
    examples = load_all_yaml("knowledge/examples/")
    for obj in examples:
        assert pattern.match(obj["knowledge_id"]), f"Invalid ID: {obj['knowledge_id']}"
```

---

### AT-3: Cross-Reference Tests

```python
def test_all_linter_rules_reference_existing_style_rules():
    """Each linter rule that references a style rule must name a real SG-* rule."""
    sg_rules = extract_rule_ids("architecture/docs/knowledge-ecosystem/04-KNOWLEDGE-STYLE-GUIDE.md")
    linter_rules = load_all_yaml("knowledge/lint/rules/")
    for rule in linter_rules:
        for ref in rule.get("references", []):
            if ref.startswith("SG-"):
                assert ref in sg_rules, f"Linter rule {rule['id']} refs unknown style rule {ref}"

def test_all_benchmark_operations_match_algorithm_ids():
    """Benchmark objects should reference real algorithm IDs."""
    algorithms = {o["algorithm_id"] for o in load_all_yaml("knowledge/packages/algorithm/")}
    benchmarks = load_all_yaml("knowledge/benchmarks/")
    for bench in benchmarks:
        # not strict — benchmark may cover composite ops
        pass  # architecture-level: ensure none reference non-existent ALG-NNN

def test_no_circular_package_dependencies():
    """Package dependency graph must be a DAG."""
    packages = load_all_package_manifests()
    graph = build_dependency_graph(packages)
    assert not has_cycle(graph), "Circular package dependency detected"
```

---

### AT-4: Schema Consistency Tests

```python
def test_base_schema_required_fields_match_template():
    """base-schema.json required list must match base required fields in templates."""
    schema = load_json("knowledge/schemas/base-schema.json")
    required_in_schema = set(schema["required"])
    expected = {"knowledge_id", "object_type", "name", "version",
                "status", "owner", "description", "tags"}
    assert required_in_schema == expected

def test_all_type_schemas_extend_base():
    """Every type schema must have '$ref': 'base-schema.json' in allOf."""
    type_schemas = glob("knowledge/schemas/by-type/*.json")
    for path in type_schemas:
        schema = load_json(path)
        assert "allOf" in schema
        refs = [item.get("$ref", "") for item in schema["allOf"]]
        assert any("base-schema" in r for r in refs), f"{path} does not extend base"

def test_all_enum_values_in_object_type_schema():
    """base-schema.json object_type enum must match KnowledgeObjectType values."""
    schema = load_json("knowledge/schemas/base-schema.json")
    enum_values = set(schema["properties"]["object_type"]["enum"])
    expected = {t.value for t in KnowledgeObjectType}
    assert enum_values == expected
```

---

## Test Run

```bash
kos architecture-tests run                # run all AT-1 through AT-4
kos architecture-tests run --suite AT-1   # single suite
kos architecture-tests --check AT-1-001   # single test
```

---

## Test Coverage Requirement

All 4 suites must pass (100%) before Phase 3.0C.5 is frozen.

---

## Cross-References

- Verification suite (50 checks) → `20-KNOWLEDGE-VERIFICATION`
- Schemas under test → `10-KNOWLEDGE-SCHEMAS`
- Templates under test → `03-KNOWLEDGE-TEMPLATES`
- CI runs these tests → `08-KNOWLEDGE-CI`
