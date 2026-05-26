# Phase 7 — hard-cut backlog (claudeOS → AtelierOS rebrand)

Status snapshot at v0.13.0 release (2026-05-10). Phase 7 is the
strangler-fig endgame: every `CLAUDEOS_*` alias resolver, every
`~/.claudeOS/` fallback path, and every "formerly claudeOS"
footnote gets removed. Triggers the v1.0 major-version bump.

See `CLAUDE.md` "Project rebrand" section for the full roadmap and
inventory snapshot trail. This file lists what specifically still
blocks Phase 7.

## Trigger gates (all three must hold)

- [ ] Zero `CLAUDEOS_*` env-var reads in the operator's `service.env`
      for ≥ 30 days (currently fail-open compat resolver still active).
- [ ] Zero `claudeos`-substring matches in user-facing docs (currently
      455 files contain the substring; many are legitimate doc anchors,
      a final sweep needs to distinguish historical from live).
- [ ] No `~/.claudeOS/` directory left on the operator's box (currently
      coexists with `~/.atelier/` — see Phase 6 below).
- [ ] `voice-audit verify` clean across the unified hash chain.

## Phase 7-1 — DONE (commit `ede70a6`, 2026-05-09)

- [x] `secret_vault.py` legacy `CLAUDEOS_SECRET_VAULT` env alias
      dropped. `ATELIER_SECRET_VAULT` is now the only spelling.

## Phase 7-2 through 7-N — env-var alias removal (per-resolver)

Six remaining canonical aliases. Each removal is one PR/commit:
delete the legacy spelling from the resolver, drop the once-per-process
`[deprecation]` log line, remove the legacy E2E case, run
`bash operator/bridges/run-all-tests.sh`.

| Env var to remove        | Resolver location                                       | Status |
|--------------------------|---------------------------------------------------------|--------|
| `CLAUDEOS_HOME`          | `operator/forge/forge/paths.py` (3 byte-identical copies)| open   |
| `CLAUDEOS_FORCE_SCOPE`   | `operator/forge/forge/scope.py`                          | open   |
| `CLAUDEOS_DEFAULT_SCOPE` | `operator/forge/forge/scope.py`                          | open   |
| `CLAUDEOS_CHANNEL_ID`    | `operator/forge/forge/scope.py` + `adapter._build_spawn_env` | open |
| `CLAUDEOS_TASK_ID`       | `operator/forge/forge/scope.py`                          | open   |
| `CLAUDEOS_CALLER_PERSONA`| `operator/bridges/shared/adapter.py::_build_spawn_env` | open |
| `CLAUDEOS_PLUGIN_SLOT_DIR` | `operator/skill-forge/skill_forge/registry.py::plugin_slot_dir` | open |
| `CLAUDEOS_SESSION_TTL_DAYS` | `operator/voice/scripts/session_timeout_sweep.py`     | open   |
| `CLAUDEOS_AGENTS_SKIP_LIVE` | `operator/bridges/shared/agents/test_engines_e2e.py` | open |
| `CLAUDEOS_USE_ENGINE_LAYER` | `operator/bridges/shared/adapter.py`            | open   |
| `CLAUDEOS_INIT_NO_AUTOSTART` | `core/init/init.py`                      | open   |
| `CLAUDEOS_INIT_TICK_INTERVAL`| `core/init/init.py`                      | open   |

Current production-code count (excluding `test_*` files):
- `ATELIER_*` references: 78
- `CLAUDEOS_*` references: 97 (legacy still dominates)

Pre-flight before removing each: grep the operator's `service.env`
for the legacy spelling. If present → flip the user's env file FIRST,
then remove the resolver in the next release cycle.

## Phase 7-3 — `~/.claudeOS/` fallback path removal

Fallback chains in path resolvers (`paths.py` × 3, `scope.py`,
`registry.py`, plus `notification_relay.py`, `path_gate.py`,
`auth_elevation_gate.py`) currently try `~/.atelier/` first and fall
back to `~/.claudeOS/`. Phase 7 removes the fallback.

Pre-flight: confirm `~/.claudeOS/` is empty / migrated on every operator
box. The Phase-4 `atelier_migrate.py` helper does the move; verify via
`MIGRATED` marker presence in the legacy directory.

On this dev box (2026-05-10): both `~/.claudeOS/` and `~/.atelier/`
coexist. Migration helper needs one boot to consolidate.

## Phase 7-4 — systemd unit cleanup

Currently enabled on operator boxes:
- `claudeos-audit-verify.timer` (enabled)
- `claudeos-session-timeout.timer` (enabled)

Phase 4 added `atelier-audit-verify.timer` and
`atelier-session-timeout.timer` as siblings. `bridge.sh up` enables
all four; Phase 7 drops the `claudeos-*` units after a soak period
where atelier-* fired cleanly.

Verify on this box (2026-05-10): atelier-* units NOT yet enabled —
either bridge hasn't restarted since Phase 4 landed, or `bridge.sh up`
unit-enabling needs to be re-run.

