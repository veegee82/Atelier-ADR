# ADR-0037 — Console Re-Launch & Operator-UI Unification

**Status:** Proposed (2026-05-17)
**Plugin:** `core/console/` (existing)
**Mount:** `/v1/console/*` (REST, unchanged) + `/console/*` (SPA, **new build artifact**), same gateway port (8765)
**Sister-ADR:** ADR-0014 (atelier-admin), ADR-0015 (console v1, vanilla SPA), ADR-0034 (operator bundle)

---

## Context

ADR-0015 shipped `atelier-console` as a vanilla-HTML/CSS/JS SPA with 17
backend route modules covering Personas, Tools, Skills, Memory, Audit,
Settings, Sessions etc. The backend is mature; the frontend is
ergonomically functional but visually undistinguished — it does not
present AtelierOS as the polished, inviting *single front door* the
operator-bundle deserves.

Three concrete gaps motivate this re-launch:

1. **Visual identity.** The current SPA is utility-styled. The operator
   bundle (12 personas, 7 bridges, voice, forge, skill-forge, cowork,
   LDD layers) is the product surface most outsiders touch first — and
   it looks like an internal tool, not an *atelier*.

2. **Plugin / bundle coverage is split across surfaces.** Bridge
   settings live in `operator/bridges/<channel>/settings.json` edited
   on disk; voice profile is `/voice-user-*` chat-commands; LDD layer
   toggles are `/ldd-set`. There is no one place where an operator
   sees the whole configurable surface at a glance.

3. **No conversational entry point.** The console is read+config; the
   only way to *talk to* AtelierOS from a browser is through one of
   the six bridge channels (Telegram/Discord/Slack/WhatsApp/Email/Signal).
   Operators on a desktop session have no first-party messenger.

The owner wants: one web UI, visually polished, that owns the whole
bundle + plugin configuration surface AND lets them chat with the
running engine (voice in / voice out, sessions, slash-commands) —
running on boot, opened automatically in the browser on desktop login.

---

## Decision

Re-launch the console frontend in **Vite + React (TypeScript) +
Tailwind + shadcn-react** under `core/console/atelier_console/web-next/`,
served from `web-next/dist/` after `npm run build`. The 17 existing
backend route modules stay byte-identical; this is a frontend rewrite
only. A new bridge channel `web` is added under `operator/bridges/web/`
for in-browser chat. A new system-wide systemd unit and an
`/etc/xdg/autostart/` entry make the UI always-on and
opened-on-desktop-login.

### Stack

| Layer | Tech | Rationale |
|---|---|---|
| Build | Vite 5 | fast dev-server, esbuild prod build, tree-shaken |
| UI | React 18 + TypeScript 5 | mature, large component ecosystem |
| Styles | Tailwind CSS 3 | utility-first, easy theme tokens, dark+light via `data-theme` |
| Components | shadcn-react (Radix primitives) | accessible, copy-into-repo (not a npm runtime dep) |
| Routing | react-router-dom 6 | data-router, SSR-not-needed since FastAPI serves index.html |
| Forms | react-hook-form + zod | matches FastAPI/pydantic schema validation |
| Data | TanStack Query 5 | cache + retry + suspense fits owner-self-service load |
| Icons | Lucide-react | tree-shaken, matches shadcn defaults |
| Testing | vitest (unit) + Playwright (E2E) | aligns with existing console pytest E2E pattern |

### Visual identity — "Atelier"

- **Palette:** Off-White `#FBF7F0`, Deep Navy `#0E1320`, Brass-Gold accent
  `#C8A24A`, neutral mid-grays. Dark theme inverts background/foreground;
  accent stays warm.
- **Typography:** Inter (sans, body + UI) + Fraunces (serif, only Hero + H1).
- **Tone:** ruhig, warm, "Werkstatt" — no SaaS-purple-gradients, no
  generic stock-illustration.

### Backend integration contract

- `mount_static(app, url_prefix="/console")` in `atelier_console/app.py`
  is extended with an env-toggle:
  - `ATELIER_CONSOLE_UI=next` (default once Iteration 1 ships and `dist/`
    exists) → mount `web-next/dist/`
  - `ATELIER_CONSOLE_UI=legacy` → mount the existing `web/` directory
    (escape hatch during migration)
  - Both directories ship in the repo until ADR-0037 Phase 4 completes;
    `web/` is deleted after the 14-day soak and an explicit operator ack.
