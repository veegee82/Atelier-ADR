# ADR-0013 — Compute-Worker Plugin (Iterative Big-Data Computation, Opt-In, Subscription-Native)

**Status:** Accepted (2026-05-14 — 10 sub-phases shipped, 98 tests green)
**Date:** 2026-05-14
**Companion to:** Layer 6 (Forge), Layer 7 (Skill-Forge), Layer 10 (path-gate),
Layer 11 (dialectic — `cli`-mode authentication pattern), Layer 16 v3
(secret-vault), L22 (`WorkerEngine` — engine-layer separation), L23
(voice-transcribe — metadata-only-audit precedent), ADR-0007 Phase 3.1
(`tenant.atelier.yaml`), ADR-0012 (Large-Data Snapshot Layer — hard
prerequisite)
**Implements:** all 10 sub-phases of `0013-implementation-plan.md`
(closure narrative in CLAUDE.md Layer 25; plugin tree under
`core/compute/`; Forge MCP routing in
`operator/forge/forge/mcp_server.py`; path-gate extension; tenant-config
schema slot).

## Context

Forge (Layer 6) generates runtime Python tools that run sandboxed under
bwrap. Today every tool invocation is **one-shot** — the LLM calls
`mcp__forge__<tool>`, gets one result, decides what to do next, calls
again. For iterative computations (parameter sweeps, optimisation,
convergence tasks) this means:

1. **Each iteration pollutes the LLM context.** A 100-step parameter
   sweep produces ~100 intermediate results that the LLM must hold,
   summarise, or carry forward. Even at 1 KB per result that's 100 KB
   of context burnt on bookkeeping rather than reasoning.
2. **The LLM is the wrong loop driver.** Parameter sweeps are
   deterministic Math (grid, Bayesian-Opt, random). Asking an LLM to
   "pick the next parameter" is high-latency, non-deterministic, and
   expensive when a 200-line sklearn implementation does it better.
3. **No structural stop semantics.** The LLM decides when to stop
   iterating; there is no convergence detector, no stall detector,
   no budget enforcer. The user pays for "looks done to me" calls.
4. **No parallelism story.** Even when 80 iterations are independent
   (grid search), they run serially because the LLM issues them
   serially.

The Agent Workflow Protocol (AWP, ADR-0001) solves the same shape with
a **DelegationLoopRunner**: an out-of-LLM-loop driver that owns the
iteration, tracks state in a `ROLLING_SUMMARY.md`, evaluates stop
criteria deterministically, and exposes only the final scalar back to
the manager-LLM. AtelierOS has every supporting piece (Forge for the
deterministic tool, Skill-Forge for promotable strategy code, bwrap for
isolation, audit-chain for forensics) except the loop driver itself.

The compliance baseline (CLAUDE.md `## Compliance baseline`) constrains
the solution shape:

- **No external LLM credentials.** The driver MUST authenticate any
  LLM call through the operator's Claude Code subscription (via
  `claude -p ...` subprocess), never via `ANTHROPIC_API_KEY` or
  OpenRouter / other-provider SDKs. Layer 11 dialectic.py already
  ships this pattern.
- **Audit-chain metadata-only.** Following the L23 precedent, the
  iteration history MUST NOT bleed PII into the chain. Parameters
  may carry sensitive values (endpoints, file paths, business
  thresholds).
- **Opt-in per tenant.** The plugin is structurally separated; default
  off; operator activates per tenant.

## Decision

A new **opt-in plugin** at `core/compute/` that ships a
**separate long-running worker process** and three new MCP tools.
The worker owns the iteration loop, persists state on disk, enforces
stop criteria, and parallelises across a configurable batch size. The
LLM sees only a handle, a Top-K progress view, and the final result.

### A) Plugin-level separation

The plugin lives under `core/compute/` with its own venv,
own `bootstrap.sh`, own test suite skipped from `run-all-tests.sh` when
the venv is absent (mirror of the Phase-2 `atelier-gateway/` pattern).
Default state of a fresh AtelierOS install: **plugin not bootstrapped,
worker not running, MCP tools not advertised**. Operator opt-in is a
two-step process documented in §3.

