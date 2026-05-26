# ADR-0032 — AWPKG: Portable Workflow Package Format

**Status:** Proposed
**Date:** 2026-05-17
**Author:** Maintainer
**Depends on:** ADR-0011 (workflow plugins), ADR-0030 (plugin system),
ADR-0007 (multi-tenant), ADR-0006 (DAG walker)
**Extends:** ADR-0011 §C (`.atelier-pkg` placeholder), ADR-0030 (lifecycle contract)
**Governs:** Package format, install/remove/export CLI, path-gate integration,
optional registry surface

---

## Context

### The gap ADR-0011 left open

ADR-0011 introduced the single-file workflow plugin (`workflow.awp.yaml` + `triggers:` block)
and mentioned `.atelier-pkg` as a future bundling artefact for cases where a workflow ships
alongside helper Forge tools, custom Skills, or Personas. That placeholder was never defined.

Today there is no standard answer to:

- "I built a strategy that uses three Forge tools, two Skills and a custom Persona — how do I
  send it to a colleague who also runs AtelierOS?"
- "How does an operator install / uninstall a third-party workflow bundle without touching the
  file system by hand?"
- "What is the audit trail for 'user installed package X on 2026-05-17'?"

The copy-paste workaround (share raw YAML/JSON over chat) breaks the path-gate invariant,
skips the SkillForge linter, leaves no audit event, and has no version or dependency metadata.

### What needs to be bundled together

A non-trivial AWP-workflow typically requires:

| Component | Defined by | Current sharing story |
|---|---|---|
| Workflow DAG | `workflow.awp.yaml` | Manual file copy |
| Forge tools | `code.*.json` via MCP | No sharing story |
| SkillForge skills | `SKILL.md` | No sharing story |
| Cowork personas | `persona.yaml` | No sharing story |
| Default config / parameter defaults | Inline YAML | Embedded in workflow |

A **package format** that bundles all four into one distributable file, with a machine-readable
manifest and an install/remove lifecycle that the existing audit and path-gate infrastructure
can observe, is the missing piece.

---

## Decision

Introduce **AWPKG** (`*.awpkg`) as the canonical distribution format for AtelierOS workflow
bundles. An `.awpkg` file is a ZIP archive with a structured layout and a mandatory
`manifest.yaml`. The existing `AtelierPlugin` protocol (ADR-0030) is the lifecycle contract;
AWPKG is the **transport and installation format** for that contract.

---

## Format specification

### Archive layout

```
<name>-<version>.awpkg          ← ZIP archive (standard deflate, no password)
├── manifest.yaml               ← mandatory; version, id, components, permissions
├── workflows/                  ← AWP workflow YAML files (ADR-0011 shape)
│   └── *.awp.yaml
├── tools/                      ← Forge tool schema definitions
│   └── code_*.json
├── skills/                     ← SkillForge skill bodies
│   └── <slug>/SKILL.md
├── personas/                   ← Cowork persona definitions
│   └── <name>.yaml
├── data/                       ← Optional non-PII default config / seed data
│   └── defaults.yaml
└── README.md                   ← Human-readable, displayed by `pkg inspect`
```

No executable scripts, compiled binaries, or hook directories are permitted. The package
*declares* — the AtelierOS runtime *installs*. Any entry outside the above list causes
the installer to abort before extraction.

### `manifest.yaml` schema

```yaml
awpkg: "1.0"                          # format version (semver-like, breaking = major bump)

# ── Identity ────────────────────────────────────────────────────────────────
id: "com.example.btc-momentum"        # reverse-domain, globally unique
name: "BTC Momentum Trader"
version: "1.2.3"                      # SemVer
description: "Trendfolge-Strategie für BTC/USD mit Compute-gestütztem Backtest."
author: "Alice <alice@example.com>"
license: "Apache-2.0"
homepage: "https://github.com/alice/btc-momentum"   # optional

# ── Compatibility ────────────────────────────────────────────────────────────
min_atelier_version: "0.9.0"
max_atelier_version: null             # null = unbounded

# ── Components (paths relative to archive root) ───────────────────────────
components:
  workflows:
    - workflows/momentum.awp.yaml
  forge_tools:
    - tools/code_backtest.json
    - tools/code_plot_equity.json
  skills:
    - skills/momentum_analysis/SKILL.md
  personas:
    - personas/quant_trader.yaml
  data:
    - data/defaults.yaml              # optional

# ── Security declarations ────────────────────────────────────────────────────
permissions:
  network: false                      # sandbox policy — forwarded to Forge tool sandbox
  compute: true                       # may schedule compute runs (ADR-0013 / ADR-0029)
  secrets:                            # required secret names (values live in vault)
    - EXCHANGE_API_KEY

# ── Dependencies (other installed AWPKG packages) ─────────────────────────
dependencies:
  - id: "com.atelier.base-data-tools"
    version: ">=1.0.0"
  - id: "com.atelier.plot-utils"
    version: "^2.0.0"

# ── Optional signature (Ed25519 over SHA-256 manifest digest) ─────────────
signature:
  algorithm: "ed25519"
  public_key: "base64url-encoded DER"
  value: "base64url-encoded signature"
```

