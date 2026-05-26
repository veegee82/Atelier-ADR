# ADR-0002 — Phase 2 Migration: `adapter.py` → `WorkerEngine` Protocol

**Status:** Implemented (sub-phases 2.1 – 2.4); 2.5 pending 14-day soak
**Date:** 2026-05-09
**Builds on:** ADR-0001
**Time estimate:** 5–8 hours focused work; best done in one fresh session

> **Naming note (added Phase 3 of the AtelierOS rebrand):** This ADR
> predates the project rename from `claudeOS` to `AtelierOS`. The
> "Phase 2" referenced here is the AWP/engine-migration Phase 2,
> distinct from the rebrand's Phase 2 (internal-callsite migration).
> The `CLAUDEOS_USE_ENGINE_LAYER` env var is now `ATELIER_USE_ENGINE_LAYER`
> with the legacy spelling kept as a Phase-1 alias until rebrand-Phase 7.
> See CLAUDE.md "Project rebrand" section for the strangler-fig plan.

**Implementation log:**

* **2026-05-09** — sub-phase 2.1 shipped. Commit
  `feat(awp-phase2.1): ClaudeCodeEngine feature-complete + adapter argv
  delegation`. `ClaudeCodeEngine._build_args` is the single source of
  truth for claude argv shape; the adapter's `_build_claude_args` is a
  thin wrapper. 36 engine tests green, 76 bridge suites green.
* **2026-05-09** — sub-phase 2.2 shipped. Commit
  `feat(awp-phase2.2): adapter env-flagged engine streaming path`. New
  `_call_claude_streaming_via_engine` mirrors legacy behaviour 1:1;
  opt-in via `CLAUDEOS_USE_ENGINE_LAYER=1`. Engine `_normalise_all`
  emits `tool_call` events; `_iter_stream` no longer breaks on
  turn_completed. `test_adapter_engine_path.py` adds 3 cases. 77 bridge
  suites green in both modes.
* **2026-05-09** — sub-phase 2.3 shipped. Commit
  `feat(awp-phase2.3): /btw mid-stream injection through engine.inject()`.
  `_running_engines` registry routes `inject_btw` through
  `engine.inject()`. 4th E2E case in `test_adapter_engine_path.py`
  verifies engine.inject was the actual write path. 77 suites green
  in both modes.
* **2026-05-09** — sub-phase 2.4 shipped. Commit
  `feat(awp-phase2.4): flip CLAUDEOS_USE_ENGINE_LAYER default to ON`.
  Default flipped from `"0"` to `"1"`; `=0` is the explicit opt-out
  knob for the 14-day soak. CLAUDE.md Layer 22 section reflects
  engine-as-default; this ADR marked Implemented for sub-phases 2.1–
  2.4. 77 suites green at the new default AND with explicit opt-out.

## Goal

Move `bridges/shared/adapter.py::call_claude_streaming` from direct
`claude` binary spawn onto the `WorkerEngine` protocol established in
Phase 1. **100 % behavior compatibility** is the load-bearing
requirement: every existing `test_adapter_*.py` must stay green
across the migration.

After Phase 2, `call_claude_streaming` is a thin orchestration shell
around `ClaudeCodeEngine`; switching `engine = ClaudeCodeEngine()` to
`engine = CodexCliEngine()` (or any future backend) becomes a one-line
adapter change.

## Non-goals

- AWP integration itself (Phase 3)
- Multi-engine routing per worker (Phase 4)
- Compliance-zone routing (Phase 5)
- Removing `ADAPTER_FAKE_CLAUDE` test fixture (must continue to work)

## Sub-phases

Each sub-phase ships with its own per-subtask E2E and is committed
separately. K_MAX = 5 iterations per sub-phase.

### **2.1 — `ClaudeCodeEngine` feature-complete**

Extend `ClaudeCodeEngine.spawn()` to cover the full adapter spawn
surface: persona, profile, working_dir, system prompt, channel/chat env
vars, MCP config materialization, `--add-dir` list, `--permission-mode`,
`--dangerously-skip-permissions`. The engine becomes the single owner
of *building the claude argv* — adapter just hands it inputs.

