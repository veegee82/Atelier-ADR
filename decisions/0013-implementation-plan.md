# ADR-0013 Implementation Plan — Compute-Worker Plugin

**Companion to:** [`0013-compute-worker-plugin.md`](./0013-compute-worker-plugin.md)
**Status:** Done (2026-05-14 — all 10 sub-phases landed; 98 tests green)

This plan breaks ADR-0013 into ten sub-phases. Each sub-phase ships as
one Claude Code session with a per-subtask E2E gate, follows the
existing strangler-fig pattern (additive, opt-in, never breaks single-
operator deployments), and updates the closure narrative in
`CLAUDE.md` once landed.

## Phase ordering rationale

Phases 13.1–13.5 are the **minimum viable opt-in path**. After 13.5 an
operator can install the plugin, start the worker, and run grid-search
computations end-to-end. Phases 13.6–13.7 add the **production
hardening** (audit chain, parallelism, parametric cache). Phases
13.8–13.10 add the **advanced surface** (Bayesian strategy, crash
recovery, full E2E).

Phases 13.1 → 13.5 should land before any closure narrative in
`CLAUDE.md` claims the feature exists — until the MCP surface is wired,
no operator can actually use it.

## Sub-phase table

| Phase | Goal | Hard deps | Status |
|---|---|---|---|
| 13.1 | Plugin skeleton + venv skip | — | ✓ |
| 13.2 | Driver core (sequential, on-disk state) | 13.1 | ✓ |
| 13.3 | Grid + Random strategies | 13.2 | ✓ |
| 13.4 | Worker daemon + Unix-socket protocol | 13.2 | ✓ |
| 13.5 | MCP bridge + Forge-side socket discovery | 13.4 | ✓ |
| 13.6 | Audit chain + path-gate + `spec.compute` schema | 13.5 | ✓ |
| 13.7 | Parallelism + parametric cache | 13.6 | ✓ |
| 13.8 | Bayesian strategy (sklearn q-EI) | 13.7 | ✓ |
| 13.9 | Crash recovery + systemd-user template | 13.7 | ✓ |
| 13.10 | E2E with real timeseries + closure | 13.8, 13.9, ADR-0012 ≥ 12.5 | ✓ |

---

## Phase 13.1 — Plugin skeleton + venv skip

**Goal**: bring the plugin tree into existence with an empty but
loadable Python package, a self-contained venv via `bootstrap.sh`, and
a `run-all-tests.sh` entry that skips cleanly when the venv is absent.

### Files

```
core/compute/
├── bootstrap.sh
├── requirements.txt
├── .gitignore                                  # .venv/, __pycache__/
├── README.md                                   # opt-in installation recipe
├── atelier_compute/
│   ├── __init__.py
│   └── version.py                              # __version__ = "0.0.1"
└── tests/
    └── test_plugin_skeleton.py                 # 4 cases — see below
```

### `requirements.txt` (frozen at 13.1)

```
# Runtime — no LLM SDKs, no network libs
# (sklearn / numpy land in 13.8 if Bayesian opt-in)
```

Empty. The minimum subset is stdlib-only. 13.8 adds sklearn + numpy.

### Test gates

1. `test_plugin_directory_exists` — repo has `core/compute/`.
2. `test_bootstrap_creates_venv` — running `bash bootstrap.sh` against
   a clean checkout produces `.venv/bin/python` (skip when CI lacks
   Python venv module).
3. `test_no_anthropic_sdk_import` — AST walk of every `.py` under
   `atelier_compute/` fails the test if `import anthropic` / `from
   anthropic` is present. **This is the cost-contract regression
   gate; it MUST exist from 13.1 onward.**
4. `test_run_all_tests_sh_skips_gracefully` — invoke `bash
   operator/bridges/run-all-tests.sh` with the compute venv
   absent; suite count goes up by 1 (the skip entry), all green.

### `run-all-tests.sh` integration

Add one entry mirroring the gateway pattern:

```bash
if [[ -x core/compute/.venv/bin/python ]]; then
  core/compute/.venv/bin/python -m pytest core/compute/tests/ -q
else
  echo "Python: atelier-compute (skipped — venv not bootstrapped)"
fi
```

### Acceptance

