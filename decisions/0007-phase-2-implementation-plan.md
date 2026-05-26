# ADR-0007 Phase 2 — Implementation Plan (REST API)

**Status:** Active
**Started:** 2026-05-11
**Tracks:** docs/decisions/0007-multi-tenant-integration-surface.md § Phasing → Phase 2
**Predecessor:** docs/decisions/0007-implementation-plan.md (Phase 1, closed)

Phase 2 adds the **first programmatic third-party-facing surface**:
a REST API gated by static bearer tokens, dispatching AWP-shaped runs
into the existing engine layer, with HMAC-signed webhook callbacks.

The single-operator path stays untouched: Phase 2 is **opt-in** — gated
on `gateway.enabled = true` in `service.env`. Operators who never flip
the flag never pay any Gateway cost, never expose any port, never see
any new bridge dependency.

OIDC / SCIM / per-tenant IdP integration is **Phase 3**, not Phase 2.
Container image is Phase 4. Phase 2 is the structural cut — REST + auth
+ dispatch + webhook — nothing more.

## Sub-phase fanout

| Sub-phase | Scope | E2E gate |
|---|---|---|
| **2.1** | Plugin skeleton + bearer-token auth module | New tests green; `run-all-tests.sh` 87/87 still green (no integration with adapter yet) |
| **2.2** | FastAPI app + `POST /v1/tenants/{tid}/runs` (Pydantic AWP-Run schema, run-id persistence per tenant, 202 Accepted, `GET /runs/{rid}`) | New tests green; in-process TestClient round-trip |
| **2.3** | Run-dispatch via existing engine layer (persona invocation through cowork resolver → ClaudeCodeEngine); async worker | New tests green; FAKE_CLAUDE-based E2E from POST → run completes |
| **2.4** | Webhook dispatch with HMAC-SHA256, at-least-once delivery, `run.completed` / `run.failed` / `run.budget_exceeded` | New tests green; stub-server receives webhook with verifiable signature |
| **2.5** | SSE streaming `GET /v1/tenants/{tid}/runs/{rid}/events` | New tests green; SSE client reads streamed events |
| **2.6** | End-to-end smoke (real uvicorn on a port, real HTTP client) + audit-events into the unified hash chain + CLAUDE.md update for Phase 2 closure | curl/httpx POST → run completes → audit chain `verify_chain` green |

Each sub-phase ships **one per-subtask E2E** that runs in
`bash operator/bridges/run-all-tests.sh` and turns the bridge
suite from `87/87 → 88/88` → … → `93/93`.

## Hard rules (carried from CLAUDE.md + ADR Phase 1 closure)

1. **Per-subtask E2E** — every sub-phase ships one E2E. Real subprocess
   for the engine layer (or `ADAPTER_FAKE_CLAUDE=1`), real filesystem
   for the token store, real HTTP for the webhook test.
2. **Docs-as-DoD** — behaviour / public-API / CLI-flag changes land
   with their CLAUDE.md update in the same commit.
3. **Single-operator path stays free** — `gateway.enabled = true` is
   the only switch. Tests in this phase MUST cover both the
   gateway-enabled and the gateway-disabled (no-op) path.
4. **No new `CLAUDEOS_*` env vars** — gateway infra is born under
   `ATELIER_*` only.
5. **Tenant axis is structurally honoured** — every run, every token,
   every webhook is tied to a single tenant via the Phase 1
   `validate_tenant_id` contract. Cross-tenant leaks are E2E-tested
   in 2.3 and 2.6.
6. **Bearer tokens never enter the audit chain in plaintext** —
   `tenant.token_issued` / `tenant.token_revoked` events carry a
   prefix-fingerprint (first 8 chars + length), never the full secret.
   Mirrors the `secret_vault.py` redaction contract from Layer 16 v3.

## Sub-phase 2.1 — Plugin skeleton + bearer-token auth

### Files

