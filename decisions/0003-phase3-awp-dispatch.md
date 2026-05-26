# ADR-0003 — Phase 3: AWP-Dispatch im Adapter

**Status:** ROLLED BACK on 2026-05-10 by ADR-0005. AWP-runtime is no
longer a dispatch path in AtelierOS. AWP is consumed only as a
*declarative standard* (engine_policy schema, task_complexity
classifier, engine_registry catalog, compliance zones — see ADR-0004
which survives the rollback). The original Phase-3 skeleton
(`awp_runtime.py`, `test_awp_integration.py`,
`test_adapter_awp_dispatch.py`, `_try_awp_dispatch`) was deleted in
the same change. This ADR remains as historical record of the
rejected direction. **Do not implement.** See ADR-0005.

**Original status:** Skeleton implemented, full conformance pending
**Date:** 2026-05-10
**Builds on:** ADR-0001 (AWP-Adoption), ADR-0002 (Engine-Migration)
**Time estimate:** Skeleton 2-3 h; full conformance 8-12 h (separate session)

## Architecture invariant (load-bearing)

AWP is the **orchestration layer** — DAG, Delegation-Loop, 5D-Budget,
Worker-Routing. It is *not* the LLM-execution layer. Every actual LLM
call flows through a `WorkerEngine` adapter (`ClaudeCodeEngine`,
`CodexCliEngine`, future `GeminiCliEngine` / `OllamaEngine` / `VLLMEngine`).

When the Phase-3 adapter wiring lands, `awp_runtime.dispatch()` must
hand AWP a **worker-engine factory**, not an LLM API. AWP-workers
invoke the engine themselves when they need an LLM step. This honors
the three-layer hexagonal split from ADR-0001:

  Body (AtelierOS) ─── Brain (AWP) ─── Engines (LLM-CLIs / SDKs)

Integrity check after wiring: every audit event with `engine_id` set
has an enclosing `awp_task_id`, and every `awp_task_id` has at least
one event with `engine_id`. If either side is empty, AWP made a direct
LLM call OR an engine ran outside AWP — both are bugs at this level.

## Goal

Add **opt-in AWP-DAG-Dispatch** in `bridges/shared/adapter.py::process_one`.
For complex multi-step tasks where a per-chat profile (or a global env-var)
opts in, the adapter routes through `awp.AWPAgent` instead of a single
`call_claude_streaming` turn. AWP itself reaches AtelierOS' Forge,
SkillForge, Personas, MCP and Sandbox through the existing surfaces — the
hexagonal-architecture promise from ADR-0001 holds.

## Non-goals (this phase)

- Removing the legacy single-turn path. Default stays single-turn; AWP is
  opt-in per chat or per env-var.
- Multi-engine routing per AWP worker (Phase 4).
- Compliance-zone routing (Phase 5).
- Passing the full AWP conformance suite. Skeleton aims at *one* end-to-end
  smoke that proves the wiring; full suite is a follow-up session.

## Sub-phases

K_MAX = 5 per sub-phase, each ships its own per-subtask E2E.

### **3a — Optional-import wrapper (`awp_runtime.py`)**

A thin module that:
- tries to `import awp` (the actual module name; PyPI package ships as
  `awp-agents 1.0.56`)
- exposes `is_available() → bool` so the adapter can capability-gate
- exposes `dispatch(task, state, *, persona, channel, chat_key,
  on_chunk) → dict | None` as the *single* entry point. Failure is
  always graceful — never raises into the adapter.

E2E: import-success path (anaconda PYTHONPATH includes the AWP src dir)
+ import-failure path (PYTHONPATH cleared) + dispatch-success +
dispatch-failure + dispatch-empty-task.

### **3b — Task-complexity classifier (`task_complexity.py`)**

Pure-Python heuristic:
- multi-step verbal markers ("erst …, dann …", "first …, then …",
  enumerated lists "1. ... 2. ...")
- DAG / pipeline / parallel keywords
- length-based hint (>500 chars + ≥3 verbs)
- conservative: false-negative is fine (assistant fallback handles), false-
  positive is bad (would route trivial tasks through the heavier AWP loop)

E2E: covers ~20 phrases — simple commands, multi-step builds, parallel
analysis requests, edge cases (long but trivial, short but multi-step).

### **3c — Adapter dispatch hook**