- All four tests pass.
- `bash run-all-tests.sh` count goes from N → N+1 (the skip entry).
- A fresh clone followed by `bash bootstrap.sh` and re-running the
  suite goes from N+1 → N+1+M where M is the actual compute test
  count.

---

## Phase 13.2 — Driver core + on-disk state

**Goal**: implement `ComputeRun` as a sequential loop. No worker, no
MCP, no parallelism yet. Driver runs in-process for testability.

### Files

```
atelier_compute/
├── driver.py                                   # ComputeRun + run loop
├── state.py                                    # RunRecord + RunRegistry (on-disk JSON)
├── budget.py                                   # Budget validation + stop criteria
└── iteration.py                                # IterRecord dataclass + history helpers
```

### Driver contract

```python
class ComputeRun:
    def __init__(self, tenant_id, tool_name, param_grid, loss_metric,
                 strategy, budget, *, atelier_home, runner_fn): ...

    def run(self) -> RunRecord:
        """Block until terminal. Returns the final RunRecord."""
```

`runner_fn` is injected — a callable matching the signature of
Forge's `runner.run_tool()`. Tests inject a stub that returns
deterministic results given inputs; the worker (phase 13.4) injects
the real Forge runner.

### State layout (final shape — used by every later phase)

```
<atelier_home>/tenants/<tid>/compute/runs/<run_id>/
├── manifest.json                  # tool_name, strategy, budget, accepted_at
├── summary.json                   # rolling: best_iter, best_loss, state, last_iteration_at
├── iterations/
│   ├── 0001.json                  # {iter, params (clear), loss, wall_ms, ts, cache_hit, fingerprint}
│   └── ...
└── artifacts/                     # populated by tool S8-fallback writes
```

Atomic-replace writes on `manifest.json` and `summary.json` (tempfile
+ rename, mode 0o600). `iterations/` is append-only — each file
written once, never modified.

### Stop criteria (deterministic)

1. **Convergence**: last iteration's loss within `convergence_eps` of
   best-so-far for `stall_after_n / 2` consecutive iters.
2. **Stall**: no improvement in `stall_after_n` iters.
3. **Budget exhaust**: `iter >= max_iterations` OR `wall_clock_s >=
   max_wall_clock_s`.
4. **Strategy stop**: `strategy.should_stop(history)` returns `(True,
   reason)`.

Tracked in `budget.py::evaluate_termination(history, budget)` →
returns `("running", "")` or `("converged" | "stalled" |
"budget_exhausted", reason)`.

### Test gates

1. `test_run_with_stub_runner_converges` — 3-axis grid, stub runner
   returns deterministic loss, run terminates with `converged` state.
2. `test_budget_max_iterations_clamps` — set `max_iterations=5`, stub
   never converges, terminate with `budget_exhausted`.
3. `test_wall_clock_budget` — stub returns after 200 ms,
   `max_wall_clock_s=1`, terminate ≤ 1100 ms.
4. `test_iteration_files_atomic` — kill driver mid-iteration via
   monkeypatched stub raising mid-call, on-disk state has no
   half-written iteration file.
5. `test_terminal_summary_complete` — terminal `summary.json` carries
   `state`, `best_loss`, `best_iter`, `total_iterations`,
   `total_wall_s`, `convergence_reason`.

### Acceptance

- All five tests pass.
- Driver runs against a stub `runner_fn`; no Forge dependency yet.
- State directory layout matches §3 of the ADR exactly.

---

## Phase 13.3 — Grid + Random strategies

**Goal**: implement the two bundled non-LLM strategies as plain Python
modules under `atelier_compute/strategies/`. Skill-Forge promotion
landing is deferred to 13.10's closure.

### Files

```
atelier_compute/strategies/
├── __init__.py                                 # registry + load_strategy(name)
├── base.py                                     # Strategy Protocol + helper utilities
├── grid.py                                     # Cartesian product walker
└── random.py                                   # per-axis distribution sampler
```

### Grid strategy

Lazy enumeration: `suggest_batch(history, n)` returns the next `n`
points by index. State carried internally is `_next_index`. Total
budget is `len(grid_points)`; `should_stop` returns `(True,
"grid-exhausted")` when index ≥ len.

### Random strategy

Per-axis distributions:

