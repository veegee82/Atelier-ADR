# ADR-0014 Implementation Plan — Admin-UI Plugin

**Companion to:** [`0014-admin-ui.md`](./0014-admin-ui.md)
**Status:** Done (2026-05-14 — all 8 sub-phases shipped, 163 tests green)

This plan breaks ADR-0014 into eight sub-phases. Each sub-phase ships
as one Claude Code session with per-subtask E2E gates, follows the
existing strangler-fig pattern (additive, opt-in, never breaks
single-operator deployments), and updates the closure narrative in
`CLAUDE.md` once landed.

## Phase ordering rationale

Phases 14.1–14.2 are the **minimum visible surface**. After 14.2 an
operator can install the plugin, log in, and view tenants / runs /
audit / quota — read-only — through a browser. This is the
demo-ready milestone.

Phases 14.3–14.4 add **mutations** in two trust tiers: tenant-scope
first (token issue/revoke, webhook rotate, quota set, role grant)
then operator-scope (tenant CRUD, OIDC, engine policy). Splitting
them lets us ship the lower-blast-radius surface to beta users while
hardening the cross-tenant surface separately.

Phases 14.5–14.7 are the **advanced surface** (realtime SSE, setup
wizard, evidence-bundle export). Phase 14.8 is the closure: headless
E2E coverage, Grafana panel embedding, README + ADR status flip.

Phases 14.1 → 14.2 should land before any operator demo is given —
until the read-only viewers work, the UI is not yet a real product.

## Sub-phase table

| Phase | Goal | Hard deps | Status |
|---|---|---|---|
| 14.1 | Plugin skeleton + auth bridge (login/CSRF/sessions/audit) | — | ✓ (`fdce6bc`) |
| 14.2 | Read-only viewers (tenants/audit/runs/quota/roles) | 14.1 | ✓ (`fdce6bc`) |
| 14.3 | Tenant-scope mutations (tokens/webhooks/quota/roles/settings) | 14.2 | ✓ (`337b923`) |
| 14.4 | Operator-scope mutations (tenant CRUD/OIDC/engine policy) | 14.3 | ✓ (`36f75fa`) |
| 14.5 | Realtime SSE (runs + audit tail) | 14.2 | ✓ (`20f4362`) |
| 14.6 | Setup wizard (first-run UX, bridge pairing) | 14.4 | ✓ (`680137c`) |
| 14.7 | Evidence-bundle export (signed .atelier-pkg) | 14.3 | ✓ (`b8b759c`) |
| 14.8 | Real-uvicorn E2E smoke + ADR closure | 14.7 | ✓ (this commit) |

---

## Phase 14.1 — Plugin skeleton + auth bridge

**Goal:** bring the plugin tree into existence with a self-contained
venv, register six new audit-event types, extend the path-gate with
four new protected globs, and ship a working session-cookie auth
flow on top of the existing `atlr_`-bearer token contract. After
14.1 a developer can `POST /v1/admin/auth/login` with an operator
token and receive a session cookie; mutations are protected by
CSRF; sessions expire on schedule.

### Files

```
core/admin/
├── bootstrap.sh
├── requirements.txt
├── .gitignore                          # .venv/, __pycache__/, frontend/build/, static/
├── README.md                           # opt-in installation recipe
├── atelier_admin/
│   ├── __init__.py
│   ├── version.py                      # __version__ = "0.1.0"
│   ├── app.py                          # APIRouter exposing /v1/admin
│   ├── auth.py                         # session store + cookie bridge + CSRF
│   ├── operator_tokens.py              # atlr_op_ token store (sibling of auth.py)
│   ├── audit.py                        # admin-specific event emitters
│   ├── paths.py                        # tenant-aware resolvers (mirror of forge/paths.py)
│   └── routes/
│       ├── __init__.py
│       └── auth_routes.py              # /login, /logout, /whoami
└── tests/
    ├── __init__.py
    ├── test_plugin_skeleton.py
    ├── test_no_anthropic_import.py
    ├── test_operator_tokens.py
    ├── test_session_store.py
    ├── test_csrf.py
    └── test_auth_routes.py
```

Plus changes outside the plugin tree:

```
operator/forge/forge/security_events.py    # +6 admin.* event types
operator/voice/hooks/path_gate.py          # +4 protected globs + 4 hints
operator/voice/hooks/test_path_gate.py     # +4 admin path-gate cases
operator/bridges/run-all-tests.sh    # +1 entry skipping when venv absent
```

