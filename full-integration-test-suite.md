# Full Integration Test Suite

This document catalogues every load-bearing test in the AtelierOS repo,
grouped by integration boundary. Tests that cross plugin boundaries
(voice ↔ forge ↔ atelier-gateway ↔ skill-forge ↔ cowork) are called
out explicitly — they are the regression gates that catch ADR-level
breakage that single-plugin suites cannot see.

The canonical runner is
[`operator/bridges/run-all-tests.sh`](../operator/bridges/run-all-tests.sh)
— one shell entry point that fans out into every suite. Gateway-side
suites are skipped (not failed) when `core/gateway/.venv/`
is absent, so single-operator deployments without the gateway
dependencies stay green.

```bash
bash operator/bridges/run-all-tests.sh
```

---

## Tenant integration tests (ADR-0007)

The fifth scope above `(task, session, project, user)` —
`tenant_id` — fans across every plugin. This section catalogues every
test that exercises a slice of the tenant contract, plus the two
cross-plugin integration tests that close the gap between per-phase
isolation tests and the production handoff.

### Phase 1.1 — Identity contract

| File | Cases | Coverage |
|---|---|---|
| `operator/forge/tests/test_tenants.py` | 28 | `validate_tenant_id` charset, `current_tenant` precedence (explicit > env > `_default`), `tenant_home` path shape, no-FS-side-effects invariant. |

Run: `cd operator/forge && python3 tests/test_tenants.py`

### Phase 1.2 — Tenant-aware path resolvers

| File | Cases | Coverage |
|---|---|---|
| `operator/forge/tests/test_paths_tenant.py` | 30 | All three `paths.py` copies (forge, voice/bridges/shared, cowork/lib) grow `tenant_home` / `tenant_global_dir` / `tenant_sessions_dir` / `tenant_forge_dir` / `tenant_skill_forge_dir` / `tenant_voice_dir` / `tenant_cowork_dir` identically. |

Run: `cd operator/forge && python3 tests/test_paths_tenant.py`

### Phase 1.3 — State-stores accept `tenant_id` kwarg

| File | Cases | Coverage |
|---|---|---|
| `operator/bridges/shared/test_state_stores_tenant.py` | 40 (4 skipped) | Eight state-store modules (roles, consent, quota, disclosure, proposal, auth_elevation, dialectic, ldd) accept `tenant_id=None` (legacy path) and `tenant_id="<tid>"` (new path under `tenants/<tid>/global/`). |

Run: `python3 operator/bridges/shared/test_state_stores_tenant.py`

### Phase 1.4 — Migration helper

| File | Cases | Coverage |
|---|---|---|
| `operator/forge/tests/test_tenant_migrate.py` | 10 | E2E_A (two-tenant smoke), E2E_B (legacy→symlink), E2E_C (`ATELIER_TENANT_MIGRATE=0` opt-out), idempotency, dry-run, no-op on fresh install, partial migration tolerance, force flag, audit-event-first ordering. |

Run: `cd operator/forge && python3 tests/test_tenant_migrate.py`

### Phase 1.3 + 1.4 join — Migration → state-store roundtrip (**NEW**)

| File | Cases | Coverage |
|---|---|---|
| `operator/forge/tests/test_tenant_migration_roundtrip.py` | 9 | Closes the strangler-fig gap: legacy state pre-migration → migrate → read/write via state-store **default branch** (`tenant_id=None`) → confirm bytes arrive in `tenants/_default/<sub>/` via the symlink. |

Cases:

