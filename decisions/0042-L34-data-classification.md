# ADR-0042 — Layer 34: Data Classification + Flow Guard

**Status:** Accepted
**Date:** 2026-05-19
**Companion to:** [0041-EU-Compliance-Atelier-OpenCode.md](0041-EU-Compliance-Atelier-OpenCode.md), [0041-GAP-ANALYSIS.md](0041-GAP-ANALYSIS.md)

---

## Context

ADR-0041 § Phase 2.1 calls for a "Data Flow Guard" that refuses to route
sensitive data to inappropriate engines. The gap analysis identified
this as the load-bearing structural gap: the existing
`compliance_zone_classifier` (Phase 5 / ADR-0004) reasons about
**content type** (PII present? code only?), but not about
**sensitivity grade**. The two axes are orthogonal and both must be
satisfied at every engine-spawn callsite.

Existing pieces this builds on:

* `compliance_zone_classifier.py` — PII-regex + persona-hint zoning.
* `engine_policy.py` — per-zone `allow_engines` / `deny_engines`.
* `engine_registry.py` — engine_id → `WorkerEngine` factory.
* L16 hash-chained `audit.jsonl` — append-only tamper-evident audit.
* L22 `WorkerEngine` Protocol — Claude / Codex / OpenCode engines.

The data-classification axis is what the ADR-0041 reviewer (DPO,
legal, CISO) needs to see when they ask "what happens if a developer
pastes a customer credential into a prompt?"

## Decision

Introduce **Layer 34** as a separate structural layer alongside
(not inside) the compliance-zone classifier and engine policy:

* New module `operator/bridges/shared/data_classification.py`.
* New `DataClassification` IntEnum: PUBLIC < INTERNAL < CONFIDENTIAL < SECRET.
* New `EngineCompliance` dataclass tagging each engine_id with
  `locality` (local / eu_cloud / us_cloud / unknown) and
  `network_egress` (none / local / external).
* New `DataFlowGuard` class enforcing the sensitivity × locality matrix
  with fail-closed semantics.
* New audit events: `data_flow.approved` (INFO),
  `data_flow.blocked` (CRITICAL).
* New tenant config sub-key
  `tenant.atelier.yaml::spec.data_classification.{matrix,engine_compliance}`.

The classification axis is **orthogonal** to the compliance-zone axis.
Both must permit the spawn; either can deny it.

### Default matrix (overridable per tenant)

| Classification | Allowed localities | Extra rule |
|---|---|---|
| PUBLIC | local + eu_cloud + us_cloud | — |
| INTERNAL | local + eu_cloud | — |
| CONFIDENTIAL | local | — |
| SECRET | local | `network_egress == "none"` |

### Default engine compliance

| engine_id | locality | network_egress | Notes |
|---|---|---|---|
| `claude_code` | us_cloud | external | api.anthropic.com |
| `codex_cli` | us_cloud | external | api.openai.com |
| `opencode` | unknown | external | provider-dependent; pin per-tenant |
| `opencode_ollama` | local | local | qwen3:8b via local Ollama |
| `opencode_http` | local | local | self-hosted OpenCode HTTP profile |

`opencode` defaults to `unknown` deliberately — operators must declare
intent via the tenant override. The bundled aliases
`opencode_ollama` and `opencode_http` are the EU-safe pinnings that
the EU_PRODUCTION preset (L35) will use.

### Fail-closed contract

`DataFlowGuard.validate()` returns a `FlowDecision`; never raises.
`DataFlowGuard.validate_or_raise()` is the strict variant that raises
`DataFlowDenied` on deny.

Unknown engine_id, unknown classification, and matrix-miss all deny.
The deny `matched_rule` discriminates them (`unknown_engine`,
`unknown_classification`, `matrix`, `secret_egress`) so audit
reviewers can tell *why* a spawn was refused.

### Audit allow-list

Every audit event details dict is structurally constrained to:
`classification` (enum name), `engine_id`, `persona`, `channel`,
`chat_key`, `reason`, `matched_rule`. **Never** the task text, the
prompt, or the engine output. A regression test
(`test_audit_allow_list`) walks every emitted event and fails if a
key escapes the list.

### Heuristic classifier