The plugin's Python code MUST NOT `import anthropic`. A CI lint
(`test_no_anthropic_sdk_import`) walks the AST of every module under
`atelier_compute/` and fails the suite if the import is present.
Mirror of `bridges/shared/dialectic.py`'s cost contract.

### B) Worker process model

The worker is a long-running Python process bound to a Unix socket at
`/run/atelier/compute-<tenant_id>.sock` (configurable). It is NOT
embedded into the Forge MCP server because:

- The Forge MCP server is request/response; embedding a loop driver
  would block the MCP event loop during long runs.
- Worker lifetime is independent of any single MCP session.
- A failing worker must not crash the Forge MCP server.

The worker's responsibilities:

1. Accept run-submissions over the Unix socket (length-prefixed JSON,
   no FastAPI / HTTP — the socket is host-local, the same machine the
   MCP server already runs on).
2. For each submission: spawn a `ComputeRun` task in an asyncio loop
   that drives the iteration.
3. Schedule iterations via a `ThreadPoolExecutor` with `max_workers ==
   tenant.compute.max_parallel_iterations` (default 4, clamp [1, 16]).
4. Each iteration calls the Forge runner's existing `run_tool()` —
   **no second bwrap sandbox is introduced**. The compute plugin
   reuses Forge's tool-execution path verbatim.
5. Persist iteration outcomes to `<atelier_home>/tenants/<tid>/compute/
   runs/<run_id>/iterations/<n>.json` synchronously, then update the
   rolling summary.
6. On terminal state (converged / stalled / budget-exhausted /
   aborted), write the final summary and emit terminal audit events.

The worker is a **daemon process** launched by the operator. It does
not auto-start. Possible launch paths:

- Manual: `atelier-compute serve --tenant <tid>` (foreground).
- systemd-user: `atelier-compute@<tid>.service` template unit;
  enabled per tenant via `systemctl --user enable atelier-compute@acme`.
- bridge-integrated: when the bridge adapter detects an active
  `spec.compute.enabled: true` for the tenant and the worker socket
  is absent, it emits a one-time `compute.worker_unreachable`
  WARNING into the audit chain. It does NOT auto-spawn the worker —
  operator action remains the gate.

### C) MCP-tool surface (the three tools)

```
mcp__forge__compute_run(
    tool_name:     str,                          # name of an already-forged Forge tool
    param_grid:    dict[str, list],              # per-axis values OR distribution spec
    loss_metric:   str,                          # path into tool output, e.g. "result.sharpe"
    strategy:      "grid" | "random" | "bayesian",
    budget:        {
        max_iterations:  int,                    # default 100, clamp [1, tenant.max_iterations_per_run]
        max_wall_clock_s: int,                   # default 600, clamp [1, tenant.max_wall_clock_per_run_s]
        convergence_eps: float,                  # default 1e-3
        stall_after_n:   int                     # default 10
    },
    minimise:      bool,                         # default True (smaller loss is better)
    data_handle:   str | None,                   # optional handle from ADR-0012 data_register
)
→ {
    compute_handle:           str,              # "compute_<22 url-safe chars>"
    estimated_iterations:     int,
    estimated_wall_s:         int,
    accepted_at:              float (unix ts)
}

mcp__forge__compute_status(compute_handle: str)
→ {
    iterations_done:          int,
    iterations_budget:        int,
    best_loss:                float | None,
    eta_s:                    int | None,
    state:                    "queued" | "running" | "converged" | "stalled" | "budget_exhausted" | "failed" | "aborted",
    top_k:                    list[                # K = 5 default, max 10 (operator-configurable)
        {iter: int, loss: float, param_fingerprint: str}
    ],
    stall_count:              int,
    started_at:               float | None,
    last_iteration_at:        float | None
}

mcp__forge__compute_result(compute_handle: str)
→ {
    state:                    "converged" | "stalled" | "budget_exhausted" | "failed" | "aborted",
    best_params:              dict,                # the actual winning params in clear text
    best_loss:                float,
    total_iterations:         int,
    total_wall_s:             float,
    convergence_reason:       str,                 # "eps-threshold-reached" | "stalled-after-N" | "budget-exhausted" | "external-abort"
    artifact_path:            str,                 # absolute path to <run_id>/ — operator-only inspection
    error:                    str | None
}

mcp__forge__compute_abort(compute_handle: str)
→ {state: "aborted", iterations_done: int}
```

