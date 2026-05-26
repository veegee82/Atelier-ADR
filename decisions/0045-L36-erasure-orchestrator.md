# ADR-0045 — Layer 36: GDPR Art. 17 Erasure Orchestrator

**Status:** Accepted
**Date:** 2026-05-19
**Companion to:** [0041](0041-EU-Compliance-Atelier-OpenCode.md), [0041-GAP-ANALYSIS](0041-GAP-ANALYSIS.md), [0042 L34](0042-L34-data-classification.md), [0043 L35](0043-L35-egress-lockdown.md), [0044 L37](0044-L37-audit-at-rest.md)

---

## Context

GDPR Art. 17 grants a "right to erasure". The gap analysis identified
that AtelierOS has scattered erasure capability (L28 `/forget`, L33
artifact pre-warn) but no **cross-layer orchestration**. A real Art. 17
request from a data subject must reach every store that holds their
data: L7 skill-forge, L24 data-snapshot, L28 recall, L33 artifacts,
plus the identity-mapping that backs the audit-chain pseudonyms.

The design tension this ADR resolves:

* **Art. 17(1)** — erase personal data without undue delay.
* **Art. 17(3)(b)** — exemption when processing is necessary "for
  compliance with a legal obligation" (Art. 30 record-keeping, Art. 32
  audit obligations).
* **L16 hash chain** — rewriting sealed audit segments to remove a
  pseudonym would break tamper-evidence or require resigning every
  subsequent segment.

**Resolution (this ADR):** AtelierOS uses **pseudonymous subject_id**
strings everywhere. The Art. 17 erasure deletes (a) the *identity
mapping* (subject_id → real-world identity) and (b) all *content
stores* (recall text, artifacts, user-scoped skills). The audit chain
itself is preserved — once the mapping is gone, the chain's
`subject_id="user_42"` is no longer traceable to a person. This
matches EDPB guidance on pseudonymisation as a sufficient Art. 17
measure when full deletion conflicts with Art. 30 / 32 obligations.

## Decision

Introduce **Layer 36** as a separate structural layer:

* New module `operator/bridges/shared/erasure_orchestrator.py`.
* `ErasureRequest` (subject_id, requester, scope, ts, request_id).
* `ErasureHandler` Protocol — one per layer that holds subject data.
* `ErasureOrchestrator` — registers handlers, walks them in declared
  order, persists trail to `<tenant>/global/erasure/<request_id>.json`.
* Five new audit events on the L16 chain.
* `StubHandler` + `builtin_stub_chain()` — placeholders that audit-only
  no-op, so the operator sees a "TODO" trail per layer before real
  handlers ship.

### subject_id shape

```python
_SUBJECT_ID_RE = re.compile(r"^[A-Za-z0-9._\-]{1,128}$")
```

Forbids whitespace, `@`, `/`, control chars. Operators who try to
pass raw email get a `ValueError` — must hash or alias first. This
is the structural defence against accidental PII landing in the
audit `subject_id` field.

### Audit contract

Five event types on the L16 hash chain:

| Event | Severity | When |
|---|---|---|
| `erasure.requested` | WARNING | BEFORE any handler runs |
| `erasure.applied` | INFO | per layer, on each successful purge |
| `erasure.skipped` | INFO | per layer, when handler finds nothing |
| `erasure.failed` | CRITICAL | per layer, on handler exception |
| `erasure.completed` | WARNING | AFTER all handlers ran |

Allow-list keys: `request_id`, `subject_id`, `requester`, `scope`,
`layer_id`, `status`, `count`, `reason`, `duration_ms`,
`overall_status`, `applied_count`, `failed_count`. **Never** notes,
free-form descriptions, layer-internal state, PII content. The
`ErasureRequest.notes` field is preserved on the trail file but
**not** in the audit chain — explicit invariant tested by
`test_audit_emission_only_uses_allow_list`.

### Aggregate-status logic

```
all handlers APPLIED | SKIPPED       → COMPLETED
mix of APPLIED/SKIPPED and FAILED    → PARTIAL
all handlers FAILED OR no handlers   → FAILED
```

PARTIAL is the most interesting case — the DPO gets a result that
records which layers cleaned and which didn't, with the failure
reason captured per-handler. Operator decides retry vs accept
partial.

### Handler invariants

`ErasureHandler.purge(subject_id, request_id) -> ErasureLayerResult`
contract:

* MUST NOT raise on common cases (subject not found, layer disabled);
  return `SKIPPED` instead.
* MAY raise on infrastructure failures (DB unreachable, permission
  denied); orchestrator captures the traceback and marks `FAILED`.
* MUST return an `ErasureLayerResult`. Wrong-type return is coerced
  to `FAILED` with `reason="handler returned <type>, expected
  ErasureLayerResult"`.
* SHOULD set `layer_id` matching the registered handler's `layer_id`.
  Mismatch is corrected by the orchestrator (the registered id wins)
  with a corrective note appended to `reason`.

### Trail persistence

`<tenant>/global/erasure/<request_id>.json` (mode 0600, lock-protected
atomic-rename). Contains full request + per-layer results. Survives
across sessions so a DPO can audit "what was actually deleted for
this subject" weeks later.