In `process_one`, after persona-resolution, before
`call_claude_streaming`:

1. Check feature gate:
   - `chat_profile.awp_enabled is True` (per-chat opt-in), OR
   - env `ATELIER_AWP_DEFAULT=on` (global opt-in)
2. Check capability: `awp_runtime.is_available()` AND
   `task_complexity.is_complex(prompt)`.
3. If both pass: call `awp_runtime.dispatch(...)`, stream chunks via
   the existing `on_chunk` callback, fall back to `call_claude_streaming`
   on dispatch failure.
4. Audit: emit `awp.task_dispatched` (or `awp.task_declined` with reason)
   into the unified hash chain.

The dispatch never widens permissions — AWP workers reach AtelierOS
through existing layers; the path-gate, secret-vault, sandbox and audit
chain remain authoritative.

### **3d — Per-subtask E2E**

`test_awp_integration.py`:
- module surface (`is_available`, `dispatch`, `is_complex_task`)
- dispatch with stub AWPAgent (monkeypatched) — round-trip
- dispatch path declines when classifier says "not complex"
- dispatch path declines when feature gate is off
- dispatch path declines gracefully when AWP module is missing
- audit emission `awp.task_dispatched` lands in the hash chain

### **3e — Run-all-tests integration + soak**

Add the new test to `run-all-tests.sh`. Run a soak under
`ATELIER_AWP_DEFAULT=off` to confirm zero behavioural drift on the legacy
path; then a smoke under `=on` to confirm dispatch fires when the
classifier triggers.

## Phase 4 — Multi-engine per worker (NEXT after Phase 3 conformance)

YAML workflow declares `engine: codex_cli` per worker step. Persona has
`default_engine: claude_code` field. Adapter resolves the engine per
dispatch, passes it to `awp_runtime.dispatch(engine_id=...)`.

Sub-phases:
- 4a — `engine_registry.py` module that maps `engine_id → ClaudeCodeEngine
  | CodexCliEngine` instance
- 4b — Persona-field `default_engine` propagation through cowork resolver
- 4c — YAML-workflow `engine:` field honoured by `awp_runtime.dispatch`
- 4d — Per-subtask E2E with both engines in one workflow

## Phase 5 — Enterprise compliance + zone routing (final phase)

`engine_policy.json` per workspace:

```jsonc
{
  "default_engine": "claude_code",
  "fallback_chain": ["claude_code", "vllm_eu_west", "ollama_local"],
  "compliance_zones": {
    "personal_data": {"allow_engines": ["azure_openai_eu", "vllm_eu_west"]},
    "code_only":     {"allow_engines": ["claude_code", "codex_cli"]}
  },
  "task_zone_classifier": "regex_pii"
}
```

Sub-phases:
- 5a — `engine_policy.py` schema + parser + validator
- 5b — `compliance_zone_classifier.py` (PII regex, scope hints from
  persona)
- 5c — Adapter routes through policy: per-task zone-classify → engine
  selection → fallback-on-failure
- 5d — Audit-Tag `engine_id` + `compliance_zone` per event in the hash
  chain
- 5e — Per-subtask E2E with deliberate engine outage triggering fallback

This is the **enterprise-adoption threshold** — the moment a regulated
customer can sign the contract without negotiating around the data path.

## What ships in this PR (Phase 3 skeleton)

- `bridges/shared/awp_runtime.py` — optional-import wrapper, `dispatch()`
  entry point (sync + state-passthrough)
- `bridges/shared/task_complexity.py` — heuristic classifier
- `bridges/shared/test_awp_integration.py` — per-subtask E2E
- `run-all-tests.sh` — new suite wired in
- `docs/decisions/0003-phase3-awp-dispatch.md` — this ADR

**Not in this PR (deliberately):**
- Adapter-side dispatch wiring (would be the actual cut-over; needs
  full conformance review first to avoid behavioural drift)
- AWP-conformance-suite execution
- AWP-worker-side reach into Forge / SkillForge / MCP (those work today
  *via subprocess* — Phase 3 doesn't change that contract)

The skeleton is honest: importable, testable, plug-and-play for the
adapter wiring in the next session. Until that wiring lands,
`awp_runtime.dispatch()` is reachable only via direct call (e.g. from
a forged tool or a user script), not from the bridge automatically.