All fields are validated against a JSON Schema (`core/awpkg/schema/manifest.v1.json`)
before any file in the archive is touched.

---

## Security model

### Pre-extraction checks (fail-closed)

| Check | Failure action |
|---|---|
| `manifest.yaml` present and valid against JSON Schema | Abort, no extraction |
| All declared component paths exist inside the archive | Abort |
| No undeclared paths exist in the archive | Abort |
| No absolute paths (`/…`) in any entry | Abort |
| No path-traversal sequences (`../`) | Abort |
| Tool names match `^code\.[a-z0-9_]+$` | Abort |
| `permissions.network: false` honoured — tools MUST NOT declare `network: allow` | Abort |

### Post-extraction checks (via existing linters)

| Component | Linter |
|---|---|
| SkillForge skills | Existing SkillForge linter (injection, secrets, persona-boundary) |
| Forge tool schemas | Existing Forge schema validator |
| Cowork personas | Existing persona YAML validator |
| Workflow DAGs | Existing AWP schema validator (ADR-0006) |

### Audit events

Every install and remove writes into the unified audit chain (`audit.jsonl`):

```jsonc
// install
{"event": "package.installed", "id": "com.example.btc-momentum",
 "version": "1.2.3", "scope": "user", "tenant_id": "_default",
 "ts": "...", "prev_hash": "..."}

// remove
{"event": "package.removed", "id": "com.example.btc-momentum",
 "scope": "user", "tenant_id": "_default",
 "ts": "...", "prev_hash": "..."}
```

### Path-gate integration

The package installer (`core/awpkg/installer.py`) extracts files into the target scope
directory. The path-gate hook (`operator/voice/hooks/path_gate.py`) extends its protected
subtree to include `<atelier_home>/**/packages/**` so that no agent subprocess can write
into an installed package directory directly — only the installer may do so, through the
explicit MCP or CLI path.

---

## Installation scopes

| Scope | Location | Visibility |
|---|---|---|
| `user` | `~/.atelier/packages/<id>/` | All projects, all tenants of this user |
| `project` | `.atelier/packages/<id>/` | This repository only |
| `session` | `<atelier_home>/sessions/<chan>:<chat>/packages/<id>/` | This chat session only (for testing) |

Default scope: `user`. Project- and session-scoped installations never promote automatically.

After installation the runtime mounts components into the appropriate registries:

- Forge tools → available in MCP namespace of the current persona
- Skills → injected into future bridge turns (project/user scope only, matching the existing
  slot-mirror scope-gate from ADR Layer 7)
- Personas → available to cowork persona resolver
- Workflows → registered in the scheduler (if `triggers.schedule` present) and slash-command
  dispatcher (if `triggers.slash` present)

---

## CLI surface

```bash
# Install from local file
atelier pkg install btc-momentum-1.2.3.awpkg
atelier pkg install btc-momentum-1.2.3.awpkg --scope project

# Install from registry (Phase 2)
atelier pkg install com.example.btc-momentum
atelier pkg install com.example.btc-momentum@1.2.3

# Inspect without installing
atelier pkg inspect btc-momentum-1.2.3.awpkg

# List installed packages
atelier pkg list
atelier pkg list --scope user

# Remove
atelier pkg remove com.example.btc-momentum
atelier pkg remove com.example.btc-momentum --scope project

# Build a package from a local workspace
atelier pkg init                          # creates awpkg.yaml skeleton
atelier pkg build                         # validates + bundles → *.awpkg
atelier pkg build --out ./dist/

# Sign a package (maintainer workflow)
atelier pkg sign btc-momentum-1.2.3.awpkg --key ~/.atelier/keys/signing.pem

# Export an already-installed package back to a file
atelier pkg export com.example.btc-momentum ./dist/
```

### `atelier pkg build` source config (`awpkg.yaml`)

```yaml
# Lives in the workflow's source directory; not shipped inside the .awpkg
id: "com.example.btc-momentum"
version: "1.2.3"
name: "BTC Momentum Trader"
description: "..."
author: "Alice <alice@example.com>"
license: "Apache-2.0"

include:
  workflows:
    - src/momentum.awp.yaml
  forge_tools:
    - forge/code_backtest.json
    - forge/code_plot_equity.json
  skills:
    - skills/momentum_analysis/SKILL.md
  personas:
    - personas/quant_trader.yaml

permissions:
  network: false
  compute: true
  secrets:
    - EXCHANGE_API_KEY

dependencies:
  - id: "com.atelier.base-data-tools"
    version: ">=1.0.0"
```

---