```python
param_grid = {
    "window_size": {"type": "int_uniform", "low": 5, "high": 200},
    "method":      {"type": "categorical", "values": ["std", "downside"]},
    "threshold":   {"type": "float_log_uniform", "low": 1e-3, "high": 1e-1},
}
```

`suggest_batch` draws `n` independent samples; `should_stop` always
returns `(False, "")` (random has no intrinsic stop).

### Test gates

1. `test_grid_enumerates_cartesian` — 2×3×2 grid → 12 unique points
   in 4 batches of 3.
2. `test_grid_batch_smaller_than_remaining` — request batch=10 with 4
   points left → returns 4, next call returns `[]` and
   `should_stop=(True, "grid-exhausted")`.
3. `test_random_independent_samples` — 100 samples from seeded RNG,
   reproducible across re-runs.
4. `test_random_respects_distributions` — `int_uniform[5, 200]` never
   produces a value outside [5, 200].
5. `test_strategy_protocol_satisfied` — both strategies satisfy the
   Protocol via `runtime_checkable`.
6. `test_strategy_registry_load` — `load_strategy("grid")` and
   `load_strategy("random")` return concrete instances;
   `load_strategy("unknown")` raises `UnknownStrategy`.

### Acceptance

- All six tests pass.
- Driver from 13.2 runs end-to-end with each strategy against a stub
  runner.
- `tenant.compute.strategies_allowed` allowlist enforcement deferred
  to 13.6.

---

## Phase 13.4 — Worker daemon + Unix-socket protocol

**Goal**: lift the driver into a separate daemon process. Define the
client/server protocol over Unix sockets. No MCP integration yet —
this phase ships a Python-level client for tests.

### Files

```
atelier_compute/
├── worker.py                                   # WorkerServer (asyncio)
├── transport.py                                # length-prefixed JSON framing
├── client.py                                   # WorkerClient (sync, for MCP bridge in 13.5)
└── cli.py                                      # `python -m atelier_compute serve` entry
```

### Protocol (length-prefixed JSON over Unix socket)

Frame: `<4-byte big-endian length><JSON bytes>`.

Request shape:

```json
{"op": "submit_run", "params": { ... ComputeRun args ... }}
{"op": "get_status", "params": {"compute_handle": "compute_..."}}
{"op": "get_result", "params": {"compute_handle": "compute_..."}}
{"op": "abort_run",  "params": {"compute_handle": "compute_..."}}
```

Response: `{"ok": true, "result": {...}}` or `{"ok": false, "error":
"...", "error_class": "BudgetExhausted" | "UnknownHandle" | ...}`.

### Worker server contract

```python
class WorkerServer:
    def __init__(self, *, tenant_id, atelier_home, socket_path,
                 max_concurrent_runs, runner_fn): ...

    async def serve_forever(self) -> None: ...
    async def stop(self) -> None: ...
```

`socket_path` mode is `0o600`; owner-only. `tenant_id` is hardcoded
per-worker — a worker serves exactly one tenant. Cross-tenant
isolation is achieved by running one worker per active tenant on
different socket paths.

### CLI

```bash
python -m atelier_compute serve \
    --tenant acme \
    --socket /run/atelier/compute-acme.sock \
    --atelier-home /home/user/.atelier
```

### Test gates

1. `test_protocol_roundtrip` — client submits run, polls status,
   reads result; all frames parse correctly.
2. `test_concurrent_run_limit` — submit 3 runs with
   `max_concurrent_runs=2`, third returns `state: "queued"`.
3. `test_unknown_handle_returns_typed_error` — `get_status` on bogus
   handle returns `error_class: "UnknownHandle"`.
4. `test_abort_terminates_run` — submit slow stub run, abort
   mid-flight, result state = `aborted`.
5. `test_worker_serves_only_its_tenant` — request with mismatched
   `tenant_id` in `params` rejected with
   `error_class: "TenantMismatch"`.
6. `test_socket_mode_0600` — after `serve_forever()` boot, socket
   inode has mode 0o600.

### Acceptance

- All six tests pass with a real `asyncio` loop and real socket I/O.
- Manual smoke: `python -m atelier_compute serve` in one terminal,
  `python -m atelier_compute submit ...` in another, full round-trip.

---

