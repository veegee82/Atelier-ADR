# ADR-0028 — Hierarchical Adaptive Compute (HAC)

**Status:** Proposed  
**Date:** 2026-05-17  
**Author:** Maintainer  
**Supersedes:** None  
**Implements:** ADR-0029 (Compute Engine Protocol) as `HACEngine` (engine_id = "hac")  
**Extends:** ADR-0027 (Adaptive Compute Pipelines)  
**Depends on:** ADR-0027, ADR-0013, ADR-0026, ADR-0006 (AWP DAG Walker), ADR-0001 (AWP Orchestration)

---

## Context

### What ADR-0027 solved — and what it left open

ADR-0027 (Adaptive Compute Pipelines) gave the LLM the ability to:
- Run multiple compute stages in sequence
- Forge new tools mid-run based on what intermediate results reveal
- Steer the next stage at each Forge Gate without pre-planning the entire pipeline

That is sufficient for problems with a **discoverable linear structure**: you do not know all the
steps upfront, but once you have seen step N you can plan step N+1.

It is not sufficient for problems with **deeply interdependent sub-problems** — where the quality
of sub-problem B depends on sub-problem A, sub-problem C reads from both, and failure anywhere
degrades the whole without telling you *where* the failure originated.

A concrete example: optimising an end-to-end ML pipeline.

```
feature engineering ──→ architecture search ──→ hyperparameter tuning ──→ final eval
       ↑                        ↑                          ↑
  depends on raw data      depends on features        depends on arch+features
```

If the final loss is 0.45 (too high), the linear ACP answer is: steer the next stage.
But which stage? Architecture search may look fine in isolation while silently poisoning
hyperparameter tuning because the features it received were miscalibrated. A linear pipeline
cannot answer this without running the entire thing again from scratch.

The missing capability: **attribute the root loss back to the sub-problem that caused it,
then selectively re-run that sub-problem with more resources and a better-fitted tool.**

This is what a neural network's backward pass does. HAC brings the same mechanism to Compute.

### The AWP sub-manager insight

ADR-0001 introduced AWP's Delegation Loop — a manager that adaptively spawns sub-workers based
on the current state. AWP also supports **nested delegation loops**: a sub-manager is itself a
`delegation_loop` node that can spawn its own workers, each with its own budget and termination
criterion.

```yaml
# AWP pattern (from news_sentiment.awp.yaml)
- id: sentiment_analysis
  type: delegation_loop
  config:
    manager: sentiment_manager
    budget: { max_loops: 3, max_total_workers: 5 }
    termination: { window: 2, min_confidence_delta: 0.05 }
```

HAC maps this pattern onto Compute: the manager is the LLM at the Backprop Gate, the workers
are compute sub-pipelines (each themselves ACP pipelines from ADR-0027), and the termination
criterion is cross-round convergence of the root loss rather than confidence delta.

The theoretical anchor: **HAC is AWP's Delegation Loop for Compute.**
AWP manages LLM-call sub-agents adaptively. HAC manages compute sub-pipelines adaptively.
The pattern is identical; only the workers differ.

---

## Decision

Introduce **Hierarchical Adaptive Compute (HAC)** as an extension to ACP (ADR-0027):

- A HAC pipeline is a **tree** of compute sub-problems, each itself an ACP pipeline.
- Each sub-pipeline has its own budget, strategy, Forge Gates, and **sub-loss function**.
- The root computes a **composite loss** from the weighted sub-losses of its children.
- After the composite root loss is computed, a **Backprop Gate** opens:
  the system performs attribution analysis (`∂L_root / ∂sub_i`) to identify the sub-pipeline
  most responsible for the current root loss.
- The LLM at the Backprop Gate can: forge new tools, reallocate budget from converged to
  struggling sub-pipelines, re-run the highest-attribution sub-pipeline, and resume or stop.
- **Fluid budget**: unused budget from converged sub-pipelines is reallocated at each Backprop
  Gate to the sub-pipeline with the highest attribution score.
- Convergence is declared when `|L_root(n) − L_root(n−1)| < ε` holds across
  `convergence_window` consecutive Backprop Gate rounds, or when the budget is exhausted.
