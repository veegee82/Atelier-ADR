# ADR-0046 — v1.0 Public Release Readiness

**Status:** Proposed (2026-05-21)
**Owner:** Silvio Jurk (Maintainer)
**Predecessor:** ADR-0018 (enterprise launch readiness), IMPLEMENTATION-ROADMAP.md (EU compliance M1–M6)
**Sister-ADRs:** 0045 (L36 erasure), 0044 (L37 audit-at-rest), ADR-001 (OpenCode EU AI Act)

---

## Context

AtelierOS is at **v0.13.0** (2026-05-10) with a large `[Unreleased]` block
in `CHANGELOG.md` that contains the complete EU compliance build-out
(L34–L37, M1–M6, including `_compliance_gate()` adapter wiring, the
EU_PRODUCTION presets, the erasure orchestrator, the compliance docs
package, and the full EU E2E test suite). The compliance work is done in
code; it just has not been versioned yet.

What separates the current state from a credible public OSS release:

1. **Unreleased code** — M1–M6 compliance work sits unversioned.
   EU AI Act claim is structurally sound but marketing-unshippable without
   a release tag to point at.

2. **Phase 7 backlog open** — 12 `CLAUDEOS_*` env-var aliases, legacy
   `~/.claudeOS/` fallback paths, dual-fire `CLAUDEOS_SIGNAL:` marker,
   old systemd unit names, and a 455-file docs substring sweep all remain.
   Phase 7-8 (final cut + `v1.0` bump) is gated on all of these.
   See `docs/phase7-backlog.md` for the full inventory.

3. **No public external presence** — no standalone website or landing page
   is live. The web-next console frontend contains a production-quality
   landing page but it is only reachable from a running self-hosted stack.
   First-time GitHub visitors see no live demo, no pricing, no screenshots.

4. **User-facing documentation gaps** — developer-oriented docs (52 ADRs,
   18 `claude-ref` files) are thorough. User-facing docs are thin.
   `docs/README.md` itself acknowledges five missing files under
   "Coming next": `setup.md`, per-bridge guides, `troubleshooting.md`,
   `memory.md`, `extending.md`.

5. **Responsible disclosure policy missing** — no `SECURITY.md` exists.
   Standard requirement for any open-source project accepting external
   contributions or running on third-party infrastructure.

6. **Privacy notice unpublished** — `docs/compliance/PRIVACY-NOTICE-TEMPLATE.md`
   exists but is a template, not a filled and committed document. GDPR
   Art. 13/14 obligation falls on every operator who deploys; the repo
   should ship an example they can adapt and a clear note on what to
   customise.

7. **GitHub repository hygiene** — git remote still points at the redirect
   URL (`veegee82/claudeOS`). No issue templates beyond the beta-tester
   PR template. No README screenshots or demo GIF. README badges reference
   a hardcoded test count (183) that will drift.

8. **Console access undocumented** — the web-next console runs on
   `127.0.0.1:8765` (uvicorn, via `bridge.sh`) but no user-facing guide
   exists for: how to reach it, how to generate an owner token, how to
   secure it behind a reverse proxy for remote access.

9. **Persona editing is read-only in the console** — the web-next
   `personas.tsx` page renders personas but editing is deferred to
   "Iteration 2". This is the first thing a new user wants to do.

10. **No CHANGELOG release entry for the compliance build-out** — the
    `[Unreleased]` block will become the v0.14.0 (or v1.0-rc1) entry.
    Without it, the compliance work is invisible to evaluators who check
    the changelog before adopting.

This ADR organises the gap into **five tracks** with sequencing constraints,
acceptance criteria, and a clear v1.0 definition. It supersedes the
"Coming next" items in `docs/README.md` and the informal launch notes
in `docs/phase7-backlog.md § Phase 7-8`.

---

## Decision

Five tracks, sequenced by dependency. Tracks 2–4 run in parallel with
Track 1. Track 5 runs last.

| Track | Name | Effort | Blocking v1.0? |
|---|---|---|---|
| **T1** | Release v0.14.0 pre-release | 1 session | Unblocks T3 marketing claim |
| **T2** | Phase 7 completion | 3–5 sessions | Yes — gates v1.0 tag |
| **T3** | External presence | 2–3 sessions | Yes — gates public announcement |
| **T4** | User documentation | 2–3 sessions | Yes — gates honest "usable" claim |
| **T5** | v1.0 cut + launch prep | 1 session | Final step |

