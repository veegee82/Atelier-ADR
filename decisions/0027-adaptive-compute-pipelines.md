# ADR-0027 — Adaptive Compute Pipelines (ACP)

**Status:** Proposed  
**Date:** 2026-05-17  
**Author:** Maintainer  
**Supersedes:** None  
**Implements:** ADR-0029 (Compute Engine Protocol) as `PipelineEngine` (engine_id = "pipeline")  
**Depends on:** ADR-0013 (Compute Worker), ADR-0026 (Compute Fabric), ADR-0006 (AWP DAG Walker), ADR-0001 (AWP as Orchestration Layer)

---

## Context

### The gap in ADR-0026

ADR-0026 (Compute Fabric, L25) solves the iteration-burns-LLM-tokens problem for **single-objective sweeps**:
one forge tool, N parameter combinations, zero LLM tokens during execution. Three MCP calls total
(`compute_run` → `compute_status` → `compute_result`).

What it cannot do: **pipelines** where the optimal structure isn't known at submission time.

A real optimisation workflow looks like this:

```
"Run feature engineering, see what the data actually looks like,
 then decide which model family to try,
 then tune that specific model based on what the feature space revealed,
 then — if the model shows signs of overfitting — add a regularisation pass
 that might not have been needed at all."
```

The plan only exists because earlier steps revealed it. No human expert writes a 4-stage sweep
before running step 1. They look at what they get, then decide what comes next.

The current answer is manual chaining: the LLM drives `compute_result → data_register → compute_run`
between stages. This works but has two failure modes:

1. **Token burn between stages.** The LLM pays round-trip cost to inspect results, decide on the
   next stage, and submit it. For a 10-stage pipeline this is 10 × 3 = 30 LLM calls, all pulling
   the full context.
2. **No tool creation.** The next stage's tool must already exist. If the data reveals a pattern
   that needs a bespoke tool — a clustering variant, a custom distance metric, a dataset-specific
   normaliser — the LLM cannot create it inside the compute loop. It must break out, forge the
   tool, then re-submit.

### The AWP Delegation Loop insight

ADR-0001 introduced the AWP Delegation Loop as a key orchestration primitive. From
`core/workflows/atelier_workflows/examples/news_sentiment.awp.yaml`:

```yaml
- id: sentiment_analysis
  type: delegation_loop
  config:
    manager: sentiment_manager
    budget:
      max_loops: 3
      max_total_workers: 5
    termination:
      enabled: true
      window: 2
      min_confidence_delta: 0.05
```

The manager LLM does not plan all workers upfront. It sees the current state, decides what worker
to spawn next, checks the termination criterion, and either spawns or stops. The loop is adaptive.