- All ADR-0027 invariants hold: zero Anthropic SDK imports, zero LLM tokens during stage
  execution, metadata-only audit chain, path-gate protection, bwrap sandbox.

---

## Architecture

![HAC — Sub-Manager tree, composite loss, backpropagation gate, fluid budget](../assets/hac-concept.svg)

---

## Design

### Mental model

```
compute_hac(root_problem, sub_managers, loss_weights, budget, backprop_gate=true)
    │
    ├─ Sub-Manager A  (e.g. "feature selection")
    │    ├─ Stage A1: raw_feature_eng  (N iterations, zero LLM tokens)
    │    └─ Stage A2: outlier_filter   (forged at Gate A1 after bimodal signal seen)
    │    └─ L_A: { primary: 0.14, coverage: 0.06, confidence: 0.88 }
    │
    ├─ Sub-Manager B  (e.g. "architecture search")
    │    reads: $A.artifacts/features.parquet
    │    ├─ Stage B1: arch_candidate   (N iterations)
    │    └─ Stage B2: arch_pruning     (N iterations)
    │    └─ L_B: { primary: 0.29, coverage: 0.18, confidence: 0.71 }
    │
    └─ Sub-Manager C  (e.g. "hyperparameter tuning")
         reads: $A.artifacts, $B.artifacts
         └─ Stage C1: bayesian_hparam_sweep
         └─ L_C: { primary: 0.09, coverage: 0.04, confidence: 0.94 }
    │
    ▼
L_root = 0.5·L_A.primary + 0.3·L_B.primary + 0.2·L_C.primary
       = 0.5·0.14 + 0.3·0.29 + 0.2·0.09  =  0.195

BACKPROP GATE opens:
    Attribution:  ∂L_root/∂B = 0.68  ←── highest
                  ∂L_root/∂A = 0.24
                  ∂L_root/∂C = 0.08  (converged, donates budget to B)
    LLM: forge_tool("better_arch_search")
    LLM: B.add_budget(C.surplus)
    LLM: resume(re_run=["B"])

    → Sub-Manager B re-runs with new tool + extra budget
    → L_root recalculated: 0.131   (ΔL = 0.064 > ε)

BACKPROP GATE opens (round 2):
    Attribution shifts: ∂L_root/∂A = 0.51  ←── now A is the bottleneck
    LLM: A.add_stage("feature_cross", strategy="random")
    LLM: resume(re_run=["A", then "B", then "C"])

    → L_root: 0.089   (ΔL = 0.042 > ε, continue)
    ...
    → L_root: 0.081   (ΔL = 0.008 < ε over window=2)

CONVERGENCE — compute_hac_result() returned
```

### Loss profile — extended tool contract

The existing forge tool contract returns a scalar. HAC extends this to a **loss profile**,
backward-compatible: if the tool returns a plain scalar `loss`, it is treated as
`{ "primary_loss": <value> }` with all auxiliary losses defaulting to zero.

```json
{
  "primary_loss":   0.14,
  "auxiliary": {
    "coverage":     0.06,
    "stability":    0.03,
    "efficiency":   0.01
  },
  "confidence":     0.88
}
```

`confidence` is the AWP R17 output-contract field (ADR-0001 § R17): a float in [0, 1]
indicating how certain the sub-pipeline is about its result. The Backprop Gate factors
confidence into attribution: a sub-pipeline with low confidence and high primary loss
is re-run before one with high confidence and the same primary loss.

The `auxiliary` fields are operator-defined. Common candidates:

| Field | Meaning |
|---|---|
| `coverage` | What fraction of the declared param space was explored |
| `stability` | Variance of the top-K results (low = robust minimum found) |
| `efficiency` | Wall time per iteration relative to budget |
| `data_quality` | Operator-injected quality signal from a data-validation stage |

### Composite loss function

Defined at pipeline submission time:

```json
{
  "type":    "weighted_sum",
  "weights": { "A": 0.5, "B": 0.3, "C": 0.2 },
  "field":   "primary_loss"
}
```

Three supported modes:

| Mode | Semantics |
|---|---|
| `weighted_sum` | `L_root = Σ w_i · L_i.field` (default) |
| `pareto` | Root succeeds when ALL sub-losses satisfy per-manager thresholds |
| `cascade` | `L_root = L_C` if `L_A < τ_A AND L_B < τ_B`, else `1.0` |