---

## Track 1 — Release the compliance build-out as v0.14.0

**Status:** Next (1 Claude session)

The entire M1–M6 compliance build is implemented and tested. The only
thing missing is a git tag and a `CHANGELOG.md` release block.

### T1.1 — Promote `[Unreleased]` → `[0.14.0]`

Move the existing `[Unreleased]` block in `CHANGELOG.md` to
`## [0.14.0] — 2026-05-21` (or the actual date of the commit).

Add a one-paragraph summary header at the top of the 0.14.0 block:

> **EU compliance build-out (M1–M6) + Layer 29.5 helper-model cost-split.
> AtelierOS is now structurally complete for EU AI Act Art. 50 and GDPR
> Art. 6, 7, 17, 30, 32 deployments. All four enforcement layers (L34
> data classification, L35 egress lockdown, L37 audit-at-rest encryption,
> L36 erasure orchestrator) ship with tests, audit events, and the
> EU_PRODUCTION preset pair.**

Add a fresh empty `[Unreleased]` section above it so future commits
have a place to land.

### T1.2 — Git tag `v0.14.0`

```bash
git tag -a v0.14.0 -m "chore(release): EU compliance build-out M1-M6, Layer 29.5 cost-split"
git push origin v0.14.0
```

**Acceptance:** `git describe --tags --abbrev=0` returns `v0.14.0`;
GitHub Releases page shows the new tag with the CHANGELOG entry.

---

## Track 2 — Phase 7 completion (gates v1.0)

**Status:** 3–5 Claude sessions. Inventory at v0.13.0: 12 open
`CLAUDEOS_*` aliases, `~/.claudeOS/` fallback paths active, dual-fire
signal marker, four `claudeos-*` systemd units, 455 files containing
the `claudeos` substring.

Each sub-task below maps to an entry in `docs/phase7-backlog.md`.
Do them in order; run `bash operator/bridges/run-all-tests.sh` after
each step before committing.

### T2.1 — Remove CLAUDEOS_* env-var aliases (Phase 7-2)

Twelve aliases remain. Remove them in three batches to keep diffs
reviewable:

**Batch A — forge/paths.py (CLAUDEOS_HOME × 3):**
- `operator/forge/forge/paths.py` (3 byte-identical copies — consolidate
  into one canonical import if possible, then drop the alias)
- Pre-flight: `grep -r CLAUDEOS_HOME ~/.atelier/ ~/.config/claude-voice/`
  confirms no live use.

**Batch B — forge/scope.py (4 aliases):**
- `CLAUDEOS_FORCE_SCOPE`, `CLAUDEOS_DEFAULT_SCOPE`,
  `CLAUDEOS_CHANNEL_ID` (also in `adapter._build_spawn_env`),
  `CLAUDEOS_TASK_ID`

**Batch C — adapter + remaining (5 aliases):**
- `CLAUDEOS_CALLER_PERSONA` (`adapter._build_spawn_env`)
- `CLAUDEOS_PLUGIN_SLOT_DIR` (`registry.py::plugin_slot_dir`)
- `CLAUDEOS_SESSION_TTL_DAYS` (`session_timeout_sweep.py`)
- `CLAUDEOS_AGENTS_SKIP_LIVE` (test-only; drop from `test_engines_e2e.py`)
- `CLAUDEOS_USE_ENGINE_LAYER` (`adapter.py` — already defaulting to `"1"`;
  remove the branch and make the engine layer unconditional)
- `CLAUDEOS_INIT_NO_AUTOSTART` + `CLAUDEOS_INIT_TICK_INTERVAL` (`core/init/init.py`)

For each removal: delete the fallback `os.environ.get("CLAUDEOS_*", ...)`
branch; keep only the `ATELIER_*` read; update the docstring/comment;
remove any test case that set the legacy spelling.

**Must NOT:** Remove before confirming the operator's `service.env`
does not reference the legacy spelling.

**Acceptance per batch:** `grep -r "CLAUDEOS_" operator/ core/ --include="*.py"
--include="*.js" --include="*.sh" | grep -v test_ | wc -l` decreases
to zero (non-test production code).

### T2.2 — Remove ~/.claudeOS/ fallback paths (Phase 7-3)

