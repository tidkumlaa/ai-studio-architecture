# Platform Kernel — Governance Rules
# Phase 2.0D.2.6A

---

## 1. Overview

Kernel governance defines the rules that all kernel code, extensions, and
callers must follow. Governance is enforced at three levels:

1. **Architecture rules** — what the kernel may and may not do
2. **Extension rules** — what plugins and modules are permitted to do
3. **Compatibility rules** — how the kernel evolves without breaking callers
4. **Upgrade rules** — how the kernel is deployed and migrated

Governance violations are not compile-time errors — they are detected by
the kernel's governance tooling (analogous to `platform_sdk.governance`).

---

## 2. Architecture Rules

### 2.1 Import Rules

| Rule | Description |
|------|-------------|
| AR-01 | Kernel code may only import from `platform_sdk.*` and Python stdlib |
| AR-02 | Kernel code must never import from runtime, product, desktop, or agent packages |
| AR-03 | Kernel modules must not import from each other in circular fashion |
| AR-04 | Kernel modules in lower groups (A, B) must not import from higher groups |

**Group dependency order (lower may import from lower-or-equal only):**
```
A (Object Foundation) → none
B (Events) → A
C (Services) → A, B
D (Configuration) → A, B
E (Scheduling) → A, B, C, D
F (Diagnostics) → A, B, C
G (Logging) → A, B, F
H (Security) → A, B, C, D, F
I (Context & Workspace) → A, B, C, D, E, F, H
J (Execution) → A, B, C, D, E, F, H, I
K (Infrastructure) → A, B (only)
```

### 2.2 State Rules

| Rule | Description |
|------|-------------|
| SR-01 | No mutable global state outside KernelContainer |
| SR-02 | All shared mutable state must be protected by threading.RLock |
| SR-03 | Configuration is read-only after KernelContainer.boot() except via PlatformConfiguration.set() |
| SR-04 | No state held in Python module-level globals by kernel components |

### 2.3 Interface Rules

| Rule | Description |
|------|-------------|
| IR-01 | All public methods must have type annotations |
| IR-02 | All public methods must document their exception contracts |
| IR-03 | Abstract methods must not have implementation |
| IR-04 | No method may return None where a typed value is expected — use PlatformResult |
| IR-05 | No method may accept **kwargs in a public interface |

### 2.4 Performance Rules

| Rule | Description |
|------|-------------|
| PR-01 | No blocking I/O on kernel-managed threads (use async or background workers) |
| PR-02 | No sleep() in hot paths |
| PR-03 | Every operation must emit a timing metric |
| PR-04 | Any operation exceeding its SLO target must emit kernel.performance.degraded |
| PR-05 | Histograms must use pre-defined bucket sizes (no dynamic bucket creation) |

---

## 3. Extension Rules

Rules governing what plugins and modules are permitted to do:

### 3.1 Plugin Rules

| Rule | Description |
|------|-------------|
| EX-01 | Plugins must not import kernel internal modules (only public kernel API) |
| EX-02 | Plugins must declare all required capabilities in their manifest |
| EX-03 | Plugins must not access Python sys.modules directly |
| EX-04 | Plugins must not modify Python builtins or sys.path |
| EX-05 | Plugins must not spawn OS threads without registering them with PlatformScheduler |
| EX-06 | Plugins must use PlatformLogger (not print() or logging.*) |
| EX-07 | Plugins must not catch and suppress PlatformException — they must re-raise or return PlatformResult.fail |
| EX-08 | Plugin manifests must have a valid semver version string |
| EX-09 | Plugin manifests must have a license field |
| EX-10 | Plugins must not access secrets they did not declare in their manifest |

### 3.2 Module Rules

| Rule | Description |
|------|-------------|
| MR-01 | Module names must be globally unique (reverse-DNS style) |
| MR-02 | Modules must declare their full dependency graph in their manifest |
| MR-03 | Module version constraints must use semver range notation |
| MR-04 | Breaking changes must increment the major version |
| MR-05 | A deprecated module must not be removed until all dependents have migrated |

### 3.3 Event Rules for Extensions

