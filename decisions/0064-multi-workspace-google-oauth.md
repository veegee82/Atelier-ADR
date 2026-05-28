# ADR-0064 — Multi-Workspace Cloud Console: Google OAuth + Per-User Container Management

**Status:** Proposed — 2026-05-29
**Date:** 2026-05-29
**Authors:** Claude Code (maintainer session)
**Type:** Architecture — SaaS Control Plane + Identity + Provisioning
**Extends:** [ADR-0007 (Multi-tenant)](0007-multi-tenant-integration-surface.md),
[ADR-0015 (Console)](0015-console-self-service-ui.md),
[ADR-0017 (Enterprise Control Plane)](0017-enterprise-control-plane.md),
[ADR-0037 (Console Relaunch)](0037-console-relaunch.md)
**Depends on:** L16 (consent gate), L36 (erasure), L35 (egress lockdown),
ADR-0007 Phase 4 (OIDC), cloud-ops provisioning webhook
**Scope:** `cloud-ops/` (auth, registry, proxy, provisioner) +
`core/console/atelier_console/` (workspace-aware routing) +
deploy pipeline (container lifecycle)

---

## Context

AtelierOS currently runs as a **single-tenant** deployment: one Docker container
serves one operator via a bearer-token login. `atelier-labs.net` exposes only a
public landing/booking page; the operator console is unreachable from the domain.

### Problems this ADR solves

| Problem | Current state |
|---|---|
| No identity | Token-only login, no user account concept |
| No multi-workspace | One container = one operator, forever |
| Domain gap | `/console/` not reachable via `atelier-labs.net` (fixed ad-hoc 2026-05-29, needs proper architecture) |
| No self-service | Provisioning is manual (SSH + docker run) |
| No lifecycle | No create / rename / delete workflow for containers |

### Why Google OAuth (not a custom password flow)

- Zero credential storage risk (no password DB to breach)
- OIDC provides a signed, auditable `sub` claim (stable Google user ID)
- Enterprise users expect SSO; Google Workspace covers most B2B scenarios
- Extensible: same OIDC flow works for GitHub, Microsoft, custom IdP later

### Why multiple workspaces per user

Operators run different AtelierOS instances for different purposes:
production, staging, customer-A, customer-B, experiments. A single login
must not collapse all of those into one container. Each workspace is an
independent, isolated AtelierOS instance with its own:

- `ATELIER_TENANT_ID`
- audit chain
- forge/skill-forge scope
- data classification zone
- network egress rules

---

## Decision

### Layer model

```
atelier-labs.net  (Cloudflare Tunnel → cloud-ops:8080)
        │
        ├── /                  Landing page (existing)
        ├── /auth/google       OAuth entry + callback
        ├── /workspaces        Workspace dashboard (new SPA page)
        ├── /app/{wid}/**      Dynamic reverse-proxy to workspace container
        └── /v1/**             cloud-ops REST API (plans, checkout, identity)

Per-user containers (Docker network: atelier-workspaces)
        atelier-{wid}:8000     isolated AtelierOS instance
```

### Component overview

| Component | File | Responsibility |
|---|---|---|
| OAuth handler | `cloud-ops/auth/google.py` | Login, callback, session cookie |
| Identity registry | `cloud-ops/registry.py` | SQLite: users + workspaces + sessions |
| Provisioner | `cloud-ops/provisioner.py` | spawn / stop / destroy containers |
| Workspace proxy | `cloud-ops/proxy.py` | dynamic `httpx` reverse-proxy |
| Workspace dashboard | landing `web/` SPA extension | list + create + switch workspaces |

---

## Component 1 — Google OAuth handler

### Flow

```
User → GET /auth/google
  └─→ cloud-ops builds Google authorization URL
        (scope: openid email profile, state: CSRF nonce, PKCE)
  └─→ Redirect to accounts.google.com

Google callback → GET /auth/google/callback?code=…&state=…
  └─→ Validate state (anti-CSRF)
  └─→ Exchange code for id_token + access_token
  └─→ Verify id_token signature (Google JWKS)
  └─→ Extract: sub (stable ID), email, name, picture
  └─→ Registry.upsert_identity(sub, email, ...)
  └─→ Create session (random 32-byte token, httpOnly Secure SameSite=Lax cookie)
  └─→ Redirect → /workspaces
```

### Session cookie

```
Name:      atelier_cloud_sid
Value:     <32-byte random hex, stored as SHA-256 in registry>
Flags:     HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=604800 (7 days)
Rotation:  Re-issued on each /workspaces load if > 24 h old
```

