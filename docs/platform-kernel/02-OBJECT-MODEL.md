# Platform Kernel — Object Model
# Phase 2.0D.2.6A

---

## 1. Object Hierarchy

Everything in the kernel derives from PlatformObject.

```
PlatformObject
├── PlatformEntity          (has persistent identity + version)
│   ├── PlatformResource    (owns a bounded system resource)
│   │   ├── PlatformWorkspace
│   │   ├── PlatformProject
│   │   └── PlatformRepository
│   ├── PlatformSession
│   ├── PlatformExecution
│   └── PlatformContext
├── PlatformService         (stateless, injectable, long-lived)
│   ├── PlatformRegistry
│   ├── PlatformEventBus
│   ├── PlatformScheduler
│   ├── PlatformLogger
│   ├── PlatformMetrics
│   ├── PlatformAudit
│   ├── PlatformSecurity
│   ├── PlatformConfiguration
│   └── PlatformServiceLocator
├── PlatformComponent       (stateful, lifecycle-managed, short/long-lived)
│   ├── PlatformPlugin
│   ├── PlatformModule
│   ├── PlatformWorker
│   └── PlatformExtension
├── PlatformMessage         (immutable, value-like, routing metadata)
│   ├── PlatformEvent
│   ├── PlatformCommand
│   └── PlatformQuery
└── PlatformValueObject     (immutable, identity-less)
    ├── PlatformIdentity
    ├── PlatformVersion
    ├── PlatformCapability
    ├── PlatformPermission
    ├── PlatformFeatureFlag
    ├── PlatformResult
    ├── PlatformException
    ├── PlatformManifest
    └── PlatformSettings
```

---

## 2. PlatformObject — Base Type

### 2.1 Purpose
The root of the kernel type system. Every kernel-managed object carries
a kernel-assigned identity, metadata, and lifecycle state.

### 2.2 Responsibilities
- Carry a stable `object_id` (UUID v7, time-ordered)
- Carry `created_at`, `modified_at` timestamps
- Expose lifecycle state (via embedded PlatformLifecycle)
- Hold arbitrary typed metadata (key → typed value map)
- Support equality by `object_id` only
- Emit `object.created` and `object.disposed` events

### 2.3 Public Interface

```python
class PlatformObject(ABC):
    # Identity
    @property
    def object_id(self) -> str: ...          # UUID v7
    @property
    def object_type(self) -> str: ...        # fully-qualified type name
    @property
    def created_at(self) -> datetime: ...
    @property
    def modified_at(self) -> datetime: ...

    # Metadata
    def get_meta(self, key: str) -> Any: ...
    def set_meta(self, key: str, value: Any) -> None: ...
    def has_meta(self, key: str) -> bool: ...
    def all_meta(self) -> dict[str, Any]: ...

    # Lifecycle
    @property
    def lifecycle(self) -> "PlatformLifecycle": ...
    @property
    def state(self) -> "LifecycleState": ...

    # Disposal
    def dispose(self) -> None: ...
    @property
    def is_disposed(self) -> bool: ...

    # Equality
    def __eq__(self, other: object) -> bool: ...   # by object_id
    def __hash__(self) -> int: ...                  # hash(object_id)
    def __repr__(self) -> str: ...
```

### 2.4 Thread Safety
All property reads are thread-safe. `set_meta` uses an internal RLock.
`dispose()` is idempotent and safe to call from any thread.

### 2.5 Invariants
- `object_id` is immutable after construction
- `object_type` is immutable after construction
- Metadata values must be serializable (JSON-compatible)
- Calling any method on a disposed object raises `PlatformException`
  with code `OBJECT_DISPOSED`

---

## 3. PlatformIdentity — Value Object

### 3.1 Purpose
Immutable value object capturing the full identity of any kernel object.
Safe to store, compare, serialize, and transmit across process boundaries.

### 3.2 Fields

```python
@dataclass(frozen=True)
class PlatformIdentity:
    object_id: str          # UUID v7
    object_type: str        # e.g. "platform.kernel.PlatformWorkspace"
    namespace: str          # e.g. "ai-studio.workspace"
    display_name: str       # human-readable label
    version: "PlatformVersion"
    owner_id: str | None    # owning object_id or None for root objects
    tags: frozenset[str]    # e.g. {"workspace", "active", "user:alice"}
    created_at: datetime
```

### 3.3 Operations