## Phase 13.5 — MCP bridge + Forge-side socket discovery

**Goal**: expose the three MCP tools (`compute_run`, `compute_status`,
`compute_result`) on the Forge MCP server, but only when the worker
socket is reachable. Adds `compute_abort` as a fourth tool.

### Files

```
atelier_compute/
└── mcp_bridge.py                               # tool definitions consumed by forge MCP server

operator/forge/forge/
├── mcp_server.py                               # MODIFIED — socket discovery on tool listing
└── _compute_discovery.py                       # NEW — checks compute socket reachability
```

### Discovery semantics

On every `mcp_server._handle_tools_list` call, the server:

1. Resolves the active tenant via `forge.tenants.current_tenant()`.
2. Computes the expected socket path: `<atelier_home>/tenants/<tid>/
   compute/worker.sock`.
3. Attempts a non-blocking connect with 100 ms timeout.
4. On success: appends the four compute tools to the tools list.
5. On failure: silently omits them. Emits `compute.worker_unreachable`
   exactly once per process (deduplicated by a module-level set
   keyed on tenant_id), severity WARNING.

The discovery is **cached for 5 seconds** to avoid burning the connect
syscall on every list-tools poll.

### MCP-tool schemas

Defined in `mcp_bridge.py` as plain dicts (no Pydantic at this layer
— the worker re-validates server-side). Per-tool input schemas
mirror the ADR §C surface byte-for-byte.

### Test gates

1. `test_mcp_tools_advertised_when_socket_present` — start worker
   in same test, list tools via MCP, four compute tools present.
2. `test_mcp_tools_hidden_when_socket_absent` — no worker, list
   tools, none of the four are present.
3. `test_mcp_tools_hidden_when_socket_unreachable` — socket file
   exists (stale) but no one listening, list tools, none present,
   `compute.worker_unreachable` emitted.
4. `test_one_shot_audit_emission` — three list-tools calls with no
   worker, exactly one `compute.worker_unreachable` event in the
   chain.
5. `test_mcp_call_routes_to_worker` — list tools, call
   `compute_run`, worker receives request, response routes back to
   MCP caller.
6. `test_discovery_cache_5s` — list twice within 5 s, only one
   actual connect attempt (mock socket factory counts).

### Acceptance

- All six tests pass with both worker-present and worker-absent
  configurations.
- An LLM running against the Forge MCP server with the worker
  running sees the four compute tools in `tools/list`.
- An LLM running against the same Forge MCP server with the worker
  stopped sees no compute tools.

**After this phase the plugin is end-to-end usable for an operator
opt-in path: bootstrap → start worker → operate.**

---

## Phase 13.6 — Audit chain + path-gate + tenant-config schema

**Goal**: wire the structural compliance layer. Five new audit-event
types into `forge/security_events.py`, two new protected globs into
`operator/voice/hooks/path_gate.py`, one new nested model into
`atelier_gateway/tenant_config.py`.

### Files modified

```
operator/forge/forge/security_events.py          # +5 entries in EVENT_SEVERITY
operator/voice/hooks/path_gate.py                # +2 protected glob patterns
core/gateway/atelier_gateway/tenant_config.py  # +ComputeConfig model
operator/voice/hooks/test_path_gate.py           # +per-glob deny + allow cases
core/gateway/tests/test_tenant_config.py       # +compute-block validation cases
```

### `atelier_compute/audit.py` (new)

Central emitter for the five event types. Mirrors the L23
voice-transcribe pattern — calls `security_events.write_event(...)`
with the per-event allow-listed details.

Critical: `compute.iteration_completed` MUST NOT carry params in
clear. The emitter validates the details dict against an allow-list
(`{"run_id", "iter", "loss", "wall_ms", "strategy",
"param_fingerprint", "cache_hit", "tenant_id"}`); any extra key
raises `AuditFieldNotAllowed`.

### Tier-3 sensitivity

Tool-schema reader extension: when a tool is registered with
`input_schema.properties.<field>.x-sensitive: true`, the driver
records the field name in the run's `manifest.json::sensitive_fields`
list. Every `iterations/<n>.json` write goes through
`audit.redact_sensitive_fields(params, sensitive_fields)` which
replaces the value with `<hash:<12-char-sha256>>`.

### Path-gate extension

