# ADR-0029 — Compute Engine Protocol: Pluggable Compute Orchestrators

**Status:** Proposed  
**Date:** 2026-05-17  
**Author:** Maintainer  
**Supersedes:** None  
**Governs:** ADR-0027 (ACP, becomes `PipelineEngine`), ADR-0028 (HAC, becomes `HACEngine`)  
**Parallel:** ADR-0022 (WorkerEngine — same pattern for LLM backends)  
**Depends on:** ADR-0013, ADR-0026

---

## Context

### The pattern already exists

ADR-0022 introduced the `WorkerEngine` protocol — a three-method interface
(`spawn`, `cancel`, `inject`) that lets the Bridge Adapter swap LLM backends
per persona without touching anything above or below:

```
Bridge Adapter
      │
      ▼
WorkerEngine (Protocol)
      ├── ClaudeCodeEngine
      ├── CodexCliEngine
      └── OpenCodeEngine
```

Adding a new LLM backend means implementing three methods. The rest of the
system — audit, hooks, TTS, routing, personas — never changes.

### ADR-0027 and ADR-0028 sit next to each other, not on top of each other

ADR-0027 (Adaptive Compute Pipelines / ACP) introduced linear pipelines with
Forge Gates where the LLM steers stage by stage.

ADR-0028 (Hierarchical Adaptive Compute / HAC) introduced sub-manager trees
with composite loss functions, Shapley attribution, and a Backprop Gate.

Both are structurally different orchestration strategies for the same underlying
primitive: a forge tool run N times with a strategy. Both deserve to live in
production simultaneously — simple sweeps use the flat engine, multi-stage workflows
use ACP, complex sub-problem decompositions use HAC.

Without a protocol layer, each ADR would add its own independent MCP tools,
its own job-store, its own tenant-config block, and its own audit emitter.
The result is three parallel implementations of the same infrastructure.

### Decision: apply the WorkerEngine pattern to Compute

Define a `ComputeEngine` protocol. `FlatEngine` (existing L25), `PipelineEngine`
(ADR-0027), and `HACEngine` (ADR-0028) implement it. A `ComputeEngineRegistry`
sits in the worker daemon and dispatches by engine-id. The MCP bridge exposes a
**unified surface** plus engine-specific extensions only where the semantics
genuinely differ (gate interactions).

---

## Decision

Introduce the **`ComputeEngine` protocol** as the pluggability layer for compute
orchestrators:

- Five protocol methods: `submit`, `status`, `result`, `gate_action`, `abort`.
- A `ComputeEngineRegistry` in the worker daemon maps engine-ids to instances.
- A unified MCP surface (`compute_submit`, `compute_status`, `compute_result`,
  `compute_gate`, `compute_abort`) dispatches by `job_id` prefix.
- Engine-specific gate vocabularies are typed via `GateAction` subtypes.
- Tenant config controls `engines_allowed` and `default_engine`.
- Custom engines (company-built) register via `registry.register(engine)` and
  become available immediately without daemon restart.
- ADR-0027 (`PipelineEngine`) and ADR-0028 (`HACEngine`) are conforming
  implementations; the existing flat compute runner becomes `FlatEngine`.
- All existing per-ADR tools (`compute_run / compute_pipeline_* / compute_hac_*`)
  remain as **thin shims** over the unified surface for backward compatibility.
  They are deprecated in favour of `compute_submit(engine=..., spec=...)`.

---

## Architecture

![Compute Engine Registry — unified MCP bridge, three engines, custom extension point](../assets/compute-engine-registry.svg)

---

## Design

### The `ComputeEngine` protocol

```python
# atelier_compute/engine_protocol.py

from typing import Protocol, runtime_checkable

@runtime_checkable
class ComputeEngine(Protocol):
    # Identity
    engine_id: str          # "flat" | "pipeline" | "hac" | custom
    display_name: str       # shown in compute_status, tenant config docs
    job_id_prefix: str      # "run_" | "pipeline_" | "hac_" | "my_"

    # Core lifecycle
    def submit(self, spec: "ComputeSpec") -> str:
        """Submit a job. Returns a job_id. Non-blocking."""
        ...

    def status(self, job_id: str) -> "ComputeStatus":
        """Poll current state. Engine-specific payload in ComputeStatus.detail."""
        ...

    def result(self, job_id: str, wait_s: int = 30) -> "ComputeResult":
        """Fetch terminal result. Blocks up to wait_s for completion."""
        ...

    def gate_action(self, job_id: str, action: "GateAction") -> None:
        """Apply a gate interaction. No-op for engines with no gates (FlatEngine)."""
        ...

    def abort(self, job_id: str) -> None:
        """Request graceful termination. Finishes current iteration batch."""
        ...
```