**Important**: `compute_run` returns IMMEDIATELY with the handle. The
LLM does not block. Iteration happens in the worker; the LLM polls
`compute_status` at its own cadence or waits and calls
`compute_result` directly (which blocks server-side up to a 30 s
long-poll cap, then returns the current state).

### D) Strategy interface (promotable via Skill-Forge)

Strategies are pure-Python modules with a fixed contract:

```python
class Strategy(Protocol):
    name: str                                     # "grid" | "random" | "bayesian" | <custom>

    def __init__(self, param_grid: dict, *, minimise: bool, seed: int | None) -> None: ...

    def suggest_batch(self, history: list[IterRecord], n: int) -> list[ParamSet]:
        """Return up to n parameter sets to evaluate next. Empty list = stop."""

    def update(self, history: list[IterRecord], new_results: list[IterRecord]) -> None:
        """Incorporate the results of the last batch; mutate internal state."""

    def should_stop(self, history: list[IterRecord]) -> tuple[bool, str]:
        """Return (True, reason) if the strategy itself wants to terminate."""
```

The three bundled strategies:

- **`grid`** — Cartesian product of axis values; batch-trivial; finite
  budget by definition. Stops when grid is exhausted.
- **`random`** — Sampled from per-axis distributions; needs explicit
  iteration budget. Stops only on budget / convergence / stall.
- **`bayesian`** — sklearn `GaussianProcessRegressor` + Expected
  Improvement acquisition; supports batch-EI (q-EI) up to `batch_size`
  parallel suggestions. Falls back to random for the first `2 *
  n_axes` iterations to seed the GP.

Strategies are **Skill-Forge artefacts** under
`operator/skill-forge/skills/dyn/compute_strategy_<name>/`. The bundled
three ship as user-scope skills at install time. New strategies can be
forged by an operator via the existing skill-creation MCP flow, graded
through the standard promotion gates (task → session → project →
user), and become available to the worker on next strategy reload.

A strategy's Python body is **executed inside the worker process**, NOT
inside bwrap. This is a deliberate trust boundary: strategies are
operator-curated code with promotion gates, equivalent in trust to the
audit chain or the path-gate hook itself. The skill linter
(`operator/skill-forge/skill_forge/linter.py`) rejects strategies that
import `subprocess` / `os.system` / `socket` / `urllib` / `anthropic` /
network-touching modules.

### E) Parallelism model

Three knobs:

1. **`tenant.compute.max_parallel_iterations`** — concurrent worker
   threads per run. Default 4, clamp [1, 16]. Each thread spawns one
   bwrap-isolated tool subprocess.
2. **`tenant.compute.max_concurrent_runs`** — number of distinct
   `compute_run` invocations alive at once. Default 2, clamp [1, 8].
   Excess submissions are queued, status `queued`.
3. **Strategy batch size** — `suggest_batch(history, n=...)` is
   called with `n = min(max_parallel_iterations, remaining_budget)`.
   Grid is trivially batched; Bayesian uses q-EI; random emits N
   independent samples.

The parametric cache (cf. §3 of the implementation plan) accelerates
strategies that re-suggest already-evaluated points (common in
Bayesian as the GP narrows). The cache key is
`sha256(tool_sha || canonical_json(params_subset))[:32]` where
`params_subset` is the subset of input fields marked `x-cache-key:
true` in the tool schema.

### F) Audit-chain model (three-tier)

The audit problem: PII-bearing parameters (endpoints, file paths,
business thresholds) must NOT enter the hash-chain (DSGVO Art. 5,
mirrored from L23). But forensics requires that an operator can
reconstruct a run end-to-end. The solution is three tiers:

**Tier 1 — hash chain** (LLM-visible via `voice-audit verify`):