## Phase 7-5 — `[CLAUDEOS_SIGNAL: <name>]` magic-prefix marker

**Status: DONE 2026-05-21** — adapter.py CLAUDEOS_SIGNAL dual-fire removed; no personas found using the legacy marker.

`adapter.py::_signal` handler dual-fires
`[ATELIER_SIGNAL: <name>] [CLAUDEOS_SIGNAL: <name>]` on the same line
so persona `append_system` blocks that grep for either form keep
working. Phase 7 drops the `CLAUDEOS_SIGNAL:` half.

Pre-flight: grep every `personas/*.json::append_system` for the
legacy marker. Migrate any persona still using the old spelling.

## Phase 7-6 — docs / READMEs / ADRs final sweep

Phase 3 rewrote prose mentions but left historical anchors. Phase 7
needs:
- A final `grep -ri claudeos docs/` audit, distinguishing historical
  ADR notes ("predates rebrand") from live references.
- README header `AtelierOS (formerly claudeOS)` simplification —
  drop the parenthetical once `claudeos` substring count is at zero
  in non-historical docs.
- `CLAUDE.md` "Project rebrand" section: shrink to a one-line
  historical anchor pointing at the v1.0 changelog entry.
- SVG diagram re-renders (`docs/diagrams/*.svg`, `assets/banner.svg`)
  — explicitly deferred from Phase 3 because re-rendering needs the
  source-of-truth (Mermaid? hand-edited?). Separate session.

Current count (2026-05-10): 455 files contain the `claudeos`
substring across `*.py`, `*.js`, `*.sh`, `*.md`, `*.json`. Most are
test files (legacy E2E coverage), legitimate path strings inside
fallback resolvers, or doc anchors. The non-fallback non-test count
is the meaningful number — needs a precise grep recipe.

## Phase 7-7 — repo-folder rename (Phase 6 in the original roadmap)

Operator action, not a Claude Code commit. Status as of v0.13.0:

- [x] **GitHub repo renamed** — `veegee82/claudeOS` →
      `veegee82/AtelierOS` (push to old URL still works via redirect;
      remote prints "This repository moved. Please use the new
      location: https://github.com/veegee82/AtelierOS.git").
- [x] **README H1 renamed** — commit `c4cbb70` flipped `# Atelier-OS`
      → `# AtelierOS` (no hyphen). Phase 7-9 below resolves the
      tree-wide alignment.
- [ ] `git remote set-url origin https://github.com/veegee82/AtelierOS.git`
      (currently still pointing at the redirect URL).
- [ ] `mv /home/shumway/projects/claudeOS /home/shumway/projects/atelier`
- [ ] Update `~/.claude/CLAUDE.md` paths.
- [ ] Update IDE bookmarks.
- [ ] Update `bridge.sh` symlinks if any.
- [ ] Update systemd unit `WorkingDirectory=` paths in
      `operator/bridges/setup.sh`-managed units.

Strangler-fig sequence: Phase 6 must run AFTER Phase 4 (data
migration) and BEFORE Phase 7-3 (fallback path removal) — otherwise
a renamed repo points at unmoved data.

## Phase 7-9 — DONE (commit lands with v0.13.x patch, 2026-05-10)

Naming spelling resolved: **`AtelierOS`** (one word, no hyphen) wins,
matching the GitHub repo name and README H1 from commit `c4cbb70`.
Tree-wide sweep replaced 148 occurrences of `Atelier-OS` →
`AtelierOS` across 34 files (CLAUDE.md, all `docs/*.md`, ADRs,
README, source comments, persona JSON, bridge scripts).

The `atelier-init` / `atelier-pipe` plugin folder names, the
`~/.atelier/` data directory, the `ATELIER_*` env vars, and the
`atelier-audit-verify` / `atelier-session-timeout` systemd units
keep their hyphenated technical-identifier spelling — only the
brand word "AtelierOS" loses the hyphen. (Hyphenation is the Unix
convention for filesystem and shell identifiers; CamelCase is the
brand-word convention.)

## Phase 7-8 — final cut commit + v1.0 bump

After 7-2 through 7-7 are clean:
- One sweep commit removing every remaining `CLAUDEOS_*` literal.
- One CHANGELOG entry "Phase 7 hard cut, claudeOS → AtelierOS
  rebrand complete."
- Bump version to `v1.0.0`.
- The string `claudeos` should only survive in git history and in
  the v1.0 CHANGELOG entry.

## Inventory snapshot to update at each Phase-7 step

| Date | Step | files w/ `CLAUDEOS_*` | files w/ `claudeos` substring | `~/.claudeOS/` exists | claudeos-* timers enabled |
|------|------|-----------------------|-------------------------------|----------------------|---------------------------|
| 2026-05-10 | v0.13.0 release | 89 | 455 | yes | yes |
| 2026-05-21 | Phase 7 partial (ADR-0046 T2) | TBD after grep | TBD | yes (migrated) | claudeos-* units removed |
| _next_ | _Phase 7-2 step_ | _ | _ | _ | _ |