Six files contain the dual-path try-atelier-then-claudeOS pattern:
`paths.py` (× 3), `scope.py`, `registry.py`, `notification_relay.py`,
`path_gate.py`, `auth_elevation_gate.py`.

Pre-flight on the operator's box: confirm
`~/.claudeOS/` is empty or the `MIGRATED` marker is present (from
`atelier_migrate.py`). If not, run `python3 operator/bridges/shared/atelier_migrate.py`
first.

For each file: delete the `if Path("~/.claudeOS/...").expanduser().exists()`
branch; keep only the `~/.atelier/` path. Update path-gate tests.

**Acceptance:** `grep -r "claudeOS" operator/ core/ --include="*.py"
--include="*.js" | grep -v "# formerly" | grep -v test_` returns zero
hits outside of historical ADR references and git log.

### T2.3 — systemd unit cleanup (Phase 7-4)

`bridge.sh` installs both `atelier-*` and `claudeos-*` unit files.

- Remove the `claudeos-audit-verify.timer` + `claudeos-session-timeout.timer`
  template generation from `bridge.sh` (lines that write the legacy unit
  files).
- After the next `bridge.sh up` on the operator box, the legacy units will
  be absent from the generated set. Disable them manually on the operator box:
  ```bash
  systemctl --user disable claudeos-audit-verify.timer claudeos-session-timeout.timer
  systemctl --user stop   claudeos-audit-verify.timer claudeos-session-timeout.timer
  ```
- Verify `atelier-audit-verify.timer` and `atelier-session-timeout.timer`
  are enabled and firing (check `systemctl --user status atelier-audit-verify.timer`).

**Acceptance:** `grep -r "claudeos-" operator/bridges/bridge.sh` returns
zero hits.

### T2.4 — CLAUDEOS_SIGNAL dual-fire removal (Phase 7-5)

`adapter.py::_signal` currently writes:
```
[ATELIER_SIGNAL: <name>] [CLAUDEOS_SIGNAL: <name>]
```

Pre-flight: `grep -r "CLAUDEOS_SIGNAL" operator/cowork/personas/` and
`grep -r "CLAUDEOS_SIGNAL" ~/.atelier/cowork/personas/` to find any
persona `append_system` block still grepping for the legacy marker.
Update those personas first.

Then strip the `[CLAUDEOS_SIGNAL: ...]` half from `_signal()`.

**Acceptance:** `grep -r "CLAUDEOS_SIGNAL" operator/ core/` returns zero
hits.

### T2.5 — Docs final sweep (Phase 7-6)

Run:
```bash
grep -ril "claudeos" docs/ --include="*.md" | xargs grep -l "claudeos" \
  | grep -v "CHANGELOG" | grep -v "phase7-backlog"
```

For each file: distinguish live references (update to AtelierOS) from
historical ADR footnotes (keep but add an italicised `_(formerly claudeOS)_`
parenthetical). Do NOT touch `CHANGELOG.md` or `docs/phase7-backlog.md`
— these are historical records.

Update `README.md`:
- Remove the `(formerly claudeOS)` parenthetical from the project header
  and About section once the `claudeos` substring count in non-historical
  docs reaches zero.
- The `README.md` banner `<picture>` block already references
  `docs/assets/banner.svg` — fine, no change needed.

SVG re-renders (Phase 7-6 deferred item): the source `.mmd` or hand-edited
files for `docs/assets/banner.svg` and `docs/diagrams/*.svg` may contain
stale strings. Audit and re-render; if the SVG sources are hand-edited,
do a direct text substitution. This is low-risk because the banner SVG
was verified clean in the 2026-05-21 review.

**Acceptance:** `grep -ric "claudeos" docs/ --include="*.md" | grep -v "CHANGELOG\|phase7"
| awk -F: '{sum += $2} END {print sum}'` returns `0`.

### T2.6 — git remote URL correction

```bash
git remote set-url origin https://github.com/veegee82/AtelierOS.git
```

Verify: `git remote -v` shows the canonical URL (no redirect).

Also update the `WorkingDirectory=` references in any systemd unit files
generated by `bridge.sh` if they still reference `/home/shumway/projects/claudeOS/`.
Check: `grep -r "claudeOS" operator/bridges/bridge.sh`.

**Acceptance:** `git remote -v` shows `veegee82/AtelierOS.git`; no stale
`claudeOS` path in generated unit files.