### `requirements.txt` (frozen at 14.1)

```
fastapi>=0.110
pydantic>=2.6
itsdangerous>=2.1    # signed cookies + CSRF
python-multipart>=0.0.9   # form parsing for login
```

No SDK dependencies (no `anthropic`, no `openai`). No password
library (we never store passwords). No JWT library (sessions are
opaque random IDs, not signed JWTs — re-validating the underlying
bearer token per request is structurally simpler than rotating JWT
signing keys).

### Operator-token contract

Shape: `atlr_op_<32 hex>` (40 chars). Distinct from the tenant-tier
`atlr_<32 hex>` (37 chars) so a leaked token's tier is identifiable
at a glance.

Storage: `<atelier_home>/global/admin/operator_tokens.json`
(mode `0o600`). Same shape as `auth.gateway_tokens.json`:
`{sha256, issued_at, label, revoked_at}`.

Resolver: `operator_tokens.resolve(token) -> Optional[OperatorIdentity]`.
Returns `(token_fingerprint, label)` on match, `None` on miss.
`hmac.compare_digest` for the lookup (constant-time compare, same as
Phase 2.1).

CLI (`python -m atelier_admin.cli operator issue|list|revoke`):
operator-scope tokens are not tenant-bound, so the CLI does not
take a `tenant_id` argument. Issue prints the plaintext once;
revocation marks `revoked_at` and never deletes the entry (so
fingerprint forensics survive).

### Session store

On-disk: `<atelier_home>/global/admin/sessions/<sid>.json`
(mode `0o600`). Session record:

```json
{
  "sid_fingerprint":   "<first 12 chars of sha256(sid)>",
  "tier":              "operator" | "tenant-admin",
  "tenant_id":         "<id>" | null,
  "token_fingerprint": "<first 8 chars of token sha256>",
  "csrf_secret":       "<32 hex>",
  "created_at":        <epoch>,
  "last_seen_at":      <epoch>,
  "expires_at":        <epoch>
}
```

`sid` itself is a `secrets.token_urlsafe(32)` (43-char base64url).
The file is named after the cleartext SID but the SID itself never
lands in audit events — only the fingerprint does. Mode `0o600`
prevents UID-local enumeration of valid SIDs from the filenames.

Idle timeout: 1 hour from `last_seen_at`. Absolute timeout: 8 hours
from `created_at`. Both enforced on every request:
`session.is_alive() == False` deletes the file and returns 401.

Per-process cleanup sweep: a daily timer (mirror of
`session_timeout_sweep.py` from Layer 8) prunes expired session
files. Bound to single-process operation; the cleanup is best-
effort because the on-request check is the load-bearing gate.

### CSRF model

Every session carries a per-session CSRF secret. On a mutation
(POST/PUT/PATCH/DELETE) the client MUST send
`X-CSRF-Token: <hmac_sha256(csrf_secret, sid)>` (hex). The server
re-computes the HMAC and `compare_digest`-matches. Mismatch → 403
with `reason=csrf-mismatch` + `admin.session_denied` audit.

The CSRF token is delivered to the client on login via the
response body (NOT the cookie — a cookie-bound CSRF token defeats
the purpose). The frontend (Phase 14.2+) stores it in
`sessionStorage` and attaches it on every mutation.

GET requests are exempt (no CSRF token required). The browser's
SameSite=Strict cookie policy already prevents cross-origin GETs
from carrying the session cookie, which is the structural defence
for read-only side-effects.

### Re-auth for sensitive mutations

A curated set of actions requires the user to re-type the bearer
token in the request body as `re_auth_token`. The set:

- `token.issue`, `token.revoke` (tenant- and operator-tier)
- `webhook.set_secret`, `webhook.revoke_secret`
- `oidc.update_trust` (Phase 14.4)
- `tenant.delete` (Phase 14.4)
- `operator.issue`, `operator.revoke`

Re-auth runs the same `auth.resolve_token` / `operator_tokens.resolve`
call as login; on success the action proceeds, on failure 403 with
`reason=re-auth-required`. The re-auth event itself is **not**
audited (it's a tautology — every mutation in the set emits its own
audit event); only failures land in the chain.

### Six new audit-event types

Registered in `operator/forge/forge/security_events.py::EVENT_SEVERITY`:

| Event | Severity |
|---|---|
| `admin.session_started` | INFO |
| `admin.session_ended` | INFO |
| `admin.session_denied` | WARNING |
| `admin.action_performed` | INFO |
| `admin.action_failed` | WARNING |
| `admin.export_generated` | INFO |