| Rule | Description |
|------|-------------|
| EV-01 | Plugins must not publish events in the `kernel.*` namespace |
| EV-02 | Plugin event types must be namespaced with the plugin's reverse-DNS id |
| EV-03 | All plugin event payloads must be JSON-serializable |
| EV-04 | Plugins must not subscribe to `kernel.security.*` events (security boundary) |

---

## 4. Compatibility Rules

### 4.1 Semantic Versioning for the Kernel

The kernel follows strict semantic versioning:

| Change Type | Version Bump | Allowed Without Deprecation |
|-------------|-------------|----------------------------|
| New public method | Minor | Yes |
| New required parameter | Major | No — must be optional with default first |
| Removed public method | Major | No — must be deprecated for 1 minor version |
| Changed return type | Major | No |
| Changed exception type | Major | No |
| New event emitted | Minor | Yes |
| Removed event | Major | No — must be deprecated |
| New capability required | Minor | Only if new feature; existing flows unaffected |

### 4.2 Deprecation Protocol

```
Step 1: Mark as deprecated in current version
        - Add @deprecated annotation
        - Emit DeprecationWarning in Python when called
        - Add deprecation entry to CHANGELOG.md
        - Emit kernel.deprecation.warning event on first call per session

Step 2: Remove in next major version
        - Remove code
        - Update CHANGELOG.md with removal notice
        - Check all known consumers have migrated (governance tooling)
```

### 4.3 API Surface Contract

The following are part of the public contract and cannot change without a major bump:
- All method signatures in PlatformObject, PlatformEvent, PlatformEventBus,
  PlatformRegistry, PlatformSecurity, PlatformScheduler, PlatformConfiguration
- All event type strings in the `kernel.*` namespace
- All ErrorCode enum values
- All LifecycleState enum values
- The PlatformManifest YAML schema

The following are NOT public contract:
- Internal implementation classes (prefixed with `_`)
- Private methods (prefixed with `_`)
- The KernelContainer constructor signature
- Internal module-to-module interfaces

---

## 5. Upgrade Rules

### 5.1 Migration Safety Rules

| Rule | Description |
|------|-------------|
| UP-01 | Kernel upgrades must not require plugin code changes for minor versions |
| UP-02 | Kernel upgrades must provide a migration guide for major versions |
| UP-03 | PlatformAudit data must be readable by the new kernel version |
| UP-04 | Plugin manifests from the previous major version must be loadable (with warnings) |
| UP-05 | Configuration files from the previous major version must be loadable |

### 5.2 Boot Compatibility

During boot, the kernel checks all loaded plugin manifests:
1. If a plugin requires a kernel version > current → refuse to load (log error)
2. If a plugin requires a kernel version < current → load with compatibility mode
3. Compatibility mode: deprecated APIs remain available; new APIs not guaranteed

### 5.3 Data Migration

If PlatformAudit schema changes between versions:
1. New kernel reads old-format records (backward compatible)
2. Migration tool converts old records to new format (offline, not blocking)
3. Old kernel cannot read new-format records (forward incompatible)

---

## 6. Governance Tooling

The kernel ships a governance CLI (analogous to `platform_sdk.governance`):

```python
class KernelGovernance:
    def check_imports(self) -> list["ImportViolation"]: ...
        # Scans kernel source for AR-01 – AR-04 violations
    
    def check_extensions(
        self, plugin_manager: PluginManager
    ) -> list["ExtensionViolation"]: ...
        # Validates all loaded plugins against EX-01 – EX-10
    
    def check_compatibility(
        self, old_version: str, new_version: str
    ) -> list["CompatibilityViolation"]: ...
        # Compares API surfaces across versions
    
    def dependency_matrix(self) -> str: ...
        # Generates the kernel dependency matrix (table)
    
    def interface_catalog(self) -> str: ...
        # Generates the kernel interface catalog

@dataclass
class ImportViolation:
    rule: str                  # "AR-01"
    source_file: str
    source_line: int
    import_statement: str
    description: str
```

---

## 7. Governance Invariants

1. A kernel release with any AR-01 or AR-02 violation is blocked from merging.
2. No public interface may be removed without first being deprecated for
   at least one minor version release.
3. The kernel's own test coverage must be >= 90% for all public interfaces.
4. Every new public method must have at least one test that verifies its
   error case (PlatformException path).
5. The governance CLI must pass with 0 violations before any kernel release.
