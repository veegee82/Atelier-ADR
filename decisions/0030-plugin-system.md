# ADR-0030 — AtelierOS Plugin System: Layer-Scoped Extension Points

**Status:** Proposed  
**Date:** 2026-05-17  
**Author:** Maintainer  
**Supersedes:** None  
**Governs:** Layer-specific extension points across L22, L23, L24, L25, Bridge, L16  
**Parallel:** ADR-0022 (WorkerEngine), ADR-0029 (ComputeEngine)  
**Depends on:** ADR-0007 (multi-tenant), ADR-0013, ADR-0022, ADR-0029

---

## Context

### The pattern already exists — four times over

AtelierOS already has four independent pluggable protocols:

```
WorkerEngine    (ADR-0022) — LLM backend selection per persona
ComputeEngine   (ADR-0029) — compute orchestrator per tenant
STTProvider     (L23)      — speech-to-text chain
BridgeChannel   (Bridge)   — communication channel registration
```

Each protocol works well within its own layer. The problem is what happens
**between** layers: discovery, lifecycle, configuration injection, and health
checking are implemented independently in every case.

### The copy-paste problem

Adding a new extension today requires:

1. Writing the layer-specific protocol (already done per ADR)
2. Inventing a registration API for that layer's registry
3. Wiring tenant config by hand (different YAML structure per layer)
4. Adding health-check logic by hand (or skipping it entirely)
5. Re-implementing the `importlib.metadata` entry-point loader (or skipping it)

Steps 2–5 are identical boilerplate regardless of which layer is being extended.
Enterprise operators who want to distribute custom engines as Python packages have
no standard contract — they reverse-engineer the internal registration API.

### No tenant-level enable/disable

There is no standard mechanism for a tenant to say "install plugin X, disable
plugin Y". Each layer has its own config shape (`engines_allowed`, `stt_chain`,
etc.). Operators cannot write tooling that operates uniformly across plugin types.

### Decision: a single lifecycle contract over all extension points

Define one `AtelierPlugin` protocol that every extension implements **in addition
to** its layer-specific protocol. Add a `PluginRegistry` that handles discovery,
lifecycle, and health aggregation uniformly. The layer-specific registries
(`ComputeEngineRegistry`, `engine_factory`, etc.) remain the authoritative routing
tables — the plugin system sits one level above them as the **installation layer**.

---

## Decision

Introduce the **`AtelierPlugin` protocol** and a `PluginRegistry` as the unified
installation and lifecycle layer:

- Three lifecycle methods: `on_load`, `on_unload`, `health_check`.
- A `PluginContext` dataclass injected at load time: `plugin_id`, `tenant_id`,
  `atelier_home`, `config`, `audit_emit`, plus optional handles to layer registries.
- The plugin's `on_load()` self-registers with the appropriate layer registry using
  the handles in `PluginContext`. No out-of-band wiring required.
- Discovery via `importlib.metadata` entry points group `"atelier.plugins"`, or via
  explicit class paths in the tenant's `spec.plugins.installed` list.
- Six typed extension points (table below); each maps plugin_type to a layer
  protocol and a target registry.
- `PluginRegistry` raises on duplicate `plugin_id` at register time.
- Tenant opt-in is explicit — no auto-load without `spec.plugins.installed` entry
  or `auto_discover_entry_points: true`.

---

## Architecture

```
Tenant config
  spec.plugins.installed: [my-sa-engine, my-stt, ...]
         │
         ▼
  loader.discover_and_load(tenant_config, atelier_home=...)
         │
         │  importlib / class_path
         ▼
  PluginRegistry.register(plugin, ctx)
         │
         │  plugin.on_load(ctx)
         │
         ├─── ctx.compute_registry.register(self)   → ComputeEngineRegistry
         ├─── ctx.engine_factory.register(self)     → adapter engine_factory
         ├─── ctx.channel_registry.register(self)   → bridge channel_registry
         └─── ... (layer-specific self-registration)
```

---

## Design

### The `AtelierPlugin` protocol

```python
# atelier_plugins/protocol.py

from typing import Protocol, runtime_checkable

@runtime_checkable
class AtelierPlugin(Protocol):
    plugin_id:    str   # globally unique, matches tenant config key
    plugin_type:  str   # one of KNOWN_PLUGIN_TYPES
    version:      str   # semver string
    display_name: str   # shown in health dashboard

    def on_load(self, ctx: PluginContext) -> None:
        """Called once after discovery. Self-register with layer registry here."""
        ...

    def on_unload(self) -> None:
        """Called on graceful shutdown or tenant hot-reload."""
        ...

    def health_check(self) -> HealthStatus:
        """Called periodically; must not block for more than 2 s."""
        ...
```

Three methods. No base class. Structurally compatible with `WorkerEngine` (ADR-0022)
and `ComputeEngine` (ADR-0029).

### `PluginContext`