```python
class PlatformIdentity:
    def to_uri(self) -> str: ...
        # platform://{namespace}/{object_type}/{object_id}
    
    @classmethod
    def from_uri(cls, uri: str) -> "PlatformIdentity": ...
    
    def matches(self, pattern: str) -> bool: ...
        # glob-style: "platform.kernel.*", "*.workspace"
    
    def is_owned_by(self, owner_id: str) -> bool: ...
    
    def with_tag(self, tag: str) -> "PlatformIdentity": ...
    
    def without_tag(self, tag: str) -> "PlatformIdentity": ...
```

### 3.4 URI Scheme
```
platform://{namespace}/{type_path}/{object_id}?version={version}&owner={owner_id}

Example:
platform://ai-studio.workspace/platform.kernel.PlatformWorkspace/0196abc7-...
```

---

## 4. Metadata Model

### 4.1 Structure

```python
@dataclass
class MetadataEntry:
    key: str
    value: Any               # must be JSON-serializable
    value_type: str          # Python type name
    set_at: datetime
    set_by: str              # object_id of setter

class MetadataStore:
    def get(self, key: str) -> Any: ...
    def set(self, key: str, value: Any, set_by: str) -> None: ...
    def delete(self, key: str) -> None: ...
    def all(self) -> dict[str, MetadataEntry]: ...
    def history(self, key: str) -> list[MetadataEntry]: ...
        # full audit trail per key
```

### 4.2 Well-Known Metadata Keys

| Key | Type | Description |
|-----|------|-------------|
| `kernel.created_by` | str | object_id of creating service |
| `kernel.label` | str | display label override |
| `kernel.description` | str | free-text description |
| `kernel.tags` | list[str] | searchable tags |
| `kernel.priority` | int | scheduling priority hint |
| `kernel.ttl_seconds` | int | auto-dispose TTL |
| `kernel.owner_id` | str | owning object_id |
| `kernel.parent_id` | str | logical parent object_id |

---

## 5. Ownership Model

### 5.1 Ownership Graph
The kernel maintains a directed ownership tree. When an owner is disposed,
all owned objects are disposed in reverse dependency order.

```
PlatformWorkspace
  └── PlatformProject (owned by workspace)
        └── PlatformRepository (owned by project)
              └── PlatformSession (owned by repository)
                    └── PlatformExecution (owned by session)
```

### 5.2 Rules
1. Each object has at most one owner.
2. Root objects (no owner) are owned by the kernel itself.
3. Circular ownership is prohibited and detected at registration time.
4. Disposal propagates depth-first, children before parents.
5. A child may outlive its owner only if explicitly detached first.

### 5.3 Ownership API

```python
class OwnershipManager:
    def register(self, obj: PlatformObject, owner_id: str | None) -> None: ...
    def get_owner(self, object_id: str) -> str | None: ...
    def get_children(self, object_id: str) -> list[str]: ...
    def detach(self, object_id: str) -> None: ...
    def dispose_tree(self, root_id: str) -> None: ...
```

---

## 6. PlatformVersion — Value Object

```python
@dataclass(frozen=True)
class PlatformVersion:
    major: int
    minor: int
    patch: int
    pre_release: str = ""
    build_metadata: str = ""

    # Semver comparison
    def is_compatible_with(self, required: "PlatformVersion") -> bool: ...
    def is_breaking_change_from(self, older: "PlatformVersion") -> bool: ...

    @classmethod
    def parse(cls, s: str) -> "PlatformVersion": ...
    def __str__(self) -> str: ...
    def __lt__(self, other) -> bool: ...
    def __le__(self, other) -> bool: ...
```

Note: Delegates to `platform_sdk.compatibility.SemanticVersion` internally.

---

## 7. PlatformCapability — Value Object

```python
@dataclass(frozen=True)
class PlatformCapability:
    capability_id: str       # e.g. "kernel.workspace.read"
    name: str
    description: str
    scope: str               # "kernel" | "plugin" | "user" | "system"
    grantable: bool          # can a plugin grant this to sub-plugins?
    
    # Well-known capabilities defined in KernelCapabilities namespace
```

### 7.1 Capability Namespace

```
kernel.object.*          Object lifecycle operations
kernel.event.*           Event subscription and publishing
kernel.plugin.*          Plugin management
kernel.config.*          Configuration read/write
kernel.metrics.*         Metrics access
kernel.workspace.*       Workspace access
kernel.session.*         Session management
kernel.execution.*       Execution control
kernel.security.*        Security operations
kernel.resource.*        Resource allocation
```