Five methods. No base class. Same structural trust as the `Strategy` protocol
(ADR-0027) and the `WorkerEngine` protocol (ADR-0022).

### Typed data contracts

```python
# atelier_compute/engine_types.py

@dataclass(frozen=True)
class ComputeSpec:
    engine:            str                    # engine_id to dispatch to
    tool_name:         str                    # forge tool, or None for multi-stage
    strategy:          str | None
    param_grid:        dict | None
    budget:            dict
    data_handle:       str | None
    sensitive_fields:  list[str]
    extra:             dict                   # engine-specific spec fields (stages, sub_managers, …)

@dataclass(frozen=True)
class ComputeStatus:
    job_id:     str
    engine_id:  str
    state:      str                           # "running" | "gate_open" | "terminal" | "failed"
    progress:   dict                          # engine-specific: iterations, round, stage, …
    detail:     dict                          # engine-specific payload (top_k, attributions, …)

@dataclass(frozen=True)
class ComputeResult:
    job_id:     str
    engine_id:  str
    state:      str                           # "converged" | "budget" | "aborted" | "failed"
    result:     dict                          # engine-specific: best_params, pipeline_tree, hac_tree
    audit_ref:  str                           # audit-chain event id for the terminal event

# GateAction is a tagged union — engines declare which subtypes they accept
@dataclass
class GateAction:
    action_type: str                          # "resume" | "abort" | engine-specific
    payload:     dict
```

Engines that don't support gates implement `gate_action` as a no-op (or raise
`EngineDoesNotSupportGates`). The MCP bridge checks `engine.supports_gates` before
advertising `compute_gate`.

### `ComputeEngineRegistry`

```python
# atelier_compute/engine_registry.py

class ComputeEngineRegistry:
    def __init__(self): self._engines: dict[str, ComputeEngine] = {}
    def register(self, engine: ComputeEngine) -> None: ...
    def get(self, engine_id: str) -> ComputeEngine: ...
    def get_by_job_id(self, job_id: str) -> ComputeEngine: ...   # prefix routing
    def engines_for_tenant(self, tenant_id: str) -> list[ComputeEngine]: ...
    def discover(self) -> list[str]: ...  # returns engine_ids of registered engines
```

**Prefix routing:** `job_id` encodes the engine. `pipeline_AbCd…` → `PipelineEngine`.
`hac_AbCd…` → `HACEngine`. `run_AbCd…` → `FlatEngine`. Custom engines declare their prefix;
the registry rejects duplicates at `register()` time.

**Hot-reload:** the registry calls `engines_for_tenant()` on every MCP `tools/list`
request (same 5 s TTL cache as the existing worker-socket discovery). Adding or removing
an engine from `engines_allowed` in `tenant.atelier.yaml` takes effect within 5 s.

### Unified MCP surface

Five tools replace the growing per-engine tool families:

```
compute_submit(engine, spec)   → job_id
compute_status(job_id)         → ComputeStatus
compute_result(job_id, wait_s) → ComputeResult
compute_gate(job_id, action)   → (engine dispatches GateAction)
compute_abort(job_id)          → void
```

`compute_gate` is only advertised when the current job_id maps to an engine with
`supports_gates = True`. This keeps the flat engine interface clean — LLMs working
with simple sweeps never see gate-related tools.

**Backward-compat shims** (deprecated, not removed):

| Old tool | Shim behaviour |
|---|---|
| `compute_run(tool_name, strategy, …)` | `compute_submit(engine="flat", spec={tool_name, strategy, …})` |
| `compute_pipeline(stages, …)` | `compute_submit(engine="pipeline", spec={stages, …})` |
| `compute_pipeline_resume(id)` | `compute_gate(id, GateAction("resume", {}))` |
| `compute_hac(sub_managers, …)` | `compute_submit(engine="hac", spec={sub_managers, …})` |
| `compute_hac_steer(id, action)` | `compute_gate(id, GateAction("backprop_steer", action))` |

Shims emit a `compute.deprecated_tool_used` INFO event in the audit chain so operators
can track migration progress.

### The three conforming engines

