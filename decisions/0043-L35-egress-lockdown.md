# ADR-0043 — Layer 35: Network Egress Lockdown + EU_PRODUCTION preset

**Status:** Accepted
**Date:** 2026-05-19
**Companion to:** [0041-EU-Compliance-Atelier-OpenCode.md](0041-EU-Compliance-Atelier-OpenCode.md), [0041-GAP-ANALYSIS.md](0041-GAP-ANALYSIS.md), [0042-L34-data-classification.md](0042-L34-data-classification.md)

---

## Context

ADR-0041 § Phase 3.1 + Phase 4.1 require that, under the EU
deployment profile, no outbound network call may reach `api.anthropic.com`
or `api.openai.com`. The gap analysis identified two parts of this:

1. **In-process refusal** — refuse to spawn an engine whose
   network_egress would target a forbidden host. This is what L35
   delivers, alongside the L34 classification × locality matrix.
2. **Real network isolation** — iptables / docker network / cloud
   security-group rules. This is **out of claude scope** and belongs
   to the operator's deployment.

L35 is the structural defence; the perimeter firewall is the
operational defence. Both are required for the EU compliance claim
to hold up under audit.

## Decision

Introduce **Layer 35** as a separate structural layer:

* New module `operator/bridges/shared/egress_gate.py`.
* New `EgressPolicy` dataclass (`enabled`, `default_action`,
  `allowed_hosts`, `forbidden_hosts`).
* New `EgressGate` class with fail-closed semantics (`validate` /
  `validate_or_raise`), default-disabled for back-compat.
* New audit events: `egress.approved` (INFO),
  `egress.blocked` (CRITICAL), `egress.preset_loaded` (INFO).
* New tenant config sub-key `tenant.atelier.yaml::spec.egress`.
* Two shipped EU_PRODUCTION tenant templates:
  - `tenant.atelier.eu-production-ollama.yaml` — default (recommended).
  - `tenant.atelier.eu-production-http.yaml` — self-hosted HTTP variant.

### Policy precedence

1. `forbidden_hosts` (explicit deny) — always wins.
2. `allowed_hosts` (explicit allow).
3. `default_action` (allow | deny) for unmatched hosts.

When `enabled=False` the gate is a pass-through (matched_rule
`egress_disabled`, no audit emission). This is the back-compat
default for pre-L35 tenants.

### Host canonicalisation

Operator-curated entries are lower-cased and stripped; entries that
fail the hostname regex (`^[A-Za-z0-9](?:[A-Za-z0-9.\-]{0,253})$`)
raise `ValueError` at load time. Host strings that fail the regex at
validation time are denied (fail closed) and audit-logged.

The same canonicalisation is applied to both lists *and* the
validation argument — a forbidden `API.Anthropic.com` and an inbound
`api.anthropic.com` match.

### Audit allow-list

Audit details: `host`, `engine_id`, `persona`, `channel`, `chat_key`,
`reason`, `matched_rule`. **Never** the URL path, the query string,
the request body, or any response content. Regression test asserts
the set.

### EU_PRODUCTION presets

Both shipped presets carry:

* `spec.data_residency.{zone: EU, allowed_engines, forbid_engines}` —
  engine identity gate (ADR-0007).
* `spec.data_classification.{matrix, engine_compliance}` — L34
  sensitivity × locality gate, with matrix tightened so that
  *every* row maps to `[local]` (no row admits a cloud engine).
* `spec.egress.{enabled: true, default_action: deny, allowed_hosts,
  forbidden_hosts}` — L35 network-level refusal.
* Forward-declared `spec.audit.{retention_years: 7,
  encryption_at_rest, rotation}` for L37 (M3 in the roadmap).

The matrix-row + engine-allowlist + egress-allowlist triple is the
"three layers of defence" property that an auditor can verify:

1. *Identity:* the only allowed engine is local.
2. *Classification:* even if a future engine were added with locality
   `us_cloud`, the matrix would refuse spawns for any classification
   ≥ INTERNAL.
3. *Network:* even if a spawn somehow occurred, the egress gate
   would refuse `api.anthropic.com` host resolution.

A single layer can be bypassed by misconfiguration; passing all three
requires *deliberate* multi-step weakening that no single environment
variable or runtime flag enables.

### Preset consistency check

`EgressGate.validate_preset_consistency()` is the function called by
the boot self-test to emit warnings on common operator mistakes:

* `forbidden_hosts` listed but `enabled=False` → nothing is actually
  blocked.
* `default_action=deny` + empty `allowed_hosts` → policy denies
  everything including loopback.
