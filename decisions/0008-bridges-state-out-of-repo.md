# ADR-0008 — Bridges State Out Of Repo

**Status**: Proposed
**Date**: 2026-05-11
**Supersedes**: nothing (additive to ADR-0007)
**Depends on**: ADR-0007 (multi-tenant axis; `<atelier_home>` is canonical)

## Context

Today (2026-05-11) we discovered a `.gitignore` gap: the path
`operator/bridges/email/attachments/` was not gitignored, and two
user-uploaded chat-export files had landed there as untracked
artefacts. They were never committed, but a single `git add -A` would
have shipped private conversation data into the repository.

Inspection of the wider bridge runtime layout shows the same structural
pattern across every channel:

| Path (current) | Content |
|---|---|
| `operator/bridges/<channel>/inbox/` | Daemon → adapter queue |
| `operator/bridges/<channel>/outbox/` | Adapter → daemon queue |
| `operator/bridges/<channel>/processed/` | Done-state archive |
| `operator/bridges/<channel>/attachments/` | User-uploaded files |
| `operator/bridges/<channel>/settings.json` | Credentials + tokens |
| `operator/bridges/whatsapp/auth/` | WhatsApp pairing state |
| `operator/bridges/shared/{inbox,outbox,processed}/` | Legacy shared queues |
| `operator/bridges/**/voice.log` | Per-daemon log files |

Every one of these directories holds **user-private** runtime state.
Every one is currently protected by a **separate** entry in
`.gitignore`. Each new bridge channel and each new state-shape requires
a new gitignore entry — and the gitignore being the single line of
defense means **one missed entry is a privacy incident**.

The `.atelier/` directory (ADR-0007 multi-tenant root) already
demonstrates the right pattern: one top-level entry in `.gitignore`,
all user-data lives below it, structurally impossible to forget.
Bridge runtime state should follow the same pattern.

## Decision

All bridge runtime state moves out of the repository tree and into
`<atelier_home>/bridges/<channel>/`. The repository tree retains only
code and templates.

**Target layout — in repo (public, forkable):**

```
<repo>/operator/bridges/<channel>/
├── daemon.js                         ← code
├── settings.json.example             ← template (no secrets)
├── <channel>-daemon.service.yaml     ← systemd unit template
├── install.sh                        ← installer
├── package.json / package-lock.json  ← node deps
├── node_modules/                     ← gitignored build artefact
└── systemd/                          ← static unit-file fragments
```

No `inbox/`, no `outbox/`, no `processed/`, no `attachments/`, **no**
`settings.json`, no `auth/`, no `voice.log`. The repository
tree contains **zero** files that hold user-private data.

**Target layout — out of repo (user-private, gitignored as a unit):**

```
<atelier_home>/                       ← canonical: ~/.atelier/
├── bridges/<channel>/
│   ├── settings.json                 ← credentials (mode 0o600)
│   ├── auth/                         ← whatsapp pairing only
│   ├── inbox/                        ← runtime queue
│   ├── outbox/
│   ├── processed/
│   ├── attachments/                  ← user uploads
│   └── voice.log
├── tenants/_default/                 ← ADR-0007, unchanged
├── voice/sessions/                   ← unchanged
├── cowork/                           ← unchanged
└── ...                               ← unchanged
```

**Structural defense**: `.gitignore` now needs a single line —
`.atelier/` — to protect every present and future bridge state file.
Adding a new channel, a new state-shape, or a new log file is
structurally safe: it lands under `.atelier/`, it is gitignored, no
operator action required.

## Consequences

**Pro:**
- Single-line gitignore protects everything bridge-state — class of
  bug "forgot to gitignore a new state dir" becomes impossible.
- Code/state separation matches ADR-0007's tenant-root pattern.
- Repository forks/clones contain zero runtime state — clean cloning
  semantics regardless of how the source was used.
- Pre-commit hook (Phase 8.7) becomes much simpler: just check that
  no staged path starts with `operator/bridges/<channel>/{inbox,
  outbox,processed,attachments,auth}/` or ends with `settings.json`
  (vs. settings.json.example).
- One canonical home directory — easier for operators to back up, to
  audit, to move between machines.

**Con:**
- Migration touches 5 daemons (`telegram`/`discord`/`slack`/
  `whatsapp`/`email`), the Python adapter, `paths.py`, and the test
  suites.
- Breaking change for existing installs — needs an idempotent
  migration helper at daemon/adapter boot that detects legacy
  in-repo state and moves it to the new location atomically.
- Path-resolution becomes slightly more complex (resolver with
  fallback during the migration window).
- Symlinks are NOT a fallback option — committing symlinks into the
  repo would re-introduce a privacy surface (symlink target reads
  the file content into the worktree).