| File | Purpose |
|---|---|
| `core/gateway/plugin.json` | Plugin manifest (name, version, description) |
| `core/gateway/atelier_gateway/__init__.py` | Package init |
| `core/gateway/atelier_gateway/auth.py` | Bearer-token resolver: load, validate, identify-tenant |
| `core/gateway/atelier_gateway/cli.py` | `python -m atelier_gateway.cli token {issue,list,revoke}` |
| `core/gateway/tests/test_auth.py` | Per-subtask E2E (real FS, real tenant resolver) |

### Auth contract

* **Storage:** `<atelier_home>/tenants/<tid>/global/auth/gateway_tokens.json`
  — per-tenant token registry. Mode `0600`. Atomic-replace pattern
  identical to `consent.py` / `roles.py`.
* **Token shape:** `atlr_<32-hex-char-secret>`. Generated via
  `secrets.token_hex(16)` → 32 chars; the `atlr_` prefix is a
  human-recognisable disambiguator (so a token leaking into logs is
  immediately greppable).
* **Storage shape:** every token is stored as
  `{"sha256": "<hexdigest>", "issued_at": <epoch>, "label": "<free text>", "revoked_at": null | <epoch>}`.
  The plaintext token is shown to the operator **once** at issue time
  and **never** persisted. Compare-on-resolve via constant-time
  `hmac.compare_digest(sha256(presented), stored.sha256)`.
* **Resolution:** `resolve_token(presented_token) -> (tenant_id, label) | None`.
  The function walks every tenant's `gateway_tokens.json` and returns
  the first match. O(tenants × tokens) per request — acceptable for
  the Phase 2 traffic envelope (≤ 1k req/s); Phase 7 will add an
  in-memory cache when rate-limiting + gRPC land.
* **Fail-closed:** missing file, malformed file, mode > 0600, expired,
  revoked → `None`. Audit events log the failure class without ever
  echoing the presented token.
* **Tenant existence check:** the resolver MUST NOT create a tenant
  directory. A token issued for a tenant that has not been provisioned
  by Phase 1's migration helper or a future provisioning CLI returns
  `None`. This keeps the migration helper as the only `mkdir` owner.

### Audit events (Layer 16 unified chain)

| Event | Severity | Trigger |
|---|---|---|
| `gateway.token_issued` | INFO | CLI `token issue` (carries fingerprint, never plaintext) |
| `gateway.token_revoked` | INFO | CLI `token revoke` |
| `gateway.token_resolve_failed` | WARNING | Bad / unknown / revoked / expired token at runtime |
| `gateway.token_resolved` | DEBUG | Successful resolve (carries tenant + fingerprint, NOT plaintext) |

All four go through `forge.security_events.write_event` so they land
in the same hash chain `voice-audit verify` already covers.

### E2E (`test_auth.py`)

Cases:

1. Issue token → file mode 0600, file shape valid, audit event written.
2. Resolve presented token → returns (tenant_id, label).
3. Resolve wrong token → returns None + `token_resolve_failed` audit.
4. Resolve revoked token → returns None + `token_resolve_failed`.
5. Resolve token for nonexistent tenant → returns None.
6. Two tenants, one token each — cross-tenant lookup MUST NOT leak.
7. Constant-time compare — verify via `hmac.compare_digest` smoke
   (not timing-attack-grade, but assert the API is used).
8. File mode > 0600 → fail-closed with `vault_malformed`-style audit.
9. Malformed JSON → fail-closed.
10. CLI `token issue` + `token list` + `token revoke` round-trip.
11. Audit chain integrity — `verify_chain` green over all events.

Wired into `run-all-tests.sh` as
`Python: atelier-gateway auth (ADR-0007 Phase 2.1)`.

### What this sub-phase deliberately does NOT do

- No FastAPI app yet — auth is a pure-Python module with no HTTP layer.
- No webhook code.
- No engine-dispatch wiring.
- No new env-vars exposed in `service.env` — `gateway.enabled` is
  introduced by 2.2 when there's actually something to enable.