Each emitter lives in `atelier_admin/audit.py` and calls
`security_events.write_event` with the right severity + the
fingerprint-only payload. The emitter helpers validate that no
cleartext token / SID / CSRF secret lands in the details (mirror of
Layer 25's `_ALLOWED_FIELDS` pattern).

### Path-gate extension

`operator/voice/hooks/path_gate.py` gains four new entries in the
`_PROTECTED_REL_GLOBS`-style data structure:

| Glob | Reason |
|---|---|
| `<atelier_home>/**/admin/operator_tokens.json` | Operator-token store |
| `<atelier_home>/**/admin/sessions/**` | Session records |
| `<atelier_home>/**/admin/csrf_secrets.json` | Reserved for Phase 14.3+ |
| `<repo>/core/admin/atelier_admin/static/**` | Built frontend bundle |

Plus four new strings in `_PROTECTED_HINTS`: `operator_tokens`,
`admin/sessions`, `csrf_secrets`, `admin/static`.

Test gates in `operator/voice/hooks/test_path_gate.py`:

- `test_admin_operator_tokens_protected` — Write/Edit/Bash redirect
  to `<atelier_home>/global/admin/operator_tokens.json` all deny.
- `test_admin_sessions_dir_protected` — Write to any file under the
  sessions dir denies.
- `test_admin_static_dir_protected` — Write to the static bundle
  path denies.
- `test_admin_bash_hint_denies` — Bash with `eval` + `operator_tokens`
  hint string fails closed.

### Auth routes

| Method | Path | Body | Codes |
|---|---|---|---|
| `POST` | `/v1/admin/auth/login` | `{token: str}` | 200 (session) · 401 (invalid) · 429 (rate-limited) |
| `POST` | `/v1/admin/auth/logout` | — | 204 |
| `GET` | `/v1/admin/auth/whoami` | — | 200 (identity) · 401 |

Login response body:
```json
{
  "tier":         "operator" | "tenant-admin",
  "tenant_id":    "<id>" | null,
  "csrf_token":   "<hex>",
  "expires_at":   <epoch>
}
```

Rate-limit: 10 login attempts per IP per minute (token-bucket; same
shape as ADR-0007 Phase 7's `rate_limit.py` but per-IP instead of
per-tenant since failed logins have no tenant binding yet). Failures
emit `admin.session_denied`.

### `app.py` mount contract

The plugin exposes a single `router: APIRouter`. The gateway side
opt-in lives in `atelier_gateway/app.py`:

```python
try:
    from atelier_admin import app as admin_app
    app.include_router(admin_app.router, prefix="/v1/admin")
except ImportError:
    pass  # plugin not installed, single-operator path
```

No second port, no second ASGI process. Single-operator deployments
that never bootstrap the admin plugin see `ImportError` here, the
include is silently skipped, and the gateway works exactly as
before.

### Test gates (Phase 14.1)

| Test | What it covers |
|---|---|
| `test_plugin_skeleton.py` | Plugin tree exists, version importable, `__init__.py` present |
| `test_no_anthropic_import.py` | AST walk of every `.py` under `atelier_admin/` — no `import anthropic` |
| `test_operator_tokens.py` | Issue/resolve/revoke round-trip, mode 0o600, constant-time compare, audit event |
| `test_session_store.py` | Create/load/expire/cleanup, mode 0o600, idle vs absolute timeout, sid never leaks into audit |
| `test_csrf.py` | Token derivation, mutation requires header, mismatch returns 403 + audit |
| `test_auth_routes.py` | E2E via FastAPI `TestClient`: login → cookie issued, whoami works, logout clears, bad token → 401 + audit, rate-limit kicks in after N attempts |

Plus the 4 path-gate cases above.

### `run-all-tests.sh` integration

```bash
if [[ -x core/admin/.venv/bin/python ]]; then
  core/admin/.venv/bin/python -m pytest core/admin/tests/ -q
else
  echo "Python: atelier-admin (skipped — venv not bootstrapped)"
fi
```

### Acceptance

- All test gates above pass.
- `bash run-all-tests.sh` with the admin venv absent: suite count
  goes up by 1 (the skip entry), all suites green.
- `bash run-all-tests.sh` with the admin venv present: suite count
  goes up by N (one per test file), all green.
- Manual smoke: `bootstrap.sh` → `uvicorn atelier_gateway.app:app` →
  curl `POST /v1/admin/auth/login` with an issued operator token
  returns 200 + a Set-Cookie header.
- `voice-audit verify` over a chain that includes admin events
  returns `(ok, [])`.

---

## Phase 14.2 — Read-only viewers

**Goal:** the operator can log in to a browser, see a tenants list,
audit timeline, run list, quota dashboard, and role browser. No
mutations yet.

### New files (Python)

```
atelier_admin/routes/
├── tenants.py          # GET /v1/admin/tenants{,/<tid>}
├── audit_chain.py      # GET /v1/admin/tenants/<tid>/audit  (paginated + filtered)
├── runs.py             # GET /v1/admin/tenants/<tid>/runs{,/<rid>}
├── quota.py            # GET /v1/admin/tenants/<tid>/quota
└── roles.py            # GET /v1/admin/tenants/<tid>/roles
```

Each route consults the existing modules (Layer 16 `consent.py`,
Layer 18 `roles.py`, Layer 20 `quota.py`, Phase 2 `runs.py`,
Phase 6 `audit_metrics.py`). Routes enforce tier:

- `tenant-admin` tier: only its own `tenant_id` is reachable.
- `operator` tier: must include `?tenant=<tid>` (no implicit
  cross-tenant aggregation; explicit context is the structural
  rule).

### New files (Frontend)

```
core/admin/frontend/
├── package.json
├── svelte.config.js
├── tailwind.config.cjs
├── postcss.config.cjs
├── tsconfig.json
├── src/
│   ├── app.html
│   ├── routes/
│   │   ├── +layout.svelte
│   │   ├── +page.svelte              # Login form
│   │   ├── (dash)/
│   │   │   ├── +layout.svelte        # Sidebar + auth gate
│   │   │   ├── tenants/+page.svelte
│   │   │   ├── audit/+page.svelte
│   │   │   ├── runs/+page.svelte
│   │   │   ├── quota/+page.svelte
│   │   │   └── roles/+page.svelte
│   └── lib/
│       ├── api.ts                    # Fetch wrapper with CSRF
│       ├── auth.ts                   # Session store
│       └── components/               # shadcn-svelte primitives
└── tests/
    ├── e2e.spec.ts                   # Playwright headless E2E
    └── ...
```

`bootstrap.sh` (extended) runs `npm install && npm run build` after
the venv is created, producing the static bundle into
`atelier_admin/static/`. The bundle is gitignored.

### Acceptance

- Headless E2E with Playwright: login → tenants page renders →
  audit page filters work → logout returns to login.
- Bundle size < 250 KB gzipped.
- Every page works keyboard-only (Tab navigation, Enter submits).
- Every page passes `axe-core` accessibility audit at AA level.

---

## Phase 14.3 — Tenant-scope mutations

**Goal:** every operation that was reachable today via a tenant-
scoped CLI invocation (token issue/revoke, webhook secret rotation,
quota set/reset, role grant/revoke, settings edit) is reachable
through the UI.

### Mutation endpoints

| Method | Path | Audit event |
|---|---|---|
| `POST` | `/v1/admin/tenants/<tid>/tokens` | `admin.action_performed` (action=token.issue) |
| `DELETE` | `/v1/admin/tenants/<tid>/tokens/<fp>` | (token.revoke) |
| `POST` | `/v1/admin/tenants/<tid>/webhooks/<ref>` | (webhook.set_secret) |
| `DELETE` | `/v1/admin/tenants/<tid>/webhooks/<ref>` | (webhook.revoke_secret) |
| `PATCH` | `/v1/admin/tenants/<tid>/quota/<uid>` | (quota.set) |
| `POST` | `/v1/admin/tenants/<tid>/quota/<uid>/reset` | (quota.reset) |
| `POST` | `/v1/admin/tenants/<tid>/roles/grants` | (role.grant) |
| `DELETE` | `/v1/admin/tenants/<tid>/roles/grants/<uid>` | (role.revoke) |
| `PATCH` | `/v1/admin/tenants/<tid>/bridges/<channel>/settings` | (bridge.settings_updated) |

Every mutation in the sensitive set (`token.*`, `webhook.*`) requires
a `re_auth_token` field in the body. Failures emit
`admin.action_failed` with `reason`.

### Frontend additions

Forms with confirmation modals; sensitive actions show a re-auth
prompt before submission. Live success/failure toasts.

### Acceptance

- Each mutation has a per-subtask E2E (FastAPI TestClient + Playwright)
  that drives login → mutation → verify audit event → verify
  side-effect on disk.
- The Phase 2 `webhook.dispatched` chain integrity holds across
  admin-side rotations (test gate: rotate secret → next webhook
  uses new secret).

---

## Phase 14.4 — Operator-scope mutations

**Goal:** create / update / delete tenants, manage OIDC trust, edit
engine policy. These actions require operator-tier tokens.

### Endpoints

| Method | Path | Re-auth | Notes |
|---|---|---|---|
| `POST` | `/v1/admin/tenants` | yes | Wraps `tenant_migrate.migrate_to_default_tenant_if_needed` for `_default`; creates fresh tenant tree for new IDs |
| `PATCH` | `/v1/admin/tenants/<tid>/config` | no | tenant.atelier.yaml editor (zone, engine allowlist, budget) |
| `DELETE` | `/v1/admin/tenants/<tid>` | yes | Requires confirmation string match; preserves audit chain (renames tree to `.deleted-<epoch>`, never rm) |
| `PUT` | `/v1/admin/tenants/<tid>/oidc` | yes | Replace OIDC trust file |
| `PUT` | `/v1/admin/tenants/<tid>/scim/users` | no | Bulk SCIM provisioning (mirrors Phase 3.5) |

Tenant deletion is **structurally non-destructive**. The directory
is renamed, never removed. A 30-day operator-side cleanup is a
separate sub-phase if it ever lands. The audit chain for the
deleted tenant is preserved in the renamed tree; chain verification
still passes on it.

### Acceptance

- E2E for tenant create → token issue → run dispatch → tenant
  rename-on-delete; chain stays verifiable across the lifecycle.
- An operator with no tenants on a fresh install can provision
  their first tenant through the UI.

---

## Phase 14.5 — Realtime SSE

**Goal:** the operator can watch a run's events live in the browser
and tail the audit chain as new events arrive.

### Endpoints

| Method | Path | Behaviour |
|---|---|---|
| `GET` | `/v1/admin/tenants/<tid>/runs/<rid>/events` | Proxy to Phase 2.5 SSE endpoint, identical wire format |
| `GET` | `/v1/admin/tenants/<tid>/audit/tail` | New SSE: emit audit events as they land |

The audit-tail SSE re-reads the chain file on a 1-second interval
(no in-process pub/sub; the file is already append-only) and emits
events newer than the consumer's last cursor. Cursor is the byte-
offset of the chain file as of subscribe time.

### Frontend

EventSource-driven views with reconnect-on-network-blip. The audit-
tail page has live filtering identical to Phase 14.2's audit page.

### Acceptance

- Live run view in the browser shows the same events as
  `curl /v1/tenants/<tid>/runs/<rid>/events`.
- Audit-tail keeps up with a synthetic 100-events/sec test load.

---

## Phase 14.6 — Setup wizard

**Goal:** a fresh AtelierOS install plus a fresh admin operator-
token bootstraps to "ready to use" in five clicks.

### Wizard steps

1. **Welcome + license acknowledgement.** Operator confirms the
   compliance baseline (EU AI Act 2026 + GDPR). Audited as
   `admin.compliance_acknowledged`.
2. **First tenant.** Pick ID (validated via `validate_tenant_id`),
   display name, zone, engine allowlist.
3. **First token.** Issue one tenant-admin token; show plaintext
   once with a "copy to clipboard" button.
4. **Bridge pairing (optional).** Per channel: input the bridge
   secret, run a connectivity probe (the daemon's HTTP
   `/healthz`-equivalent), confirm.
5. **Done.** Redirect to the dashboard with a notification banner
   for any non-default decision.

### Acceptance

- E2E: empty `<atelier_home>/` → wizard → first tenant alive →
  successful test run dispatched through the new token.
- Wizard never bypasses the audit chain; every step lands at least
  one event.

---

## Phase 14.7 — Evidence-bundle export

**Goal:** the operator can export a signed `.atelier-pkg` carrying
a slice of audit events plus a verification proof, hand it to an
auditor, and the auditor can verify the chain on a separate
machine.

### Surface

| Method | Path | Body | Returns |
|---|---|---|---|
| `POST` | `/v1/admin/tenants/<tid>/exports/audit` | `{since: epoch, until: epoch, event_types: [..]?}` | Streaming `.atelier-pkg` |

The bundle contains:
- `manifest.atelier.yaml` (Phase 5 shape)
- `payload/audit_chain.jsonl` — the filtered slice
- `payload/audit_chain.verify.txt` — the `verify_chain` output
- `payload/SUMMARY.md` — event counts by type + severity + date range

Signature uses the operator's ed25519 key (Phase 5 packaging) so
the auditor's `python -m atelier_gateway.cli package verify` works
unchanged.

### Acceptance

- Generated bundle verifies on a machine that has only the
  `atelier-gateway` plugin (no admin plugin needed for verification).
- Filtered slices preserve chain integrity (the verify proof
  documents the filter boundary so partial-chain checks succeed).

---

## Phase 14.8 — Headless E2E + Grafana + closure

**Goal:** the surface is production-ready, the Grafana panels for
admin metrics are merged into the existing dashboards, and the
ADR's status flips to Accepted.

### Headless E2E (Playwright)

Test cases (~30):
- Login → logout round-trip.
- CSRF gate trips on missing header.
- Idle timeout redirects to login.
- Absolute timeout redirects to login.
- Tier enforcement: tenant-admin cannot reach operator-only routes.
- Every page renders without console errors.
- Every page passes `axe-core` AA.
- Bundle size < 250 KB gzipped (CI gate).
- Setup wizard E2E on an empty `<atelier_home>/`.
- Evidence-bundle export → verify on a fresh subprocess.

### Grafana panels

Two additions on `docs/observability/grafana/atelier-security.json`:

- **Admin sessions** (gauge timeseries) — `atelier_admin_sessions_active`
  by tier.
- **Admin actions / 1h** (counter timeseries) — by action + outcome.

One addition on `docs/observability/grafana/atelier-overview.json`:

- **Admin session denials** (timeseries) — by reason. Yellow > 5/h,
  red > 20/h.

### Closure

- ADR-0014 status: Proposed → Accepted.
- CLAUDE.md gains a `## Layer 26 — Admin UI` section mirroring
  Layer 25's structure (compliance posture, audit events,
  Prometheus metrics, "must NOT do" list, references).
