# ADR-0014 — Admin-UI Plugin (Operator-facing Web Console, Opt-In)

**Status:** Accepted (2026-05-14 — all 8 sub-phases shipped, 163 tests green)
**Date:** 2026-05-14
**Companion to:** ADR-0007 (Multi-Tenant Gateway), ADR-0008
(Bridges-state out-of-repo), ADR-0009 (Declarative Tenant Surface),
ADR-0010 (Operator Observability Surface), Layer 10 (path-gate),
Layer 16 (Security hardening, esp. consent + audit chain), Layer 18
(role-based delegation), Layer 20 (quota + audit-view).
**Implementation plan:** `0014-implementation-plan.md` (TBD)

## Context

AtelierOS today is a **CLI- and slash-command-shaped** system. Every
operator-facing surface — tenant provisioning, token issuance, audit
chain inspection, webhook secret rotation, bridge settings,
skill/forge management, quota review — is reachable only via:

1. Shell commands (`python -m atelier_gateway.cli ...`,
   `voice-audit verify`, `bridge.sh up/down/restart`).
2. Slash-commands inside a connected chat (`/grant`, `/quota`,
   `/role`, `/consent`, `/audit me`, `/settings`).
3. Direct YAML / JSON edits to `tenant.atelier.yaml`,
   `data_policy.yaml`, `bridges/<channel>/settings.json`, etc.

This is sufficient for the single-operator deployment that
AtelierOS started as, and it is intentional for the audit-defensible
posture (every state-change is recorded by a deterministic CLI call
or a daemon hot-reload, both of which land in the unified hash
chain). But it is **insufficient for the product step**:

