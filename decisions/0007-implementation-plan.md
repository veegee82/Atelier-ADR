# ADR-0007 — Implementation Plan

**Status:** Active
**Started:** 2026-05-10
**Tracks:** docs/decisions/0007-multi-tenant-integration-surface.md

This plan breaks the 7-phase ADR-0007 rollout into atomic sub-phases
that each ship a per-subtask E2E and pass `bash operator/bridges/run-all-tests.sh`
before the next sub-phase starts. The single-operator legacy path
(`<atelier_home>/global/…` direct, no tenant axis) MUST stay byte-identical
until Phase 1.4 lands.

## Phase 1 — Per-tenant scope-root (4 sub-phases)

**Goal:** add `tenant_id` as a fifth scope above (task, session,
project, user). Default tenant `_default` covers existing
single-operator paths. Audit chains, state-stores, skill-forge slots
all become tenant-aware. No new public API; the change is structural.

### Phase 1.1 — Tenant module + identity contract

| File | Action |
|---|---|
| `operator/forge/forge/tenants.py` | new — `current_tenant()`, `tenant_home(tid)`, `default_tenant_id()`, `validate_tenant_id()` |
| `operator/forge/tests/test_tenants.py` | new — per-subtask E2E for the four functions above |

**E2E gate:** new tests green; `run-all-tests.sh` still 87/87 green
(no integration yet — module exists in isolation).

**Tenant-id rules:**
- Charset: `[a-z0-9][a-z0-9_-]{0,62}` (DNS-label-like; one leading
  underscore allowed for reserved internal IDs like `_default`).