ADR-0005 rolled back AWP as runtime ("AWP is consumed strictly as protocol / declarative standard,
never as runtime"). But the **Delegation Loop pattern** survives as an architectural principle.

ACP applies that same principle to the Compute Layer: the LLM is the Delegation Loop manager,
each compute stage is a worker, and a **Forge Gate** between stages gives the manager the ability
to mint entirely new tools before deciding what to run next.

---

## Decision

Introduce **Adaptive Compute Pipelines (ACP)** as an extension to the L25 Compute Worker:

- A pipeline is a **sequence of compute stages**, each running a forge tool N times.
- Between stages, the worker pauses at a **Forge Gate** and the LLM gets a structured steering turn.
- At each gate the LLM can: inspect results, **forge a new tool** (or not), add/replace/skip the
  next stage, and resume or stop.
- The pipeline manifest is **mutable at runtime** — stages that do not exist at submission time
  can be added by the LLM mid-run, including stages backed by tools that were forged at that gate.
- A **5D budget** (borrowed from AWP) caps stages, tool creations, total iterations, wall clock,
  and gate timeout.
- All existing Compute Worker invariants hold: zero Anthropic SDK imports, zero LLM tokens during
  execution, metadata-only audit chain, path-gate protection.

---

## Concept diagram

![ACP — LLM as Conductor, adaptive stages, forge gates between each stage](../assets/acp-concept.svg)

---

## Design

### Mental model

```
compute_pipeline(initial_stages, budget, steering_gate=true)
       │
       └─ Stage 0 runs (Worker, N iterations, zero LLM tokens)
                │
                └─ FORGE GATE 1 ──────────────────────────────────────────
                    LLM calls: pipeline_status()         ← inspect
                    LLM calls: forge_tool("kmeans_v2")   ← create (optional)
                    LLM calls: pipeline_add_stage(...)   ← plan
                    LLM calls: pipeline_resume()         ← go
                │
                └─ Stage 1 runs (newly forged tool, N iterations, zero tokens)
                        │
                        └─ FORGE GATE 2 ───────────────────────────────────
                            LLM refines param space based on cluster shapes
                            LLM adds Stage 2 using pre-existing train_model
                        │
                        └─ Stage 2 runs (Bayesian, converges)
                                │
                                └─ GATE 3: auto-stop (loss Δ < threshold)
                                        │
                                        └─ pipeline_result() → full tree
```

### New MCP tools (6)

```
compute_pipeline           submit pipeline, initial_stages optional, returns pipeline_id
compute_pipeline_status    current stage + progress + Top-K per completed stage + artifacts
compute_pipeline_steer     override next stage params before resume (no forge required)
compute_pipeline_add_stage add a new stage at the tail or after a named stage
compute_pipeline_resume    release the gate, run next stage
compute_pipeline_abort     graceful shutdown (finish current iteration batch first)
compute_pipeline_result    full result tree across all completed stages
```

The existing `compute_run / compute_status / compute_result / compute_abort` are **unchanged**.
ACP is additive.

### The Forge Gate in detail

When a stage completes, the worker emits `compute.stage_completed` into the audit chain and
**pauses**. The LLM's steering turn is structured as a **miniature Delegation Loop iteration**:

```
Gate opens
  ↓
LLM receives (via compute_pipeline_status):
  {
    "current_stage": "feature_eng",
    "best_loss":     0.031,
    "best_params":   { "n_components": 20, "method": "pca" },
    "top_k":         [...],
    "artifacts": [
      { "name": "features.parquet",
        "path": "$feature_eng.artifacts/features.parquet",
        "snapshot": { /* L24 schema + stats, PII-redacted */ }
      }
    ],
    "budget_remaining": { "stages": 17, "tool_creations": 9, "iterations": 870 }
  }

LLM inspects artifact snapshot (column stats, distinct counts, distribution hints)
LLM sees: bimodal distribution in col "sensor_delta"

LLM forks:
  Option A — data fits a known pattern → use existing tool, call pipeline_steer()
  Option B — data needs bespoke handling → call forge_tool(), then pipeline_add_stage()

Gate closes when LLM calls pipeline_resume() or pipeline_abort()
```

**Gate timeout:** `steering_gate_timeout_s` (default 3600). If the LLM session ends or the gate
times out, the worker auto-resumes with the next pre-defined stage (if any) or terminates
gracefully. No deadlock.

### The $ref syntax for inter-stage data

Stages reference prior stage outputs via `$stage_id.xxx`:

| Reference | Resolves to |
|---|---|
| `$stage_id.artifacts/filename` | Absolute path, ro-bound into next stage's bwrap |
| `$stage_id.best_params` | JSON object of the winning parameter set |
| `$stage_id.best_loss` | Float scalar |
| `$stage_id.top_k` | List of Top-K records |

References are resolved at stage-start, not at pipeline-submit. A stage added dynamically at
Gate 2 can reference Stage 1 artifacts that didn't exist when the pipeline was submitted.

### Forge Gate + forge_tool interaction

The `forge_tool` MCP call is **unchanged**. A tool forged at Gate N is immediately available
for `pipeline_add_stage` at Gate N. The tool inherits its normal scope (session/project/user)
and can be promoted afterward via `forge_promote`.

The pipeline manifest records which stage introduced which tool:

```json
{
  "stage_id": "cluster_analysis",
  "tool_name": "kmeans_v2",
  "forged_at_gate": 1,
  "forge_run_id": "tool_AbCd..."
}
```

This creates a full audit trail: a given pipeline result can be explained by walking its
manifest and the forge audit events for each dynamically created tool.

### 5D Budget (AWP-aligned)

```yaml
budget:
  max_stages:            20    # total stages including dynamically added
  max_tool_creations:    10    # forge_tool calls during this pipeline run
  max_total_iterations: 1000   # sum of all stage iterations
  max_wall_clock_s:   86400    # 24h hard cap
  steering_gate_timeout_s: 3600 # auto-resume if LLM doesn't steer
```

Budget is checked at each gate and at each stage-terminal. Exceeding any dimension terminates
the pipeline gracefully with `convergence_reason: "budget-<dim>"`.

The 5D shape is deliberately aligned with AWP's budget schema (ADR-0001 § 5D-Budget). A pipeline
manifest expressed as `workflow.awp.yaml` maps naturally to the ACP budget fields.

### On-disk layout

```
<atelier_home>/tenants/<tid>/compute/pipelines/<pipeline_id>/
├── manifest.json               # pipeline def, budget, forged_tools per gate
├── pipeline_summary.json       # rolling: current_stage, state, best_loss per stage
│
└── stages/
    ├── feature_eng/            # Stage 0
    │   ├── manifest.json
    │   ├── summary.json
    │   ├── iterations/0001.json ...
    │   └── artifacts/
    │       └── features.parquet
    │
    ├── cluster_analysis/       # Stage 1 — forged at Gate 1
    │   ├── manifest.json       # includes forged_at_gate: 1
    │   ├── summary.json
    │   ├── iterations/
    │   └── artifacts/
    │
    └── train_model/            # Stage 2 — added at Gate 2
        └── ...
```

All modes `0600`. Path-gate (L10) extended: `**/pipelines/**` added to the protected glob set.

### Audit chain — 5 new event types

All metadata-only. No parameter values, no artifact content, no PII.

| Event | Severity | Fields |
|---|---|---|
| `compute.pipeline_started` | INFO | pipeline_id, stage_count, steering_gate, budget |
| `compute.stage_completed` | INFO | pipeline_id, stage_id, best_loss, total_iterations, tool_name |
| `compute.pipeline_gate_opened` | INFO | pipeline_id, gate_index, next_stage_id, budget_remaining |
| `compute.pipeline_tool_forged` | INFO | pipeline_id, gate_index, tool_name, forge_run_id |
| `compute.pipeline_terminal` | INFO | pipeline_id, state, stages_completed, total_iterations, total_wall_s, convergence_reason |

### Convergence auto-stop

Pipelines with `steering_gate: false` can self-terminate:

```yaml
convergence:
  window: 2                # look at last N stages
  min_loss_delta: 0.005    # stop if best_loss improved by less than this across the window
```

If the improvement plateau is detected, the worker emits `compute.pipeline_terminal` with
`convergence_reason: "cross-stage-plateau"` without opening the next gate.

---

## How this relates to existing architecture

### AWP alignment

| AWP concept | ACP equivalent |
|---|---|
| Delegation Loop | Forge Gate iteration (LLM as manager) |
| Worker | Compute stage (N iterations of a forge tool) |
| Manager decides next worker | LLM at gate decides next stage + optional forge |
| termination: min_confidence_delta | convergence.min_loss_delta |
| 5D budget | ACP budget (same shape, adapted dims) |
| state: `{state.foo}` substitution | `$stage_id.xxx` artifact references |
| R17 output contract | stage summary.json with best_loss + top_k |

ACP does **not** import `awp.runtime.*` (enforced by the existing AST lint in `test_awp_walker.py`).
It is standards-aligned, not runtime-coupled. Same contract as ADR-0005.

### Forge (L6) alignment

`forge_tool` is called by the LLM at a gate, not by the worker. The resulting tool is
immediately available to the worker for the next stage. Forge's path-gate, audit chain,
bwrap sandbox, and MCP surface are all unchanged. The pipeline just provides a structured
context in which forge_tool calls have meaning and traceability.

### Compute Worker (L25) alignment

The parallel driver, strategy protocol, budget evaluator, and RunStore are reused verbatim for
each stage. A pipeline is operationally a sequence of `compute_run` instances managed by a
thin pipeline coordinator layer (`pipeline.py`) sitting above the existing worker daemon.

The worker daemon does **not** restart between stages. It keeps the Unix socket open, runs
stages sequentially (or in parallel if `depends_on` allows), and parks at each gate waiting
for a `resume` envelope on the socket.

---

## Tenant configuration

```yaml
spec:
  compute:
    enabled: true
    pipelines:
      enabled: true
      max_pipeline_stages: 20
      max_tool_creations_per_pipeline: 10
      max_pipeline_wall_clock_s: 86400
      steering_gate_timeout_s: 3600
      allow_dynamic_stages: true      # false → LLM can steer params but not add stages
      allow_forge_at_gate: true       # false → no forge_tool calls during pipeline
```

`allow_forge_at_gate: false` is the conservative enterprise setting: the LLM can narrow param
spaces at each gate but cannot introduce new code. Useful when the operator wants all tools
pre-approved before a pipeline run.

---

## Security model

| Risk | Mitigation |
|---|---|
| LLM forges malicious tool during pipeline | Existing forge sandbox + path-gate (unchanged). Pipeline adds no new trust surface for forged tools. |
| LLM adds unbounded stages | `max_pipeline_stages` budget dim; exceeded → graceful terminal |
| LLM forges too many tools | `max_tool_creations_per_pipeline` budget dim |
| Gate deadlock (LLM session ends) | `steering_gate_timeout_s` → auto-resume or terminal |
| Artifact exfiltration across stages | Artifacts are ro-bound into bwrap; tool cannot write to prior stage dirs |
| Raw param values in audit | Same `_ALLOWED_FIELDS` allow-list + regression gate as L25 |
| LLM writes to pipeline manifest directly | Path-gate: `**/pipelines/**` is a new protected glob |

`allow_forge_at_gate: false` is available for operators who want to block dynamic tool
creation entirely — tools must be pre-promoted before a pipeline run, same as pre-ACP behaviour.

---

## Rejected alternatives

### A: Static pipeline DAG submitted at once

Submit all stages upfront, like a static AWP DAG. The LLM only steers params, not stage
structure. Simpler implementation, but misses the core use-case: the LLM cannot plan stage N+1
before it has seen stage N's results. Reduces ACP to a slightly more convenient manual-chaining
wrapper.

### B: LLM runs the iteration loop itself (no worker)

The LLM calls `forge_tool` and evaluates it directly, iterating in its own context. Unlimited
flexibility, but burns LLM tokens per iteration (the exact failure mode L25 was designed to
eliminate). Rejected.

### C: Workflow-as-code (Python script submitted to worker)

Operator submits a Python script; the worker runs it as the pipeline. Flexible but bypasses the
forge sandbox for the orchestration layer itself, creates a code-injection surface at the
pipeline level, and requires the LLM to generate correct Python rather than declarative stage
descriptions. Rejected.

### D: AWP Runtime for orchestration

Restore AWP runtime (`awp.AWPAgent.run`) as the pipeline coordinator. Rejected explicitly by
ADR-0005: AWP runtime owns its own HTTP path, bypassing the engine layer and audit hooks.
ACP achieves the same adaptivity through the Forge Gate pattern without touching AWP runtime.

---

## Consequences

### Positive

- LLM-driven pipelines become first-class citizens: the LLM is the expert who adapts the plan,
  the worker is the engine that executes it — clean separation.
- Tools emerge from data rather than from pre-planning. A pipeline over a never-before-seen
  dataset can forge the exact tool the data reveals it needs.
- Every decision is audited: which stages were added at which gate, which tools were forged,
  what the loss trajectory was across stages.
- Zero new trust surface: forge sandbox, path-gate, and audit chain all unchanged. The pipeline
  coordinator adds orchestration logic only, not execution logic.

### Costs and risks

- **Implementation complexity.** The pipeline coordinator (`pipeline.py`) is a new stateful
  component inside the worker daemon. Stage coordination, gate parking, timeout handling, and
  $ref resolution all need careful implementation.
- **Test surface.** Each gate behaviour (timeout, forge, add_stage, resume, abort, auto-stop)
  needs E2E tests. Target: ≥ 120 new test cases across the pipeline module.
- **LLM quality matters more.** A poorly steered pipeline can waste compute budget. The
  `budget` constraints are the guardrails, but the quality of gate steering depends on the LLM.
- **Upgrade path.** Existing `compute_run` users are unaffected; no migration needed.

---

## Implementation plan (sketch)

| Phase | Scope | Key deliverable |
|---|---|---|
| 0 | Schema | `pipeline_manifest.py` — dataclasses for PipelineManifest, Stage, GateBudget, RefSyntax |
| 1 | Coordinator | `pipeline.py` — stage sequencing, gate park/resume, $ref resolution, timeout |
| 2 | MCP surface | 6 new tools in `mcp_bridge.py` wired to coordinator |
| 3 | Audit | 5 new event types in `audit.py` + allow-list enforcement |
| 4 | Path-gate | `**/pipelines/**` glob in `path_gate.py` + tests |
| 5 | Tenant config | `PipelineConfig` in `tenant_config.py`, validated schema |
| 6 | E2E tests | ≥ 120 cases: gate timeout, forge-at-gate, add_stage, auto-stop, crash-recovery |
| 7 | Docs | Update `docs/compute.md` with Pipeline section, new SVG |

---

## References

- ADR-0001 — AWP as orchestration layer; Delegation Loop pattern; 5D budget
- ADR-0005 — AWP standards-only decision; no `awp.runtime.*` import
- ADR-0006 — AWP DAG Walker (engine-driven, standards-aligned)
- ADR-0013 — Compute Worker (L25); `compute_run` contract; parallel driver
- ADR-0026 — Compute Fabric; multi-tenant worker; strategy protocol
- ADR-0012 — Large-Data Snapshot (L24); `data_handle` and artifact `ro-bind` pattern
- Layer 6 (Forge) — `forge_tool`, `forge_promote`, bwrap sandbox
- Layer 10 (path-gate) — protected path glob extension required
- `core/compute/atelier_compute/` — implementation home
- `operator/bridges/shared/awp_walker.py` — Delegation Loop reference implementation