```python
@dataclass
class PluginContext:
    plugin_id:        str
    tenant_id:        str
    atelier_home:     Path
    config:           dict          # spec.plugins.installed[*].config block
    audit_emit:       Callable[[str, dict], None]
    compute_registry: Any | None = None   # atelier_compute.ComputeEngineRegistry
    engine_factory:   Any | None = None   # adapter engine_factory
    channel_registry: Any | None = None   # bridge channel_registry
    extra:            dict = field(default_factory=dict)
```

The context carries only what the plugin needs. Raw API keys are **never** passed
here — the plugin reads them from the vault via the standard secret injection
mechanism (ADR Layer 16 v3). `audit_emit` is the only observability hook.

### Extension points

| plugin_type | Layer | Layer-specific protocol | Registers with |
|---|---|---|---|
| `compute_engine` | L25 | `ComputeEngine` (ADR-0029) | `atelier_compute.engine_registry` |
| `worker_engine` | L22 | `WorkerEngine` (ADR-0022) | adapter `engine_factory` |
| `bridge_channel` | Bridge | `BridgeChannel` | `bridge.channel_registry` |
| `stt_provider` | L23 | `STTProvider` | `atelier_stt.provider_registry` |
| `data_connector` | L24 | `DataConnector` | `atelier_data.connector_registry` |
| `audit_backend` | L16 | `AuditBackend` | `audit.backend_registry` |

A plugin that implements `plugin_type = "compute_engine"` must implement **both**
`AtelierPlugin` (lifecycle) and `ComputeEngine` (capability). `on_load()` calls
`ctx.compute_registry.register(self)`. `on_unload()` calls
`ctx.compute_registry.unregister(self.engine_id)`.

### `PluginRegistry`

```python
# atelier_plugins/registry.py

class PluginRegistry:
    def register(self, plugin: AtelierPlugin, ctx: PluginContext) -> None: ...
    def unregister(self, plugin_id: str) -> None: ...
    def get(self, plugin_id: str) -> AtelierPlugin: ...
    def health_check_all(self) -> dict[str, HealthStatus]: ...
    def plugins_by_type(self, plugin_type: str) -> list[AtelierPlugin]: ...
    def discover(self) -> list[str]: ...
```

Thread-safe (`threading.Lock`). `register()` calls `plugin.on_load(ctx)` and stores
the plugin. Raises `PluginAlreadyRegistered` on duplicate `plugin_id`.
`health_check_all()` catches per-plugin exceptions and returns
`HealthStatus(ok=False, message=str(exc))` — one broken plugin never blocks the
rest.

### Discovery and loading

```python
# atelier_plugins/loader.py

def load_from_entry_points(group="atelier.plugins") -> list[type]: ...
def load_from_class_path(class_path: str) -> type: ...
def load_from_manifest(manifest_path: Path) -> dict: ...
def discover_and_load(tenant_config: dict, *, atelier_home: Path) -> list[AtelierPlugin]: ...
```

`discover_and_load` reads `tenant_config["spec"]["plugins"]["installed"]`, loads
each class via `load_from_class_path`, instantiates (no-arg constructor), and
returns a list of plugin instances ready for `PluginRegistry.register()`.

Entry-point auto-discovery is **opt-in** via
`spec.plugins.auto_discover_entry_points: true` — off by default to keep tenants
isolated.

### Tenant configuration

```yaml
spec:
  plugins:
    installed:
      - id: my-sa-engine
        config:
          temperature: 1.0
          cooling_rate: 0.95
      - id: my-stt-provider
        config:
          model: whisper-large-v3
    auto_discover_entry_points: false   # default; set true for dev/trusted envs
```

### Entry-point declaration (plugin's `pyproject.toml`)

```toml
[project.entry-points."atelier.plugins"]
my-sa-engine = "my_sa_engine:SimulatedAnnealingPlugin"
```

The entry-point name becomes the default `plugin_id` when `id` is omitted from the
tenant config.

### `HealthStatus`

```python
@dataclass
class HealthStatus:
    ok:      bool
    message: str = ""
    details: dict = field(default_factory=dict)
```

Returned by `plugin.health_check()` and aggregated by `PluginRegistry.health_check_all()`.
The operator dashboard (ADR-0017) polls `health_check_all()` via the REST API.

### Audit events

Two new events at the plugin-system level:

| Event | Severity | Fields |
|---|---|---|
| `plugin.loaded` | INFO | `plugin_id`, `plugin_type`, `version`, `tenant_id` |
| `plugin.unloaded` | INFO | `plugin_id`, `tenant_id` |
| `plugin.health_check_failed` | WARNING | `plugin_id`, `error_type` (no message — no PII) |

All three emitted via the injected `audit_emit` callable — the plugin system never
writes to the audit chain directly.

### Path-gate

Plugins run inside the existing path-gate perimeter (Layer 10). No new protected
globs are required. Plugin code that writes to `atelier_home/**` uses the same
rules as any other adapter code — forge/skill-forge/audit paths are blocked.

---

## How existing extension points change