The cookie is scoped to `atelier-labs.net`. It is **not** forwarded to
workspace containers — workspace sessions are separate (existing `atelier_console_sid`).

### Session-to-workspace auth bridge

When the user's browser hits `/app/{wid}/**` for the first time in a
session, `cloud-ops`:

1. Validates `atelier_cloud_sid` → resolves `google_id`.
2. Checks `workspace.owner_google_id == google_id` (ownership gate).
3. Issues a short-lived (5 min) `atlr_*` gateway token for the workspace
   container by calling `atelier-gateway token issue` via subprocess.
4. Sets a `atelier_console_sid` cookie scoped to `/app/{wid}/` via a
   transparent redirect through `/app/{wid}/_auth_bridge`.
5. All subsequent `/app/{wid}/**` requests are proxied with no extra
   auth step (the `atelier_console_sid` cookie carries the workspace session).

This preserves the existing console auth protocol unchanged inside the
workspace container.

---

## Component 2 — Identity registry (SQLite)

**File:** `cloud-ops/registry.py`
**Path:** `/opt/atelier-cloud/identity.db` (mode 0600, outside container, bind-mounted)

### Schema

```sql
CREATE TABLE identities (
    google_id       TEXT PRIMARY KEY,  -- OIDC `sub` claim
    email           TEXT NOT NULL,
    display_name    TEXT,
    picture_url     TEXT,
    created_at      REAL NOT NULL,
    last_login_at   REAL NOT NULL
);

CREATE TABLE workspaces (
    workspace_id    TEXT PRIMARY KEY,   -- random 8-char slug, e.g. "ws_a3f2c1"
    owner_google_id TEXT NOT NULL REFERENCES identities(google_id),
    label           TEXT NOT NULL,      -- user-chosen name
    tenant_id       TEXT NOT NULL UNIQUE, -- ATELIER_TENANT_ID inside container
    container_name  TEXT,               -- Docker container name, null = not provisioned
    container_host  TEXT,               -- "atelier-{wid}:8000" or null
    status          TEXT NOT NULL DEFAULT 'creating',
    -- 'creating' | 'running' | 'stopped' | 'error' | 'deleted'
    plan            TEXT NOT NULL DEFAULT 'free',
    data_region     TEXT NOT NULL DEFAULT 'eu-central',
    created_at      REAL NOT NULL,
    last_active_at  REAL
);

CREATE TABLE sessions (
    session_sha256  TEXT PRIMARY KEY,   -- SHA-256 of the cookie value
    google_id       TEXT NOT NULL REFERENCES identities(google_id),
    created_at      REAL NOT NULL,
    expires_at      REAL NOT NULL,
    last_seen_at    REAL NOT NULL,
    user_agent      TEXT,
    ip_addr         TEXT               -- pseudonymised (first 3 octets only)
);

CREATE INDEX idx_workspaces_owner ON workspaces(owner_google_id);
CREATE INDEX idx_sessions_expires ON sessions(expires_at);
```

### GDPR notes

- `ip_addr` stores only the /24 prefix (last octet zeroed) — not PII.
- `picture_url` is a Google CDN URL pointing to external data; no image
  data is stored locally.
- `display_name` and `email` are subject to Art. 17 erasure; see §Erasure below.

---

## Component 3 — Provisioner

**File:** `cloud-ops/provisioner.py`

### Create workspace

```
1.  Generate workspace_id: "ws_" + 8 random alphanum chars
2.  Generate tenant_id: UUID4
3.  Pull/verify image: atelier:local (already present on host)
4.  docker run --detach
      --name atelier-{workspace_id}
      --network atelier-workspaces
      --restart unless-stopped
      --memory 2G
      --env ATELIER_TENANT_ID={tenant_id}
      --env ATELIER_HOME=/home/atelier/.atelier
      --env HOME=/home/atelier
      --volume /opt/atelier-workspaces/{workspace_id}/home:/home/atelier
      atelier:local
5.  Wait for /healthz == 200 (poll 1s, timeout 60s)
6.  Issue atlr_* token via atelier-gateway CLI (exec into container)
7.  Write workspace row to registry (status='running')
8.  Return {workspace_id, console_url: "/app/{workspace_id}/"}
```

### Stop workspace

- `docker stop atelier-{workspace_id}` → registry status='stopped'
- Data volume persists at `/opt/atelier-workspaces/{workspace_id}/`

### Delete workspace

```
1.  registry status='deleted'
2.  docker stop + docker rm atelier-{workspace_id}
3.  Emit atelier_workspace.deleted audit event
4.  Schedule data purge after 30-day retention window (Art. 5 GDPR)
     (cron job: rm -rf /opt/atelier-workspaces/{workspace_id}/)
```