- **Non-operator tenant admins** (a customer's compliance officer,
  a tenant's HR lead, a workshop coordinator) cannot use AtelierOS
  without shell access to the operator's host.
- **Beta-customer onboarding** requires the operator to manually
  walk through `tenant init`, token issuance, OIDC config, webhook
  secret upload, bridge pairing — at minimum 20 minutes of CLI per
  tenant.
- **Audit-chain inspection** is read-line-by-line via
  `audit.jsonl`. For a real compliance review (a DPA audit, a SOC2
  walkthrough, a GDPR Art. 15 subject-access request) the operator
  needs filtered views, full-text search across event types, and
  exportable evidence bundles.
- **Live run monitoring** exists in the protocol (SSE on
  `/v1/tenants/{tid}/runs/{rid}/events`, ADR-0007 Phase 2.5) but
  has no human-facing renderer.

Three constraints from the compliance baseline (CLAUDE.md
`## Compliance baseline`) shape the solution:

- **The Admin-UI must not bypass any structural compliance
  mechanism.** Every gate that protects the CLI / slash-command
  surface must equally protect the HTTP surface (path-gate,
  audit-chain, consent gate, role-based delegation, engine /
  zone policy, secret vault). The UI is a render-layer over the
  same primitives.
- **Every admin action is an audit event.** Operator clicks on a
  "Revoke token" button must produce the same audit-chain entry as
  the CLI `python -m atelier_gateway.cli token revoke ...` does,
  plus a session-binding event that ties the action to the
  authenticated browser session.
- **No new external dependencies on the LLM side.** The admin is
  pure operator-facing infrastructure. It MUST NOT
  `import anthropic` (mirror of `dialectic.py` and Layer 25); any
  LLM-shaped feature inside the UI (e.g. "explain this audit
  event") goes through `claude -p` subprocess and is opt-in per
  tenant.

## Decision

A new **opt-in plugin** at `core/admin/` that ships:

1. A **FastAPI router** mounted under `/v1/admin/*` on the existing
   `atelier-gateway` ASGI app (no second server, no second port,
   no second auth surface).
2. A **SvelteKit single-page application** built to static files
   and served by the same ASGI app from `/admin/`.
3. A **session-cookie auth bridge** that wraps the existing
   `atlr_`-bearer-token contract for browser ergonomics, including
   CSRF protection, idle-timeout, and forced re-auth for sensitive
   mutations.

Default state of a fresh AtelierOS install: **plugin not
bootstrapped, routes not mounted, static bundle not built**.
Operator opt-in is a two-step process (`bootstrap.sh` plus a
config flag in `tenant.atelier.yaml`), mirroring the
`atelier-gateway` and `atelier-compute` patterns.

### A) Plugin-level separation

The plugin lives under `core/admin/` with its own venv,
own `bootstrap.sh`, own test suite skipped from
`run-all-tests.sh` when the venv is absent.

```
core/admin/
├── bootstrap.sh                # venv + npm install + frontend build
├── requirements.txt            # FastAPI extras + python-jose (CSRF + cookies)
├── package.json                # SvelteKit + Tailwind + shadcn-svelte
├── atelier_admin/
│   ├── __init__.py
│   ├── app.py                  # APIRouter mounted at /v1/admin
│   ├── auth.py                 # Cookie ↔ bearer-token bridge + CSRF
│   ├── audit.py                # admin-specific event emitters
│   ├── routes/
│   │   ├── tenants.py          # CRUD on tenant.atelier.yaml
│   │   ├── tokens.py           # issue / revoke / list (wraps auth.py)
│   │   ├── webhooks.py         # secret rotation (wraps webhooks.py)
│   │   ├── audit_chain.py      # paginated + filtered view
│   │   ├── runs.py             # list + status + SSE proxy
│   │   ├── quota_roles.py      # quota dashboards + role grants
│   │   ├── skills_tools.py     # forge + skill-forge browsers
│   │   ├── settings.py         # bridges/<channel>/settings.json edit
│   │   └── metrics.py          # embeds Prometheus / Grafana panels
│   └── static/                 # frontend build output (gitignored)
├── frontend/
│   ├── src/                    # SvelteKit source
│   ├── tests/                  # Playwright headless E2E
│   └── ...
├── tests/                      # pytest (FastAPI TestClient)
└── README.md
```

The plugin's Python code MUST NOT `import anthropic`. A CI lint in
`tests/test_no_anthropic_import.py` walks the AST and fails on
violations.

### B) Authentication model — two-tier, session-bound

The existing bearer-token contract (ADR-0007 Phase 2.1) issues
**per-tenant tokens** (`atlr_<32 hex>`) that authorise operations
against exactly one tenant. The admin needs a second tier:

| Tier | Token shape | Authority |
|---|---|---|
| **Tenant-admin** | existing `atlr_<32 hex>` | Operations on one tenant's resources |
| **Operator** | new `atlr_op_<32 hex>` | Cross-tenant view + tenant CRUD |

Operator tokens live in a separate store at
`<atelier_home>/global/admin/operator_tokens.json` (mode `0o600`,
sha256-only on disk, fingerprint-only in audit events) — the
same shape as the tenant-token store, with a distinct file so the
two trust domains cannot accidentally cross.

**Session-cookie flow:**

1. `POST /v1/admin/auth/login` with body `{token: "atlr_..."}` over
   HTTPS. Server resolves the token, mints a session record at
   `<atelier_home>/global/admin/sessions/<sid>.json` carrying
   `{token_fingerprint, tier, tenant_id?, csrf_secret, expires_at}`.
2. Response sets `Set-Cookie: atelier_admin_sid=<sid>;
   HttpOnly; Secure; SameSite=Strict; Max-Age=3600`.
3. Every subsequent admin request reads the cookie, loads the
   session, re-validates the underlying token (defence against
   "operator revoked the token but session still alive"), and
   verifies an `X-CSRF-Token` header on mutations against the
   session's CSRF secret.
4. Idle timeout: 1 hour from last activity. Absolute timeout: 8
   hours. Re-auth required for **sensitive mutations** (token
   issue/revoke, webhook secret rotation, OIDC config edit,
   tenant deletion) via a freshly-typed token in the same
   request body (`re_auth_token`).

Cookies are never set on non-HTTPS connections. Local dev uses a
mkcert-signed local CA documented in the README.

Every login emits `admin.session_started`; every logout emits
`admin.session_ended`. Every mutation emits
`admin.action_performed` with `action`, `target_tenant_id`,
`target_resource_kind`, `target_resource_id`, plus the
session-fingerprint linking it to the login. Read-only requests
emit nothing (mirror of the metrics-endpoint contract from Phase
6).

### C) Surface — what the UI exposes

| Section | Backend source (exists today) | UI affordance |
|---|---|---|
| **Tenants** | `tenant_config.py`, migration helper | CRUD form, zone + engine-allowlist editor, budget block editor |
| **Tokens** | `auth.py` + admin-tier extension | Issue (one-time plaintext reveal), revoke, list with last-seen |
| **Webhooks** | `webhooks.py` | Secret set/rotate/revoke, test-fire button hitting the operator's URL |
| **OIDC + SCIM** | `oidc.py`, `scim.py` | Trust-file editor, JWKS-URI test, SCIM user browser |
| **Audit chain** | `security_events.py` + `audit_metrics.py` | Filtered timeline (event-type / severity / time-range / actor), CSV/JSON export, chain-verify badge, evidence-bundle download |
| **Runs** | `dispatcher.py` + SSE endpoint | List + status, live event-stream view, retry / abort buttons |
| **Quota** | `quota.py` + Prometheus | Per-user usage, set-limit form, reset button |
| **Roles** | `roles.py` | Grant / revoke / leave, capability-bundle editor |
| **Consent + Disclosure** | `consent.py`, `disclosure.py` | Per-(channel, chat) viewer, revoke-by-uid |
| **Skills / Tools** | Forge + Skill-Forge registries | Per-scope browser, promote / purge, manifest viewer |
| **Bridge settings** | `bridges/<channel>/settings.json` | Form-based editor, hot-reload trigger, dry-run validate |
| **Metrics** | Phase 6 Prometheus + Grafana | Embedded Grafana iframes (URL configured via tenant settings) |
| **Compute** | Layer 25 worker | Active runs, Top-K view, abort |
| **Data handles** | Layer 24 atelier_data | Registered handles, snapshot preview (already redacted), unregister |

### D) Frontend stack

- **SvelteKit** (SSR-capable, builds to static or to a Node
  adapter — we use static + FastAPI delivery so there is exactly
  one server process to operate).
- **TailwindCSS** + **shadcn-svelte** for the component library.
- **Playwright** for headless E2E (browser navigation, form
  submission, table sorting, role-based view enforcement).
- **No global state library.** SvelteKit's stores cover everything
  the surface needs; introducing Redux / Zustand / NanoStores adds
  drift between client state and server-side reality, which is
  exactly the bug class an audit-defensible UI must avoid.

Bundle target: **< 250 KB gzipped**. Operators install AtelierOS on
self-hosted boxes that may not have wide bandwidth; a 5 MB SPA is
disqualifying.

### E) Audit-event taxonomy

Six new event types registered in `security_events.EVENT_SEVERITY`:

| Event | Severity | Carries |
|---|---|---|
| `admin.session_started` | INFO | `tier`, `tenant_id?`, `token_fingerprint`, `user_agent_class` |
| `admin.session_ended` | INFO | `sid_fingerprint`, `reason` (logout / idle / absolute / revoked) |
| `admin.session_denied` | WARNING | `reason` (bad-token / expired / csrf-mismatch / re-auth-required) |
| `admin.action_performed` | INFO | `action`, `target_kind`, `target_id`, `sid_fingerprint` |
| `admin.action_failed` | WARNING | same as `action_performed` + `reason` |
| `admin.export_generated` | INFO | `export_kind`, `bytes`, `event_count`, `sid_fingerprint` |

`token_fingerprint` is the first 8 chars of the sha256, mirroring
Phase 2.1. `sid_fingerprint` is the first 12 chars of the session
ID. Neither the cleartext token nor the cleartext session ID ever
lands in the chain.

The `User-Agent` header is **classified** before audit (one of
`browser-desktop`, `browser-mobile`, `cli-curl`, `cli-other`,
`unknown`); the raw string never enters the chain. Reasoning:
operators want forensic correlation ("the breach came from a
mobile browser at 03:00") without storing a fingerprint-grade
identifier that re-identifies users later.

### F) Path-gate extension

Layer 10's path-gate (Layer 10) gains four new protected globs:

| Glob | Reason |
|---|---|
| `<atelier_home>/global/admin/operator_tokens.json` | Operator-token store; only the admin auth.py may write |
| `<atelier_home>/global/admin/sessions/**` | Session records; LLM-side overwrite would let an admin be impersonated |
| `<atelier_home>/global/admin/csrf_secrets.json` | Per-session CSRF secrets |
| `core/admin/atelier_admin/static/**` | Built frontend; an LLM overwriting it would inject script |

Plus four new `_PROTECTED_HINTS` strings (`operator_tokens`,
`admin/sessions`, `csrf_secrets`, `admin/static`) so fail-closed
Bash detection trips on `eval` / heredoc references even when the
actual write target is opaque.

### G) Prometheus metric families

Three new metric families on the Phase 6 `/v1/tenants/{tid}/metrics`
endpoint:

| Metric | Type | Labels |
|---|---|---|
| `atelier_admin_sessions_active` | gauge | `tier` |
| `atelier_admin_actions_total` | counter | `action`, `outcome` |
| `atelier_admin_session_denials_total` | counter | `reason` |

Label allow-lists:
- `tier`: `tenant-admin` / `operator`.
- `action`: curated set of the ~30 admin actions (e.g.
  `token.issue`, `token.revoke`, `webhook.rotate`,
  `tenant.update`, `quota.set`, `role.grant`). Values outside the
  set collapse to `"other"`.
- `outcome`: `ok` / `denied` / `failed`.
- `reason`: `bad-token` / `expired` / `csrf-mismatch` /
  `re-auth-required` / `cross-tenant` / `rate-limited`.

## Consequences

### Positive

1. **Tenant admins reach AtelierOS without shell access.** The
   compliance-officer use case ("show me every consent event for
   user X in the last 90 days") moves from "operator runs grep"
   to "tenant admin clicks two filters and exports CSV".
2. **Onboarding latency collapses.** A scripted setup-wizard
   (Phase 4 of the implementation plan) turns the 20-minute CLI
   walk-through into a 5-minute browser flow.
3. **Audit-chain becomes visibly load-bearing.** Operators who
   never ran `voice-audit verify` will see the chain-verify badge
   on every UI page and start trusting (and demanding) it.
4. **Compliance evidence bundles become first-class artifacts.**
   The "Export" button produces a signed `.atelier-pkg` carrying
   audit events + a `verify_chain` proof, ready to hand to a DPA
   auditor without operator intervention.

### Negative

1. **Surface area grows.** Every admin route is a new entry point
   that needs auth, CSRF, audit emission, role-check, rate-limit.
   The implementation-plan budget must reflect this — a
   "list tenants" endpoint is ~5 LoC, but its full belt-and-
   braces costs ~80 LoC of guards.
2. **Frontend skill load.** Maintaining a SvelteKit app adds a
   second language (TypeScript) and a second build pipeline (npm)
   to a project that is otherwise pure Python + a thin Node
   bridge layer. Mitigation: bundle target stays minimal; no
   client-side rendering of any compliance state; every audit
   decision happens server-side.
3. **Session store grows.** Every login writes a file; idle
   sessions accumulate until the cleanup sweep prunes them. Bound
   the absolute lifetime to 8 hours and add a daily sweep
   (mirror of `session_timeout_sweep.py` from Layer 8).

### Neutral

- The admin is **opt-in**; operators who do not bootstrap it pay
  zero cost (no routes mounted, no static files served, no
  metrics emitted).
- Headless E2E via Playwright runs against a real uvicorn process
  on an ephemeral port — same fixture pattern as the Phase 2.6
  smoke suite.

## Implementation plan (sub-phase fanout)

The full sub-phase plan lives in `0014-implementation-plan.md`.
Headline phases:

| Phase | Scope | LoC budget |
|---|---|---|
| **14.1 — Plugin skeleton + auth bridge** | `bootstrap.sh`, `app.py`, `auth.py`, login/logout/CSRF, operator-token store, session store, six audit events, path-gate extension | ~800 Python + ~200 TS |
| **14.2 — Read-only viewers** | Tenants list, audit timeline, runs list, quota dashboard, role browser — every page hits `GET` endpoints only | ~600 Python + ~1200 TS |
| **14.3 — Mutations (tenant scope)** | Token issue/revoke, webhook rotate, quota set, role grant/revoke, settings edit | ~500 Python + ~800 TS |
| **14.4 — Mutations (operator scope)** | Tenant CRUD, OIDC config, engine policy editor | ~400 Python + ~600 TS |
| **14.5 — Realtime + SSE** | Run-event live view, audit-chain tail | ~200 Python + ~400 TS |
| **14.6 — Setup wizard** | First-run UX, tenant provisioning, OIDC connect, bridge pairing flow | ~300 Python + ~800 TS |
| **14.7 — Evidence-bundle export** | Signed `.atelier-pkg` carrying audit events + chain proof | ~250 Python |
| **14.8 — Headless E2E + Grafana embedding + closure** | Playwright suite, dashboard panels for the new metrics, README + this ADR's status flip to Accepted | ~400 TS + docs |

Each phase ships its own per-subtask E2E suite (mirror of
ADR-0013's discipline). `run-all-tests.sh` skips the admin suites
when `core/admin/.venv/` is absent.

## What you, as Claude Code, must NOT do (admin-UI)

- **Don't bypass any compliance gate from the admin path.** Path-
  gate, audit-chain, consent gate, role-based authority, engine /
  zone policy, secret vault — every gate that protects the CLI
  and slash-command surfaces must equally protect the HTTP
  surface. The admin is a render-layer; it cannot *grant* itself
  authority that the underlying primitive denies.
- **Don't put the cleartext token, CSRF secret, or session ID
  into any audit-event detail field.** Fingerprints only. The
  `test_admin_audit_carries_no_secrets` case is the regression
  gate.
- **Don't introduce a second auth layer.** The operator-tier
  token is a sibling of the tenant-tier token, *not* a separate
  account system with passwords. AtelierOS deliberately has no
  password store; bringing one in here re-introduces the
  bcrypt-rotation / forgot-password / lockout-policy surface that
  bearer-tokens were chosen to avoid.
- **Don't render audit-chain hashes client-side.** The browser
  receives an opaque `chain_intact: bool` flag plus a list of
  events; the chain math runs server-side via `verify_chain`.
  Exposing hashes lets a malicious client claim a fake "intact"
  state on a screenshot.
- **Don't make the operator-tier token cross-tenant by default.**
  Operator tokens default to no tenant scope; the operator picks
  the tenant context explicitly in the UI, and every mutation
  records which tenant was targeted. A blanket "operates on every
  tenant" mode would be the silent multi-tenancy bug ADR-0007
  exists to prevent.
- **Don't add `npm` as a runtime dependency of voice / cowork /
  forge / gateway.** The admin's `package.json` is invoked from
  `bootstrap.sh` only; the produced static bundle is what the
  ASGI app serves at runtime. Single-operator deployments that
  never enable the admin must remain Node-free at runtime.
- **Don't embed Grafana via cross-origin iframe without
  `frame-ancestors` review.** The Phase 6 dashboards expect to
  run on the operator's Grafana instance; embedding them in the
  admin requires CSP coordination and per-panel auth tokens.
  Document the operator-side Grafana configuration; do not
  smuggle Grafana credentials into the admin UI's storage.
- **Don't introduce a websocket primitive.** Long-polling and SSE
  cover every real-time need on the surface (run events, audit
  tail). Websockets add a second framing layer, a second
  auth-renewal story, and a second reconnect-on-network-blip
  behaviour. They are a Phase-15+ decision if a feature genuinely
  needs them.
- **Don't widen the `action` metric label to free-form strings.**
  Cardinality control is the load-bearing rule for Phase 6
  metrics; an admin route author who adds `action="custom-edit"`
  per click would saturate the metrics surface.
- **Don't make the admin UI a path for skill / forge tool
  *creation*.** Creation goes through the persona-bound MCP path
  (operator-curated). The admin allows browsing, promotion,
  purging, and reading — but the "I had an idea, let me write a
  Python tool here" flow is structurally separate, because it
  would re-introduce the very LLM-context bloat that Forge exists
  to keep out of the chat.
- **Don't ship a "demo data" fixture that lives next to real
  tenant data.** Demo / sandbox tenants are real tenants under
  the `_demo` ID with their own isolated chain; never a special-
  cased path that the audit chain treats differently.
- **Don't drop the absolute-8-hour session timeout in favour of
  refresh-on-activity-only.** An idle laptop with a logged-in
  admin browser is the prime physical-attack vector; the absolute
  cap is what limits the window of unattended exposure.
- **Don't render error stack traces in the admin UI.** A
  ValidationError from Pydantic is helpful to a developer and
  catastrophic to a SOC2 auditor scanning the operator's
  screenshots. Error responses carry curated `reason` codes only;
  stack traces land in the operator's server log, never in the
  HTTP response body.

## References

- ADR-0007 — Multi-tenant gateway (Phase 2 auth + REST surface
  that the admin extends)
- ADR-0008 — Bridges-state out-of-repo (the bridge-settings
  editor reads + writes through the canonical `<atelier_home>/
  bridges/<channel>/settings.json` path)
- ADR-0009 — Declarative tenant surface (tenant.atelier.yaml
  schema that the tenant CRUD form edits)
- ADR-0010 — Operator observability surface (the metrics +
  Grafana panels the admin embeds)
- ADR-0012 — Large-data snapshot layer (data-handle browser)
- ADR-0013 — Compute worker plugin (compute-run viewer)
- Layer 10 — path-gate (extended with four new protected globs)
- Layer 16 — security hardening (audit-chain integrity, consent
  gate)
- Layer 18 — role-based delegation (the authority model the
  admin renders)
- Layer 20 — quota + audit-view (the data the admin surfaces)
- `core/gateway/` — the FastAPI app this plugin
  mounts onto
- Compliance baseline (CLAUDE.md `## Compliance baseline`) — the
  set of structural mechanisms the admin must preserve, never
  weaken