* An engine in the preset's `allowed_engines` has
  `network_egress=external` → L35 can refuse the spawn but cannot
  enforce runtime confinement; perimeter firewall is required.

Boot self-test emits `egress.preset_loaded` (INFO) when the load
passes; failing checks surface as WARNING via the unified self-test
output.

### Wiring status

* **M2 (this milestone — done):** Module + tests + two tenant
  presets + EVENT_SEVERITY + ADR + ref doc. Standalone; opt-in.
* **M2.5 (next):** Adapter wires `EgressGate.from_tenant_config()`
  alongside `DataFlowGuard.from_tenant_config()` (L34) into
  `_pre_spawn_gate()`. Same wire-in point; deliberate symmetry.
* **M2.6 (next):** Boot self-test gains an `egress.*` check group
  per `bridge.sh doctor`.

## Consequences

### Positive

* Three layers of structural defence (identity / classification /
  egress) — auditor-verifiable claim.
* Declarative policy in tenant config — hot-reloadable, no code
  changes per tenant.
* Audit events trace every approval and refusal to the L16 hash chain
  with the same allow-list pattern as L34.
* Two presets cover the two realistic EU deployment shapes; operators
  pick one and ship.

### Negative / trade-offs

* Real network isolation is *not* delivered by L35 — operator must
  also configure perimeter firewall. L35 documents this explicitly
  (see "Out of scope" in the module docstring + the preset
  templates).
* `default_action=deny` requires the operator to list every legitimate
  host. The cost is fragility on new infrastructure (forgot to
  whitelist the local Ollama LAN host); the benefit is fail-closed
  semantics on new hostile infrastructure.
* Both EU_PRODUCTION presets tighten the L34 matrix so that even
  PUBLIC is `[local]`-only. Operators who want a "default-EU plus
  occasional public bulletins" profile need a third preset.

### Out of scope

* In-process Network-Hook (Python `socket` monkeypatch / urllib
  intercept) — brittle, by-passable by any subprocess, false sense
  of security. Documented as a deliberate non-goal.
* Cloud security-group / iptables generation. Operator's job.
* The `OpenCodeHttpEngine` worker registration — needs a builder in
  `operator/bridges/shared/agents/`. Marked TODO in the
  `eu-production-http.yaml` preset header.

## Compliance mapping

| Requirement | ADR-0041 reference | This ADR delivers |
|---|---|---|
| No outbound calls to Anthropic / OpenAI | § Phase 3.1 | `forbidden_hosts` covers the canonical hosts + `default_action=deny` covers everything else |
| Audit every approved / refused egress | § Phase 2.2 | `egress.approved` + `egress.blocked` audit events |
| Tenant-configurable allow/deny | § Phase 3.1 | `tenant.atelier.yaml::spec.egress` |
| EU_PRODUCTION deployment mode | § Phase 3.1 | Two shipped preset templates |
| DPIA-mappable claim "no data leaves the host" | § Phase 3.2 | Three-layer defence: identity + classification + egress |

## Must NOT do

* Don't enable `default_action=deny` without listing the legitimate
  hosts the engine needs — would deny localhost loopback too.
* Don't add `api.anthropic.com` / `api.openai.com` to `allowed_hosts`
  in any EU_PRODUCTION preset — the preset must fail loudly if it
  contradicts its own forbidden list.
* Don't put URL paths, request bodies, or response content in audit
  `details` — allow-list enforces.
* Don't make L35 the *only* defence — it refuses spawns, the
  perimeter firewall enforces the network boundary. Documenting
  both is load-bearing.
* Don't make `egress.blocked` advisory; fail-closed contract
  matching L34.
* Don't `import anthropic` from `egress_gate.py` (CI lint).

## Validation

* `python3 operator/bridges/shared/test_egress_gate.py` — 33 unit
  tests pass (canonicalisation, disabled passthrough, forbidden
  precedence, default-deny / default-allow, malformed host fail-
  closed, validate_or_raise, audit allow-list, tenant config loader,
  flat / wrapped JSON file shape, preset consistency, no
  anthropic import).
* `EVENT_SEVERITY` table carries the three new event types with
  correct severities.
* Two preset templates validate as YAML.

## Future work (later milestones)

* **M2.5:** Adapter `_pre_spawn_gate()` wiring for L34 + L35.
* **M2.6:** `bridge.sh doctor` adds an `egress.*` check group calling
  `EgressGate.validate_preset_consistency()`.
* **M3 (L37):** `spec.audit.encryption_at_rest` becomes enforced.
* **M6:** E2E test `test_no_egress_to_anthropic` exercises the full
  preset → adapter → engine path.