`classify_task(task, persona)` is the *default* classifier:

1. Explicit `[class:secret|confidential|internal|public]` marker
   (operator override; wins).
2. Secret-pattern regex bank (AWS / OpenAI / GitHub / Stripe / Slack /
   PEM private key / `password=`). Match → SECRET.
3. compliance_zone classifier hit → PII → CONFIDENTIAL.
4. Default INTERNAL.

Operators may install a custom classifier in the adapter — the guard
treats the classification argument as opaque so any caller can produce
one through any logic.

## Consequences

### Positive

* Sensitivity grading is **enforced** at every spawn site (once M2
  wires the guard into the adapter), not advisory.
* Auditors (DPO, legal) get a clear matrix to review per tenant.
* The matrix is **declarative** in tenant.atelier.yaml — operator
  changes hot-reload without touching code.
* Audit events carry only metadata; the chain itself never needs
  redaction for L34 reasons.
* The OpenCode `unknown` locality default forces operators to make an
  explicit pinning decision; no silent default routes to the wrong
  jurisdiction.

### Negative / trade-offs

* The classifier is heuristic; a missed PII regex routes a CONFIDENTIAL
  task as INTERNAL. The conservative direction (false-positive)
  routes harmless code as CONFIDENTIAL and just sends it to the local
  engine — survivable.
* Two layers (compliance_zone and data_classification) now both need
  to permit the spawn; misconfiguration in either denies. Operators
  must understand both axes.
* `make_factory()` in `engine_registry.py` is **not** yet wired to
  the guard (M1 ships the module standalone; M2 wires the adapter).
  Until M2, the guard is opt-in for code paths that construct it
  explicitly.

### Out of scope (for this ADR)

* Adapter integration — deferred to M2 along with the
  EU_PRODUCTION preset (L35), where the guard is constructed once
  per tenant and called at every spawn site.
* Egress-level enforcement (network firewall analogue) — L35.
* Tenant-level cross-layer erasure (Art. 17) — L36.

## Compliance mapping

| Requirement | ADR-0041 reference | This ADR delivers |
|---|---|---|
| Block sensitive data to external endpoints | § Phase 2.1 | `SECRET` × external denied (fail-closed) |
| Audit every flow approval / denial | § Phase 2.2 | `data_flow.approved` + `data_flow.blocked` audit events |
| Tenant-configurable allow/deny | § Phase 3.1 | `tenant.atelier.yaml::spec.data_classification` |
| Forbid Anthropic for sensitive code | § Phase 3.1 | `claude_code` locality=us_cloud → matrix denies anything ≥ INTERNAL |
| Explicit DPIA-mappable matrix | § Phase 3.2 | Tenant-config matrix + engine_compliance |

## Must NOT do

* Don't fold classification into `compliance_zone` — they are
  orthogonal axes.
* Don't make `data_flow.blocked` advisory; the guard returns an
  allowed=False decision and `validate_or_raise` raises.
* Don't put task text, prompt content, or engine output in
  audit `details` — the allow-list rejects smuggled keys at
  emission time.
* Don't ship a `claude_code` override that flips it to `local` — the
  default is load-bearing for the threat model.
* Don't widen the matrix without an ADR amendment.

## Validation

* `python3 operator/bridges/shared/test_data_classification.py` — 33
  unit tests pass (matrix, allow-list, tenant override, classifier
  heuristic, no-anthropic-import lint).
* `EVENT_SEVERITY` table in `security_events.py` carries the two new
  event types with the correct severities.
* `tenant.atelier.yaml` template carries the new sub-key with
  defaults commented out (preserves backwards compatibility for
  existing tenants).

## Future work (later milestones)

* **M2 (L35):** Wire `DataFlowGuard.from_tenant_config()` into
  `adapter.py::_pre_spawn_gate()` alongside the engine-trust gate.
  Construct the guard once per tenant; call `validate_or_raise` at
  every `engine.spawn()` callsite.
* **M2 (L35):** EU_PRODUCTION preset that pins
  `data_classification.matrix.INTERNAL` to `[local]` only, blocking
  even claude_code for general work.
* **M4 (L36):** Erasure orchestrator consumes `data_flow.*` audit
  events when redacting a user's audit trail per Art. 17 request.