```json
{
  "event": "compute.iteration_completed",
  "ts": 1778204770.0,
  "run_id": "compute_abc123",
  "iter": 42,
  "loss": 0.847,
  "wall_ms": 4230,
  "strategy": "bayesian",
  "param_fingerprint": "sha256:7f3a91b4c2e89f01",
  "cache_hit": false,
  "tenant_id": "acme"
}
```

The `param_fingerprint` is `sha256(canonical_json(params))[:16]` (16
hex chars). Same fingerprint = same params = same tool outcome —
operationally correlatable across the chain, but no PII leak.

**Tier 2 — artifact directory** (operator-only filesystem access):

```
<atelier_home>/tenants/<tid>/compute/runs/<run_id>/
├── summary.json                  # rolling: best_so_far, loss-curve, convergence state
├── iterations/
│   ├── 0001.json                 # {iter, params: {...real values...}, loss, wall_ms, ts}
│   ├── 0002.json
│   └── ...
└── artifacts/                    # tool-emitted artefacts (S8 full-stdout, etc.)
```

Mode `0o600` on every file. Path-gate (Layer 10) protects the tree
from LLM-side `Write` / `Edit` / `Bash` (§ G).

**Tier 3 — per-field sensitivity at the tool schema**:

```jsonc
{
  "input_schema": {
    "type": "object",
    "properties": {
      "window_size": {
        "type": "integer"
      },
      "api_endpoint": {
        "type": "string",
        "x-sensitive": true        // ← new
      }
    }
  }
}
```

