# ADR-0035 — Repo Structure: Core vs. Operator Layer

**Status:** Accepted  
**Date:** 2026-05-17  
**Author:** Maintainer  
**Depends on:** ADR-0034 (operator bundle), ADR-0032 (AWPKG),
ADR-0030 (plugin system)  
**Governs:** Top-level directory layout of the AtelierOS repository  

---

## Context

### The problem: `plugins/` contains everything

The current repo has 19 plugins under `plugins/`. Some are core engine
code that operators should never touch (compute, gateway, admin, …).
Others are operator-facing runtime layers (cowork, forge, skill-forge,
voice). Everything looks the same from the outside.

```
plugins/
├── atelier-admin/          ← CORE engine (compliance UI)
├── atelier-compute/        ← CORE engine (optimizer)
├── atelier-gateway/        ← CORE engine (REST API, OIDC)
├── atelier-plugins/        ← CORE engine (plugin loader)
├── cowork/                 ← OPERATOR layer (personas, router)
├── forge/                  ← OPERATOR layer (tool generation)
├── skill-forge/            ← OPERATOR layer (skill generation)
└── voice/                  ← OPERATOR layer (bridges, adapter)
```

An operator cloning the repo sees no structural signal about what to
customise and what to leave alone. A contributor sees no signal about
blast radius. Both must read `CLAUDE.md` to understand the split.

### The current split

After ADR-0034, the conceptual layers are already defined:

| Layer | Meaning | Current location |
|---|---|---|
| **Core engine** | Powers the runtime; operator never modifies | `plugins/atelier-*`, `core/awpkg/` |
| **Runtime layer** | Operator-facing features; operator configures via bundle | `operator/cowork/`, `operator/forge/`, `operator/skill-forge/`, `operator/voice/` |
| **Operator bundle** | Everything the operator actually owns | `operator/bundle/` |
| **Runtime state** | Generated, never committed | `.atelier/`, `.ldd/`, `.claude/` |

The problem: "Core engine" and "Runtime layer" both live in `plugins/`
with no visual distinction.

---

## Decision

Reorganise the repository into **three top-level directories**:

```
atelier-os/
├── core/          ← CORE ENGINE — never operator-modified
├── operator/      ← OPERATOR LAYER — bundle + runtime plugins
└── docs/          ← unchanged
```

Plus the existing roots that stay: `.atelier/`, `.ldd/`, `.claude/`,
`LICENSE`, `README.md`, `CLAUDE.md`, `bootstrap.sh`.

---

## Target Layout

```
atelier-os/
│
├── core/                                ← Engine plugins (atelier-*)
│   ├── compute/                         ← was core/compute/
│   ├── gateway/                         ← was core/gateway/
│   ├── admin/                           ← was core/admin/
│   ├── compliance/                      ← was core/compliance/
│   ├── console/                         ← was core/console/
│   ├── delegate/                        ← was core/delegate/
│   ├── init/                            ← was core/init/
│   ├── license/                         ← was core/license/
│   ├── pipe/                            ← was core/pipe/
│   ├── plugins/                         ← was core/plugins/
│   ├── workflows/                       ← was core/workflows/
│   └── awpkg/                           ← was core/awpkg/
│
├── operator/                            ← Everything operator-owned
│   ├── bundle/                          ← was operator/bundle/ (ADR-0034)
│   │   ├── personas/
│   │   ├── skills/
│   │   ├── tools/
│   │   ├── workflows/                   ← user AWP workflows
│   │   ├── bridge-config/
│   │   ├── config-templates/
│   │   └── manifest.yaml
│   │
│   ├── bridges/                         ← was operator/bridges/
│   │   ├── shared/                      (adapter.py, shared JS modules)
│   │   ├── discord/
│   │   ├── telegram/
│   │   ├── slack/
│   │   ├── email/
│   │   └── whatsapp/
│   │
│   ├── cowork/                          ← was operator/cowork/
│   │   ├── lib/                         (resolver, router — engine pieces)
│   │   └── personas/                    (default personas — shadowed by bundle)
│   │
│   ├── forge/                           ← was operator/forge/
│   │   └── (MCP server, sandbox engine)
│   │
│   ├── skill-forge/                     ← was operator/skill-forge/
│   │   └── (MCP server, linter, grader)
│   │
│   └── voice/                           ← was operator/voice/ (non-bridge parts)
│       ├── hooks/                       (path-gate, auth hooks)
│       ├── scripts/                     (stt, tts, summarize, say)
│       └── skills/                      (built-in slash commands)
│
└── docs/                                ← unchanged
    ├── decisions/
    ├── claude-ref/
    └── ...
```

---

## Visual clarity achieved

**Before (everything in `plugins/`):**
```
plugins/
  atelier-compute/   ← touch this?
  atelier-gateway/   ← touch this?
  cowork/            ← touch this?
  forge/             ← touch this?
  voice/             ← touch this?
```

**After:**
```
core/                ← Don't touch.  Engine internals.
  compute/
  gateway/
  admin/

operator/            ← This is yours. Bundle, bridges, forge.
  bundle/
  bridges/
  cowork/
  forge/
```