- No API changes. The new SPA is a strict consumer of the 17 existing
  route modules + the two new endpoints introduced in Iteration 2/3
  (landing-personas, web-chat WebSocket).
- Auth is unchanged: cookie `atelier_console_sid`, CSRF via
  `X-CSRF-Token` (HMAC-derived per ADR-0015).

### Iteration plan

| Iteration | Scope | Status |
|---|---|---|
| 1 | ADR + frontend foundation + Landing + Dashboard | **shipped** (PR #3) |
| 2 | Settings modules (Personas, Bridges, Voice, Forge, SkillForge, Cowork, LDD, Compliance) | **shipped** (PR #3 + PR #4) |
| 3 | Web-bridge channel + in-browser Messenger + Voice (req/resp) | **shipped v1 minimal** (PR #4); see § "Iteration 3a v1 scope" below |
| 4 | systemd-system unit + xdg/autostart + hardening | **shipped** (PR #3) |
| 3* (follow-up) | Full bridge-adapter integration for the `web` channel — hooks, path-gate, audit hash-chain, persona routing, engine selection, /btw inject | queued — ADR-0037 amendment will track scope |

### Iteration 3a v1 scope (explicit non-goals)

The first delivery of the web bridge channel uses an
`asyncio.create_subprocess_exec` spawn of `claude -p --output-format stream-json`
inside `chat_runtime.py`, with `--continue` for per-workdir session
resume. Authentication uses the same console session cookie. The
following are deliberately **not** wired up in v1 — they require
extracting parts of `operator/bridges/shared/adapter.py` (5,300 lines)
into a re-usable runtime, which is a multi-day refactor and lands as
an ADR-0037 amendment:

- persona pinning + auto-routing (always uses adapter defaults today)
- mid-stream `/btw` inject
- multi-engine dispatch (claude only; codex/opencode TBD)
- compliance hash-chain integration (v1 writes to a separate
  `web_chat.jsonl` to make the gap obvious)
- consent / disclosure / quota / observer-transcript gates

These are honored by every other bridge channel and MUST be honored
by `web` before it is considered production-grade. The v1 ships now
so the operator's first-look UX is a polished messenger, not a
"coming soon" tile.

#### Iter 3a addendum — session titles (2026-05)

The sidebar shows a human-readable label per session instead of the raw
22-char `sid`. Two paths produce a title; both write to the same
`title` field already persisted in `<sid>.json`:

1. **Auto-title on first user turn** — `chat_runtime._derive_auto_title()`
   takes the first non-empty line of the prompt, collapses whitespace,
   trims at a word boundary near 60 chars, strips trailing punctuation.
   Pure-Python heuristic — no LLM call, no helper-model dependency
   (an LLM-summariser stays out of v1 scope; if added later it becomes
   a separate iteration). The gate only fires when `turn_count == 0`
   AND `title` is empty, so manual renames are sticky.
2. **Manual rename** — `PATCH /v1/console/chat/sessions/{sid}` body
   `{"title": "..."}`, CSRF-protected. Frontend offers double-click /
   pencil-icon inline edit with optimistic update.

The `title` field cap stays at 120 chars (matches `CreateSessionRequest`
schema). Auto-derived titles cap at 60 to keep the sidebar readable.
The audit log gains nothing new — title is operator-supplied UI state,
not a compliance-relevant event.



### New bridge channel `web` (Iteration 3 preview)

A new daemon under `operator/bridges/web/` exposes a WebSocket on the
gateway (`ws://…/v1/console/chat/<sid>`) that funnels into the existing
adapter the same way Telegram/Discord/Signal do. Session keys take the
shape `web:<sid>`; the channel honours the full Layer-8 reset lifecycle,
Layer-10 path-gate, Layer-16 consent gate, Layer-19 disclosure card,
Layer-20 quota. Voice in/out is request/response: browser-recorded
audio (MediaRecorder API) → `POST /v1/console/voice/transcribe` →
existing `operator/voice/scripts/stt/` chain; assistant text → optional
`POST /v1/console/voice/tts` → audio blob played by an HTML `<audio>`
element.

### Always-on + auto-open (Iteration 4 preview)

- `/etc/systemd/system/atelier-operator-ui.service` (NEW), `User=atelier`,
  `WantedBy=multi-user.target`, `Restart=always`,
  `ExecStart=… uvicorn atelier_gateway.app:app --host 127.0.0.1 --port 8765`.
- `/etc/xdg/autostart/atelier-operator-ui.desktop` (NEW),
  `Exec=xdg-open http://127.0.0.1:8765/` — fires for every desktop user.
- The existing `core/gateway/systemd/atelier-webui.service` (user-mode,
  `default.target`) remains for development use; the new system-wide
  unit is the production path.

---

## Non-Goals

- **No auth-system change.** Owner-tier `atlr_<32 hex>` bearer remains.
  OIDC stays an atelier-admin concern.
- **No backend rewrite.** Routes, dependency-injection, audit schema
  unchanged.
- **No replacement for atelier-admin.** Cross-tenant operator work
  remains in `core/admin/`. The console deliberately surfaces
  *single-tenant owner* concerns.
- **No bundling of additional engines into the web bridge.** The web
  channel uses whichever engine the chat's persona picks via the
  existing engine-layer dispatch (`ATELIER_USE_ENGINE_LAYER`).
- **No streaming voice (WebRTC) in Iteration 3.** Request/response
  only; WebRTC may follow in a separate ADR if latency demands it.

---

## Consequences

**Positive.**
- One visually coherent front door for the whole operator bundle.
- Browser-first chat removes the "I need Telegram on this device" friction.
- Always-on systemd + auto-open materially lowers the activation cost.
- Frontend rewrite isolated from backend means a feature-flag flip is
  the only switchover cost.

**Costs.**
- npm + Node toolchain become required on machines that build the UI
  (CI + maintainer's box). Production deploy is the pre-built
  `dist/` directory — no Node at runtime.
- Two SPAs (legacy `web/` + new `web-next/dist/`) coexist for the
  migration window; ~50 KB extra repo weight until Phase 4 deletes
  the legacy.
- shadcn components are copy-in (not a runtime dep), so they live in
  `web-next/src/components/ui/` and are owned by this repo for
  security review / patching.

**Risks.**
- Frontend complexity rises. Mitigation: TanStack-Query for cache,
  zod schemas mirrored from pydantic models, Playwright E2E per
  settings module before declaring the module done.
- The new web bridge is a fresh attack surface. Mitigation: it
  re-uses the existing adapter (path-gate, consent, audit) without
  any bypass; the WebSocket endpoint shares the console cookie-auth.
- Auto-open in `/etc/xdg/autostart/` is system-wide and may surprise
  multi-user systems. Mitigation: install script asks before placing
  the `.desktop` file, and an opt-out env (`ATELIER_NO_AUTOSTART=1`)
  is honoured by the autostart hook.

---

## Compliance review (EU AI Act 2026 + GDPR)

| Mechanism | Affected by ADR-0037? | Action |
|---|---|---|
| Bot-disclosure card (Art. 50) | Yes — `web` channel must emit it on first turn | Iteration 3: web bridge sends disclosure-card via Layer-19 path |
| Per-user consent gate (Art. 6/7) | Yes — `web:<sid>` is a uid | Iteration 3: `/consent` works in web channel |
| Hash-chained audit log (Art. 30/32) | Yes — every settings mutation + chat turn | No change: existing `console_audit` + adapter audit already emit |
| Compliance-zone routing (Art. 14) | No | unchanged |
| Engine-policy allowlist | No | unchanged |
| Secret-vault capability split | No | UI shows masked values + reveal-button gated by re-auth |
| Path-gate hook (L10) | No | path-gate fires regardless of channel |
| Voice-transcribe metadata-only | Yes — web `/voice/transcribe` must conform | Iteration 3: must NOT include transcript in audit |

**No structural-compliance guarantee is weakened by this ADR.**

---

## Migration / rollback

- Iteration 1 ships `web-next/dist/` but leaves `web/` as default.
  Switch is a single env-var.
- Rollback at any point: set `ATELIER_CONSOLE_UI=legacy`, restart unit.
- After 14-day soak with `web-next/dist/` as default and zero
  console-side errors, `web/` is deleted (Phase 4 follow-up commit).

---

## References

- ADR-0014 (`atelier-admin`)
- ADR-0015 (console v1, vanilla SPA — superseded by this ADR for the
  frontend layer only)
- ADR-0017 (Enterprise control plane phases I–VI)
- ADR-0034 (operator bundle structure)
- `core/console/atelier_console/app.py::mount_static`
- `core/console/atelier_console/auth.py::COOKIE_NAME`
- `operator/cowork/personas/*.json` (12 personas — Landing-page source)