**Explicitly out of scope:**
- The `<atelier_home>/tenants/_default/voice/sessions/` tree (the
  per-chat working dir Claude Code spawns into) is already off-repo
  via ADR-0007. Phase 8 does not touch it.
- The `~/.config/claude-voice/` directory (TTS keys, voice config)
  stays where it is. It is XDG-canonical and already off-repo.

## Sub-phases

| Phase | What | Files touched |
|---|---|---|
| 8.1 | Path-resolver: `bridge_runtime_dir(channel, kind)` in `paths.py` (Python) and a sibling `bridgeRuntimeDir(channel, kind)` in `shared/js/bridge_paths.js` (Node). Both honour `ATELIER_HOME` and emit a deprecation log when falling back to the legacy in-repo path. | `shared/paths.py`, new `shared/js/bridge_paths.js`, `forge/forge/paths.py` (mirror) |
| 8.2 | Migration helper: detect `<repo>/operator/bridges/<channel>/{inbox,outbox,processed,attachments,auth}/` and `settings.json`; atomically move into `<atelier_home>/bridges/<channel>/`. Idempotent, audit-first, opt-out via `ATELIER_BRIDGES_MIGRATE=0`. Pattern matches ADR-0007 Phase 1.4. | `shared/bridges_migrate.py` (new), boot wiring in adapter.py + each daemon |
| 8.3 | Adapter rewire: replace the `ROOT / "inbox"` fallback in `adapter.py` with `bridge_runtime_dir(channel, "inbox")`. ENV overrides (`ADAPTER_INBOX` etc.) stay as the per-test sandbox path. | `shared/adapter.py` |
| 8.4 | Daemons rewire: each of the 5 daemons' `writeInbox()` / outbox-poller / attachments-write call now resolves through `bridgeRuntimeDir(channel, ...)`. Per-daemon settings.json read also moves. | `discord/daemon.js`, `telegram/daemon.js`, `slack/daemon.js`, `whatsapp/daemon.js`, `email/daemon.js` |
| 8.5 | Tests: every test that read from the old in-repo paths now points at a tempdir via `ATELIER_HOME`. New tests for the resolver + migration helper. | `shared/test_*.py`, `shared/js/test_modules.js`, new `shared/test_bridges_migrate.py` |
| 8.6 | `.gitignore` cleanup: remove the no-longer-needed lines (every `operator/bridges/*/{inbox,outbox,processed,attachments}/`). Replace with one comment block that points to `.atelier/`. | `.gitignore` |
| 8.7 | Pre-commit hook + closure narrative + CLAUDE.md update. | new `scripts/git-hooks/pre-commit`, `CLAUDE.md` |

Total estimated complexity: ~ ADR-0007 Phase 1's level (mechanical
path-rewire across many files, but each step small and audit-able).

## What you, as Claude Code, must NOT do (ADR-0008)

- **Don't commit symlinks** from `<repo>/operator/bridges/<channel>/
  inbox/` (etc.) to `<atelier_home>/bridges/<channel>/inbox/`. A symlink
  inside the repo tree means a future `tar`/`zip` of the worktree
  follows the symlink and packages user data. The point of the move is
  that the repo tree contains zero private data — symlinks defeat that.
- **Don't fall back silently to the legacy in-repo path forever.** The
  fallback is a migration courtesy with a one-time deprecation log,
  same pattern as ADR-0007's `CLAUDEOS_HOME` alias. Phase 8.6+ removes
  the fallback once installs have migrated.
- **Don't write `settings.json` (with credentials) anywhere under the
  repo tree.** The template stays at `settings.json.example` (which is
  whitelisted in `.gitignore`); the real file lives at
  `<atelier_home>/bridges/<channel>/settings.json` and never under the
  repo.
- **Don't create the `<atelier_home>/bridges/` tree from the path
  resolver.** Same rule as ADR-0007 Phase 1.1: resolvers are identity-
  only, no FS side effects. The migration helper (Phase 8.2) is the
  single owner of `mkdir` for the bridges tree.
- **Don't widen the migration's source-path list without an E2E
  proving the new path's content survives.** ADR-0007 Phase 1.4
  established this discipline; ADR-0008 inherits it. A silently-added
  source breaks idempotency for operators who already ran the helper.
- **Don't migrate `node_modules/`.** It's a build artefact, gitignored
  already, and re-installable via `install.sh`. Moving it into
  `<atelier_home>` would just clutter the user-private tree.

## References

- ADR-0007 — multi-tenant integration surface (the canonical
  `<atelier_home>` pattern this ADR extends).
- Current `.gitignore` entries that this ADR makes redundant.
- Today's incident: `operator/bridges/email/attachments/` was
  missing from `.gitignore`. Two user-uploaded chat-exports landed
  there as untracked files. Fixed in the same commit chain as this
  ADR (`.gitignore` widening — Phase 1 of today's privacy cleanup).