Operator-defined composite functions are allowed as Python callables registered in the
strategy module (same trust level as custom strategies in ADR-0027).

### Attribution analysis

The Backprop Gate computes attribution with a Shapley-approximation over the completed
sub-pipeline losses. The computation runs **in the worker**, not in an LLM subprocess:

```
For each sub-manager i:
    baseline_L = L_root(all sub-managers at best_loss)
    counterfactual_L = L_root(sub_i replaced by its median_loss, others at best_loss)
    attribution_i = counterfactual_L − baseline_L

Normalise: attribution_i / Σ attribution_j
```

This is a lightweight, deterministic approximation (no SK-learn, no LLM call). It is accurate
enough to rank sub-managers; the LLM at the gate does the nuanced interpretation.

The attribution scores are included in `compute_hac_status()` and in the
`compute.backprop_gate_opened` audit event.

### Budget — hierarchical and fluid

Budget is allocated top-down at submission:

```yaml
budget:
  total_iterations:      2000
  total_wall_clock_s:   14400     # 4h
  max_backprop_rounds:     5      # re-run cycles after root loss
  convergence_window:      2      # consecutive rounds below ε to declare convergence
  convergence_epsilon:  0.005

  sub_managers:
    A: { fraction: 0.30, min_iterations: 100 }
    B: { fraction: 0.45, min_iterations: 200 }
    C: { fraction: 0.25, min_iterations:  80 }

  fluid_reallocation:
    enabled:           true
    trigger:           "convergence"      # or "gate", "attribution_threshold"
    max_transfer:      0.50               # max fraction of a sub-budget that may move
```

At each Backprop Gate, the worker computes how much budget remains unused in converged
sub-managers and offers it to the LLM for reallocation. The LLM can direct surplus to any
sub-manager; the worker enforces `min_iterations` as a floor.

**Budget inheritance**: a sub-manager that is itself an ACP pipeline (ADR-0027) receives its
budget as a self-contained `compute_pipeline` submission. It cannot request more than its
allocated share; the root coordinator blocks over-allocation.

### Recursive sub-managers

A sub-manager may be declared as a full HAC pipeline, enabling recursive nesting:

```json
{
  "manager_id": "B",
  "type": "hac",
  "sub_managers": ["B_search", "B_prune"],
  "loss_weights": { "B_search": 0.6, "B_prune": 0.4 },
  "max_nesting_depth": 3
}
```

`max_nesting_depth` (default 3, max 5) prevents runaway recursion.
At depth > `max_nesting_depth`, the sub-manager is automatically downgraded to a flat
ACP pipeline (ADR-0027) with no further nesting.

### New MCP tools (5 additional)

All existing ACP tools (`compute_pipeline_*`) and all existing L25 tools (`compute_run / *`)
are **unchanged**.

```
compute_hac                submit a HAC tree (returns hac_id, non-blocking)
compute_hac_status         current state: sub-manager progress, attributions, budget
compute_hac_steer          at backprop gate: add budget, forge tool, add stage to sub-manager
compute_hac_resume         release the backprop gate, re-run attributed sub-managers
compute_hac_result         full result tree: per-manager top-K, root loss history, convergence
```

`compute_hac_steer` is the gate-level equivalent of ACP's `compute_pipeline_steer`,
extended with two new operations:
- `reallocate_budget(from_manager, to_manager, fraction)`
- `add_stage_to_manager(manager_id, stage_def)` — adds a stage into any sub-manager,
  including one that already completed (it will be re-run from that stage onward)

### On-disk layout

```
<atelier_home>/tenants/<tid>/compute/hac/<hac_id>/
├── manifest.json              # full tree definition, loss_weights, budget
├── hac_summary.json           # rolling: round, L_root history, attributions, state
│
└── sub_managers/
    ├── A/                     # each is a full ACP pipeline workspace (ADR-0027 layout)
    │   ├── manifest.json
    │   ├── pipeline_summary.json
    │   └── stages/
    │       ├── raw_feature_eng/
    │       │   ├── iterations/
    │       │   └── artifacts/
    │       └── outlier_filter/    ← forged at Gate A1
    │           ├── iterations/
    │           └── artifacts/
    │
    ├── B/
    │   └── stages/ ...
    │
    └── C/
        └── stages/ ...
```