One glance tells you which tree to open.

---

## What changes per layer

### `core/` rules
- Engine code only — no user-configurable content
- Import paths stay identical (Python packages don't move, only the
  containing directory moves within the repo)
- CI/CD config paths updated
- `CLAUDE.md` updated to reflect new paths
- Compat symlinks NOT needed (Python import paths are package-internal,
  not filesystem-relative)

### `operator/` rules
- Operators are expected to read and edit content here
- `operator/bundle/` (ADR-0034) is the canonical place for versioned config
- `operator/bridges/*/settings.json` = gitignored, lives in `~/.atelier/`
  after ADR-0008 Phase 8.3 — repo contains only daemon code here
- `operator/cowork/personas/` = Core defaults (shadowed by bundle install)
- New contributors know: "everything under `operator/` is where to look
  for what the operator can change"

### `docs/` unchanged
All ADRs, layer references, diagrams stay at `docs/`.

---

## Migration path

This is a **high-blast-radius rename** — do NOT rush it. Phases:

| Phase | Action | Risk |
|---|---|---|
| 0 | This ADR + directory concept sketch | None |
| 1 | CI inventory: list every file referencing `plugins/<name>` | Read-only |
| 2 | Move `plugins/atelier-*` → `core/<name>` (12 dirs) | Medium — update CI, Dockerfiles, service units |
| 3 | Move `operator/bridges` → `operator/bridges` (5 daemon dirs) | Medium — update systemd, bridge.sh |
| 4 | Move `operator/cowork` + `operator/forge` + `operator/skill-forge` → `operator/` | Medium — update import paths, test harnesses |
| 5 | Move `operator/voice/{hooks,scripts,skills}` → `operator/voice/` | Low |
| 6 | Move `operator/bundle/` → `operator/bundle/` | Low |
| 7 | Delete now-empty `plugins/` | Cleanup |
| 8 | Update `CLAUDE.md` references | Documentation |

**Pre-condition for Phase 2:** All 186 test suites green on current
structure before any rename.

**Each phase = one PR, one green CI run before merging the next.**

---

## What this ADR does NOT change

- Import paths inside Python packages (`atelier_compute`, `atelier_plugins`,
  etc.) — these are package-internal, unaffected by which parent dir they
  live in
- `.atelier/` runtime state — unchanged
- `operator/bundle/` content — already defined by ADR-0034
- `docs/` — unchanged
- `.gitignore` patterns — paths updated but semantics unchanged

---

## Operator-bundle completeness (updated from ADR-0034)

After this restructure, `operator/bundle/` is the **single source of truth**
for everything an operator configures:

| Component | Location in bundle | Installed to |
|---|---|---|
| Personas | `personas/*.json` | `<atelier_home>/cowork/personas/` |
| LDD skills | `skills/ldd/*/SKILL.md` | `~/.claude/skills/` |
| Forge tools | `tools/*.json` | forge MCP via `forge_tool()` |
| AWP workflows | `workflows/*.awp.yaml` | `<atelier_home>/tenants/_default/workflows/` |
| Bridge config (Discord) | `bridge-config/discord.json` | `<atelier_home>/bridges/discord/settings.json` |
| Shared adapter | `bridge-config/shared.json` | `operator/bridges/shared/settings.json` |
| Compute config | `config-templates/tenant.atelier.yaml` | `<atelier_home>/tenants/_default/global/` |
| LDD toggles | `config-templates/ldd.json` | `<atelier_home>/global/ldd.json` |
| Voice TTS/STT | `config-templates/voice.config.json` | `~/.config/claude-voice/config.json` |
| Voice audience | `config-templates/voice.profile.json` | `~/.config/claude-voice/profile.json` |
| Forge policy | `config-templates/forge.policy.json` | `<atelier_home>/global/forge/policy.json` |
| Data policy | `config-templates/data_policy.yaml` | `<atelier_home>/global/data_policy.yaml` |

**Everything configurable by the operator is in the bundle. Nothing else.**

---

## Must NOT (during migration)

- Don't rename Python package names (only the directory containing them
  moves — `import atelier_compute` stays valid)
- Don't run Phase 2+ before Phase 1 inventory is complete
- Don't merge any phase before CI is fully green
- Don't update `CLAUDE.md` path references until the actual rename
  is committed (stale doc is worse than no doc during a migration)
- Don't move `.atelier/` — it is runtime state, not source code

---

## Implementation checklist

- [x] Concept written (this ADR)
- [x] `operator/bundle/` complete (v1.2.0 — all components present)
- [x] Phase 1: CI/test inventory of `plugins/<name>` references
- [x] Phase 2: `core/` — move 12 atelier-* plugins
- [x] Phase 3: `operator/bridges/` — move 5 bridge daemons
- [x] Phase 4: `operator/cowork|forge|skill-forge` — move operator plugins
- [x] Phase 5: `operator/voice/` — move hooks/scripts/skills
- [x] Phase 6: `operator/bundle/` — rename from repo root
- [x] Phase 7: delete `plugins/`
- [x] Phase 8: update CLAUDE.md + CI configs
