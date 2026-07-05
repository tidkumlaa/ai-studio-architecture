# Platform Kernel — Service Infrastructure
# Phase 2.0D.2.6A

---

## 1. Overview

The kernel's service infrastructure provides the wiring that connects
components without creating hard dependencies. It comprises:

1. **Dependency Injection** — constructor-based, resolved at boot
2. **PlatformRegistry** — runtime catalog of all active objects
3. **PlatformPlugin / PluginManager** — dynamic extension loading
4. **PlatformModule** — deployment unit grouping plugins
5. **PlatformServiceLocator** — governed service lookup facade
6. **PlatformManifest** — declarative plugin descriptor
7. **PlatformConfiguration** — hierarchical settings
8. **PlatformCapability** — access token for service use

Full module-level specs in `03-MODULE-CATALOG.md §§ C, D`.

---

## 2. Dependency Injection Model

### 2.1 Principle

All kernel services are injected via constructor parameters. No service
creates its own dependencies. No global singletons.

```python
# Correct — injected
class MyComponent(PlatformComponent):
    def __init__(
        self,
        event_bus: PlatformEventBus,
        config: PlatformConfiguration,
        registry: PlatformRegistry,
        security: PlatformSecurity,
    ) -> None: ...

# Forbidden — self-locating
class BadComponent(PlatformComponent):
    def __init__(self) -> None:
        self._bus = get_global_event_bus()   # NEVER do this
```

### 2.2 Kernel Service Graph (Boot Order)

```
[1] PlatformAudit             (no deps)
[2] PlatformMetrics           (no deps)
[3] PlatformLogger            (metrics)
[4] PlatformConfiguration     (no deps)
[5] PlatformEventBus          (audit, metrics)
[6] PlatformRegistry          (event_bus, audit)
[7] PlatformSecurity          (registry, audit, config)
[8] PlatformHealth            (registry, metrics, event_bus)
[9] PlatformScheduler         (event_bus, metrics, config)
[10] PlatformServiceLocator   (registry, security)
[11] PluginManager            (registry, event_bus, security, config, manifest)
[12] LifecycleOrchestrator    (registry, event_bus)
```

### 2.3 Kernel Container

The KernelContainer assembles and holds all kernel services.
It is the only place where kernel services are constructed.

```python
class KernelContainer:
    def __init__(self, settings: KernelSettings) -> None: ...
    
    def boot(self) -> None: ...
        # Constructs all services in dependency order
        # Starts all lifecycle-managed services
    
    def shutdown(self, timeout_ms: int = 30_000) -> None: ...
    
    # Accessors (injected into components that need them)
    @property
    def event_bus(self) -> PlatformEventBus: ...
    @property
    def registry(self) -> PlatformRegistry: ...
    @property
    def security(self) -> PlatformSecurity: ...
    @property
    def scheduler(self) -> PlatformScheduler: ...
    @property
    def config(self) -> PlatformConfiguration: ...
    @property
    def health(self) -> PlatformHealth: ...
    @property
    def audit(self) -> PlatformAudit: ...
    @property
    def metrics(self) -> PlatformMetrics: ...
    @property
    def logger_factory(self) -> LoggerFactory: ...
    @property
    def plugin_manager(self) -> PluginManager: ...
    @property
    def service_locator(self) -> PlatformServiceLocator: ...
```

---

## 3. Plugin Discovery and Loading

### 3.1 Discovery

```
Plugin search path (in order):
1. Built-in kernel plugins (bundled with kernel package)
2. Platform extension directory (configured path)
3. Runtime-registered manifests (loaded by calling code at runtime)
```

```python
class PluginManager:
    def discover(self, search_path: str) -> list[PlatformManifest]: ...
        # Scans path for manifest.yaml files; validates each
        # Returns valid manifests; logs and skips invalid ones
    
    def load_from_manifest(self, manifest: PlatformManifest) -> PlatformPlugin: ...
    def load_all(self, manifests: list[PlatformManifest]) -> list[PlatformPlugin]: ...
        # Loads in dependency order using topological sort on manifest dependencies
```

### 3.2 Loading Sequence