### Limits per user (plan-gated)

| Plan | Max workspaces | Container memory | Idle auto-stop |
|---|---|---|---|
| free | 1 | 1 GB | after 7 days |
| pro | 5 | 2 GB | after 30 days |
| enterprise | unlimited | configurable | never |

---

## Component 4 — Dynamic workspace proxy

**File:** `cloud-ops/proxy.py`  
Replaces the static `host.docker.internal:8000` proxy added 2026-05-29.

### Routing logic

```python
@app.api_route("/app/{workspace_id}/{path:path}", methods=ALL_METHODS)
async def proxy_workspace(request, workspace_id, path):
    session = require_cloud_session(request)          # 401 if missing/expired
    ws = registry.get_workspace(workspace_id)
    if ws is None or ws.status == 'deleted':
        raise HTTPException(404)
    if ws.owner_google_id != session.google_id:
        raise HTTPException(403)                      # ownership gate
    if ws.status == 'stopped':
        provisioner.start(workspace_id)               # wake on access
        await provisioner.wait_healthy(workspace_id)
    return await _proxy(request, ws.container_host, f"/{path}")
```

Path translation:
- `/app/{wid}/console/**` → `http://atelier-{wid}:8000/console/**`
- `/app/{wid}/v1/console/**` → `http://atelier-{wid}:8000/v1/console/**`

The React SPA inside the container uses `basename="/app/{wid}/console"`.
This requires the gateway to set `ATELIER_CONSOLE_BASE_PATH=/app/{wid}/console`
at container startup — a new env var read by `mount_static()` and injected
into `index.html` at serve time (or via a `<meta name="base-path">` tag
read by `main.tsx`).

---

## Component 5 — Workspace dashboard

A new page served at `atelier-labs.net/workspaces` (part of the cloud-ops
landing SPA, NOT the per-container console SPA):

```
┌──────────────────────────────────────────────────────────────────┐
│  AtelierOS  ·  silvio@googlemail.com                  [Abmelden] │
├──────────────────────────────────────────────────────────────────┤
│  Meine Workspaces                            [+ Neuer Workspace] │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐                      │
│  │ 🟢 Produktion    │  │ 🟡 Staging       │                      │
│  │ ws_a3f2c1        │  │ ws_b7d9e2        │                      │
│  │ Zuletzt: heute   │  │ Zuletzt: 3 Tage  │                      │
│  │ [Öffnen]  [···]  │  │ [Öffnen]  [···]  │                      │
│  └──────────────────┘  └──────────────────┘                      │
│                                                                  │
│  Status: 2 von 5 Workspaces aktiv  (Pro-Plan)                   │
└──────────────────────────────────────────────────────────────────┘
```

REST endpoints (cloud-ops):

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/identity/workspaces` | List user's workspaces |
| `POST` | `/v1/identity/workspaces` | Create new workspace |
| `PATCH` | `/v1/identity/workspaces/{wid}` | Rename workspace |
| `DELETE` | `/v1/identity/workspaces/{wid}` | Delete workspace |
| `POST` | `/v1/identity/workspaces/{wid}/start` | Wake stopped workspace |
| `POST` | `/v1/identity/workspaces/{wid}/stop` | Stop workspace |

---

## Docker network topology

```
Bridge network: atelier-workspaces (new, created at deploy time)

 cloud-ops ────── atelier-workspaces ─────┬── atelier-ws_a3f2c1 :8000
                                          ├── atelier-ws_b7d9e2 :8000
                                          └── atelier-ws_... :8000

cloud-ops connects to atelier-workspaces via extra_networks.
No host port is exposed per workspace container — traffic flows
container-to-container inside the bridge.
```

`cloud-ops/docker-compose.yml` addition:

```yaml
networks:
  tunnel:         # existing: cloudflared ↔ cloud-ops
    driver: bridge
  atelier-workspaces:
    name: atelier-workspaces
    driver: bridge
    attachable: true