Two new patterns in `path_gate._PROTECTED_GLOBS`:

```python
"<atelier_home>/**/compute/**",
"<atelier_home>/**/compute/worker.sock",
```

(The second is redundant given the first but is listed explicitly
because the socket has different semantics — write attempts to a
live socket are different from FS writes; the explicit pattern makes
the protection scrutable.)

### Tenant-config extension

```python
class ComputeConfig(BaseModel):
    model_config = ConfigDict(extra="forbid")
    enabled: bool = False
    max_parallel_iterations: int = Field(4, ge=1, le=16)
    max_concurrent_runs: int = Field(2, ge=1, le=8)
    max_iterations_per_run: int = Field(200, ge=1, le=10000)
    max_wall_clock_per_run_s: int = Field(600, ge=1, le=86400)
    top_k_size: int = Field(5, ge=1, le=10)
    disallow_llm_strategies: bool = False
    strategies_allowed: list[str] = Field(
        default_factory=lambda: ["grid", "random", "bayesian"]
    )
```

Slotted into `TenantSpec.compute: ComputeConfig | None = None`.

### Test gates

1. `test_audit_event_only_carries_allowed_details` — emitter rejects
   a details dict containing a param value; raises
   `AuditFieldNotAllowed`.
2. `test_param_fingerprint_deterministic` — same params → same
   fingerprint across two runs.
3. `test_param_fingerprint_canonical_ordering` — `{a: 1, b: 2}` and
   `{b: 2, a: 1}` produce identical fingerprints.
4. `test_sensitive_field_redaction_in_iter_log` — schema marks
   `api_endpoint` as `x-sensitive`, iter log replaces value with
   `<hash:...>`.
5. `test_non_sensitive_field_clear_in_iter_log` — value preserved
   verbatim.
6. `test_path_gate_denies_compute_write` — Bash `echo X >
   /home/u/.atelier/tenants/acme/compute/foo` denied.
7. `test_path_gate_denies_socket_redirect` — Bash `mv malicious.sock
   /home/u/.atelier/tenants/acme/compute/worker.sock` denied.
8. `test_tenant_config_compute_defaults` — load tenant without
   compute block, `spec.compute` is `None`.
9. `test_tenant_config_compute_extra_keys_forbidden` —
   `max_parallel_runs: 4` (typo for `max_parallel_iterations`)
   raises validation error.
10. `test_tenant_config_compute_clamps` — `max_parallel_iterations:
    100` raises (above clamp 16).
11. `test_strategies_allowed_default` — empty `compute` block
    inherits default `["grid", "random", "bayesian"]`.
12. `test_audit_chain_integrity_across_compute_events` —
    `verify_chain` returns `(True, [])` over a synthetic 50-iter run
    that emits start + 50 × iteration_completed + terminal.

### Acceptance

- All twelve tests pass.
- An iter log on disk contains clear-text non-sensitive params and
  hashed sensitive ones.
- An LLM-issued Bash command writing into the compute tree is
  denied by path-gate.
- A `tenant.atelier.yaml` with `spec.compute.enabled: true` is
  accepted; with `extra_field: X` is rejected.

---

## Phase 13.7 — Parallelism + parametric cache

**Goal**: turn the sequential driver into a parallel one via
`ThreadPoolExecutor`. Extend the Forge cache to honour
`x-cache-key` annotations for parametric hits.

### Files

```
atelier_compute/
└── parallel.py                                 # ParallelDriver wrapping sequential driver

operator/forge/forge/
└── cache.py                                    # MODIFIED — honours x-cache-key subset
```

### Parallel driver

```python
class ParallelDriver(ComputeRun):
    def _run_iteration_batch(self, batch: list[ParamSet]) -> list[IterRecord]:
        with ThreadPoolExecutor(max_workers=self.max_parallel) as pool:
            futures = [pool.submit(self._run_one, p) for p in batch]
            return [f.result(timeout=self.iter_timeout_s) for f in futures]
```

Each `_run_one` is a single iteration: cache lookup → tool spawn (if
miss) → result record. The bwrap subprocess is the per-iteration unit
of parallelism.

### Forge cache extension

