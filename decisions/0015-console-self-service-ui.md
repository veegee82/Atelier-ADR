# ADR-0015 — atelier-console: owner-self-service web UI

**Status:** Accepted (2026-05-15)
**Plugin:** `core/console/`
**Mount:** `/v1/console/*` (REST) + `/console/*` (vanilla-JS SPA), same gateway port
**Sister-ADR:** ADR-0014 (atelier-admin, operator-tier)

---

## Context

The bridge runtime (voice + cowork + forge + skill-forge) exposes
~30 slash-commands across five bridges (Telegram / Discord / Slack /
WhatsApp / Email) for inspection and configuration. Operators
typing `/whoami`, `/skills`, `/forge_list`, `/quota`, `/role`,
`/consent`, `/settings`, `/dialectic-status`, `/ldd-status` etc. into
chat windows is structurally serviceable but ergonomically poor:

- Each command lives in a different chat thread; cross-cutting
  questions ("which skills are active in which sessions and what's
  their grade-curve?") require manually correlating screenshots.
- Memory files (`~/.claude/projects/.../memory/`) are invisible to
  the bridge surface entirely.
- `voice-audit verify` is the only path to inspect the hash chain;
  `audit-tail` shows the last N events but no filter or live-tail.
- Forge tools and skill-forge skills are listed via MCP-tools, not
  through any UI an operator can scan at a glance.

**ADR-0014** introduced `atelier-admin` for operator-tier
(cross-tenant) compliance work. The owner-tier surface — for the
person who runs the AtelierOS instance for themselves — was
deferred.

This ADR covers that deferred surface.

---

## Decision

Ship `atelier-console`, a sister plugin to `atelier-admin`. Vanilla
JS SPA + FastAPI + SSE. Same Gateway port. Owner-tier
single-tenant auth via the existing `atlr_<32 hex>` bearer-token
class. **All compliance mechanisms (Disclosure / Consent / Audit /
Path-Gate / Engine-Policy / Zone-Residency) remain structurally
unchanged** — the console is a read-mostly projection with curated
mutation paths, never a bypass.

### Surface — 13 sections

| # | Section | Phase |
|---|---|---|
| 1  | Dashboard (health-card)             | A |
| 2  | Agents Live (SSE-stream)            | D |
| 3  | Sessions (bridge-session list)      | B |
| 4  | Audit Tail (hash-chain viewer)      | B + D-live |
| 5  | Runs (Gateway-runs)                 | B |
| 6  | Personas (bundle + user-override)   | B + C-detail + E-edit |
| 7  | Tools (Forge)                       | B + C-detail + E-promote |
| 8  | Skills (Skill-Forge)                | B + C-detail + E-promote |
| 9  | Memory (auto-memory store)          | C + E-edit |
| 10 | Compute Worker (Layer 25)           | F |
| 11 | Workspaces (FS tree-browser)        | F |
| 12 | Members (roles / quota / consent / disclosure) | F |
| 13 | Settings (6 known YAML/JSON files)  | F-read + E2-edit |

### Phases (chronological roll-out)

| Phase | What | Status |
|---|---|---|
| **A** | Plugin skeleton, owner-token auth, login/logout/whoami, dashboard | DONE |
| **B** | Read-only viewers: Sessions, Audit Tail, Runs, Personas | DONE |
| **C** | Drilldowns: Persona-detail, Tool-detail, Skill-detail, Memory-browser | DONE |
| **D** | Realtime SSE streams: Agents Live + Audit-Live | DONE |
| **E** | Mutations: Memory-edit, Persona-edit, Tool/Skill-promote (with CSRF + Re-Auth) | DONE |
| **F** | Compute Worker, Workspaces, Members, Settings (read-only) | DONE |
| **E2** | Settings editor with structural YAML/JSON validation + Re-Auth | DONE |
| **G** | Closure: /healthz, ADR, README, Layer-29 anchor in CLAUDE.md | DONE |

### Auth model

| Property | Choice |
|---|---|
| Bearer-token class | `atlr_<32 hex>` (37 chars) — the existing tenant-tier shape from ADR-0007 Phase 2.1 |
| Operator tokens (`atlr_op_*`) | Explicitly REJECTED at `/auth/login` — operator surface lives in atelier-admin |
| Cookie name | `atelier_console_sid` — distinct from admin so both UIs can run in the same browser |
| Session store | `<atelier_home>/global/console/sessions/<sid>.json`, mode 0o600 |
| Idle timeout | 1 h |
| Absolute timeout | 8 h |
| CSRF | per-session secret, HMAC-derived `X-CSRF-Token` header on every mutation; GET exempt (SameSite=Strict carries the structural defence) |
| Re-Auth | Sensitive mutations (Memory-Edit, Persona-Edit, Promote, Settings-Edit) require the bearer token typed fresh in the request body. Constant-time fingerprint compare against the session's `token_fingerprint`. |
| Rate limit | 10 login attempts / source-IP / 60s — same shape as admin-UI |

### Audit chain

Five new event types registered in `forge.security_events.EVENT_SEVERITY`:

```
console.session_started     INFO
console.session_ended       INFO
console.session_denied      WARNING
console.action_performed    INFO
console.action_failed       WARNING
```

All events go through `forge.security_events.write_event` into the
unified per-tenant chain at
`<tenant_home>/global/forge/audit.jsonl`. Cleartext SID, CSRF
secret, bearer token, memory body, persona JSON, settings file
content NEVER appear in audit details — emitter validates against a
curated allow-list per event-type at the boundary
(`AuditFieldNotAllowed` raised on smuggle-in).

### Compliance posture (load-bearing)

| Mechanism | Console preserves it by |
|---|---|
| EU AI Act Art. 50 (active disclosure) | Console is opt-in; operator-only. No bot-disclosure path runs through it. |
| GDPR Art. 6/7 (consent gate) | Console READS consent state via `/members`; never writes. Owner-side `/grant` / `/revoke` already lives in slash-commands. |
| GDPR Art. 30 (RoPA / audit chain) | Every mutation lands as `console.action_performed`, every failure as `console.action_failed`. Hash-chain integrity verified by `voice-audit verify` covers them. |
| GDPR Art. 32 (security of processing) | Bearer-token-only auth (no passwords, no JWT signing keys). Mutations gated by CSRF + Re-Auth. Path-gate hook (Layer 10) protects forge / skill-forge / audit / policy tree from any direct write — console mutations route through typed endpoints, never raw `Write` / `Bash`. |
| EU AI Act Art. 14 (human oversight) | Owner sees what their machine does in real-time via Agents-Live SSE. Quota / Consent / Role state is fully visible per chat. |
| EU AI Act Art. 14 / GDPR Art. 32 (zone residency, engine policy) | `tenant.atelier.yaml` is editable in Settings with structural YAML validation; the engine-policy and zone-residency gates in the dispatcher remain authoritative — console only changes the policy file. |

---

## Consequences

### What this enables

- A **single browser tab** that surfaces what was previously
  scattered across 30 slash-commands.
- **End-to-end visibility**: Agents-Live SSE shows in real-time
  which skill is being injected into which session, which forge
  tool is being called, when an audit-event lands in the chain.
- **Self-service**: the owner edits memory, promotes a skill,
  copies-and-edits a persona without touching CLI.
- **Mental-model consolidation**: 13 sections that mirror the
  layer architecture (sessions / personas / tools / skills /
  memory / compute / workspaces / settings / members) make the
  system tractable to a new operator inside the first hour.

### What it does NOT do (deliberately)

- **No cross-tenant view.** Owner sees ONLY their tenant. Cross-
  tenant rollups belong to atelier-admin.
- **No operator-tier endpoints.** OIDC config, SCIM provisioning,
  tenant CRUD all live in atelier-admin.
- **No bridge-message-send.** The console is for inspection +
  configuration, not for chatting through a web UI.
- **No evidence-bundle export.** atelier-admin already has that
  surface (Phase 14.7); duplicating it in console is unnecessary.
- **No npm / build step.** Vanilla JS by contract — same as
  admin-UI. Operators who want a fancier frontend layer it on
  themselves.
- **No websockets.** SSE handles every realtime need; websockets
  would add a parallel auth surface and a parallel bug surface.

---

## Architecture summary

```
core/console/
├── bootstrap.sh              # opt-in venv setup
├── requirements.txt          # fastapi + pydantic + httpx + uvicorn
│                             # + itsdangerous + pyyaml + pyjwt
├── atelier_console/
│   ├── __init__.py           # __version__ = "0.1.0"
│   ├── app.py                # FastAPI router + /healthz + /version + mount_static
│   ├── auth.py               # session store (mirror of admin auth.py with
│   │                         #   distinct cookie + sessions dir)
│   ├── audit.py              # 5 event-type emitters; metadata-only allow-list
│   ├── deps.py               # require_session, require_csrf, verify_reauth
│   ├── routes/
│   │   ├── auth_routes.py    # /login, /logout, /whoami; tenant-tier only
│   │   ├── dashboard.py      # health-card data
│   │   ├── sessions.py       # bridge-sessions list
│   │   ├── audit_tail.py     # GET /audit/tail (filtered)
│   │   ├── runs.py           # gateway-runs list
│   │   ├── personas.py       # list + detail + PUT (user-only) + copy-from-bundle
│   │   ├── tools.py          # list + detail + impl preview
│   │   ├── skills.py         # list + detail + grades + body
│   │   ├── memory.py         # index + GET + PUT + DELETE
│   │   ├── streams.py        # SSE /audit/stream (byte-cursor tail)
│   │   ├── promote.py        # POST /tools/{name}/promote, /skills/{name}/promote
│   │   ├── workspaces.py     # FS tree-browser
│   │   ├── members.py        # roles + quota + consent + disclosure fan-in
│   │   ├── compute.py        # Layer 25 worker probe + run-list
│   │   └── settings.py       # 6 known YAML/JSON files; GET + PUT (E2)
│   └── web/                  # vanilla-JS SPA
│       ├── index.html
│       ├── app.js            # ~900 lines, hash-routed router
│       ├── style.css         # ~250 lines
│       └── dev-login.html    # gitignored, hardcoded token, localhost dev
└── tests/                    # placeholder; integration tests deferred
```

### Wire-format summary

| Method | Path | Purpose |
|---|---|---|
| GET  | `/v1/console/version`              | Plugin version (unauth) |
| GET  | `/v1/console/healthz`              | Liveness probe (unauth) |
| POST | `/v1/console/auth/login`           | Issue session-cookie + CSRF token |
| POST | `/v1/console/auth/logout`          | Clear session |
| GET  | `/v1/console/auth/whoami`          | Inspect current session |
| GET  | `/v1/console/dashboard`            | Health-card payload |
| GET  | `/v1/console/sessions`             | All bridge-sessions |
| GET  | `/v1/console/audit/tail?...`       | Filtered audit-tail (poll) |
| GET  | `/v1/console/audit/stream?...`     | SSE audit-tail (live) |
| GET  | `/v1/console/runs`                 | Gateway runs |
| GET  | `/v1/console/personas`             | List |
| GET  | `/v1/console/personas/{name}`      | Detail (full body) |
| PUT  | `/v1/console/personas/{name}`      | Edit (user-scope only); CSRF + Re-Auth |
| POST | `/v1/console/personas/{name}/copy-from-bundle` | Bundle → user copy |
| GET  | `/v1/console/tools`                | List (multi-scope aggregation) |
| GET  | `/v1/console/tools/{name}`         | Detail + impl preview |
| POST | `/v1/console/tools/{name}/promote` | Forge-promote; CSRF + Re-Auth |
| GET  | `/v1/console/skills`               | List |
| GET  | `/v1/console/skills/{name}`        | Detail + body + grades |
| POST | `/v1/console/skills/{name}/promote`| Skill-Forge-promote; CSRF + Re-Auth |
| GET  | `/v1/console/memory`               | Auto-memory file index |
| GET  | `/v1/console/memory/{name}`        | File body |
| PUT  | `/v1/console/memory/{name}`        | Write; CSRF + Re-Auth |
| DELETE | `/v1/console/memory/{name}`      | Delete (MEMORY.md protected); CSRF + Re-Auth |
| GET  | `/v1/console/workspaces?depth=N`   | FS tree |
| GET  | `/v1/console/members`              | All chats with state |
| GET  | `/v1/console/members/{chat_key}`   | Per-uid drilldown |
| GET  | `/v1/console/compute`              | Worker probe + run-list |
| GET  | `/v1/console/compute/runs/{run_id}`| Manifest + summary |
| GET  | `/v1/console/settings`             | 6 known config files |
| PUT  | `/v1/console/settings/{label}`     | Edit; CSRF + Re-Auth + structural validation |

---

## What you, as Claude Code, must NOT do (console rules)

- **Don't add an endpoint that bypasses CSRF or Re-Auth on a
  mutation.** The two-layer protection (`require_csrf` for every
  write + `verify_reauth` for sensitive writes) is the structural
  floor; bypassing it in any new route silently breaks the
  Compliance posture above.
- **Don't accept operator-tier tokens (`atlr_op_*`) at the console
  login.** They live in atelier-admin; merging the surfaces would
  collapse the tier-separation that lets each plugin's audit-chain
  posture stay clean.
- **Don't put the cleartext SID, CSRF secret, bearer token, memory
  body, persona JSON or settings file content into any audit-event
  detail field.** The `_FORBIDDEN_FIELDS` set + per-event
  `_ALLOWED_FIELDS` allow-list in `console_audit._emit` is the
  enforcement; tests verify it doesn't drift.
- **Don't widen the protected-file set on `Settings`.** Six labels
  are the contract: `tenant.atelier.yaml`, `data_policy.yaml`,
  `ldd.json`, `dialectic.json`, `relay.json`, `branding.yaml`.
  Adding files needs an ADR amendment because each new file
  potentially touches a Compliance gate.
- **Don't add a 7th file to settings without enforcing structural
  validation per kind.** YAML and JSON each have one parse-call;
  unknown kinds default to "permissive" which silently lets a
  broken file land on disk. Always add the `_validate_body` branch
  in the same commit that adds the file.
- **Don't drop the `MEMORY.md` protection on `DELETE /memory/{name}`.**
  The index file is structurally distinct — its absence orphans
  every typed entry's pointer.
- **Don't accept a Persona PUT against a bundle persona without a
  user-override already on disk.** The bundle integrity is
  structural; the documented escape hatch is
  `POST /personas/{name}/copy-from-bundle`.
- **Don't auto-create the `<atelier_home>/global/console/` tree
  outside `auth.create_session`.** The session module is the sole
  owner of the subtree; manual `mkdir` from other endpoints would
  race with concurrent provisioning.
- **Don't make the Agents-Live SSE stream open-ended.** The
  `max_seconds=1800` cap is the load-bearing safety against
  per-connection cursor-drift; the SPA reconnects via EventSource
  automatically.
- **Don't add an endpoint that writes to the audit chain on a
  GET.** Read endpoints are read-only by contract; emitting on
  every page-load saturates the chain.
- **Don't introduce `prometheus_client` or websockets.** The
  metrics path is admin-UI's concern (Phase 14.5 + ADR-0007 Phase
  6). The realtime path is SSE-only.
- **Don't ship the dev-login.html file in commits.** It's
  gitignored and contains a plaintext token; it's a localhost-dev
  convenience only.

---

## References

- `outputs/atelier-console-konzept.md` — original design document
  (12 sections + 7 open questions)
- ADR-0007 Phase 2.1 — tenant-tier bearer-token store (atelier_gateway.auth)
- ADR-0014 — atelier-admin operator-UI (parallel sister)
- Layer 6 (forge), Layer 7 (skill-forge), Layer 10 (path-gate),
  Layer 14 (LDD), Layer 16 (security hardening), Layer 18 (roles),
  Layer 20 (quota + audit-view), Layer 25 (compute) — every
  console section is a read-projection of one or more of these
  layers.
