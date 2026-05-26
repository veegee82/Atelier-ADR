# ADR-0007 Phase 6 — Implementation Plan (Observability)

**Status:** Draft
**Started:** 2026-05-11
**Tracks:** docs/decisions/0007-multi-tenant-integration-surface.md § Phasing → Phase 6
**Predecessors:** Phase 1 (closed), Phase 2 (closed), Phase 3 (closing)

Phase 6 surfaces the AtelierOS runtime as queryable metrics for
operators. The single architectural commitment: **observability is a
read-side projection of the existing tenant-scoped audit chain, not a
parallel telemetry pipeline.**

## Design principle — chain-as-source-of-truth

The unified hash chain at `<tenant_home>/global/forge/audit.jsonl` is
already:

- structured (typed events with curated `details` fields),
- tenant-scoped (one chain per tenant, isolated by construction),
- hash-verified (`verify_chain` covers the full lifecycle),
- append-only (no in-place mutation),
- the canonical record every other layer already writes through.

Phase 6 does NOT introduce a second collection path. The metrics
endpoint reads the chain, projects it into Prometheus exposition
format, and serves it. No new write site; no parallel aggregator
fed by hooks scattered across the codebase.

## Sub-phase fanout

| Sub-phase | Scope | E2E gate |
|---|---|---|
| **6.1** | `audit_metrics.py` core library — pure-Python aggregator, Prometheus text exposition format, dimension whitelist | Curated metric set renders correctly from a synthetic chain; cardinality bounds enforced |
| **6.2** | `GET /v1/tenants/{tid}/metrics` Gateway endpoint — bearer-auth, tenant-scoped, TTL-cached | TestClient + uvicorn E2E; scrape returns 200 + parseable exposition; cross-tenant 403 |
| **6.3** | `voice-audit metrics` CLI subcommand — same aggregator, table/JSON/prom rendering for single-operator deployments | CLI round-trip against a sandbox chain; default tenant resolved correctly |
| **6.4** | Grafana dashboard templates + operator docs under `docs/observability/` | Templates valid JSON; documented panel-to-metric mapping |
| **6.5** | Closure — CLAUDE.md + README updates | Docs-as-DoD passes |

## Hard rules carried from CLAUDE.md

1. **Per-subtask E2E** — every sub-phase ships one E2E against a
   real audit chain. Synthetic events generated through
   `security_events.write_event`, never hand-crafted JSONL.
2. **Docs-as-DoD** — every behaviour change lands with its
   CLAUDE.md update in the same commit.
3. **Single-operator path stays free** — Phase 6 adds no required
   dependency. Operators without Prometheus use the CLI; operators
   without the Gateway never see the endpoint.
4. **No new `CLAUDEOS_*` env vars** — `ATELIER_*` only. Likely
   knobs: `ATELIER_METRICS_TTL_S` (cache), `ATELIER_METRICS_WINDOW_S`
   (default lookback).
5. **No new audit events for metric scrapes.** Scrapes are
   observability, not state. Adding scrape-audit would inflate the
   chain at scrape-rate × tenant-count cardinality.

## Phase 6.1 — `audit_metrics.py` core library

### Files

| File | Purpose |
|---|---|
| `core/gateway/atelier_gateway/audit_metrics.py` | Aggregator + Prometheus renderer |
| `core/gateway/tests/test_audit_metrics.py` | Per-subtask E2E (~15 cases) |

### Metric families (curated set)

| Name | Type | Labels | Source event |
|---|---|---|---|
| `atelier_gateway_runs_total` | counter | `status` | `gateway.run_status_changed` (terminal) |
| `atelier_gateway_run_duration_seconds` | histogram | `status` | derived from `run_created` → terminal |
| `atelier_gateway_webhooks_total` | counter | `outcome` (`delivered`, `failed`) | `gateway.webhook_dispatched` / `_failed` |
| `atelier_gateway_auth_failures_total` | counter | `reason` | `gateway.token_resolve_failed`, `gateway.oidc_resolve_failed` |
| `atelier_gateway_cross_tenant_denied_total` | counter | — | `gateway.cross_tenant_denied` |
| `atelier_gateway_engine_denied_total` | counter | `reason` | `gateway.engine_denied` |
| `atelier_gateway_zone_denied_total` | counter | — | `gateway.zone_denied` |
| `atelier_forge_tools_created_total` | counter | `persona` | `tool.created` |
| `atelier_skills_created_total` | counter | `scope` | `skill.created` |
| `atelier_dialectic_decisions_total` | counter | `site`, `mode`, `choice` | `decision.dialectical` |
| `atelier_consent_drops_total` | counter | — | `consent.observer_dropped` |
| `atelier_quota_exceeded_total` | counter | `bundle` | `quota.over_limit` |
| `atelier_path_gate_denied_total` | counter | `tool_name` | `path_gate.denied` |
| `atelier_audit_chain_events_total` | counter | — | sanity: total events read |

### Dimension whitelist

Every label value MUST come from a curated set in
`_LABEL_ALLOWLIST`. Cardinality bound per label ≤ 32. Values not in
the allowlist collapse to `"other"`. Specifically forbidden as
labels:

- token fingerprints (would be unbounded)
- `run_id` (cardinality = run count)
- `tenant_id` on per-tenant endpoint (implicit in URL)
- `target` paths, snippets, user emails