Today's cache key (per CLAUDE.md): `sha256(tool_sha ‖
json(payload_sorted) ‖ python_tag)[:32]`.

New: tools may declare `x-cache-key: true` per field. When ANY field
has the annotation, the cache key becomes:

```python
key_payload = {k: v for k, v in payload.items()
               if schema.properties[k].get("x-cache-key")}
sha256(tool_sha ‖ json(key_payload, sorted) ‖ python_tag)[:32]
```

If NO field has the annotation, the existing behaviour applies (full
payload). This is back-compat — existing tools see no change.

### Test gates

1. `test_parallel_driver_4_workers_4x_speedup_floor` — stub runner
   sleeps 100 ms, batch of 8, 4 workers → wall ≤ 350 ms (parallel)
   vs 800 ms (serial); ratio ≥ 2× (allow for thread overhead).
2. `test_parallel_preserves_iteration_order` — batches in order,
   even if individual futures complete out of order.
3. `test_iteration_timeout_terminates_run` — stub sleeps 10 s,
   `iter_timeout_s=1`, run terminates with `state: "failed"`,
   `error_class: "IterationTimeout"`.
4. `test_parametric_cache_hit_on_partial_key` — tool with `window`
   (cache-key) and `_artifacts_dir` (not), two calls with same
   `window` but different artifacts dir → second is cache hit.
5. `test_parametric_cache_miss_on_key_field_change` — change
   `window` value → miss.
6. `test_parametric_cache_backcompat` — tool with no `x-cache-key`
   annotations behaves identically to pre-13.7 cache.
7. `test_strategy_batch_size_respects_max_parallel` —
   `max_parallel_iterations=4`, `should_stop` not triggered → every
   `suggest_batch(history, n)` call receives `n=4`.

### Acceptance

- All seven tests pass.
- Benchmark in `test_parallel_driver_4_workers_4x_speedup_floor`
  demonstrates real parallelism gain.
- Existing Forge tools without `x-cache-key` annotations exhibit
  byte-identical cache behaviour to pre-13.7.

---

## Phase 13.8 — Bayesian strategy (sklearn q-EI)

**Goal**: implement the third bundled strategy. Sklearn
`GaussianProcessRegressor` with q-Expected Improvement acquisition
for batch suggestions. Lands as a Skill-Forge skill.

### Files

```
atelier_compute/strategies/
└── bayesian.py                                 # GP + q-EI

core/compute/
└── requirements.txt                            # +scikit-learn>=1.4, +numpy>=1.26

operator/skill-forge/skills/dyn/compute_strategy_bayesian/
└── SKILL.md                                    # promoted to user-scope at install
```

### Strategy details

- **Seeding**: first `2 * n_axes` iterations use random sampling
  (warm-up).
- **GP kernel**: Matérn-5/2 default; configurable via strategy params
  passed through `compute_run.params.strategy_config.kernel`.
- **Acquisition**: Expected Improvement; batch via constant-liar
  (CL-min heuristic) — quick, deterministic, well-documented.
- **Numerical stability**: normalise observed losses to [0, 1] before
  GP fit; map suggestions back.
- **Strategy CPU budget**: `suggest_batch` measured against a
  per-call 5 s wall budget; exceed → terminate run with `state:
  "failed"`, `error_class: "StrategyTimeout"`. The budget
  protects against GP-fit explosion at high iteration counts.

### Bootstrap installation

`bootstrap.sh` (from 13.1) is extended:

```bash
if [[ "${ATELIER_COMPUTE_MINIMAL:-0}" == "1" ]]; then
    pip install -r requirements-minimal.txt
else
    pip install -r requirements.txt