Keeping 2.1 narrow lets us prove the token resolver in isolation
before any HTTP surface depends on it.

## Sub-phase 2.2 — FastAPI app + run-submission endpoint

Outline only (full plan when 2.1 is green):

- `atelier_gateway/app.py` — `FastAPI()` instance, dependency-injected
  `resolve_token` from 2.1.
- `atelier_gateway/runs.py` — `Run` Pydantic model (AWP-shaped),
  per-tenant run persistence at
  `<atelier_home>/tenants/<tid>/global/gateway/runs/<run_id>.json`.
- `POST /v1/tenants/{tid}/runs` → 202 + `{"run_id": "..."}`.
  Tenant id in the URL MUST match the tenant identified by the bearer
  token; mismatch → 403.
- `GET /v1/tenants/{tid}/runs/{rid}` → run status + result.
- In-process `TestClient` E2E.

## Sub-phase 2.3 — Run dispatch via engine layer

Outline:

- Bridge to the existing `ClaudeCodeEngine` via the cowork resolver.
- Async worker (a single `asyncio.Task` per gateway process is enough
  for Phase 2; thread-pool comes in Phase 7 when rate-limiting lands).
- E2E uses `ADAPTER_FAKE_CLAUDE=1` for hermetic CI runs.

## Sub-phase 2.4 — Webhook dispatch

Outline:

- `atelier_gateway/webhooks.py` — `send(url, event, payload, secret)`.
- Signature header: `X-Atelier-Signature: sha256=<hexdigest>` over the
  raw JSON body. Mirrors GitHub's webhook signing convention.
- At-least-once: max 3 retries with exponential backoff (1s, 4s, 16s);
  give-up audited as `webhook.delivery_failed`.
- E2E uses a stub HTTP server bound to an ephemeral port.

## Sub-phase 2.5 — SSE streaming

Outline:

- `GET /v1/tenants/{tid}/runs/{rid}/events` returns
  `Content-Type: text/event-stream`.
- Re-uses the existing `StreamEvent` vocabulary from
  `bridges/shared/agents/__init__.py` — text_delta, tool_call,
  turn_completed, error.
- E2E: SSE client reads N events, asserts ordering matches what the
  engine emitted.

## Sub-phase 2.6 — End-to-end smoke + audit chain + CLAUDE.md

Outline:

- Real `uvicorn` on an ephemeral port for one test case.
- Verify `audit.jsonl` chain integrity across the whole flow.
- CLAUDE.md gains a "Layer 22b — Gateway REST API" section (analogous
  to Layer 22 — `WorkerEngine` protocol).

## Loss function for Phase 2

```
loss(phase2) = 1 - (
    bearer_token_isolation_e2e_pass_rate
    * single_operator_no_gateway_regression_pass_rate
    * cross_tenant_leak_e2e_pass_rate
    * webhook_signature_verifiable_pass_rate
)
```

Reaches 0 when all four invariants hold simultaneously. The
`single_operator_no_gateway_regression_pass_rate` term is the
regression gate that protects existing operators from Phase 2's
blast radius — same role E2E_B played in Phase 1.

## Cadence

One sub-phase per session. Estimated wall-clock:

| Sub-phase | Estimated effort |
|---|---|
| 2.1 | 1 session (~3 h — module + 11-case E2E + CLI) |
| 2.2 | 1 session (~3 h — FastAPI scaffolding + run persistence) |
| 2.3 | 2 sessions (~5 h — engine bridge + async worker + cross-tenant E2E) |
| 2.4 | 1 session (~3 h — webhook + retry + signature) |
| 2.5 | 1 session (~2 h — SSE re-using StreamEvent) |
| 2.6 | 1 session (~3 h — full E2E + audit + docs) |
| Phase 2 total | ~7 sessions, ~19 h focused work |

Phases 3–7 cadence will be set after Phase 2 closes.