**ComputeEngine (ADR-0029):** unchanged internally. `ContribEngine` or any custom
engine that also implements `AtelierPlugin` gets lifecycle and discovery for free.
Single-file migration: add three methods (`on_load`, `on_unload`, `health_check`)
and an entry-point declaration.

**WorkerEngine (ADR-0022):** same pattern. Existing `ClaudeCodeEngine`,
`CodexCliEngine`, `OpenCodeEngine` are not migrated (they are built-in, not
distributed as packages). New third-party engines use `AtelierPlugin`.

**STTProvider (L23):** no changes to the provider chain itself. A custom provider
distributed as a package implements `AtelierPlugin` + `STTProvider`; `on_load`
registers with `atelier_stt.provider_registry`.

---

## Implementation plan

| Phase | Scope | Deliverable |
|---|---|---|
| 0 | Protocol | `protocol.py` — `AtelierPlugin`, `PluginContext`, `HealthStatus`, exceptions |
| 1 | Registry | `registry.py` — thread-safe register/unregister/get/health/discover |
| 2 | Loader | `loader.py` — entry_points, class_path, manifest, discover_and_load |
| 3 | Package | `__init__.py` — public surface; `pyproject.toml`; `atelier-plugins` plugin package |
| 4 | Templates | `templates/compute_engine_plugin.py`, `worker_engine_plugin.py`, `bridge_channel_plugin.py` |
| 5 | Tests | `tests/test_plugin_system.py` — protocol conformance, registry, loader |
| 6 | Docs | Update `docs/compute.md`; add `docs/plugins.md` quickstart |
| 7 | Integration | Wire `PluginRegistry` into worker boot; health endpoint in admin REST API |

---

## Must NOT

- Don't call any plugin method before `on_load()` completes.
- Don't allow `plugin_id` collisions — registry raises `PluginAlreadyRegistered` on duplicate.
- Don't let plugins `import anthropic` — CI AST lint enforces (same rule as `ldd.py`, `dialectic.py`).
- Don't pass raw API keys in `PluginContext` — `audit_emit` and `atelier_home` only.
- Don't auto-load plugins without explicit tenant opt-in in `spec.plugins.installed`
  or `auto_discover_entry_points: true`.
- Don't run `on_load()` on the main thread if the plugin declares `async_load: true`
  (Phase 7 async extension point — reserved).
- Don't store plugin instances outside `PluginRegistry` — always use `get(plugin_id)`.
- Don't emit `plugin.health_check_failed` with the exception message — `error_type`
  (class name) only, no PII.

---

## Why this is the right structural decision

### One pattern, six extension points

The WorkerEngine and ComputeEngine protocols proved that a minimal Protocol
(`submit/status/result` or `spawn/cancel`) is enough to make an extension point
pluggable. `AtelierPlugin` adds exactly the missing cross-cutting concerns —
lifecycle (`on_load`/`on_unload`) and observability (`health_check`) — without
touching the layer-specific protocols at all.

### Enterprise distribution path

A company can now ship a custom compute orchestrator or STT provider as a standard
Python package on PyPI. The only contract is: implement `AtelierPlugin` + the
relevant layer protocol, declare the entry point in `pyproject.toml`, and add the
package to the tenant config. No AtelierOS source changes required.

### Tenant isolation preserved

Auto-discovery is opt-in. Each tenant's `spec.plugins.installed` is the explicit
allowlist. The `PluginContext` carries the tenant's `atelier_home` — plugins cannot
cross tenant boundaries.

---

## Consequences

### Positive

- New extensions require zero boilerplate beyond implementing the two protocols.
- Operator tooling can query `health_check_all()` uniformly across all plugin types.
- The six typed extension points give contributors a clear map of where to plug in.
- Third-party package distribution is supported from day one.

### Costs

- **Two-protocol requirement.** Plugins must implement both `AtelierPlugin` and the
  layer protocol. For simple extensions this is six methods total — manageable, but
  more than a single-protocol design.
- **No base class.** The Protocol approach avoids tight coupling but gives
  contributors no default implementation to inherit. The templates in
  `core/plugins/templates/` compensate.
- **`on_load` coupling.** If a layer registry is not available in `PluginContext`
  (e.g. the compute registry is absent in a bridge-only deployment), the plugin must
  handle the `None` case gracefully. Templates demonstrate the pattern.

---

## References

- ADR-0007 — Multi-tenant axis; `PluginContext.tenant_id` / `atelier_home` follow its resolver
- ADR-0013 — Compute Worker; `FlatEngine` is the reference compute extension
- ADR-0022 — WorkerEngine protocol (structural template for lifecycle pattern)
- ADR-0029 — ComputeEngine protocol (structural template for capability pattern)
- `core/plugins/atelier_plugins/protocol.py` — Protocol definition (Phase 0)
- `core/plugins/atelier_plugins/registry.py` — Registry (Phase 1)
- `core/plugins/atelier_plugins/loader.py` — Loader (Phase 2)
- `core/plugins/templates/compute_engine_plugin.py` — Reference template