- README gets a screenshot + 3-line description of the admin UI.
- `docs/for-companies.md` gains a paragraph on tenant-admin
  delegation: "your compliance officer reaches the audit chain
  through the admin UI, no shell required".

---

## Cross-phase non-negotiables

- **No phase weakens any structural compliance mechanism.** Every
  PR for every sub-phase must walk through the CLAUDE.md
  `## Compliance baseline` table and confirm preservation.
- **Per-subtask E2E discipline applies.** Every sub-task ships its
  own E2E that drives the new behaviour through real HTTP /
  real filesystem / real bwrap. No mocks where a subprocess is
  the actual external dependency.
- **No phase adds `import anthropic` anywhere in the admin tree.**
  The AST lint from 14.1 fires on every test run.
- **No phase changes the wire format of `/v1/tenants/<tid>/...`
  endpoints.** The admin extends with `/v1/admin/*`; the tenant
  surface stays byte-stable so existing SDKs and CLI tools
  continue working.
- **Every phase updates `0014-implementation-plan.md`'s status
  table** alongside the code changes.

## Open questions (resolved by Phase 14.1 close)

- **Cookie name.** Final: `atelier_admin_sid`. Documented in 14.1
  so the frontend (14.2+) can hard-code it.
- **Re-auth UX.** Final: a modal that asks for the bearer token
  fresh; never cached, never auto-filled. Documented in 14.1's
  README so 14.3 can implement it consistently.
- **Operator-token revocation cascade.** Final: revoking an
  operator token MUST invalidate all sessions that were issued
  from it (the per-request re-validation handles this; no
  separate sweep needed). Documented in 14.1's auth module.

## References

- ADR-0014 (parent decision)
- ADR-0007 Phase 2.1 — bearer-token auth contract (extended by
  this plan with the operator tier)
- ADR-0007 Phase 5 — `.atelier-pkg` package format (consumed by
  Phase 14.7 export)
- ADR-0008 — bridges-state out-of-repo (canonical settings path
  for the Phase 14.3 settings editor)
- ADR-0009 — declarative tenant surface (schema the Phase 14.4
  tenant editor reads/writes)
- ADR-0010 — operator observability surface (metrics the Phase
  14.5 dashboards embed)
- Layer 10 — path-gate (extended in Phase 14.1)
- Layer 16 — audit chain + consent gate (consumed everywhere)
- Layer 18 — roles (consumed by Phase 14.2 and 14.3)
- Layer 20 — quota (consumed by Phase 14.2 and 14.3)