- Forbidden: path-traversal sequences (`.`, `..`, `/`, `\`), uppercase,
  unicode, whitespace.
- Resolution order: `ATELIER_TENANT_ID` env → caller-passed arg →
  `_default`. Same precedence pattern as `ATELIER_CALLER_PERSONA`.

### Phase 1.2 — Path-resolver tenant-awareness

| File | Action |
|---|---|
| `operator/forge/forge/paths.py` | add `tenant_home(tid)` returning `<atelier_home>/tenants/<tid>/`; keep `atelier_home()` unchanged |
| `operator/cowork/lib/paths.py` | mirror (byte-identical to forge) |
| `operator/bridges/shared/paths.py` | mirror |
| `operator/forge/tests/test_paths_tenant.py` | new — per-subtask E2E |

**Backward-compat rule:** every existing `<atelier_home>/global/…`
resolver continues to return the legacy path unchanged. Tenant-aware
resolvers are NEW functions, not replacements. Phase 1.4 flips the
default once the migration is in place.

**E2E gate:** new tests green; `run-all-tests.sh` 87/87 (legacy paths
still resolve identically).

### Phase 1.3 — State-stores grow tenant-axis (optional kwarg)

| Module | Function | Change |
|---|---|---|
| `roles.py` | `_store_path`, `_audit_path` | accept `tenant_id` kwarg, default `_default` |
| `consent.py` | same | same |
| `quota.py` | same | same |
| `disclosure.py` | same | same |
| `proposal.py` | same | same |
| `auth_elevation.py` | same | same |
| `dialectic.py` | `_config_path` | same |
| `ldd.py` | `_config_path` | same |

Per-subtask E2E in each module's existing test file — assert
`tenant_id="_default"` returns the legacy path, `tenant_id="other"`
returns a fresh path.

**E2E gate:** `run-all-tests.sh` 87/87 (legacy callers pass no tenant
arg → `_default` → legacy path → tests pass).

### Phase 1.4 — Audit-chain split + on-disk migration

| Path | Old | New |
|---|---|---|
| Unified audit chain | `<atelier_home>/global/forge/audit.jsonl` | `<atelier_home>/tenants/_default/global/forge/audit.jsonl` |
| All state-store dirs | `<atelier_home>/global/<store>/` | `<atelier_home>/tenants/_default/global/<store>/` |
| Sessions | `<atelier_home>/sessions/<key>/` | `<atelier_home>/tenants/_default/sessions/<key>/` |

**Migration helper** (`operator/forge/forge/tenant_migrate.py`):
- Boot-time check, idempotent.
- If `<atelier_home>/global/` exists AND `<atelier_home>/tenants/_default/global/`
  does NOT, atomic `git mv`-equivalent into `_default`.
- Leaves a `MIGRATED` marker pointing to the new location (same
  pattern as `atelier_migrate.py` from Phase 4 of the rebrand).
- Emits `tenant.path_migrated` into the audit chain BEFORE the move
  so the migration is recorded in the chain it's about to rewrite.
- Opt-out via `ATELIER_TENANT_MIGRATE=0` for tests / sensitive setups.

**E2E gate** (load-bearing for Phase 1 completion):
- E2E_A: two-tenant smoke — `_default` runs in parallel with `acme`,
  each writes own audit-chain, `verify_chain` passes independently on
  both.
- E2E_B: legacy single-tenant path — boot from clean `<atelier_home>/global/…`,
  migration fires, post-boot all 87 existing tests still pass against
  the new layout.
- E2E_C: roll-back safety — `ATELIER_TENANT_MIGRATE=0` keeps the
  legacy layout addressable; deprecation log fires once per process.

## Phase 2 — REST API (FastAPI, bearer-token)

Skeleton-only outline at this stage; full plan lands when Phase 1.4 ships.

- `core/gateway/` new plugin
- `POST /v1/tenants/{tid}/runs` accepts AWP `run.yaml`
- Webhook dispatch with HMAC-SHA256 signing
- Static bearer-token auth first; OIDC in Phase 3

## Phase 3 — OIDC + SCIM (Keycloak in CI)
## Phase 4 — OCI image + Helm chart
## Phase 5 — `.atelier-pkg` package format (Sigstore-signed)
## Phase 6 — Prometheus `/metrics` + OTLP export
## Phase 7 — gRPC API + Layer-20 quota integration

Phases 2–7 will each get their own implementation plan when their
predecessor's E2E gates are green.

## Hard rules carried from CLAUDE.md

1. **Per-subtask E2E** — every sub-phase ships one E2E hitting real
   subprocesses for MCP, real filesystem for run-workspaces, real
   `bwrap` for sandbox isolation. Mocks only where the dependency is
   pure network.
2. **Docs-as-DoD** — behaviour / public API / CLI flag changes land
   with their CLAUDE.md update in the same commit.
3. **No new `CLAUDEOS_*` env vars** — tenant infra is born under
   `ATELIER_*` only; the rebrand strangler-fig stays Phase-7-gated.
4. **Strangler-fig for legacy paths** — Phase 1.4 keeps the
   `<atelier_home>/global/` reads valid via the migration marker;
   nothing reads the legacy path after migration but the migration
   itself records its work to the chain.
5. **Single-operator path stays free** — the `_default` tenant is
   implicit, no operator action required. Setups that never enable
   the Gateway never see the multi-tenant complexity.

## Loss function for Phase 1

```
loss(phase1) = 1 - (
    cross_tenant_audit_isolation_e2e_pass_rate
    * single_tenant_legacy_path_pass_rate
)
```

reaches 0 when E2E_A AND E2E_B both pass simultaneously. E2E_B is the
regression gate that protects existing operators from the multi-tenant
refactor's blast radius.

## Cadence

One sub-phase per session. Estimated wall-clock:

| Sub-phase | Estimated effort |
|---|---|
| 1.1 | 1 session (~2 h) |
| 1.2 | 1 session (~3 h) |
| 1.3 | 2 sessions (~6 h — 8 modules) |
| 1.4 | 2 sessions (~6 h — migration + 3 E2E gates) |
| Phase 1 total | ~6 sessions, ~17 h of focused work |

Phases 2–7 cadence will be set after Phase 1 closes.