### Time window + cache

- Default window: since-process-start (rolling counters).
- Optional `?since=<duration>` (e.g. `1h`, `24h`) for bounded
  windows.
- TTL-cache: 15 s per `(tenant_id, window)`. Scrape interval
  ≥ 15 s recommended; faster scrapes hit cache.

### No SDK dependency

Prometheus exposition format is ~100 LOC to emit by hand. The
`prometheus_client` SDK pulls in a parallel collection model
(in-process registry) that doesn't fit the chain-as-source design.
Plain text formatter only.

## Phase 6.2 — Gateway `/metrics` endpoint

### Files touched

| File | Purpose |
|---|---|
| `core/gateway/atelier_gateway/app.py` | New route `GET /v1/tenants/{tid}/metrics` |
| `core/gateway/tests/test_metrics_endpoint.py` | Per-subtask E2E |

### Contract

| Method | Path | Auth | Response |
|---|---|---|---|
| `GET` | `/v1/tenants/{tid}/metrics` | Bearer (existing) | `text/plain; version=0.0.4` (Prometheus 0.0.4) |

- Tenant binding identical to Phase 2.2 (token-tenant must match
  URL-tenant; 403 on mismatch).
- Optional `?since=24h` query param.
- No state mutation; no audit event; idempotent.

## Phase 6.3 — `voice-audit metrics` CLI

### Files touched

| File | Purpose |
|---|---|
| `operator/voice/scripts/voice_audit.py` | New `metrics` subcommand |
| `operator/voice/scripts/test_voice_audit_metrics.py` | Per-subtask E2E |

### Contract

```
voice-audit metrics [--tenant <tid>] [--since <duration>]
                    [--format prom|json|table]
```

- Default `--tenant`: resolved via `current_tenant()` (`_default`
  for single-operator).
- Default `--format`: `table` (terminal-friendly).
- Same aggregator as 6.1; only the renderer differs.

## Phase 6.4 — Grafana templates + docs

### Deliverables

| Path | Purpose |
|---|---|
| `docs/observability/README.md` | Operator guide: scrape config, auth, dashboard import |
| `docs/observability/grafana/atelier-overview.json` | Per-tenant gateway health |
| `docs/observability/grafana/atelier-security.json` | Auth failures, cross-tenant denials, path-gate, consent drops |

## Phase 6.5 — Closure

- CLAUDE.md gains a "Phase 6 — Observability" section under
  ADR-0007 in the existing style.
- README.md links to `docs/observability/README.md`.
- Inventory snapshot row added.

## Out of scope for Phase 6

- OTLP traces export → Phase 7 alongside gRPC parity.
- Pushgateway / real-time push → use the existing SSE stream
  (`/v1/tenants/{tid}/runs/{run_id}/events`).
- Cross-tenant rollups → separate `/admin/metrics` endpoint with
  a different auth class; a Phase 7+ decision.
- Live dashboards inside the bridge → operator concern, not
  agent-runtime concern.

## What you, as Claude Code, must NOT do (Phase 6)

- **Don't add token fingerprints, user emails, full webhook URLs,
  `run_id`, IP addresses, or text snippets to Prometheus labels.**
  Labels are cardinality-sensitive and visible to every scraper. The
  whitelist is the structural defence; black-listing always lags
  behind new fields.
- **Don't make the metrics endpoint mutate state or write to the
  chain.** Scrapes are read-only by contract. Audit-event emission
  on scrape would inflate the chain at `scrape_rate × tenant_count`
  cardinality; for a 15 s scrape × 100 tenants that's 576k events
  per day of zero-information noise.
- **Don't introduce the `prometheus_client` Python SDK.** The text
  exposition format is stable + documented + ~100 LOC to emit.
  Adding the SDK pulls in a parallel in-process registry model that
  competes with the chain-as-source-of-truth invariant.
- **Don't widen the metric family set without an ADR amendment.**
  The current ~14 families cover the load-bearing operator
  questions (uptime, security incidents, cost-driving events). New
  families risk turning the surface into a debug-log dump.
- **Don't move aggregation into the chain-write path.** Aggregation
  is read-side projection. Write-side aggregation would create
  state that drifts from the chain and forces every event-writing
  site to know about every aggregator.
- **Don't bypass bearer-token auth on `/metrics`.** A "public
  scrape" path leaks tenant activity volume + security-incident
  cadence to anyone with network reach to the Gateway.
- **Don't fold cross-tenant aggregates into the per-tenant
  endpoint.** Per-tenant scrapes return per-tenant data only.
  Cross-tenant rollups belong on a Gateway-operator endpoint with
  a separate auth gate; mixing them would let a tenant infer
  another tenant's activity from a shared aggregate.
- **Don't read the chain without `verify_chain` integrity
  checking.** A corrupted chain produces meaningless aggregates;
  the metrics endpoint MUST surface `atelier_audit_chain_intact{ok}`
  as a gauge so operators see chain breaks via Prometheus
  alongside the `voice-audit verify` exit code.
- **Don't emit per-`run_id` histograms or gauges.** `run_id` is
  unbounded over time; pre-aggregated histograms (`_bucket`,
  `_sum`, `_count`) are the right shape, not per-run series.
