# ADR-0007 Phase 3 — Implementation Plan (OIDC + policy)

**Status:** Active
**Started:** 2026-05-11
**Tracks:** docs/decisions/0007-multi-tenant-integration-surface.md § Phasing → Phase 3
**Predecessors:** Phase 1 (closed), Phase 2 (closed)

Phase 3 adds **identity-provider integration** plus **per-tenant
policy** on top of the Phase-2 REST surface. It is the bridge from
"static bearer tokens" (Phase 2.1) to "operator-managed identity"
that enterprise adoption requires.

Two orthogonal strands. Phase 3 ships them in sequence to keep each
sub-phase shippable on its own:

| Strand | Scope | Dependency |
|---|---|---|
| **A — Tenant config + engine policy** | `tenant.atelier.yaml` loader; per-tenant `engine_policy.json`; dispatcher gates by allowlist | Pure Python; no external service |
| **B — OIDC + SCIM** | JWT validation (via the existing bearer surface, second auth class); SCIM-stub for user provisioning; Keycloak in CI | Authlib + a running OIDC provider |

Strand A first — it is hermetic, directly extends Phase 2, and
delivers operator-visible value without spinning up Keycloak.
Strand B follows in a dedicated sub-phase fan-out when A is green.

## Sub-phase fanout (Strand A)

| Sub-phase | Scope | E2E gate |
|---|---|---|
| **3.1** | `tenant.atelier.yaml` loader + Pydantic schema; per-tenant config at `<tenant_home>/global/tenant.atelier.yaml`; CLI to inspect / template | New tests green; loader fail-closed on bad shape; Phase 2 unchanged |
| **3.2** | Engine policy: per-tenant `engine_policy.json` declares allowed engines + per-engine kwargs; dispatcher consults it before `engine_factory()` | New tests green; cross-engine denial → `run.failed` with `engine-not-allowed`; audit event emitted |
| **3.3** | Data-residency zones (re-use ADR-0004 shape from the YAML); validate run requests against zone constraints | New tests green; out-of-zone engine for tenant → 403 + audit |

## Sub-phase fanout (Strand B — outline only, full plan when 3.3 ships)

| Sub-phase | Scope |
|---|---|
| **3.4** | OIDC adapter (Authlib): accept JWTs alongside `atlr_` bearer tokens; tenant binding from JWT claims |
| **3.5** | SCIM 2.0 stub for `Users` resource (create / list / get); per-tenant user store |
| **3.6** | Keycloak-in-CI E2E: two tenants with disjoint OIDC issuers; JWT from issuer A cannot operate on tenant B |
| **3.7** | Closure: documentation, real-IdP-smoke with `docker-compose up keycloak` |

## Hard rules carried from CLAUDE.md

1. **Per-subtask E2E** — every sub-phase ships one E2E. For Strand A,
   pure Python is sufficient. For Strand B, Keycloak in a CI sidecar
   is the closure gate; integration tests run against a real OIDC
   issuer.
2. **Docs-as-DoD** — every behaviour change lands with its CLAUDE.md
   update in the same commit.
3. **Single-operator path stays free** — the Gateway is already
   opt-in; Phase 3 does not add new switches. A tenant without a
   `tenant.atelier.yaml` keeps Phase 2 defaults (every engine allowed,
   no zone constraint, every Phase-2.1 bearer token works).
4. **No new `CLAUDEOS_*` env vars** — `ATELIER_*` only.
5. **Audit-first** — every policy decision (allow / deny) lands in
   the unified hash chain. Phase 3 introduces:
   - `gateway.engine_denied` (WARNING) — run rejected by engine policy
   - `gateway.zone_denied` (WARNING) — out-of-zone engine attempted
   - `gateway.policy_loaded` (INFO) — operator-visible config reload

## Phase 3.1 — Tenant config (this commit if scope is just 3.1)

### Files

| File | Purpose |
|---|---|
| `core/gateway/atelier_gateway/tenant_config.py` | Pydantic-v2 `TenantConfig` model + YAML loader + writer |
| `core/gateway/atelier_gateway/cli.py` | New `tenant {init,show}` subcommand |
| `core/gateway/tests/test_tenant_config.py` | Per-subtask E2E (~15 cases) |

### Schema

```yaml
apiVersion: atelier/v1
kind: Tenant
metadata:
  id: acme
  display_name: ACME Corporation
  created_at: "2026-05-11T00:00:00Z"
spec:
  data_residency:
    zone: eu-west                       # optional
    allowed_engines: [claude_code]       # optional; empty = no restriction
    forbid_engines: []                   # optional
  budget:
    max_runs_per_day: 5000               # optional
    max_tokens_per_day: 10_000_000       # optional
    max_wall_clock_per_run_s: 300        # optional
```

### Loader contract

- File path: `<tenant_home>/global/tenant.atelier.yaml`.
- Missing file → `TenantConfig.default(tenant_id)` (no restriction).
- Malformed YAML / wrong shape → `TenantConfigMalformed` (fail-closed
  in the caller's discretion; Phase 3.1 just exposes the error).
- The loader does NOT create the tenant directory. Same rule as
  Phase 2's auth / runs modules.

### CLI

```
python -m atelier_gateway.cli tenant init <tenant_id>
  --display-name "ACME Corporation"
  --zone eu-west
python -m atelier_gateway.cli tenant show  <tenant_id>
```

`init` writes the YAML; `show` pretty-prints it. Subsequent
sub-phases extend `tenant` with allowlist / budget management.

### What Phase 3.1 does NOT do

- It does not enforce the policy. 3.2 wires the dispatcher; 3.3
  wires the residency zone. 3.1 is the data layer only.
- It does not gate runs by tenant config. A tenant without a config
  file behaves exactly as in Phase 2.
- It does not validate the engine names against an authoritative
  list. The list is the engine layer's concern (Phase 3.2 reads it
  from `bridges/shared/agents/`).

### Loss function for Phase 3.1

```
loss(3.1) = 1 - (
    config_round_trip_pass_rate          # write -> read identical
    * malformed_yaml_fail_closed_rate    # bad input never poisons defaults
    * phase_2_unchanged_pass_rate        # 113/113 stays green
)
```

Reaches 0 when all three invariants hold.

## Cadence

| Sub-phase | Estimated effort |
|---|---|
| 3.1 | 1 session (~2 h) |
| 3.2 | 1 session (~3 h — dispatcher integration + cross-engine E2E) |
| 3.3 | 1 session (~3 h — residency zones, AWP engine_policy compat) |
| 3.4 | 2 sessions (~5 h — Authlib integration, JWT validation) |
| 3.5 | 2 sessions (~5 h — SCIM stub) |
| 3.6 | 1 session (~4 h — Keycloak-in-CI smoke) |
| 3.7 | 1 session (~2 h — docs + real-IdP closure) |
| Phase 3 total | ~9 sessions, ~24 h focused work |

Each sub-phase is its own commit + (when shipped to a public
branch) its own PR.