fi
```

`requirements-minimal.txt` is the empty 13.1 file; `requirements.txt`
gains sklearn + numpy. Operators on disk-constrained hosts opt out of
Bayesian via `ATELIER_COMPUTE_MINIMAL=1 bash bootstrap.sh`.

### Test gates

1. `test_bayesian_warmup_uses_random` — first `2 * n_axes` iters
   match seeded-random samples.
2. `test_bayesian_post_warmup_uses_gp` — iter `2*n_axes + 1`
   onwards, internal `_used_gp` flag set per suggestion.
3. `test_bayesian_qei_batch_size` — batch=4 returns 4 distinct
   points.
4. `test_bayesian_converges_on_synthetic_function` — known quadratic
   `loss = (x - 0.7)^2 + (y - 0.3)^2`, 30 iters Bayesian beats 30
   iters random by ≥ 30% on best-loss (statistical test, fixed
   seed).
5. `test_bayesian_strategy_timeout` — patch GP fit to sleep 10 s,
   strategy budget 1 s, run fails with `StrategyTimeout`.
6. `test_skill_file_at_user_scope` — after install,
   `operator/skill-forge/skills/dyn/compute_strategy_bayesian/
   SKILL.md` exists and is grade-marked at user scope.
7. `test_minimal_bootstrap_skips_bayesian` —
   `ATELIER_COMPUTE_MINIMAL=1 bash bootstrap.sh` produces a venv
   without sklearn; `import bayesian` raises a clean
   `ModuleNotFoundError` with operator-readable hint.
8. `test_strategies_allowed_blocks_bayesian` — tenant config
   `strategies_allowed: ["grid", "random"]`, submit run with
   `strategy: "bayesian"` → server rejects with `error_class:
   "StrategyNotAllowed"`.

### Acceptance

- All eight tests pass.
- Bayesian strategy outperforms random on the synthetic-quadratic
  benchmark.
- Operator can install without sklearn via `ATELIER_COMPUTE_MINIMAL=1`.
- `strategies_allowed` allowlist enforced.

---

## Phase 13.9 — Crash recovery + systemd-user template

**Goal**: a worker that died mid-run resumes on next boot. systemd-
user unit template makes restart-on-crash an operator-managed
concern.

### Files

```
atelier_compute/
└── recovery.py                                 # scan compute/runs/, resume non-terminal