**Scope**:
- Extend `ClaudeCodeEngine.spawn()` signature: add `mcp_config`,
  `permission_mode`, `add_dirs`, `persona`, `channel`, `chat_key`,
  `dangerously_skip_permissions`.
- Extract `_build_claude_args()` from `adapter.py` into `claude_code.py`
  as a static method `ClaudeCodeEngine._build_args()`.
- Engine respects `ADAPTER_FAKE_CLAUDE=1` for the test-fixture path.

**E2E**:
- `test_engines_e2e.py` extended with a "feature-complete" case:
  spawn with persona + system prompt + working_dir + add_dirs +
  permission_mode, verify all flags arrive at the spawned process
  via the existing `ADAPTER_FAKE_ARGS_DUMP` mechanism.
- Existing 22 tests stay green.

**Risk**: argv shape drift between engine and existing
`_build_claude_args`. **Mitigation**: golden-snapshot test that
diffs the two argv lists for several scenarios before any adapter
change.

### **2.2 — Adapter optional engine path (env-flagged, opt-in)**

Add a code path in `call_claude_streaming` that delegates to the engine
when `CLAUDEOS_USE_ENGINE_LAYER=1`. Default OFF — production behavior
unchanged.

**Scope**:
- Adapter does budget preflight + session-dir + spawn-env (unchanged).
- Then either: (a) legacy direct-spawn path, or (b)
  `engine = ClaudeCodeEngine(); collect(engine.spawn(...))` based on
  env flag.
- On-status callback shim: legacy path called `on_status(...)` per
  tool_use event; engine path needs a `for ev in engine.spawn():`
  loop that fires `on_status` on `text_delta` boundaries.

**E2E**:
- Run **the entire** `test_adapter_*.py` suite **twice**: once with
  `CLAUDEOS_USE_ENGINE_LAYER=0` (legacy), once with `=1` (engine).
  Both must pass identically.
- Add a new `test_adapter_engine_path.py` with three cases: simple
  prompt, prompt with tool_use, prompt with mid-stream cancel.

**Risk**: divergent timing between legacy and engine paths breaks
`test_adapter_progress.py`'s dedup logic. **Mitigation**: profile
both paths against the same fixture before flipping defaults.

### **2.3 — Mid-stream `/btw` injection through the engine**

The trickiest single feature. Engine must own the stdin pipe;
adapter delegates injection.

**Scope**:
- `ClaudeCodeEngine.inject(text)` writes a stream-json `user` message
  to the live process stdin. Engine maintains the stdin pipe lifecycle
  (open at spawn, close on first `result` event, EOF/raise on inject
  after close).
- Adapter's `inject_btw(chat_key, text)` resolves to
  `_running_engines[chat_key].inject(text)` (parallel to the existing
  `_running_stdins[chat_key]`).
- The race window between `inject()` and `cancel()` stays the same:
  engine's internal lock guards both calls.

**E2E**:
- Existing `test_adapter_btw.py` runs under both
  `CLAUDEOS_USE_ENGINE_LAYER=0` and `=1`. Both must pass.
- New live test: spawn real `claude`, inject via engine path mid-stream,
  verify the second reply lands in `final_text`.

**Risk**: stdin closure races with cancel. **Mitigation**: re-use the
existing `_running_stdins_guard` pattern; the engine becomes the new
owner of that lock.

### **2.4 — Flip default to engine path**

One-line change: `CLAUDEOS_USE_ENGINE_LAYER` default goes from `"0"`
to `"1"`. Legacy path opt-out via `=0` for emergency rollback.

**Scope**: env-default flip + CLAUDE.md note.

**E2E**: full `run-all-tests.sh` + manual bridge smoke test (one
WhatsApp/Discord turn each).

**Risk**: production regression on a path that wasn't covered by
tests. **Mitigation**: leave the env-flag opt-out alive; if anything
breaks in production, set `=0` as immediate rollback while debugging.

### **2.5 — Remove legacy direct-spawn path**