### Built-in handlers

L36 ships `StubHandler` (skipped no-op) and `builtin_stub_chain()`
covering the five expected layers:

* L28-recall
* L33-artifacts
* L7-skill-forge
* L24-data-snapshot
* L16-identity-mapping

Operators register real handlers per-layer as implementations land.
Until then, an erasure call returns `COMPLETED` with five `SKIPPED`
entries — visible in both the trail file and the audit chain.

## Consequences

### Positive

* Art. 17 erasure is now a **single command** that touches every
  store — the DPO sees one trail, one set of audit events.
* `subject_id` shape regex is the **structural defence** against
  accidental PII in the audit chain.
* Handler failures don't cascade; each layer reports independently.
* `PARTIAL` status preserves the audit-of-the-audit even when one
  layer is broken — DPO can decide retry vs accept.
* No silent failure modes — every state transitions through an
  audit event with a known severity.

### Negative / trade-offs

* Audit chain is **not** redacted — pseudonyms remain. Documented as
  a deliberate compromise per EDPB pseudonymisation guidance. The
  effective Art. 17 mechanism is the identity-mapping delete.
* Real per-layer handlers are not in this ADR — they ship as
  follow-ups in the respective layers (L7, L24, L28, L33, identity-
  mapping). Until then, `builtin_stub_chain()` audits the gap.
* Operators must manage the subject_id ↔ identity mapping themselves.
  L36 does not own this mapping; it only deletes from a registered
  `L16-identity-mapping` handler that the operator provides.

### Out of scope

* The identity-mapping itself — operator-owned (could be an LDAP
  attribute, a separate user-management plugin, a column in an
  external IAM system). L36 only invokes its handler.
* Bulk erasure (multiple subjects in one call). Single-subject API
  is the documented surface; bulk is a follow-up.
* Async / queued erasure. The orchestrator runs synchronously;
  parallelism is a follow-up if real handlers ever exceed wall-time
  budgets.
* Cross-tenant erasure. ADR-0007 silos tenants; `ErasureScopeError`
  is reserved for this case (currently unused — the orchestrator
  is constructed per tenant by the caller).

## Compliance mapping

| Requirement | Source | This ADR delivers |
|---|---|---|
| Right to erasure (single subject) | GDPR Art. 17(1) | `ErasureOrchestrator.execute()` |
| Documented erasure trail | GDPR Art. 30 | `<tenant>/global/erasure/<request_id>.json` |
| Audit of erasure request | GDPR Art. 32 | `erasure.{requested,applied,skipped,failed,completed}` events |
| Pseudonymisation as Art. 17 measure | EDPB guidance | subject_id shape regex + audit chain preservation |
| Tenant isolation | ADR-0007 | One orchestrator per tenant (constructed by caller); `ErasureScopeError` reserved |

## Must NOT do

* Don't redact sealed audit segments — pseudonymisation is the
  Art. 17 mechanism here.
* Don't put `ErasureRequest.notes` or any free-form descriptor in
  audit `details` — allow-list enforces.
* Don't accept a raw email / name as `subject_id` — regex enforces
  pseudonymous shape.
* Don't make `erasure.failed` advisory; it is CRITICAL and the
  overall_status carries the consequence.
* Don't run an erasure handler across tenants — ADR-0007 silo.
* Don't make the trail file world-readable — mode 0600.
* Don't auto-retry on `FAILED` — DPO decides; the orchestrator
  records and returns.
* Don't `import anthropic` (CI lint).

## Validation

* `python3 operator/bridges/shared/test_erasure_orchestrator.py` —
  30 tests pass. Coverage:
  - subject_id shape (accepts pseudonyms, rejects PII shapes,
    length cap, non-string).
  - `ErasureRequest` validation (auto request_id, requester
    required, scope required).
  - Happy path two handlers (overall COMPLETED, audit emission
    order, severities, trail file written + mode 0600).
  - Failure paths (raising handler caught + CRITICAL audit; all
    failed → FAILED; no handlers → FAILED; bad return coerced;
    mis-attributed layer_id corrected).
  - Skipped status (INFO event).
  - Duplicate handler registration raises.
  - Audit allow-list (smuggled keys rejected at emission;
    `notes` field NOT in audit despite being in `ErasureRequest`).
  - `_aggregate_status` (every combination).
  - `StubHandler` + `builtin_stub_chain` covers expected layers.
  - CI lint (no `import anthropic`).
* `EVENT_SEVERITY` table carries the five new event types.

## Future work (later milestones)

* **L7 / L24 / L28 / L33 follow-up commits:** each layer ships its
  real `ErasureHandler`. The orchestrator already accepts them via
  `register_handler` — no L36 changes needed.
* **`atelier-erasure` CLI:** thin wrapper around
  `ErasureOrchestrator.execute()` for DPO use. M4.5 in the roadmap.
* **Console route:** admin-UI route at `/admin/erasure/<subject_id>`
  for non-CLI DPO workflow.
* **Identity-mapping plugin:** operator-chosen store for the
  subject_id ↔ real identity link. Out of L36 scope by design.