All modes `0600`. Path-gate (L10) extended: `**/hac/**` added to protected glob set,
alongside the existing `**/pipelines/**` from ADR-0027.

### Audit chain — 6 new event types

All metadata-only. No parameter values, no artifact content, no attribution scores with raw values.

| Event | Severity | Fields |
|---|---|---|
| `compute.hac_started` | INFO | `hac_id`, `manager_count`, `budget`, `backprop_gate` |
| `compute.hac_round_started` | INFO | `hac_id`, `round`, `managers_to_run` |
| `compute.hac_root_loss_computed` | INFO | `hac_id`, `round`, `root_loss`, `sub_losses` (manager→scalar only) |
| `compute.backprop_gate_opened` | INFO | `hac_id`, `round`, `attributions` (manager→fraction), `budget_remaining` |
| `compute.hac_budget_reallocated` | INFO | `hac_id`, `from_manager`, `to_manager`, `fraction`, `trigger` |
| `compute.hac_terminal` | INFO | `hac_id`, `state`, `rounds_completed`, `final_root_loss`, `convergence_reason` |

`sub_losses` in `compute.hac_root_loss_computed` carries only the scalar primary loss per
sub-manager — no auxiliary breakdown, no artifact paths, no parameter fingerprints.
Those are already covered by the per-stage events from ADR-0027.

`attributions` in `compute.backprop_gate_opened` carries fractions summing to 1.0 — no raw
Shapley values, no intermediate counterfactual losses that could fingerprint the data.

### Convergence and termination

```
Round n:  L_root(n) = composite(sub-losses after re-runs)
          Δn = |L_root(n) − L_root(n−1)|

Convergence:  Δn < ε  AND  Δ(n-1) < ε    (window = 2, configurable)
Budget stop:  rounds_completed ≥ max_backprop_rounds
              OR total_iterations_used ≥ budget.total_iterations
              OR wall_clock_elapsed ≥ budget.total_wall_clock_s

On stop: emit compute.hac_terminal, write hac_summary.json, release all sub-manager resources
```

Auto-convergence without LLM intervention is available when `backprop_gate: false`:
the worker runs all sub-managers, computes attribution, re-runs the highest-attribution
sub-manager automatically (no steering turn), repeats until convergence or budget.

`backprop_gate: true` (default) is recommended for complex problems where the LLM's
judgment about *what to do at each round* is valuable (forge new tools, redesign a
sub-manager, stop early because the result is already good enough for the use-case).

---

## Tenant configuration

```yaml
spec:
  compute:
    enabled: true
    hac:
      enabled: false                     # default OFF — opt-in per tenant
      max_nesting_depth: 3
      max_backprop_rounds: 10
      max_sub_managers: 8
      fluid_reallocation: true
      allow_forge_at_backprop_gate: true # false → lock down tool creation during HAC
      convergence_epsilon: 0.005
      convergence_window: 2
      backprop_gate_timeout_s: 7200      # auto-resume if LLM disappears
```

`allow_forge_at_backprop_gate: false` is the conservative enterprise setting: sub-managers
may still forge tools at their own Forge Gates (ADR-0027), but no new tools can be created
during the backpropagation round. All tools must exist before the HAC run starts.

---

## Security model

| Risk | Mitigation |
|---|---|
| Unbounded recursion | `max_nesting_depth` (default 3, max 5) hard-capped by coordinator |
| Budget explosion across sub-managers | Budget is hierarchical; sub-managers cannot exceed their allocated share; `max_transfer` caps fluid reallocation |
| Attribution gaming (LLM forces re-run of all managers) | Attribution is computed by the worker (deterministic Shapley approx), not by the LLM; LLM can choose which to re-run but not which attribution scores to see |
| Path-gate bypass via nested pipeline | `**/hac/**` is a new protected glob; sub-manager artifact dirs inherit path-gate coverage through `**/pipelines/**` from ADR-0027 |
| Raw parameter leakage in attribution event | `attributions` field carries fractions only; per-stage parameter fingerprints are already in stage-level events |
| Forge-at-gate creating unbounded new tools | `max_tool_creations` from ADR-0027 budget applies per sub-manager; root HAC adds `max_total_tool_creations` across all sub-managers |
| LLM reallocation creates runaway budget flow | `min_iterations` floor per sub-manager; `max_transfer` fraction cap per reallocation event |
| Backprop gate deadlock | `backprop_gate_timeout_s` → auto-resume with highest-attribution sub-manager re-run, no user intervention needed |

