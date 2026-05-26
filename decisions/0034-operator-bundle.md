# ADR-0034 — Operator Bundle: Reproducible Operator Stack as AWPKG

**Status:** Proposed  
**Date:** 2026-05-17  
**Author:** Maintainer  
**Depends on:** ADR-0032 (AWPKG format), ADR-0030 (plugin system),
ADR-0008 (bridges state out of repo), ADR-0033 (provider abstractions)  
**Governs:** `operator/bundle/` directory, bundle install lifecycle,
the split between Core (AtelierOS engine) and Operator Layer (personas,
skills, tools, bridge config)

---

## Context

### The problem: everything is everywhere

An operator who sets up AtelierOS today must:

1. Clone the repo (Core engine)
2. Manually configure personas in `operator/cowork/personas/` (committed to Core repo)
3. Copy LDD skills into `~/.claude/skills/` (no install path, manual symlinks)
4. Forge tools materialise at runtime in `~/.atelier/global/forge/tools/` (not reproducible)
5. Write `settings.json` for each bridge in the repo directory (gitignored but co-located with code)
6. There is no single artifact that says "this is my complete operator setup"

If a second operator (or a second machine) wants the same stack, they must:
- Know what personas were customised
- Know which skills are installed
- Know which forge tools exist
- Manually replicate bridge config

There is no version, no diff, no "install from scratch" path.

### What "operator layer" means

The AtelierOS stack has two layers:

```
┌────────────────────────────────────────────┐
│            Operator Layer                  │  ← This ADR
│  Personas, Skills, Tools, Bridge Config   │
│  LDD methodology, Domain forge tools      │
├────────────────────────────────────────────┤
│            AtelierOS Core                  │  ← Never modified by operator
│  adapter.py, daemons, audit, path-gate    │
│  plugin infra, AWPKG installer, bridges   │
└────────────────────────────────────────────┘
```

"Core" is the engine. "Operator Layer" is everything the operator chose,
configured, or built on top of the engine. The operator layer changes
frequently (new personas, tuned skills, added tools). Core changes via
releases.

### The solution: Operator Bundle (AWPKG)

Package the entire Operator Layer as a signed, versioned `.awpkg`
(ADR-0032 format). Installation is one command:
```
atelier pkg install operator/bundle-v1.awpkg
```

Upgrade is one command:
```
atelier pkg upgrade com.shumway.operator
```

A second machine gets the exact same setup:
```
git clone atelier-os && atelier pkg install com.shumway.operator.awpkg
```

---

## Decision

Introduce the **Operator Bundle** as a first-class AWPKG type.

1. `operator/bundle/` in the repo is the **source directory** for the
   bundle. The AWPKG installer reads it directly (directory install, no
   zip required for local use). For distribution, `atelier pkg build`
   zips it to `com.shumway.operator-v1.awpkg`.

2. The bundle has five component types:
   - `personas/` — Cowork persona YAML/JSON definitions
   - `skills/` — SkillForge/Claude-Code skill bodies (Markdown)
   - `tools/` — Forge tool definitions (JSON with inline impl)
   - `bridge-config/` — Settings templates (no secrets) per channel
   - `data/` — Optional operator defaults (LDD config, profile.json)

3. **Bridge config install contract** (extends ADR-0032):
   `bridge-config/<channel>.settings.template.json` is copied to
   `<atelier_home>/bridges/<channel>/settings.json` **only if that file
   does not already exist** (never overwrites live secrets). The operator
   fills in tokens after first install.

4. **ADR-0008 Phase 8.3 fully implemented**: all five bridge daemons now
   read `SETTINGS_FILE` from `<atelier_home>/bridges/<channel>/settings.json`
   via `bridgeSettingsPath()` (JS) / `bridge_runtime_dir()` (Python).
   On first boot each daemon auto-migrates its legacy in-repo
   `settings.json` to the canonical path if found.

5. **Personas** from the bundle are installed to
   `<atelier_home>/cowork/personas/` (user-override layer — same as
   operator-supplied personas today, but now versioned and reproducible).

6. **Skills** are installed to `~/.claude/skills/<name>/SKILL.md`.
   The bundle ships the skill bodies as Markdown; the installer writes
   them. This replaces the current manual-symlink workflow for LDD.

7. **Tools** are installed via the Forge MCP server. The bundle's
   `tools/<name>.json` format is:
   ```json
   {
     "name": "code.backtest_multi",
     "description": "...",
     "input_schema": {...},
     "runtime": "python",
     "meta": {"deterministic": true, "side_effects": false},
     "impl": "<full python source>"
   }
   ```
   The installer calls `forge_tool(name, description, input_schema, impl, ...)`
   for each entry — same as if the user forged the tool manually.

---

## Bundle layout

```
operator/bundle/
├── manifest.yaml                          ← required (AWPKG §1)
├── README.md
│
├── personas/                              ← 12 Cowork persona definitions
│   ├── assistant.json
│   ├── browser.json
│   ├── coder.json
│   ├── forge.json
│   ├── homeassistant.json
│   ├── inbox.json
│   ├── jarvis.json
│   ├── local-coder.json
│   ├── orchestrator.json
│   ├── orchestrator-haiku.json
│   ├── os.json
│   └── research.json
│
├── skills/
│   └── ldd/                               ← Loss-Driven Development suite
│       ├── using-ldd/SKILL.md
│       ├── loop-driven-engineering/SKILL.md
│       ├── dialectical-reasoning/SKILL.md
│       ├── root-cause-by-layer/SKILL.md
│       ├── e2e-driven-iteration/SKILL.md
│       ├── reproducibility-first/SKILL.md
│       ├── loss-backprop-lens/SKILL.md
│       ├── docs-as-definition-of-done/SKILL.md
│       ├── iterative-refinement/SKILL.md
│       ├── drift-detection/SKILL.md
│       └── method-evolution/SKILL.md
│
├── tools/                                 ← Forge tool definitions (inline impl)
│   ├── code.backtest_btc.json
│   ├── code.backtest_multi.json
│   └── code.plot_backtest.json
│
├── bridge-config/                         ← Settings templates (no secrets)
│   ├── discord.settings.template.json
│   ├── telegram.settings.template.json
│   ├── slack.settings.template.json
│   ├── email.settings.template.json
│   └── whatsapp.settings.template.json
│
└── install.sh                             ← Bootstrap helper
```