core/compute/systemd/
├── atelier-compute@.service                    # systemd-user template unit
└── README.md                                   # operator setup recipe
```

### Recovery semantics

On worker startup:

1. Scan `<atelier_home>/tenants/<tid>/compute/runs/*/manifest.json`.
2. For each run whose `summary.json::state` is `queued` or `running`:
   - Mark state `recovering` (audit: `compute.run_recovering`).
   - Load history from `iterations/*.json` (sorted by iter number).
   - Reconstruct strategy state via `strategy.update(history,
     history)` (idempotent contract).
   - Continue from `iter = max(history) + 1`.
3. Runs whose strategy state cannot be reconstructed (e.g. strategy
   no longer installed, schema migration) → mark `failed` with
   `error_class: "RecoveryFailed"`.

### systemd-user template

```ini
[Unit]
Description=AtelierOS Compute Worker (tenant %i)
After=default.target

[Service]
Type=simple
ExecStart=%h/.local/bin/atelier-compute serve --tenant %i --atelier-home %h/.atelier
Restart=on-failure
RestartSec=5s
PrivateTmp=yes
NoNewPrivileges=yes

[Install]
WantedBy=default.target
```

Enable per tenant: `systemctl --user enable atelier-compute@acme`.

### Test gates

1. `test_recovery_resumes_interrupted_run` — write a manifest +
   half the iter logs + non-terminal summary, boot worker, run
   completes from the next iter.
2. `test_recovery_with_terminal_run_ignored` — terminal-state runs
   are not resumed.
3. `test_recovery_with_missing_strategy_fails_cleanly` — manifest
   declares `strategy: "neural_arch_search"` not installed → marks
   run failed.
4. `test_recovery_strategy_update_idempotent` — same history
   replayed through `strategy.update` produces same internal state
   (grid index, random RNG seed, Bayesian GP posterior).
5. `test_systemd_unit_template_renders` — manual smoke (skipped
   in CI without systemd) — `systemctl --user start
   atelier-compute@test` launches a worker.
6. `test_recovery_audit_chain_event` — `compute.run_recovering`
   event lands in chain on resume.

### Acceptance

- All six tests pass (5 ran, 1 skipped without systemd).
- Manual operator smoke: kill worker mid-run, restart, run
  completes correctly.

---

## Phase 13.10 — E2E with real timeseries + closure

**Goal**: prove the full stack works end-to-end against a realistic
workload. Update `CLAUDE.md` Layer-section. Push to main.

### Hard prerequisite

ADR-0012 (Large-Data Snapshot Layer) must be ≥ Phase 12.5
(`data_register` + `data_snapshot` MCP tools available). Without
those, the E2E can't pass a `data_handle` to the tool.

### E2E test surface

A synthetic 1-million-row OHLCV CSV (deterministic seed), a Forge
tool `compute_rolling_sharpe` declared with `x-data: large_dataset`
on the input + `x-sensitive: true` on a fake `api_endpoint` field +
`x-cache-key: true` on the `window` field. The test:

1. Registers the CSV via `data_register` (Layer 0012).
2. Forges `compute_rolling_sharpe` via existing forge MCP tools.
3. Submits a Bayesian `compute_run` with `param_grid` = {`window`:
   range(5, 200), `method`: ["std", "downside"]}, budget 80 iters.
4. Polls `compute_status` 5 times during execution; asserts Top-K
   carries fingerprints (not values).
5. Calls `compute_result`; asserts converged within budget, best
   `window` in `[20, 80]` (sanity check on synthetic data).
6. Inspects `<run_id>/iterations/0042.json`; asserts `window` is
   clear-text integer, `api_endpoint` is hashed.
7. Inspects audit chain; asserts every `iteration_completed` event
   carries `param_fingerprint` and no clear params.
8. Asserts `verify_chain` returns `(True, [])` across the full run.

### Closure narrative

`CLAUDE.md` gains a section titled **"Layer 24 — Compute Worker
(opt-in iterative big-data)"** at the end of the structural-layers
sequence (after L23 STT). The section follows the existing template
shape: what was added, what files, what audit events, the "What you,
as Claude Code, must NOT do" list, and references.

The bot-disclosure card (L19) is updated to mention that some tenants
may have additional compute capabilities — a one-line addition,
gated on the tenant's `spec.compute.enabled`.

### Acceptance

- E2E test passes against a real worker + real Forge MCP server +
  real `data_register` snapshot.
- Wall-clock: 80 iters Bayesian × 5 s tool wall, 4 parallel → ≤ 130 s
  total (allow generous slack for cold start + GP fit).
- `CLAUDE.md` Layer-24 section drafted, reviewed, committed.
- `run-all-tests.sh` count goes from current N to N + (compute-suite
  count); all green.
- ADR-0013 status flips from `Proposed` to `Accepted` in this
  commit.

---

## Cross-cutting concerns

### Test isolation

Every compute test that touches `<atelier_home>` MUST sandbox via
`tempfile.mkdtemp(prefix="atelier-compute-test-")` and override
`ATELIER_HOME` for the duration. Mirror of the existing forge /
skill-forge test pattern (CLAUDE.md "Test isolation" guidance).

### Audit chain integrity

Every phase that adds audit events MUST include a
`verify_chain` round-trip case in its E2E gates. Phase 13.6's case
12 is the template; 13.7 (parametric cache events if added), 13.8
(strategy events), 13.9 (recovery event) repeat the pattern.

### Performance regression guards

Phases 13.2, 13.7, 13.8 ship benchmarks (with generous slack to
avoid flake) measuring wall-clock for canonical scenarios. The
benchmarks are advisory — they log timing but don't fail the suite
on slow CI. Operators reviewing PRs can see the numbers; CI
machines (which may be slow + shared) don't gate on them.

### Closure rule

No phase claims its slot in CLAUDE.md until its E2E gate is green AND
the commit lands. Drafting CLAUDE.md prose ahead of green tests is
the documented anti-pattern (`docs-as-definition-of-done` LDD skill).

### Backout strategy

If a phase ships and we discover a regression that requires backout:

- Phases 13.1 – 13.5: `git revert` works clean — the plugin is
  additive, no other module imports it.
- Phase 13.6: backout means reverting the path-gate / tenant-config /
  audit-events delta. The audit chain remains intact (the new event
  types are simply never emitted again). Existing chain entries
  with the new types stay valid.
- Phase 13.7: backout means dropping `x-cache-key` honour in the
  Forge cache. Tools that opted in keep working (their key falls
  back to full-payload, which is the pre-13.7 behaviour).
- Phase 13.8: backout means removing the bayesian module + bumping
  `strategies_allowed` default. Tenants with bayesian-only
  computations need to switch to grid or random.

No phase introduces a one-way migration. A backout at any sub-phase
is recoverable.