```

Workspace containers are created with `--network atelier-workspaces`
by the provisioner (no compose involvement — provisioner calls Docker SDK
or `docker run` subprocess).

---

## Auth-flow diagram (full)

```
Browser                cloud-ops               Google         atelier-{wid}
   │                       │                      │                  │
   │ GET /workspaces        │                      │                  │
   │──────────────────────▶│                      │                  │
   │                       │ no session cookie     │                  │
   │◀── 302 /auth/google ──│                      │                  │
   │                       │                      │                  │
   │ GET /auth/google      │                      │                  │
   │──────────────────────▶│                      │                  │
   │◀──302 accounts.google─│                      │                  │
   │                       │                      │                  │
   │ OAuth consent screen  │                      │                  │
   │──────────────────────────────────────────────▶                  │
   │◀─────────────── code ─────────────────────────                  │
   │                       │                      │                  │
   │ GET /auth/google/cb   │                      │                  │
   │──────────────────────▶│                      │                  │
   │                       │ exchange code        │                  │
   │                       │─────────────────────▶│                  │
   │                       │◀──── id_token ────────│                  │
   │                       │ verify + extract sub  │                  │
   │                       │ upsert identity       │                  │
   │                       │ create session        │                  │
   │◀── 302 /workspaces ───│                      │                  │
   │    Set-Cookie: sid    │                      │                  │
   │                       │                      │                  │
   │ GET /app/ws_a3f2/     │                      │                  │
   │──────────────────────▶│                      │                  │
   │                       │ validate sid          │                  │
   │                       │ verify ownership      │                  │
   │                       │ issue atlr_* token   │                  │
   │                       │──────────────────────────────────────▶  │
   │                       │ proxy request        │                  │
   │◀──────────────────────────────────────────────────── response ──│
```

---

## GDPR / EU AI Act compliance

### Consent gate

Before any workspace is provisioned:

1. Cloud-ops shows the **disclosure card** (EU AI Act Art. 50) — the same
   text as L19, adapted for cloud context.
2. User must click "Ich stimme zu" — stored as `consent.granted_at` in
   `identities` table alongside the displayed version hash.
3. No container is created until consent is on record.
4. Consent can be withdrawn via `/v1/identity/consent` DELETE → triggers
   full erasure (see below).

### Erasure (GDPR Art. 17)

`atelier-erasure` is extended with a cloud-ops handler:

```
atelier-cloud-erasure <google_id>
  1. Delete all workspace containers (docker stop + rm)
  2. Schedule data-volume purge (30-day window)
  3. Delete registry rows: workspaces, sessions (immediate)
  4. Zero identity row fields: email→NULL, display_name→NULL,
     picture_url→NULL, google_id pseudonymised (SHA-256 with salt)
  5. Revoke all outstanding atlr_* tokens per workspace
  6. Write erasure trail: /opt/atelier-cloud/erasure/{request_id}.json