| Engine | engine_id | job_id prefix | supports_gates | Implements |
|---|---|---|---|---|
| `FlatEngine` | `"flat"` | `"run_"` | No | ADR-0013, ADR-0026 |
| `PipelineEngine` | `"pipeline"` | `"pipeline_"` | Yes | ADR-0027 |
| `HACEngine` | `"hac"` | `"hac_"` | Yes | ADR-0028 |

**`FlatEngine`** wraps the existing `ParallelDriver` and `RunStore` from ADR-0026 verbatim.
No changes to the inner loop, strategy protocol, or audit events. The only addition is
implementing the five protocol methods as thin wrappers.

**`PipelineEngine`** implements the pipeline coordinator from ADR-0027. `gate_action` accepts
`GateAction` subtypes: `ForgeAction`, `AddStageAction`, `ResumeAction`, `AbortAction`.

**`HACEngine`** implements the HAC coordinator from ADR-0028. `gate_action` accepts:
`ForgeAction`, `AddStageToManagerAction`, `ReallocateBudgetAction`, `ReRunManagerAction`,
`ResumeAction`, `AbortAction`.

### Tenant configuration

```yaml
spec:
  compute:
    enabled: true
    engines_allowed:    [flat, pipeline, hac]   # which engines this tenant may use
    default_engine:     flat                     # used when compute_submit omits engine
    flat:     { max_parallel_iterations: 4, … } # engine-specific config blocks
    pipeline: { max_pipeline_stages: 20, … }    # (from ADR-0027)
    hac:      { max_nesting_depth: 3, … }        # (from ADR-0028)
```

The per-engine config blocks are **unchanged** from ADR-0027 and ADR-0028.
The new fields are `engines_allowed` and `default_engine`. `extra="forbid"` on the
Pydantic model for the root `compute` block; each engine's sub-block retains its own
strict schema.

### Adding a custom engine

A company implementing a custom optimisation orchestrator (e.g. a population-based
evolutionary algorithm, or a domain-specific meta-learner):

```python
# mycompany/my_engine.py
from atelier_compute.engine_protocol import ComputeEngine, ComputeSpec, ComputeStatus, ComputeResult, GateAction

class EvolutionaryEngine:
    engine_id      = "evolutionary"
    display_name   = "Evolutionary Strategy (MyCompany)"
    job_id_prefix  = "evo_"
    supports_gates = False          # or True if the engine has checkpoints

    def submit(self, spec: ComputeSpec) -> str: ...
    def status(self, job_id: str) -> ComputeStatus: ...
    def result(self, job_id: str, wait_s: int = 30) -> ComputeResult: ...
    def gate_action(self, job_id: str, action: GateAction) -> None: pass
    def abort(self, job_id: str) -> None: ...

# In worker startup:
registry.register(EvolutionaryEngine())
```

Then in `tenant.atelier.yaml`:

```yaml
spec:
  compute:
    engines_allowed: [flat, pipeline, evolutionary]
    evolutionary: { population_size: 50, generations: 100 }
```

The `evolutionary` engine key is passed to `EvolutionaryEngine` as `spec.extra` —
no changes to the registry or the MCP bridge are needed.

### Audit chain — 2 new event types (unified layer)

The existing per-engine audit events (from ADR-0027 and ADR-0028) are unchanged.
Two new events at the registry level:

| Event | Severity | Fields |
|---|---|---|
| `compute.engine_registered` | INFO | `engine_id`, `display_name`, `supports_gates`, `tenant_id` |
| `compute.deprecated_tool_used` | INFO | `old_tool_name`, `redirected_to`, `job_id` |

`compute.engine_registered` fires once per engine per worker boot (and on hot-reload
of `engines_allowed`). `compute.deprecated_tool_used` fires per shim call.

### Path-gate

No new protected globs — `**/compute/**` (ADR-0026), `**/pipelines/**` (ADR-0027),
and `**/hac/**` (ADR-0028) already cover the artifact trees for all three engines.
Custom engines whose artifact trees fall outside these globs must declare a new protected
path in `path_gate.py` and add corresponding test cases.

---

## How ADR-0027 and ADR-0028 change

**ADR-0027** status: **Proposed → Implements ADR-0029**.
The `compute_pipeline_*` tools become backward-compat shims. The pipeline coordinator
is refactored to implement `ComputeEngine` (`PipelineEngine`). No semantic changes.

**ADR-0028** status: **Proposed → Implements ADR-0029**.
The `compute_hac_*` tools become backward-compat shims. The HAC coordinator implements
`ComputeEngine` (`HACEngine`). No semantic changes.