## Dependency resolution

The installer resolves dependencies depth-first before extracting the package itself.
Resolution algorithm:

1. Read `dependencies` from manifest.
2. For each dependency: check whether a compatible version is already installed.
3. If yes → skip. If no → abort with a clear error (no auto-download in Phase 1).
4. After all dependencies are satisfied → extract and register the package.

Phase 2 adds automatic dependency fetch from the registry.

---

## Optional registry (Phase 2)

A lightweight registry at `packages.atelier.dev` (operator self-hostable):

```
GET  /v1/packages                         # search / list
GET  /v1/packages/<id>                    # metadata + version list
GET  /v1/packages/<id>/<version>/download # binary .awpkg
POST /v1/packages                         # publish (auth required)
```

The registry is **never required** — offline distribution via ZIP is the primary path and
must remain fully functional without network access.

CLI opt-in:

```bash
atelier pkg registry add https://packages.atelier.dev
atelier pkg registry list
```

---

## Relation to existing ADRs

| ADR | Relation |
|---|---|
| ADR-0011 (workflow plugins) | AWPKG *is* the `.atelier-pkg` placeholder from §C. Single-file workflow YAMLs (ADR-0011 shape) are a valid sub-set of AWPKG — they can be shipped standalone OR inside an `.awpkg`. |
| ADR-0030 (plugin system) | `AtelierPlugin` protocol is the lifecycle contract. AWPKG is the **transport** for plugins that are not Python packages (no `importlib.metadata` entry point). The installer registers components with the same `PluginRegistry`. |
| ADR-0007 (multi-tenant) | Scope-gate: `user`-scope packages are tenant-global; `project`-scope packages are workspace-local. `tenant_id` recorded in audit event. |
| ADR-0006 (DAG walker) | Workflow files inside AWPKG are validated and handed to the same walker. |
| L7 (SkillForge) | SkillForge linter runs unchanged on every `SKILL.md` in the package. Slot-mirror scope-gate applies: task/session-scope skills are never mirrored. |
| L6 (Forge) | Forge schema validator runs on every `code_*.json`. Path-gate extended to `packages/**`. |
| L10 (path-gate) | `packages/**` added to protected subtree — no direct agent writes. |
| L16 (audit chain) | `package.installed` / `package.removed` events added to the hash chain. |

---

## Must NOT do

- Don't allow executable scripts or compiled binaries inside `.awpkg`.
- Don't extract any archive entry before the full pre-extraction check set passes.
- Don't bypass the SkillForge linter for skills inside a package ("trusted source" allowlist is forbidden).
- Don't allow `permissions.network: true` in a package that also declares `compute: true` without explicit operator config override.
- Don't make the registry the install path for offline / air-gapped deployments — local file must always work.
- Don't skip the `package.installed` audit event on successful install.
- Don't auto-promote a session-scope installation to project or user scope.
- Don't allow a package to declare `secrets:` values (only names — the vault contract from L6 must hold).
- Don't write `packages/**` from bridge or adapter code — only through the installer CLI or MCP tool.

---

## Open questions (to resolve before Phase 1 merge)

1. **Namespace ownership:** Who may publish under `com.atelier.*`? Maintainer-reserved or
   open with a registration gate?
2. **Signature enforcement:** Optional for local installs; mandatory for registry installs?
   Or tenant-configurable (`require_signatures: true` in `tenant.atelier.yaml`)?
3. **Compute permission gate:** Should `compute: true` in the manifest require explicit
   operator sign-off in `tenant.atelier.yaml::compute.enabled`? (Currently compute is
   off-by-default per ADR-0013.)
4. **Tenant isolation for `user`-scope:** A `user`-scope package is currently visible to
   all tenants of that user. Should there be a per-tenant include/exclude list?
5. **Upgrade behaviour:** `atelier pkg install` over an existing version — warn + replace, or
   require explicit `--upgrade` flag?

---

## Implementation phases

### Phase 1 — Local install/remove (no registry)

- `core/awpkg/` plugin skeleton (follows ADR-0030 structure)
- JSON Schema for `manifest.yaml` (v1)
- `installer.py`: pre-extraction checks, scope resolution, component registration,
  audit events
- `builder.py`: `atelier pkg build` + `atelier pkg init`
- Path-gate extension for `packages/**`
- `atelier pkg inspect` (read-only, no extraction)
- Test suite: install/remove round-trip, path-traversal rejection, linter integration

### Phase 2 — Registry + dependency resolution

- Registry API (read-only first, then publish)
- Automatic dependency fetch
- `atelier pkg search`
- Signature verification (Ed25519)

### Phase 3 — MCP tool surface

- `mcp__awpkg__install` / `mcp__awpkg__remove` / `mcp__awpkg__list`
- Allows in-conversation install ("installiere das btc-momentum-Paket")
- Gated: requires `awpkg_mcp_enabled: true` on persona
