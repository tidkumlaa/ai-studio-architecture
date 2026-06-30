# KNW-KE-ARCH-026 — Knowledge Package Manager

**Phase:** 3.0C.5  
**Status:** CANONICAL  
**Owner:** architecture-board  
**Version:** 1.0.0

---

## Purpose

`kos` is the Knowledge Package Manager. It installs, removes, upgrades, and verifies knowledge packages. All package operations are atomic and versioned.

---

## Package Manager Commands

```bash
kos install {package}              # install a package
kos install {package}@{version}    # install specific version
kos install --all                  # install all packages in workspace
kos install --dev                  # include dev/test packages
kos install --frozen               # use lock file exactly; fail if changed

kos remove {package}               # remove a package
kos remove {package} --force       # remove even if other packages depend on it

kos upgrade {package}              # upgrade to latest version
kos upgrade {package}@{version}    # upgrade to specific version
kos upgrade --all                  # upgrade all packages

kos downgrade {package}@{version}  # downgrade to specific version

kos verify {package}               # verify package integrity
kos verify --all                   # verify all installed packages

kos list                           # list installed packages
kos info {package}                 # show package manifest

kos lock                           # generate/update lock file
kos lock --check                   # check lock file is up to date

kos clean                          # remove untracked files
kos cache clear                    # clear download cache
```

---

## Package Manifest Format

```yaml
# knowledge/packages/{name}/package.yaml
id: kos.platform.package
name: Platform Package
version: 1.0.0
namespace: plt
domain_code: PLT
status: CANONICAL
owner: team:platform
description: All Knowledge Objects for the KOS Platform layer.
license: internal
repository: knowledge/packages/platform/

requires_kos: ">=1.0.0"    # minimum kos CLI version

dependencies:
  kos.meta.package:
    version: ">=1.0.0,<2.0.0"
    optional: false

dev_dependencies:
  kos.test.package:
    version: ">=1.0.0"
    optional: true

exports:
  object_types: [MODULE, SERVICE, CONFIGURATION, DEPLOYMENT]
  namespaces: [plt]

quality:
  min_overall_score: 0.75
  min_evidence_score: 0.55
  require_canonical_for_export: true
```

---

## Lock File Format

```yaml
# knowledge/package-lock.yaml
lock_version: "1.0.0"
generated_at: "2026-06-30T00:00:00Z"
packages:
  kos.platform.package:
    version: "1.0.0"
    resolved: "knowledge/packages/platform/"
    checksum: sha256:abc123def456...
    dependencies:
      kos.meta.package: "1.0.0"

  kos.meta.package:
    version: "1.0.0"
    resolved: "knowledge/packages/meta/"
    checksum: sha256:789abc...
    dependencies: {}

  kos.algorithm.package:
    version: "1.0.0"
    resolved: "knowledge/packages/algorithm/"
    checksum: sha256:cba321...
    dependencies: {}
```

---

## Workspace File

```yaml
# knowledge/workspace.yaml (root of all packages)
kos_version: "1.0.0"
packages:
  - kos.platform.package
  - kos.runtime.package
  - kos.provider.package
  - kos.algorithm.package
  - kos.pattern.package
  - kos.api.package
  - kos.test.package
  - kos.financial.package
  - kos.meta.package
default_namespace: meta
```

---

## Install Protocol

```
kos install kos.platform.package

1. Read package.yaml
2. Resolve all dependencies (see 27-KNOWLEDGE-DEPENDENCY-MODEL)
3. Download/copy resolved packages to knowledge/packages/
4. Verify checksums against lock file
5. Run: kos verify {package}
6. Update knowledge/registry/index.yaml
7. Update knowledge/catalog/catalog.yaml
8. Write knowledge/package-lock.yaml
```

Atomicity: if any step fails, roll back all previous steps.

---

## Upgrade Protocol

```
kos upgrade kos.platform.package

1. Check current version
2. Resolve latest compatible version
3. Check for breaking changes (MAJOR bump)
   - If MAJOR: require --allow-major flag
4. Download new version
5. Run migration (if migration guide exists)
6. Verify checksums
7. Update registry and catalog
8. Update lock file
```

---

## Package Verification

```
kos verify kos.platform.package

Checks:
  1. Manifest checksum matches lock file
  2. All objects validate against schemas
  3. All objects pass kos lint
  4. All relationships reference existing objects
  5. All package dependencies are installed
  6. Package quality gates met (min_overall_score)
```

---

## CLI Configuration

```yaml
# ~/.kos/config.yaml (user-level)
registry_url: "file://knowledge/"
cache_dir: "~/.kos/cache/"
color: true
verbosity: normal

[platform]
# workspace-level config overrides
registry_url: "file://knowledge/"
```

---

## Cross-References

- Dependency model → `27-KNOWLEDGE-DEPENDENCY-MODEL`
- Package manifest schema → `10-KNOWLEDGE-SCHEMAS`
- Lock file used by CI → `08-KNOWLEDGE-CI`
- Registry updated on install → `14-KNOWLEDGE-REGISTRY`