---

## Relation to existing architecture

### ADR-0027 (ACP) — HAC extends, does not replace

Every sub-manager in a HAC tree is a full ACP pipeline (ADR-0027). HAC adds:
- The root composite loss function
- The Backprop Gate with attribution analysis
- Fluid budget reallocation between sub-managers
- Recursive nesting up to `max_nesting_depth`

A user who needs a single linear pipeline with forge gates continues to use
`compute_pipeline` from ADR-0027 unchanged.

### ADR-0006 (AWP DAG Walker) — structural parallel

The AWP DAG Walker walks nodes in topological order, validates R17 output contracts,
and passes state between nodes via `{state.foo}` substitution.

HAC is structurally equivalent at the sub-manager level:
- Sub-managers correspond to DAG nodes
- `$manager_id.artifacts/file` is the HAC analog of `{state.outputs.file}`
- `confidence` in the loss profile is the HAC analog of AWP's R17 `confidence` field
- `max_backprop_rounds` is the HAC analog of AWP's `max_loops`

HAC does **not** import `awp.runtime.*` (the existing AST lint from `test_awp_walker.py`
covers HAC modules as well). The alignment is structural, not runtime.

### The full compute stack

```
ADR-0013  Compute Worker         single stage, N iterations, scalar loss
    ↓
ADR-0026  Compute Fabric         multi-tenant, strategies, parallel driver
    ↓
ADR-0027  Adaptive Compute       linear pipeline, forge gate, LLM steers
           Pipelines (ACP)
    ↓
ADR-0028  Hierarchical           tree of sub-pipelines, composite loss,
           Adaptive Compute       attribution, backprop gate, fluid budget
           (HAC)
```

Each layer is independently usable. HAC is the layer for problems that cannot be
planned linearly upfront.

---

## Rejected alternatives

### A: LLM manages sub-problems manually (no HAC)

The LLM issues sequential `compute_pipeline` calls for each sub-problem, inspects results,
and decides which to re-run manually. Works, but: the attribution analysis must be done by
the LLM (burning tokens, risk of hallucinated blame assignment), fluid budget reallocation
is manual (error-prone), and there is no automatic convergence criterion. HAC automates
exactly the parts that are deterministic (attribution, budget) and keeps the LLM for the
parts that require judgment (tool forging, sub-manager design decisions).

### B: Single pipeline with many stages (flat ACP)

A flat ACP pipeline can approximate sub-manager structure by ordering stages carefully.
Fails when sub-problems genuinely run in parallel, when sub-manager budgets differ, or
when the attribution of a bad root loss requires replaying only one branch of the computation
(a flat pipeline would have to replay all downstream stages). HAC models this structure
explicitly rather than hiding it in stage ordering.

### C: Neural architecture search style (weight sharing)

Run all sub-managers in parallel, share intermediate representations via a differentiable
interface, compute gradients jointly. This requires the forge tools to be differentiable
functions — not a realistic constraint for general-purpose compute (the tool might be a
SQL query or a file transformation). HAC uses finite-difference attribution (Shapley approx)
which works for any black-box tool that returns a scalar loss.

### D: Spark / Dask for parallel sub-pipelines

Use an external parallel execution framework for the sub-manager tree. Requires operators
to install and configure additional infrastructure, loses the bwrap sandbox for each task,
bypasses the audit chain, and breaks the uniform MCP tool interface. HAC extends the
existing worker daemon with a coordinator layer — no new infrastructure dependency.

---

## Consequences

### Positive

- Complex, multi-sub-problem optimisations become first-class: feature engineering,
  architecture search, and hyperparameter tuning run as a single coherent HAC job with
  automatic attribution and budget flow.