- **R1 — Pre-migration seed survives the move**: legacy file content readable through symlink AND new tenant path; same applies to `roles` state.
- **R2 — Default-branch writes land under tenants/**: `tenant_global_dir()` with no arg + `roles._store_path()` with no `tenant_id` both end up in `tenants/_default/` after migration.
- **R3 — Symlink transparency**: explicit `tenant_id="_default"` and `tenant_id=None` resolve to the same physical file.
- **R4 — Audit chain continuity across migration**: events written before, during (the `tenant.path_migrated` event itself), and after migration form one verifiable hash chain via `verify_chain`.
- **R5 — Gateway-provisions-acme after migration**: cross-tenant isolation when a second tenant is mkdir'd into `tenants/acme/`; `_default` and `acme` chains do not leak.
- **R6 — Opt-out keeps legacy addressable**: `ATELIER_TENANT_MIGRATE=0` leaves `<home>/global/` as a real directory; writes via the legacy path still work.

Run: `cd operator/forge && python3 tests/test_tenant_migration_roundtrip.py`

### Phase 3.1 — `tenant.atelier.yaml` config

| File | Cases | Coverage |
|---|---|---|
| `core/gateway/tests/test_tenant_config.py` | 20 | Schema strictness (`extra="forbid"`), round-trip read/write at mode 0o600, fail-closed on world-readable file, `metadata.id == tenant_id` cross-field check, `is_engine_allowed` precedence (forbid > allowlist > default), CLI `tenant init` / `tenant show` round-trip. |

Run: `cd core/gateway && .venv/bin/python tests/test_tenant_config.py`

### Phase 1 + 6 join — Cross-plugin tenant isolation (**NEW**)

| File | Cases | Coverage |
|---|---|---|
| `core/gateway/tests/test_tenant_cross_plugin.py` | 15 | Closes the three-plugin handoff gap: voice/shared writes via state-store kwarg → forge `security_events` appends to per-tenant chain → gateway `audit_metrics` projects the same chain by `tenant_id`. |

Cases:

- **T1 — Two-tenant parallel metric projection**: acme writes 3 `tool.created` + 2 `skill.created` events, globex writes 1 of each; `audit_metrics.aggregate("acme")` and `aggregate("globex")` see only their own counter values. Prometheus body for one tenant never mentions the other.
- **T2 — Cross-tenant chain integrity**: every tenant's chain verifies independently via `verify_chain`. Tampering one (overwriting a `hash` field) breaks ONLY that tenant; the others stay clean. The `atelier_audit_chain_intact` Prometheus gauge flips per tenant accordingly.
- **T3 — Path resolver agreement**: `forge.paths.tenant_global_dir("acme") / "forge" / "audit.jsonl"` equals `audit_metrics._audit_path("acme")` (path-bytes match), and a state-store `_store_path(..., tenant_id="acme")` lands under `tenant_home("acme")`.
- **T4 — Invalid tenant id rejected at every boundary**: `../escape`, uppercase, spaces, `a/b`, very-long ID, empty — `tenants.validate_tenant_id`, `paths.tenant_global_dir`, and `audit_metrics.aggregate` all raise.
- **T5 — `ATELIER_TENANT_ID` env precedence**: env-var is the default when no kwarg passed; explicit kwarg always wins; invalid env value raises (no silent fall-through to `_default`).

Run: `cd core/gateway && .venv/bin/python tests/test_tenant_cross_plugin.py`

### Tenant test totals

| Suite | Cases |
|---|---|
| Phase 1.1 identity (`test_tenants.py`) | 28 |
| Phase 1.2 paths (`test_paths_tenant.py`) | 30 |
| Phase 1.3 state-stores (`test_state_stores_tenant.py`) | 40 (4 skip) |
| Phase 1.4 migration (`test_tenant_migrate.py`) | 10 |
| Phase 1.3+1.4 roundtrip (`test_tenant_migration_roundtrip.py`) | 9 |
| Phase 3.1 config (`test_tenant_config.py`) | 20 |
| Phase 1+6 cross-plugin (`test_tenant_cross_plugin.py`) | 15 |
| **Total** | **152** |

---

## Gateway integration tests (ADR-0007 Phases 2–7)

Every suite under `core/gateway/tests/` requires the
plugin venv (`core/gateway/.venv/`); skipped if absent.

| Phase | Suite | Focus |
|---|---|---|
| 2.1 | `test_auth.py` | Bearer-token shape, sha256-on-disk, cross-tenant rejection, fail-closed on file-mode > 0o600 |
| 2.2 | `test_app.py` | `POST /v1/tenants/{tid}/runs`, `GET /runs/{id}`, `/healthz`, Pydantic strict validation, cross-tenant 403, run-status state machine |
| 2.3 | `test_dispatcher.py` | Engine spawn → `running` → terminal, budget timeout, tenant-id env propagation into engine subprocess |
| 2.4 | `test_webhooks.py` | HMAC-SHA256 signature, retry on 5xx, no retry on 4xx, sort-keys body encoding |
| 2.5 | `test_sse.py` | Replay-for-late-subscriber, live-tap, terminal frame, process-restart fallback to disk snapshot |
| 2.6 | `test_smoke.py` | End-to-end over real uvicorn + httpx + audit-chain verify across the full lifecycle |
| 3.1 | `test_tenant_config.py` | (see Tenant section above) |
| 3.2 | `test_engine_policy.py` | Engine allowlist / forbid-list, fail-closed on malformed YAML |
| 3.3 | `test_zone_policy.py` | Data-residency `zone` gate, `"global"` escape hatch |
| 3.4 | `test_oidc.py` | JWT validation, JWKS pinning, allowed-alg, tenant-claim cross-check |
| 3.5 / 3.6 | `test_scim.py`, `test_keycloak_smoke.py` | SCIM 2.0 Users with PATCH, hermetic JWKS-URI stub |
| 4 | `test_container.py` | Dockerfile + Helm chart structural + `helm template` smoke |
| 5 | `test_packaging.py` | `.atelier-pkg` build/verify/install round-trip, ed25519 signature, path-escape rejection |
| 6.1 | `test_audit_metrics.py` | Counter projection, histogram bucket math, label whitelist, TTL cache, multi-tenant isolation |
| 6.2 | `test_metrics_endpoint.py` | `/v1/tenants/{tid}/metrics` route, auth, cross-tenant, content-type, since= filter |
| 6.4 | `test_observability_dashboards.py` | Grafana JSON parses + every panel-metric exists in the aggregator |
| 7 | `test_phase7.py`, `test_phase7_followup.py` | Durable queue, rate-limit, gRPC proto shape |

---

## Bridge / voice integration tests

Run from `operator/bridges/`:

```bash
bash run-all-tests.sh
```

Highlights of the cross-plugin tests under the bridge runner:

- **Adapter ↔ forge ↔ skill-forge end-to-end**:
  - `shared/test_adapter_security_hardening.py` — inbox re-auth (TOCTOU), `/btw` audit body snippet, chain-gap detection.
  - `shared/test_consent_gate.py` — Layer 16 Phase 4: observer-transcript flow with consent gate.
- **Skill outcome grading across turns**:
  - `shared/test_skill_outcome_grading.py` — auto-grade after bridge turn + outcome signal detection on next user turn, `/reset` and `/cancel` snapshot hygiene.
- **Voice ↔ forge secret injection**:
  - `operator/forge/tests/test_secret_injection.py` — vault → bwrap env, persona ACL, recursive envelope redaction.
- **Path-gate hook (cross-plugin enforcement)**:
  - `operator/voice/hooks/test_path_gate.py` — every Bash vector (`>`, `tee`, `eval`, `sed -i`, heredoc, `python -c open`, `xargs`, unbalanced quotes) blocked for forge / skill-forge / audit.jsonl / policy.json paths regardless of permission mode.

---

## Adding a new tenant integration test

When adding a new cross-plugin tenant test:

1. Sandbox `ATELIER_HOME` to a tempdir (`tempfile.mkdtemp`); never touch the live `~/.atelier`.
2. Provision tenants via `mkdir(parents=True)` for `<home>/tenants/<tid>/global/forge` — DO NOT call the migration helper unless that is the unit-under-test.
3. Use `forge.security_events.write_event(path, type, details=…)` for hash-chained writes; the audit chain is the single source of truth that gateway metrics project.
4. Verify isolation by writing to two tenants and asserting:
   - Their physical chain files differ.
   - `aggregate(tenant_id)` returns disjoint counter values.
   - Tampering one chain leaves the other clean (run `verify_chain` on both).
5. Wire the new file into `operator/bridges/run-all-tests.sh` next to the matching phase entry.
6. Update this document's "Total" table and add the file to the corresponding phase row.

## Audit-chain invariants every test must respect

The hash chain at `<tenant_home>/global/forge/audit.jsonl` is the
load-bearing audit boundary across the whole repo. Tests MUST:

- Never write events with `hash_chain=False` unless that is the unit-under-test (e.g. the `audit.chain_gap_detected` out-of-band event from `audit_health_check`).
- Never rewrite or truncate the chain mid-test; tampering must produce a chain that `verify_chain` correctly flags as broken — the regression value is the gate firing, not the chain getting "fixed."
- Never cross tenants in a single event's details payload — `details: {tenant: "acme"}` written into globex's chain breaks the isolation contract Phase 6's metric labels rely on.