```

The identity `sub` (Google ID) is pseudonymised rather than deleted to
maintain referential integrity in the audit chain (same mechanism as L36).

### Data residency

`data_region: eu-central` is the default for all free/pro workspaces.
The provisioner refuses to start a container on a non-EU host for these
plans. Enterprise plans may override via tenant config.

---

## Audit events

All events written to the cloud-ops audit log
(`/opt/atelier-cloud/audit.jsonl`, same hash-chain format as L16).

| Event | Trigger | Fields |
|---|---|---|
| `identity.login` | Successful OAuth callback | `google_id_prefix` (8 chars), `ip_24` |
| `identity.logout` | Session deleted | `session_prefix` |
| `identity.consent_granted` | Consent form submitted | `google_id_prefix`, `consent_version` |
| `identity.consent_withdrawn` | DELETE /consent | `google_id_prefix` |
| `workspace.created` | Provisioner completes | `workspace_id`, `plan` |
| `workspace.deleted` | DELETE endpoint | `workspace_id` |
| `workspace.started` | Wake-on-access | `workspace_id` |
| `workspace.stopped` | Stop endpoint or idle TTL | `workspace_id`, `reason` |
| `proxy.access` | Request routed to workspace | `workspace_id`, `method`, `path_prefix` (first 32 chars) |
| `auth.bridge_token_issued` | Auth bridge issues short-lived token | `workspace_id`, `token_fingerprint` |

**Must NOT appear in audit details:** email, display_name, full IP,
full path, request body, response body, full google_id.

---

## Security properties

| Threat | Mitigation |
|---|---|
| CSRF on OAuth callback | `state` parameter = random nonce, verified before code exchange |
| OAuth code interception | PKCE (`S256` challenge) |
| Session fixation | New session token issued after every login |
| Cross-workspace access | Ownership check on every proxy request |
| Container escape via path | Path is stripped at proxy, no `..` passthrough |
| Token leakage in logs | Short-lived (5 min) bridge token, audit stores fingerprint only |
| Workspace enumeration | `workspace_id` is random 8-char slug, not sequential |
| Idle resource abuse | Auto-stop per plan limits; wake-on-access with 60s startup budget |
| Google JWKS cache poisoning | JWKS fetched at startup + 1 h rotation, pinned to `accounts.google.com` |

---

## Consequences

### Positive

- Single URL (`atelier-labs.net`) covers landing, login, dashboard, and
  all workspace consoles — no per-user DNS or certificates needed.
- Multiple workspaces per user enable production/staging separation without
  separate accounts.
- Operators never see a bearer token during normal use — Google login is
  the only credential they manage.
- Full GDPR compliance: consent-first, erasure-complete, audit-chained.

### Negative

- `cloud-ops` becomes stateful (SQLite DB + Docker socket). Horizontal
  scaling now requires shared storage (phase 2 concern — out of scope here).
- Wake-on-access adds up to 60 s cold-start latency for idle workspaces
  (free plan). A status page or loading screen is required in the dashboard.
- The React console SPA needs `ATELIER_CONSOLE_BASE_PATH` support —
  a small but load-bearing change to `mount_static()` and `main.tsx`.

### Out of scope

- GitHub / Microsoft / custom OIDC login (same OIDC flow, future ADR)
- Kubernetes scheduling (this ADR targets the existing single-host Hetzner setup)
- Team workspaces / shared ownership (future ADR — identity model supports it via
  a `workspace_members` table)
- Billing integration beyond plan limits (extends existing Stripe webhook)

---

## Implementation plan

### M1 — Identity + OAuth (no containers yet)

- `cloud-ops/auth/google.py`: OAuth2 login + callback, PKCE, JWKS verify
- `cloud-ops/registry.py`: SQLite schema, upsert_identity, session CRUD
- `cloud-ops/app.py`: `/auth/google`, `/auth/google/callback`,
  `require_cloud_session()` middleware
- `cloud-ops/web/workspaces.html`: placeholder dashboard (static, no JS)
- Deploy + test: login flow end-to-end on staging

### M2 — Workspace provisioning

- `cloud-ops/provisioner.py`: create / start / stop / delete
- `atelier-workspaces` Docker network creation in deploy script
- `cloud-ops/app.py`: `/v1/identity/workspaces` REST endpoints
- `cloud-ops/web/workspaces.html`: real React-ish dashboard (minimal)
- E2E test: create workspace, wait healthy, destroy

### M3 — Dynamic proxy + auth bridge

- `cloud-ops/proxy.py`: ownership-gated reverse proxy for `/app/{wid}/**`
- Auth bridge: short-lived token issue + `atelier_console_sid` handoff
- `core/gateway/atelier_gateway/app.py`: respect `ATELIER_CONSOLE_BASE_PATH`
- `core/console/.../main.tsx`: read base path from env/meta tag
- E2E test: login → create workspace → open console → full auth flow

### M4 — GDPR + hardening

- Consent gate on first login
- `atelier-cloud-erasure` CLI
- Cloud-ops audit hash-chain (reuse L16 `AuditChain` class)
- Idle auto-stop cron (per-plan TTL)
- Plan limits enforcement in provisioner
- Security review: PKCE, CSRF, path-strip, ownership gate

---

## Must NOT do

- Don't store the Google `access_token` in the DB (only `sub` + email + session).
- Don't forward `atelier_cloud_sid` to workspace containers (separate session domains).
- Don't expose Docker socket to any container other than `cloud-ops`.
- Don't skip the consent gate on any code path (including programmatic provisioning).
- Don't use sequential workspace IDs (enumeration risk).
- Don't proxy requests before the ownership check.
- Don't issue long-lived bridge tokens — 5 min max, revoked after first use.
- Don't cache Google JWKS for more than 1 hour without re-fetching.
- Don't put email or full google_id in any audit event detail.
- Don't delete identity rows on erasure — pseudonymise instead (chain integrity).
- Don't open host ports per workspace container (container-to-container only).

---

## References

- [ADR-0007 Multi-tenant](0007-multi-tenant-integration-surface.md)
- [ADR-0015 Console self-service UI](0015-console-self-service-ui.md)
- [ADR-0017 Enterprise Control Plane (OIDC Phase 4)](0017-enterprise-control-plane.md)
- [ADR-0037 Console relaunch (web-next)](0037-console-relaunch.md)
- [ADR-0045 L36 Erasure Orchestrator](0045-L36-erasure-orchestrator.md)
- [ADR-0047 Hosted Tenant Console BYOK](0047-hosted-tenant-console-byok.md)
- Google Identity OIDC: https://developers.google.com/identity/openid-connect/openid-connect
- PKCE RFC 7636: https://datatracker.ietf.org/doc/html/rfc7636
- GDPR Art. 7 (consent), Art. 17 (erasure), Art. 32 (security of processing)
- EU AI Act Art. 50 (disclosure obligation)