Tool author marks sensitive params explicitly. In Tier-2 logs, sensitive
fields are replaced by `<hash:a3f7c1d2e4b8>` (12-char SHA-256 prefix of
the value). Non-sensitive fields keep their clear-text value in the
artifact (operator can read them, the chain still doesn't see them).
Default `x-sensitive: false`. Symmetrical with the existing
`x-redact: true` annotation for secret-vault.

Five new audit-event types, registered in `forge/security_events.py`:

| Event | Severity | Carries |
|---|---|---|
| `compute.run_started` | INFO | run_id, tool_name, strategy, budget |
| `compute.iteration_completed` | INFO | run_id, iter, loss, wall_ms, param_fingerprint, cache_hit |
| `compute.run_terminal` | INFO | run_id, state, total_iterations, total_wall_s, best_loss, convergence_reason |
| `compute.worker_unreachable` | WARNING | tenant_id, attempted_socket — one-shot per bridge boot |
| `compute.run_failed` | WARNING | run_id, iter, error_class, error_message (200-char cap) |

### G) Path-gate extension

Layer 10's `path_gate.py` adds two protected globs:

| Pattern | Why |
|---|---|
| `<atelier_home>/**/compute/**` | Tier-2 artifact tree — LLM must not direct-write |
| `<atelier_home>/**/compute/worker.sock` | Worker socket — LLM must not direct-read or direct-write |

Both follow the existing fail-closed Bash detection (eval / heredoc /
cmd-subst with hint-strings → deny). The MCP path stays open by
construction (worker uses plain Python file-IO inside its own
process, never traverses Claude's PreToolUse gate).

### H) LLM-cost contract (CRITICAL)

The whole driver MUST cost zero Anthropic tokens during execution. The
only LLM touchpoints are:

1. **`compute_run` call** — one MCP tool call, billed to the existing
   Claude Code subscription via the standard MCP path.
2. **`compute_status` calls** — one MCP tool call per poll. The user
   controls cadence.
3. **`compute_result` call** — one MCP tool call.

The worker process, the strategies, the iteration loop, the
convergence detector — all pure Python, zero LLM use.

**Future-proofing**: if a future strategy genuinely needs LLM help
(e.g. an "interpret-loss-curve" advisor strategy), it MUST authenticate
via `claude -p --max-turns 1 --no-tools <prompt>` subprocess — the same
pattern Layer 11 dialectic.py uses. This binds the cost to the
operator's subscription, not to an external API key. The strategy's
manifest declares `requires_llm: true` and the operator can disable it
via `tenant.compute.disallow_llm_strategies: true`.

The plugin's `requirements.txt` MUST NOT contain `anthropic` or
`openai` or `google-cloud-aiplatform` or any other LLM-provider SDK.
This is structural, not policy.

### I) Tenant config extension

`tenant.atelier.yaml` grows a `spec.compute` block:

```yaml
apiVersion: atelier/v1
kind: Tenant
metadata:
  id: acme
spec:
  compute:
    enabled: false                          # default OFF — opt-in per tenant
    max_parallel_iterations:    4           # clamp [1, 16]
    max_concurrent_runs:        2           # clamp [1, 8]
    max_iterations_per_run:     200         # clamp [1, 10000]
    max_wall_clock_per_run_s:   600         # clamp [1, 86400]
    top_k_size:                 5           # clamp [1, 10]
    disallow_llm_strategies:    false       # default false; strategies declaring requires_llm work
    strategies_allowed:         ["grid", "random", "bayesian"]   # operator allowlist
```

Loader extends `atelier_gateway/tenant_config.py` (already validates
the rest of the spec). Per ADR-0007 Phase 3.1: `extra="forbid"` on
the nested model — unknown keys raise.

## Consequences

### Positive

- **The LLM stays out of the iteration loop.** A 100-iteration sweep
  costs three MCP tool calls (`run`, `status`, `result`) of LLM
  tokens, not 100. The driver pays for the actual compute in CPU
  seconds.
- **Iterations parallelise cleanly.** Grid → trivial. Bayesian →
  q-EI batched. Random → independent samples. 4 cores × 80 iters
  × 5 s each ≈ 100 s wall-clock instead of 400 s serial.
- **Strategies are promotable code.** Skill-Forge's grading +
  promotion mechanism applies. A custom "neural-architecture-search"
  strategy starts at task scope, earns its way to user scope or
  gets purged at the 7-day TTL.
- **Subscription-native cost model.** The plugin is unusable for an
  attacker who steals a Claude Code session — they can submit runs,
  but they cannot exfiltrate via a second LLM provider because the
  plugin can't import one.
- **Audit forensics is preserved without PII leak.** Operator with
  filesystem access can reconstruct any run via tier-2 artefacts;
  the chain shows the iteration cadence without the values.
- **Cache-as-correctness.** The parametric cache (§E, plan §3.7)
  means a re-suggested point doesn't re-burn tool wall-clock.
  Strategies become resilient to crashes — restart, re-evaluate
  history, cache lights up.
- **Plugin separation enforces opt-in.** Operators who never want
  this never know it exists. The Forge MCP server doesn't
  advertise the tools unless the worker socket is reachable.

### Negative

- **Worker lifecycle adds operator burden.** A new process to start,
  monitor, restart. systemd-user units help but the operator does
  need to know about it.
- **Strategies trust boundary is wider than tool-author code.**
  Strategy bodies run inside the worker process, not inside bwrap.
  The linter rejects network and subprocess imports, but a clever
  pure-Python adversary could still e.g. consume CPU. Mitigation:
  strategy CPU-time budget per `suggest_batch` call (default 5 s);
  exceed → terminate the run with `failed`.
- **Audit chain volume grows.** A 100-iteration run produces 102
  events (1 start, 100 iter, 1 terminal). For tenants running 50
  compute_runs/day this is ~5000 events/day. The chain handles
  this (Phase 6 metrics aggregation is the read-side projection)
  but operators with very high run rates should review their
  storage budget.
- **Bayesian strategy needs sklearn.** The plugin venv carries
  sklearn + numpy. Disk footprint: ~150 MB. Operators on
  constrained-disk deployments can disable bayesian via
  `strategies_allowed: ["grid", "random"]` and skip the sklearn
  install via `bootstrap.sh --minimal`.
- **q-EI Bayesian is approximate.** Batch-Bayesian using
  Kriging-believer or constant-liar is not equivalent to sequential
  Bayesian on N independent runs. For high-batch-size + tight
  budgets the approximation degrades. Mitigation: default batch
  size 4; documented in the strategy's SKILL.md.

### Risks deliberately accepted

- **An operator who enables compute and forgets to start the
  worker** sees `compute.worker_unreachable` events but the LLM
  doesn't know the tools are gone. The MCP server simply doesn't
  advertise them. This is intentional — failed-loud would mean a
  bridge boot warning even for tenants who never plan to enable
  compute.
- **Strategies marked `requires_llm: true` cost subscription tokens
  per iteration.** Operators who don't want this set
  `disallow_llm_strategies: true`. The default is permissive
  because the bundled three strategies don't use LLMs.
- **A pathological `param_grid` (10^9 points) submitted with
  `strategy: grid`** hits the `max_iterations_per_run` clamp and
  aborts with `budget_exhausted`. The grid is computed lazily;
  the worker does not pre-enumerate 10^9 entries into RAM.

## What this ADR deliberately does NOT include

- **A new sandbox.** Tool execution reuses Forge's bwrap pipeline
  verbatim. The compute plugin is an iteration ORCHESTRATOR, not a
  second execution layer.
- **Cross-run state.** Each `compute_run` is self-contained. Two
  successive runs don't share GP state, don't share history.
  Operators who want hierarchical optimisation issue a second run
  with a tighter `param_grid` derived from the first's `best_params`.
- **Distributed compute across hosts.** Single-host worker. Multi-host
  is a future ADR (likely Phase 5 of an own roadmap) requiring shared
  state, network coordination, and a different audit-chain story.
- **Streaming or interactive iterations.** An iteration runs to
  completion before the next one is scheduled. No mid-iteration
  steering by the LLM — that would defeat the LLM-out-of-loop
  invariant.
- **A separate cost-accounting system.** Compute runs charge against
  the tenant's existing `spec.budget.max_runs_per_day` (ADR-0007
  Phase 3.1) and `max_tokens_per_day`. Each compute_run counts as 1
  run; each tool invocation inside it counts as 1 tool-call but 0
  tokens (no LLM use). A future ADR can add `compute_iterations_per_day`
  if operators need finer control.
- **An auto-restart supervisor for the worker.** systemd-user does
  this if the operator wants it; we don't ship our own. A worker
  crash leaves the on-disk run state intact; the next boot reads
  it and resumes interrupted runs (cf. plan §3.6 recovery
  semantics).

## Compliance-baseline interaction

- **Strengthens GDPR Art. 5 (data minimisation):** parameter values
  never enter the audit chain (tier-1 fingerprint only); LLM sees
  only Top-K progress with fingerprints, not values.
- **Preserves GDPR Art. 32 (security of processing):** tier-2
  artefacts at `0o600`; path-gate-protected; worker socket
  authenticates by tenant.
- **Reinforces compliance-zone routing (ADR-0007 Phase 3.3):** the
  worker process inherits the tenant's `data_residency.zone`; tool
  invocations route through the same engine-policy gate as direct
  Forge calls.
- **Preserves engine-policy allowlist (ADR-0007 Phase 3.2):** every
  iteration goes through the existing Forge runner, which already
  consults `tenant.atelier.yaml::spec.compute_engine`-style policy
  (currently engine-allowlist via the dispatcher; the runner is
  engine-agnostic so this is structurally inherited).
- **Does not introduce a compliance-off mode.** There is no flag
  that disables the audit emission, the path-gate, or the
  sensitivity-annotation respect. A tenant that turns compute off
  via `enabled: false` is the only "off" — the audit / path-gate /
  sensitivity layers stay structurally active for every active
  run.

The L23 metadata-only-audit rule generalises: **parameters never
enter the chain; only their fingerprint + loss do**. The L23
voice-transcribe rule, the ADR-0012 snapshot-content rule, and now
the ADR-0013 parameter-value rule converge on the same invariant.

## Repository placement: monorepo

This ADR explicitly chooses to ship the plugin inside the AtelierOS
monorepo (`core/compute/`) rather than as a separate
repository. Three structural reasons:

1. **The audit chain is shared infrastructure.** Every chain writer
   (forge, skill-forge, path-gate, voice, gateway) must agree on
   `EVENT_SEVERITY` registration, event-detail ACLs, and
   hash-chain integrity rules. Cross-repo coordination here would
   require version-locked releases; the moment compute is out of
   sync the chain integrity is at risk.
2. **The plugin reuses Forge's tool-execution path verbatim.**
   `forge.runner.run_tool()` is the single execution primitive;
   the compute worker calls into it. A separate repo would either
   vendor a copy (drift) or pin a `forge` version (release
   coordination).
3. **`tenant.atelier.yaml`'s `spec.compute` block lives in the
   gateway's Pydantic schema.** The tenant-config loader is in
   `core/gateway/atelier_gateway/tenant_config.py`;
   extending it from another repo means cross-repo PRs whenever
   the schema evolves.

The plugin still follows the **`.atelier-pkg` (ADR-0007 Phase 5)
publication path** if operators need to distribute it independently:
once feature-complete, the plugin tree is packageable into a signed
`.atelier-pkg` archive for external installs. This is the supported
path for distribution; in-repo development is the supported path
for development.

## Implementation surface (sketch — full plan in 0013-implementation-plan.md)

| Sub-phase | Module | Scope |
|---|---|---|
| 13.1 | `core/compute/{bootstrap.sh,requirements.txt,atelier_compute/__init__.py}` | Plugin skeleton + venv skip in `run-all-tests.sh` |
| 13.2 | `atelier_compute/driver.py` + `state.py` | Loop core (single-threaded), stop criteria, on-disk state |
| 13.3 | `atelier_compute/strategies/{grid,random}.py` + Strategy Protocol | Two bundled strategies (Bayesian deferred to 13.8) |
| 13.4 | `atelier_compute/worker.py` + `transport.py` | Unix-socket daemon, length-prefixed JSON protocol |
| 13.5 | `atelier_compute/mcp_bridge.py` + Forge-side socket discovery | Three MCP tools advertised iff socket reachable |
| 13.6 | `atelier_compute/audit.py` + path-gate extension + tenant-config extension | Tier-1 chain events + tier-2/3 sensitivity + path-gate globs + `spec.compute` schema |
| 13.7 | `atelier_compute/parallel.py` + parametric cache (`x-cache-key`) | ThreadPoolExecutor + Forge cache extension |
| 13.8 | `atelier_compute/strategies/bayesian.py` (sklearn GP + q-EI) | Bayesian strategy as a Skill-Forge skill |
| 13.9 | `atelier_compute/recovery.py` + systemd-user unit template | Crash recovery from on-disk state + service template |
| 13.10 | E2E with real Forge tool + ADR-0012 data_handle + closure | Full pipeline including 50-iter Bayesian sweep against a synthetic timeseries |

Each sub-phase ships its own per-subtask E2E (per the project
convention) and updates `run-all-tests.sh` with a skip-gate so single-
operator deployments without the plugin venv stay 100% green.

## References

- ADR-0001 — AWP as orchestration layer; the DelegationLoopRunner
  pattern this ADR ports.
- ADR-0007 Phase 3.1 — `tenant.atelier.yaml` schema this ADR extends.
- ADR-0007 Phase 5 — `.atelier-pkg` signed package format for future
  external distribution of this plugin.
- ADR-0012 — Large-Data Snapshot Layer; hard prerequisite for the
  `data_handle` parameter of `compute_run`.
- Layer 6 (Forge) — the `run_tool()` primitive the worker calls into.
- Layer 7 (Skill-Forge) — the strategy promotion mechanism.
- Layer 10 (path-gate) — extended with two new protected globs.
- Layer 11 (`dialectic.py`) — the `claude -p` subprocess pattern for
  subscription-native LLM use (template for future LLM-aware
  strategies).
- Layer 16 v3 (secret-vault) — `x-sensitive` mirrors `x-redact`.
- L23 (voice-transcribe) — metadata-only-audit precedent.
- L22 (`WorkerEngine`) — the engine-layer separation that this ADR
  does NOT cross; tool invocations stay engine-agnostic via the
  existing Forge runner.
- [scikit-learn GaussianProcessRegressor](https://scikit-learn.org/stable/modules/gaussian_process.html) —
  Bayesian strategy backend.
- [Frazier 2018 — A Tutorial on Bayesian Optimization](https://arxiv.org/abs/1807.02811) —
  q-EI acquisition reference.