**ADR-0013 / ADR-0026**: the existing `compute_run / compute_status / compute_result /
compute_abort` tools become shims for `FlatEngine`. The `ParallelDriver`, `RunStore`,
`Strategy` protocol, and all audit events are unchanged.

---

## Implementation plan

| Phase | Scope | Deliverable |
|---|---|---|
| 0 | Protocol | `engine_protocol.py` — `ComputeEngine`, `ComputeSpec`, `ComputeStatus`, `ComputeResult`, `GateAction` |
| 1 | Registry | `engine_registry.py` — register, get, prefix routing, tenant allowlist, TTL cache |
| 2 | FlatEngine | Wrap existing `ParallelDriver` / `RunStore` in the protocol |
| 3 | MCP surface | Unified `compute_submit / status / result / gate / abort` in `mcp_bridge.py` |
| 4 | Shims | Old per-engine tools → shims with `compute.deprecated_tool_used` audit event |
| 5 | PipelineEngine | ACP coordinator (ADR-0027) adapted to protocol |
| 6 | HACEngine | HAC coordinator (ADR-0028) adapted to protocol |
| 7 | Tenant config | `engines_allowed`, `default_engine` in `ComputeConfig` |
| 8 | Custom engine docs | Example `EvolutionaryEngine` + walkthrough in `docs/compute.md` |
| 9 | E2E tests | Protocol conformance suite: every engine must pass the same 30-case harness |
| 10 | Deprecation | Announce shim removal in next minor; add `--strict` flag that fails on shim use |

---

## Why this is the right structural decision

### Mirror of WorkerEngine (ADR-0022)

```
LLM backends:                    Compute orchestrators:
WorkerEngine (Protocol)          ComputeEngine (Protocol)
  ClaudeCodeEngine                 FlatEngine
  CodexCliEngine                   PipelineEngine
  OpenCodeEngine                   HACEngine
  <custom>                         <custom>
```

The pattern is proven. It keeps the Bridge Adapter (for LLM) and the MCP Bridge
(for Compute) stable while the implementations evolve independently. Adding a fourth
engine touches exactly one file: the new engine class.

### The protocol enforces the right boundary

Every engine's job is to: accept a spec, run compute, expose progress, handle gate
interactions, abort gracefully. That's the contract. What happens inside — flat loops,
linear pipelines, or sub-manager trees with attribution analysis — is the engine's
business. The MCP bridge does not care.

### Composability: engines can call engines

A `HACEngine` sub-manager is itself a `PipelineEngine` or `FlatEngine` job submitted
through the same registry. No special case needed — the registry handles nested
`pipeline_*` job_ids issued by the HAC coordinator the same way it handles top-level ones.

---

## Consequences

### Positive
- Three compute paradigms (flat, pipeline, hac) coexist in one worker with one MCP surface.
- Adding a company-specific compute orchestrator is a single-file change.
- Protocol conformance tests (`engine_conformance_suite.py`) verify every engine
  satisfies the contract — a new engine must pass 30 shared cases before it can be
  listed in `engines_allowed`.
- The LLM always uses the same five MCP tools regardless of which engine is underneath.
  It reads `ComputeStatus.detail` to understand engine-specific state.

### Costs
- **Migration effort.** The existing flat compute tools (`compute_run / *`) must be
  wrapped in `FlatEngine` without changing their behaviour. Careful integration testing
  required to confirm zero semantic drift.
- **Shim overhead.** Per-call shim dispatch adds one function call and one audit event.
  Negligible at compute-job scale (submitting a job takes microseconds; the job runs for
  seconds to hours).
- **Protocol friction for simple engines.** A company wanting a trivial custom strategy
  (extend `Strategy` from ADR-0026) does not need the full `ComputeEngine` protocol —
  they just register a new strategy with the flat engine. The protocol is the right level
  for orchestration-level customisation, not iteration-level customisation.

---

## References

- ADR-0022 — WorkerEngine protocol (structural template)
- ADR-0013 — Compute Worker; `FlatEngine` wraps its `ParallelDriver`
- ADR-0026 — Compute Fabric; strategy protocol; parallel driver
- ADR-0027 — ACP; `PipelineEngine` implementation
- ADR-0028 — HAC; `HACEngine` implementation
- `atelier_compute/engine_protocol.py` — Protocol definition (to be created, Phase 0)
- `atelier_compute/engine_registry.py` — Registry (to be created, Phase 1)
- `operator/bridges/shared/engine_registry.py` — WorkerEngine registry (structural reference)