---

## Track 3 — External presence (parallel with T2)

**Status:** 2–3 Claude sessions. Blocking public announcement.

### T3.1 — SECURITY.md

Create `SECURITY.md` at the repo root. Required content:

- **Supported versions** — which release(s) receive security fixes
  (v1.0+ LTS; older tags get no backports).
- **Reporting a vulnerability** — email `security@atelierOS.dev` (or
  the maintainer's contact); expected response time (≤72 h acknowledgement,
  ≤14 days triage); CVE coordination process.
- **Scope** — what is in scope (bridge daemons, adapter, forge sandbox,
  audit chain, path-gate) and what is not (the upstream Claude CLI binary,
  third-party bridge library dependencies if < 30 days old).
- **Out-of-scope** — social engineering, issues requiring physical access,
  CVSS < 4.0 on a fresh default install.
- **PGP key** — optional but recommended for the maintainer's signing key.

This file is referenced automatically by GitHub's security advisory UI
and by `npm audit` / `pip-audit` toolchains.

**Acceptance:** `SECURITY.md` exists at repo root; GitHub shows the
"Security policy" tab in the repository Insights → Security section.

### T3.2 — Published privacy notice

`docs/compliance/PRIVACY-NOTICE-TEMPLATE.md` is an Art. 13/14 template.
Create a companion `docs/compliance/PRIVACY-NOTICE-EXAMPLE.md` that is
the template **filled in for a canonical self-hosted AtelierOS deployment**
(using `operator@example.com` and `your-domain.example.com` as
placeholders). This is the document an operator copies, fills in their
own contact details, and publishes.

Add a one-paragraph note at the top of the template:
> *"Operators: copy this file, replace all `<...>` placeholders with your
> deployment-specific details, and publish it at your deployment's
> `/privacy` URL or in your user-facing documentation. The example file
> `PRIVACY-NOTICE-EXAMPLE.md` shows how the blanks should be filled."*

**Acceptance:** Both files exist; the example has no unfilled `<...>`
placeholders; `docs/compliance/COMPLIANCE-REPORT-GUIDE.md` references them.

### T3.3 — README screenshots + demo GIF

Add five visual elements to `README.md` between the "How it works"
architecture diagram and the "Four Headline Features" section:

1. **Screenshot: Discord bot receiving a message and replying** — shows
   the disclosure card (first `/join`) and a subsequent task reply with
   voice attachment.
2. **Screenshot: `bridge.sh status` output** — shows all seven bridge
   channels with a green status.
3. **Screenshot: web console dashboard** — shows the AtelierOS web-next
   dashboard with bridge status, audit chain health, and persona list.
4. **Screenshot: `voice-audit verify` output** — shows the green chain
   integrity check with event count and hash summary.
5. **GIF or static screenshot: forge tool creation** — a user asks the
   bot to create a tool; the bot responds with a forge invocation and
   the tool is callable in the next turn.

Place screenshots under `docs/assets/screenshots/` (create directory).
Reference them from README.md with `<img>` tags (same style as existing
SVG references, `width="100%"` or `width="700"`).

Hardcoded test-count badge (`183 green`) must be replaced with a dynamic
CI badge once a GitHub Actions test workflow exists, OR updated to the
current count and noted as manually maintained.