After N days of green default-on operation (suggested N = 14 days,
operator's call), delete the legacy code body. `call_claude_streaming`
becomes a thin engine wrapper.

**Scope**:
- Delete the legacy direct-spawn block from `call_claude_streaming`.
- Delete `_running_stdins` (now lives in engine).
- Delete `CLAUDEOS_USE_ENGINE_LAYER` env-flag handling — engine path
  is the only path.

**E2E**: full `run-all-tests.sh` plus a 24h soak on a non-production
bridge.

**Backout**: revert the deletion commit; legacy code is in git
history. The 14-day window before 2.5 is specifically to give that
backout option time to be useful.

## Migration order rationale

1. **2.1 first** because the engine MUST be feature-complete before
   the adapter can plausibly use it. Any missing capability and the
   adapter falls back to legacy at runtime — confusing test signals.
2. **2.2 next** — keep both paths alive in parallel. This is the
   "long greenfield" period where the test matrix doubles but
   regressions get caught while the legacy path is still authoritative.
3. **2.3 isolates the trickiest feature** so its bugs don't masquerade
   as general adapter regressions.
4. **2.4 flip is a one-line change** once everything else is green.
5. **2.5 cleanup is the last act** after the new path has earned trust
   in production.

## Risk register

| Risk | Mitigation |
|---|---|
| `/btw` mid-stream injection regresses | Dedicated regression test in 2.3 + golden-output diff for inject sequences |
| Voice-mode Stop-Hook stops working | Adapter spawn-env passes `VOICE_HOOK_RECURSION` and friends; engine.spawn() must merge `env=` with `os.environ` correctly. Add explicit env-passthrough test. |
| Skill auto-grade snapshot timing changes | Snapshot is set in adapter post-spawn (not in engine); timing should be equivalent. Verify in 2.2 against `test_skill_outcome_grading.py`. |
| `ADAPTER_FAKE_CLAUDE` test fixture stops working | Engine respects same env var for fake-mode short-circuit; covered in 2.1 E2E. |
| Stream-idle watchdog (`ADAPTER_STREAM_IDLE_TIMEOUT`) belongs in adapter, not engine | Watchdog stays in adapter loop wrapping `engine.spawn()`. Engine emits events, adapter measures gap between events. Verify via `test_adapter_stream_idle.py` under env=1. |
| Persona env-vars (`CLAUDEOS_CALLER_PERSONA`, etc.) get lost | 2.1 includes the env-passthrough explicit test. |
| Path-Gate hook stops firing | Hook registers on Claude Code subprocess regardless of who spawned it. Should be invariant. Verify with `test_path_gate.py` under env=1. |

## How this connects to LDD

- Each sub-phase is one **inner-loop** iteration with K_MAX=5.
- The full phase is one **outer-loop** step
  (`∂L/∂method = adapter architecture`).
- `docs-as-DoD` applies to every commit: CLAUDE.md "Layer 22" section
  gets updated with each sub-phase progress.
- `reproducibility-first` gate fires before any code edit:
  failing test → confirm reproduces → only then propose fix.
- `loss-backprop-lens` calibrates step size: a flaky test under
  env=1 but green under env=0 is engine-side, not adapter-side.

## How this connects to AWP

This phase is the *enabler* for AWP integration in Phase 3, not AWP
itself. Once `call_claude_streaming` is engine-routed, the AWP
manager can pick `ClaudeCodeEngine`, `CodexCliEngine`, or any other
registered engine per-worker — the adapter no longer cares.

## Definition of "Phase 2 done"

All of these must hold:

1. `run-all-tests.sh` green with `CLAUDEOS_USE_ENGINE_LAYER=1` as
   default.
2. Manual bridge smoke test green (Discord + WhatsApp + Telegram one
   turn each).
3. Legacy direct-spawn path removed (after the 14-day soak).
4. CLAUDE.md "Layer 22" section reflects engine-as-default.
5. Per-sub-phase ADR comments added to ADR-0002 (this file) marking
   each sub-phase as Implemented.

## Recommended starting cadence for the fresh session

```
1. Read ADR-0001 + ADR-0002 (this file)
2. /loop-driven-engineering — start the inner loop
3. TodoWrite the five sub-phases as discrete tasks
4. Phase 2.1 first; commit when E2E green
5. Phase 2.2 next; do NOT skip the dual-mode test matrix
6. Stop after 2.4 if you're tired — 2.5 can wait 14 days anyway
```