- The LLM's judgment is applied where it matters most: at the Backprop Gate, where it
  decides which sub-problem needs a better tool and how to reallocate compute resources —
  not in the iteration loop itself (which remains token-free).
- Convergence is automatic and auditable: the root loss history is in `hac_summary.json`
  and the per-round `compute.hac_root_loss_computed` events in the audit chain.
- The AWP alignment gives HAC a theoretical foundation: it is not an ad-hoc extension
  but an application of the Delegation Loop pattern to compute rather than LLM calls.

### Costs and risks

- **Implementation scope.** HAC adds a coordinator layer above the ACP pipeline coordinator
  from ADR-0027. Both must handle crash recovery, gate timeouts, and budget enforcement
  correctly at their respective levels.
- **Attribution quality.** Shapley approximation is fast (deterministic, no LLM) but coarse.
  For sub-managers with very similar loss profiles, the attribution may be misleading. The
  Backprop Gate exposes all sub-loss profiles to the LLM so it can override the automated
  attribution recommendation when it has domain knowledge.
- **Test surface.** Each new behaviour (fluid reallocation, recursive nesting, auto-convergence,
  backprop gate timeout, forge-at-backprop-gate) needs E2E coverage. Target: ≥ 150 new test
  cases across `hac_coordinator.py` and integration tests with the ACP pipeline coordinator.
- **Operator understanding.** HAC introduces composite loss functions, attribution, and fluid
  budgets — concepts that require operator buy-in beyond simple "run this tool N times."
  Documentation and sensible defaults (`backprop_gate: true`, conservative budget limits) are
  the mitigation.

---

## Implementation plan

| Phase | Scope | Key deliverable |
|---|---|---|
| 0 | Schema | `hac_manifest.py` — HACManifest, SubManager, LossWeights, FluidBudget, AttributionResult |
| 1 | Attribution | `hac_attribution.py` — Shapley approx, composite loss modes (weighted_sum, pareto, cascade) |
| 2 | Coordinator | `hac_coordinator.py` — sub-manager sequencing, round loop, backprop gate park/resume, timeout |
| 3 | Fluid budget | Budget tracker with `reallocate()` + `min_iterations` floor enforcement |
| 4 | MCP surface | 5 new tools in `mcp_bridge.py` wired to HAC coordinator |
| 5 | Audit | 6 new event types + allow-list enforcement + regression gate |
| 6 | Path-gate | `**/hac/**` glob in `path_gate.py` + test cases |
| 7 | Tenant config | `HACConfig` in `tenant_config.py` (extra="forbid") |
| 8 | Recursive nesting | Sub-manager type detection, depth enforcement, budget inheritance |
| 9 | E2E tests | ≥ 150 cases: attribution, reallocation, convergence, gate timeout, crash-recovery, nesting |
| 10 | Docs | `docs/compute.md` HAC section, update `docs/decisions/0027-*.md` cross-ref |

---

## References

- ADR-0001 — AWP Delegation Loop pattern; R17 output contract; 5D budget shape
- ADR-0005 — AWP standards-only line; no `awp.runtime.*` import (HAC inherits this constraint)
- ADR-0006 — AWP DAG Walker; state-passing; topological ordering (structural parallel to HAC sub-manager tree)
- ADR-0013 — Compute Worker; forge tool scalar loss contract; bwrap sandbox
- ADR-0026 — Compute Fabric; parallel driver; strategy protocol; worker daemon
- ADR-0027 — Adaptive Compute Pipelines; Forge Gate; ACP pipeline as HAC sub-manager unit
- ADR-0012 — Large-Data Snapshot (L24); `data_handle` passed to sub-managers as `ro-bind`
- Layer 6 (Forge) — `forge_tool`, `forge_promote`; called by LLM at Backprop Gate
- Layer 10 (Path-gate) — extended with `**/hac/**` protected glob
- `core/compute/atelier_compute/` — implementation home
- `operator/bridges/shared/awp_walker.py` — Delegation Loop reference implementation
- Lundberg & Lee (2017) — SHAP: A Unified Approach to Interpreting Model Predictions (Shapley approx basis)