**Acceptance:** README renders correctly on GitHub desktop and mobile
(check via GitHub's mobile preview); images are ≤200 KB each; GIF ≤2 MB.

### T3.4 — Standalone landing page

Deploy a minimal static landing page at a public URL (e.g.
`atelierOS.dev` or GitHub Pages at `veegee82.github.io/AtelierOS`).

The web-next `landing.tsx` is production-quality but requires a running
stack. A static landing page must be accessible without any backend.

**Minimal viable content:**
1. Hero: AtelierOS banner SVG + tagline + two CTA buttons (GitHub,
   Quick-Start).
2. Four compliance pillars (can reuse the `compliance.svg` diagram).
3. Seven bridges listed (icons or text list).
4. Quick-start code block: `bash setup.sh`.
5. Link to `docs/` and the GitHub repo.
6. Footer: Apache-2.0, Privacy Notice link, Security contact.

**Implementation options (pick one):**
- GitHub Pages: add `.github/workflows/deploy-pages.yml` that copies a
  `site/` directory (vanilla HTML/CSS) to the `gh-pages` branch on every
  push to `main`. No build step — hand-write the HTML once.
- Cloudflare Pages (zero-config): push the `site/` directory; Cloudflare
  picks it up automatically.
- Deploy the web-next frontend with a static mock for the `/v1/console/landing/personas`
  endpoint (hardcode the bundle personas as JSON).

Whichever option: `site/index.html` is the single source of truth.
Do NOT duplicate content between `site/` and the existing `README.md` —
the README stays comprehensive; the landing page is the 30-second hook.

**Acceptance:** A public HTTPS URL renders the landing page without errors;
`curl -s <url> | grep AtelierOS` returns a match; the page loads on a
mobile viewport (375 px wide, no horizontal scroll).

### T3.5 — GitHub repository hygiene

One-time operator actions (no code change required):

- **Topics:** add `ai-agent`, `claude`, `eu-ai-act`, `gdpr`, `self-hosted`,
  `discord-bot`, `telegram-bot`, `whatsapp-bot`, `mcp` to the GitHub repo
  topics (Settings → General → Topics).
- **About description:** "Self-hosted AI agent runtime — EU AI Act + GDPR
  by construction. Seven bridges, runtime tool generation, multi-engine."
- **Website field:** set to the landing page URL from T3.4.
- **Discussions:** enable GitHub Discussions (Settings → General → Features).
  Create two starter categories: "Show and tell" and "Q & A".
- **Issue templates:** add `.github/ISSUE_TEMPLATE/bug_report.md` and
  `feature_request.md` (standard GitHub templates are fine; no custom
  fields needed for v1.0).
- **CI badge (stretch):** add a GitHub Actions workflow
  `.github/workflows/test.yml` that runs `bash operator/bridges/run-all-tests.sh`
  on every push to `main` and on PRs. Add the workflow badge to README.md
  (replaces the hardcoded `183 green` badge).

**Acceptance:** GitHub repo page shows description, website, and ≥5 topics;
Discussions tab is visible; two issue templates appear in the "New Issue"
dialog.

---

## Track 4 — User documentation

**Status:** 2–3 Claude sessions. Blocking honest "usable" claim.
Write all files in `docs/` (English only per repo convention).

### T4.1 — setup.md

Step-by-step companion to `setup.sh` for users who prefer reading over
running. Structure:

1. **Prerequisites** (OS, Claude Code login, API keys).
2. **Running setup.sh** — what each stage asks and what to answer.
3. **Per-bridge credential guide** — one section per bridge (Telegram,
   Discord, WhatsApp, Slack, Email, Teams, Signal). Each section:
   where to get the token / how to pair, what to paste into setup.sh,
   how to verify it worked.
4. **Starting and stopping** — `bridge.sh up`, `bridge.sh status`,
   `bridge.sh tail`, `bridge.sh down`.
5. **Accessing the web console** — generate an owner token
   (`atelier-admin token create --tier owner`), open `http://127.0.0.1:8765/console/`,
   paste token at login. If running on a remote server: set up the
   Caddy reverse proxy (reference `ops/Caddyfile.template`).

### T4.2 — troubleshooting.md

Symptom → cause table. Every row: **Symptom**, **Likely cause**,
**Fix**. Minimum 15 rows covering the most common new-user failure modes:

| # | Symptom | Likely cause |
|---|---|---|
| 1 | Bot doesn't reply at all | Token missing or whitelist empty in settings.json |
| 2 | "claude: command not found" | Claude Code not on PATH; check `~/.local/bin` in `$PATH` |
| 3 | "openai module not found" | pip install failed due to PEP 668; use venv or pipx |
| 4 | WhatsApp QR code expired | Re-run `bash operator/voice/scripts/whatsapp_cli.sh pair` |
| 5 | Discord bot not in server | OAuth URL missing `bot` scope; re-generate invite URL |
| 6 | "rate limited" response | OPENAI_API_KEY exhausted; check API usage dashboard |
| 7 | No TTS audio attached | ffmpeg not installed; check `which ffmpeg` |
| 8 | Systemd service fails to start | WorkingDirectory not updated after repo move; see T2.6 |
| 9 | `bridge.sh status` shows all red | Node.js < 20 or `npm install` not run; run `bridge.sh up` |
| 10 | `voice-audit verify` fails (chain broken) | Uncommitted changes to audit.jsonl; do not edit manually |
| 11 | Forge tools not showing in Claude | Plugin not registered; run `claude plugin install voice@...` |
| 12 | Web console returns 401 | Owner token expired (90-day TTL); re-generate |
| 13 | Signal bridge not connecting | signal-cli binary not installed; see operator/bridges/signal/README.md |
| 14 | Email bridge not picking up mail | IMAP IDLE not supported by provider; switch to polling mode |
| 15 | Bot replies but TTS is cut off | ffmpeg missing; or `OPENAI_API_KEY` missing (TTS is OpenAI) |

Add a section "Reading the logs" showing `bridge.sh tail` output and what
each log prefix means.

### T4.3 — Console access guide

`docs/console.md`: how to access the web-next console in three scenarios:

1. **Local desktop** — `bridge.sh up` starts uvicorn on `127.0.0.1:8765`;
   open `http://127.0.0.1:8765/console/`; generate token via
   `python3 operator/voice/scripts/atelier_token.py --tier owner`.
2. **Remote server (self-hosted)** — Caddy reverse proxy setup referencing
   `ops/Caddyfile.template`; TLS via Let's Encrypt; access at
   `https://your-domain.example.com/console/`.
3. **Docker (ops/ stack)** — Caddy is already in `docker-compose.yml`;
   set `ATELIER_DOMAIN` in `.env`; console is at
   `https://<domain>/console/`.

Include token rotation (90-day TTL, `atelier-admin token create`).

### T4.4 — memory.md

Document the four user-facing memory commands: `/profile`, `/memory`,
`/vault`, and `/forget`. For each: what it does, syntax, example
interaction, what is stored and where.

Note the GDPR-relevant behaviour: `/forget` triggers L28 recall erasure
+ optionally the full L36 erasure orchestrator for complete right-to-
deletion; an operator can trace what data is held about a user.

### T4.5 — extending.md

Brief guide for operators who want to:

1. **Write a custom persona** — copy an existing persona JSON from
   `operator/cowork/personas/`, edit `description` / `append_system` /
   `mcp_servers`, drop it in `~/.atelier/cowork/personas/`.
2. **Create a forge tool** — invoke the forge MCP tool from a chat;
   describe what the tool should do; the agent generates and registers it.
   For manual creation: format reference in `docs/forge.md`.
3. **Add a new bridge** — minimal checklist: Node.js daemon with
   `health()` endpoint on port 789N, `settings.json` schema, `install.sh`,
   `bridge.sh` entry for the new channel.
4. **Install a workflow package** — `atelier-pkg install <name>.atelier-pkg`;
   what the package manifest looks like; how to create one.

### T4.6 — docs/README.md "Coming next" section removal

Once T4.1–T4.5 are written, remove the "Coming next" stub section from
`docs/README.md` and add the new docs to the reading-order table at the
appropriate positions.

---

## Track 5 — v1.0 cut + launch prep

**Status:** After T2 + T3 + T4 are complete.

### T5.1 — Phase 7-8 final cut commit

Run `docs/phase7-backlog.md` Phase 7-8 checklist:

1. Final grep audit: `grep -ric "claudeos" . --include="*.py" --include="*.js"
   --include="*.sh" --include="*.md" --include="*.json" | grep -v "Binary\|.git\|CHANGELOG\|phase7"
   | awk -F: '{sum+=$2} END {print sum}'` must return `0`.
2. One sweep commit removing every remaining `CLAUDEOS_*` literal that
   survived T2.1–T2.6.
3. Update `docs/phase7-backlog.md` Phase 7-8 table with today's date
   and final counts.

### T5.2 — CHANGELOG v1.0 entry

Add `## [1.0.0] — <date>` entry to CHANGELOG.md with:

- One-paragraph summary: Phase 7 complete, claudeOS → AtelierOS rebrand
  finalised; all four compliance layers live; 7 bridges; production-grade
  web console; EU AI Act + GDPR structural compliance claim.
- Sub-sections: Changed (Phase 7 removals), Removed (CLAUDEOS_* aliases,
  legacy systemd units), References (ADR-0018, ADR-0046, IMPLEMENTATION-ROADMAP).

### T5.3 — Git tag v1.0.0

```bash
git tag -a v1.0.0 -m "feat(release): v1.0.0 — AtelierOS public release"
git push origin v1.0.0
```

Create a GitHub Release (not just a tag) with the CHANGELOG v1.0 entry
as the body. Mark it as "Latest Release".

### T5.4 — Launch announcement draft

Prepare (do not publish until operator confirms timing):

- **HN / Show HN title:** `"Show HN: AtelierOS — self-hosted AI agent runtime,
  EU AI Act + GDPR by construction (seven bridges, runtime tool generation)"`
- **Three pre-drafted first-reply comments** (one technical Q&A, one
  compliance explanation, one competitive differentiation).
- **Awesome-list PRs** (T3.5 stretch): submit to `awesome-selfhosted`,
  `awesome-mcp`, `awesome-selfhosted-ai`.
- **README "Why AtelierOS" section** — add a two-paragraph punchy section
  at the top of README.md (before "How it works") explaining the unique
  value prop vs. running Claude Code directly. Focus on: compliance by
  construction, multi-channel, runtime self-improvement.

**Acceptance for T5.4:** The draft exists as `docs/launch-announcement-draft.md`;
no external publication happens until the maintainer issues explicit
approval.

---

## Code gaps found in review (not covered by T1–T5)

These items were identified during the 2026-05-21 code review as bugs or
structural gaps not covered by the Phase 7 or documentation tracks.

### CG-1 — Persona editing disabled in web console

`core/console/atelier_console/web-next/src/pages/personas.tsx` renders
persona detail but mutations (editing `description`, `append_system`,
model overrides) are deferred to "Iteration 2" (comment in source).

This is the first thing an operator wants to do after installation.

**Fix:** Implement `PUT /v1/console/personas/{name}` in
`core/console/atelier_console/routes/personas.py` (only for user-override
personas in `~/.atelier/cowork/personas/`; bundle personas remain
read-only). Wire to a form in `personas.tsx` (same pattern as
`bridges.tsx` settings editor: JSON textarea + re-auth modal). 8–12
tests covering the route, path-gate protection, and tier-scoping.

### CG-2 — Test count badge in README.md is manually maintained

`README.md` line 11 hardcodes `183 green`. This drifts silently.

**Fix (Track T3.5 stretch):** add a GitHub Actions workflow
`.github/workflows/test.yml` that runs the test suite; replace the
hardcoded badge with a dynamic workflow badge. Until CI exists: update
the count to the real current number whenever a release is tagged
(T1.2 and T5.3 should each bump it).

### CG-3 — bridge.sh systemd units still reference claudeOS paths

`bridge.sh` generates systemd `WorkingDirectory=` paths. If the git
clone is still at `/home/shumway/projects/claudeOS/` (or the symlink
equivalent), the generated units will break after T2.6 moves the remote
URL. Pre-flight for T2.6: confirm `WorkingDirectory` in all generated
unit files matches the actual clone path after any potential folder rename.

### CG-4 — setup.sh macOS TTS limitation undocumented in README

`setup.sh` warns at detection time: "macOS: audio *out* from Claude won't
work (voice note synthesis unavailable)." This constraint is not surfaced
in `README.md` or `docs/`. A user on macOS who reads the README will not
see this limitation before running setup. Add a "Platform notes" table to
`README.md` listing known per-OS limitations.

### CG-5 — Signal bridge: missing per-bridge README

Six of the seven bridge directories have either a `README.md` or inline
documentation in `daemon.js`. `operator/bridges/signal/` has neither.
The setup steps for Signal are non-trivial (requires `signal-cli` JVM
binary, phone number registration). Add `operator/bridges/signal/README.md`
covering: prerequisites, installation of `signal-cli`, registration, and
first-run pairing.

---

## What this ADR explicitly does NOT cover

- **Enterprise tier features** (ADR-0017 Phase V D.2 items: SSO wizard,
  WORM archive, SLA dashboard, cross-tenant audit search) — continue at
  their own cadence post-launch.
- **Cloud / hosted tier** (ADR-0017 D.3: Stripe, customer onboarding,
  multi-region DR) — separate Atelier-Cloud repo, post-launch.
- **AWP public registry** (`public.awp.dev`) — ADR-0036 Phase 1 ecosystem
  play; no blocking dependency on v1.0.
- **Pro tier infrastructure** (hosted model, private registry, NL
  generation without API key) — deferred until first paying Pro customer.
- **Trademark registration** — operator action, no code implication;
  recommended before the HN launch but not a code gate.
- **GmbH incorporation decision** — operator + legal; recommended before
  first €100k contracted revenue (ADR-0018 B.2).

---

## Sequencing summary

```
T1 (1 session)   — Release v0.14.0 — unblocks marketing claim today
     │
     ├── T2 (3–5 sessions, sequential per sub-task) ─────────────────┐
     │   2.1 CLAUDEOS_* env aliases (3 batches)                      │
     │   2.2 ~/.claudeOS/ fallback paths                             │
     │   2.3 systemd unit cleanup                                     │
     │   2.4 CLAUDEOS_SIGNAL dual-fire removal                       │
     │   2.5 docs final sweep                                         │
     │   2.6 git remote URL + WorkingDirectory                       │
     │                                                                │
     ├── T3 (2–3 sessions, parallel with T2) ──────────────────────┐ │
     │   3.1 SECURITY.md                                            │ │
     │   3.2 Privacy notice example                                 │ │
     │   3.3 README screenshots + GIF                               │ │
     │   3.4 Standalone landing page                                │ │
     │   3.5 GitHub repo hygiene (operator actions)                 │ │
     │                                                              │ │
     ├── T4 (2–3 sessions, parallel with T2+T3) ─────────────────┐ │ │
     │   4.1 setup.md                                             │ │ │
     │   4.2 troubleshooting.md                                   │ │ │
     │   4.3 Console access guide                                 │ │ │
     │   4.4 memory.md                                            │ │ │
     │   4.5 extending.md                                         │ │ │
     │   4.6 docs/README.md cleanup                               │ │ │
     │                                                            ▼ ▼ ▼
     └──────────────────────────────────────────── T5 (1 session)
                                                   5.1 Phase 7-8 cut
                                                   5.2 CHANGELOG v1.0
                                                   5.3 git tag v1.0.0
                                                   5.4 Launch draft
```

**Minimum elapsed time (working sessions only):**
- T1 alone: ½ day (can ship today as v0.14.0)
- T2 + T3 + T4 in parallel: 1 week (3 parallel tracks × 2–3 sessions each)
- T5 after all three: ½ day
- **Total: ~1.5 working weeks** to v1.0 tag + public announcement

**External dependencies not in this ADR:**
- Legal sign-off on EULA (ADR-0018 B.1) — not blocking OSS v1.0 but
  blocking commercial Enterprise tier
- DPIA DPO signature — operator action after pentest
- Trademark application — operator action; recommended before HN launch
- Domain registration / DNS for landing page — operator action

---

## Acceptance — when ADR-0046 is "Done"

ADR-0046 transitions from *Proposed* → *Accepted* when:

- `git describe --tags --abbrev=0` returns `v1.0.0`
- `grep -ric "claudeos" . --include="*.py" --include="*.js" --include="*.sh" | grep -v ".git\|CHANGELOG\|phase7" | awk -F: '{sum+=$2} END {print sum}'` returns `0`
- `SECURITY.md` exists at repo root
- `docs/setup.md` + `docs/troubleshooting.md` + `docs/console.md` exist
- `docs/assets/screenshots/` contains ≥3 screenshots
- A public HTTPS landing page exists and loads correctly
- `bash operator/bridges/run-all-tests.sh` exits 0

---

## References

- `docs/phase7-backlog.md` — Phase 7 task inventory (canonical source for T2)
- `docs/decisions/0018-adr-0017-public-launch-readiness.md` — enterprise tier roadmap
- `docs/decisions/IMPLEMENTATION-ROADMAP.md` — EU compliance M1–M6 (mostly done)
- `docs/decisions/ADR-001-opencode-eu-ai-act.md` — EU AI Act architecture review
- `CHANGELOG.md § [Unreleased]` — the large compliance build-out to be versioned in T1
- `core/console/atelier_console/web-next/src/pages/personas.tsx` — CG-1 (persona editing)
- `operator/bridges/bridge.sh` — systemd unit generation (CG-3)
- `docs/compliance/PRIVACY-NOTICE-TEMPLATE.md` — T3.2 source
- `docs/compliance/DSB-CHECKLIST.md` — companion to T3.2
- `README.md` — T3.3 + T3.5 target
- `docs/README.md § "Coming next"` — T4.6 target (remove stub after T4 complete)