```
1. Validate manifest (signature, schema, required fields)
2. Verify required capabilities are available in PlatformSecurity
3. Resolve and verify dependencies (all required plugins already loaded)
4. Create PluginSandbox with declared capability set
5. Import entry_module (Python import inside sandbox context)
6. Instantiate entry_class with injected KernelContainer services
7. Call plugin.initialize()
8. Register plugin in PlatformRegistry
9. Emit kernel.plugin.loaded
```

### 3.3 Dependency Resolution

Plugins declare dependencies in their manifest:
```yaml
dependencies:
  - name: "ai-studio.platform.core"
    version: ">=1.0.0,<2.0.0"
    optional: false
```

PluginManager uses topological sort to load dependencies before dependents.
Circular dependencies are rejected at load time.

### 3.4 Hot Reload

```python
class PluginManager:
    def reload(self, plugin_id: str) -> None: ...
```

Hot reload sequence:
```
1. Plugin transitions to STOPPING
2. All in-flight calls to the plugin complete or time out (10s)
3. Plugin unloaded (entry_class disposed, module removed from sys.modules)
4. Fresh copy loaded using steps in §3.2
5. Plugin re-registered in PlatformRegistry
6. Subscribers to kernel.plugin.reloading notified
```

Limitations:
- Hot reload is not guaranteed for plugins that store mutable state
  in module-level globals (which violates P3 — Constructor Injection)
- State transfer across reload is the plugin's responsibility

---

## 4. Service Registration Contract

Any object registered in PlatformRegistry must:

1. Be in RUNNING or INITIALIZED lifecycle state at registration time
2. Have a valid `object_id` (UUID v7)
3. Have a valid `object_type` (fully-qualified Python class name)
4. Declare any capabilities it provides (via manifest or registration metadata)

When an object is disposed, it is automatically unregistered.
The registry uses a weak-reference strategy for objects with TTLs.

---

## 5. PlatformServiceLocator Usage Rules

PlatformServiceLocator is a governed facade. It is NOT a general-purpose
IoC container. Usage is restricted:

| Allowed | Disallowed |
|---------|------------|
| Optional service lookup (service may not be present) | Replacing constructor injection for required deps |
| Context-dependent service selection (e.g. which provider?) | Resolving services in hot paths (< 0.05 ms budget) |
| Plugin code resolving exported kernel services | Resolving services the plugin's manifest did not declare |

```python
# Allowed use case: optional service
class MyPlugin:
    def __init__(self, locator: PlatformServiceLocator) -> None:
        self._tracer = locator.try_resolve(TracingService)  # may be None
```

---

## 6. Module Activation Model

Modules group related plugins and have a shared activation lifecycle.

```
Module.activate():
    1. Verify all required capabilities are available
    2. Load all plugins in the module (in dependency order)
    3. Module transitions to RUNNING
    4. Emit kernel.module.activated

Module.deactivate():
    1. Unload all plugins in reverse dependency order
    2. Module transitions to STOPPED
    3. Emit kernel.module.deactivated
```

Modules are the unit used by the Desktop App when enabling/disabling
feature sets. A module activation failure does not crash the kernel —
the module enters FAILED state and the kernel continues.

---

## 7. Configuration Integration

### 7.1 Plugin Configuration Access

Plugins receive their configuration via the KernelContainer injected at
construction. They declare a `config_schema` in their manifest and receive
a PlatformSettings snapshot at initialization:

```python
class MyPlugin(PlatformPlugin):
    def __init__(
        self,
        config: PlatformConfiguration,
        ...
    ) -> None:
        self._settings = (
            SettingsBuilder()
            .from_config(config, namespace=f"plugins.{self.plugin_id}")
            .with_defaults({"timeout_ms": 5000})
            .validated_by(self.manifest.config_schema)
            .build()
        )
```

### 7.2 Configuration Change Notification

Plugins subscribe to config changes for their namespace:

```python
self._config_handle = config.on_change(
    f"plugins.{self.plugin_id}.*",
    self._on_config_changed,
)
```

### 7.3 Secrets Access

Plugins declare secret names in their manifest. Secrets are resolved through
PlatformSecurity, not PlatformConfiguration:

```python
api_key = security.get_secret(
    self._settings.require("api_key_secret"),
    accessor_id=self.object_id,
)
```