---

## `manifest.yaml`

```yaml
awpkg: "1.0"
id: "com.shumway.operator"
name: "Shumway Operator Bundle"
version: "1.0.0"
description: >
  Complete operator stack: 12 Cowork personas, LDD skill suite,
  trading backtest tools, and bridge config templates.
license: "proprietary"
author: "shumway"

components:
  personas:     ["personas/*.json"]
  skills:       ["skills/**"]
  tools:        ["tools/*.json"]
  bridge_config: ["bridge-config/*.template.json"]

permissions:
  network:  true   # browser + research personas need network
  compute:  true   # backtest tools use ComputeEngine
  secrets:
    - DISCORD_TOKEN
    - TELEGRAM_TOKEN
    - SLACK_BOT_TOKEN
    - SLACK_APP_TOKEN
    - EMAIL_IMAP_USER
    - EMAIL_IMAP_PASS
    - OPENAI_API_KEY
```

---

## Install lifecycle

```
atelier pkg install ./operator/bundle/

  1. Validate manifest (schema check, no path traversal)
  2. Install personas  → <atelier_home>/cowork/personas/<name>.json
  3. Install skills    → ~/.claude/skills/<name>/SKILL.md
  4. Install tools     → forge_tool() MCP call per tools/*.json
  5. Install bridge-config:
       for each bridge-config/<ch>.settings.template.json:
         if not exists <atelier_home>/bridges/<ch>/settings.json:
           copy template → <atelier_home>/bridges/<ch>/settings.json
           log "Fill in secrets: <atelier_home>/bridges/<ch>/settings.json"
  6. Emit audit event: package.installed {id, version, scope}
```

**Step 5 is never destructive**: it only writes if the target doesn't exist.
Live settings (with real tokens) are never overwritten.

---

## What moves out of `operator/cowork/personas/`

**Phase 1 (this ADR):** Personas are **copied** into `operator/bundle/personas/`.
The originals in `operator/cowork/personas/` remain as the Core bundle
defaults — they ship with AtelierOS and work without any operator bundle.
Both paths work simultaneously; user-installed personas (from the bundle,
at `<atelier_home>/cowork/personas/`) shadow Core defaults.

**Phase 2 (follow-on ADR):** Once the install path is proven, the Core
bundle ships only a minimal set of defaults (assistant + coder). All
operator-specific personas (jarvis, inbox, homeassistant, trading-related)
move exclusively into the operator bundle.

---

## What changes per bridge daemon (ADR-0008 Phase 8.3)

All five bridge daemons (`discord`, `telegram`, `slack`, `email`,
`whatsapp`) now resolve `SETTINGS_FILE` via `bridgeSettingsPath(channel)`:

```
Before: operator/bridges/discord/settings.json   (in repo dir, gitignored)
After:  ~/.atelier/bridges/discord/settings.json       (canonical, outside repo)
```

**Auto-migration on first boot:** if the canonical path doesn't exist
and the legacy in-repo `settings.json` does, the daemon copies it over.
No operator action required for existing deployments — restart is enough.

**Security improvement:** `~/.atelier/` is mode 0700 and gitignored as a
unit. Bridge secrets can never accidentally land in a `git add .`.

---

## Must NOT

- **Must NOT** write bridge secrets (tokens, passwords) into the bundle.
  `bridge-config/` contains templates only; tokens go in vault or are
  filled in manually after first install.
- **Must NOT** auto-install a bundle without explicit operator confirmation
  (same as any AWPKG install — no silent side effects).
- **Must NOT** overwrite existing `<atelier_home>/bridges/<ch>/settings.json`
  during bundle install — `if not exists` guard is load-bearing.
- **Must NOT** include absolute paths in tool `impl` strings — impl must
  read inputs from stdin (forge sandbox contract).
- **Must NOT** put LDD skill bodies that contain prompt-injection patterns —
  SkillForge linter runs on install, not just on skill_create.
- **Must NOT** remove `operator/cowork/personas/` defaults before Phase 2.

---

## Implementation checklist

- [x] ADR-0008 Phase 8.3: all 5 bridge daemons read from canonical path
- [x] `operator/bundle/personas/` — 12 persona definitions
- [x] `operator/bundle/bridge-config/` — Discord config (with chat_profiles, no token) + shared adapter settings + 5 channel templates
- [x] `operator/bundle/manifest.yaml` — v1.1.0
- [x] `operator/bundle/install.sh` (bootstrap helper)
- [x] `operator/bundle/skills/ldd/` — 11 LDD SKILL.md files
- [x] `operator/bundle/tools/` — 3 forge tool JSON with inline impl
- [ ] AWPKG installer: `bridge_config` component type support (Phase 2)
- [ ] AWPKG installer: `skills` component → `~/.claude/skills/` path (Phase 2)
- [ ] `atelier pkg build` packs `operator/bundle/` → `.awpkg` zip (Phase 2)
- [ ] Move operator-only personas out of Core defaults (Phase 3)
