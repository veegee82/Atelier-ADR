# Implementation Roadmap — ADR-0041 EU compliance

**Status:** V2 — synchronised with [0041-GAP-ANALYSIS.md](0041-GAP-ANALYSIS.md)
**Last updated:** 2026-05-19
**Audience:** maintainer (operator)

This roadmap replaces the V1 file that was authored independently of
the gap analysis. V1 prescribed parallel structures (a second
`LLMEngine` hierarchy, a second compliance logger, a second
forbidden-endpoint list) that would have damaged load-bearing
invariants. V2 builds on what already ships and adds four structural
layers (L34–L37) plus governance docs.

The original V1 narrative is preserved as historical context in the
git history (`git log -- IMPLEMENTATION-ROADMAP.md`).

---

## Goal

> **The first agent platform structurally approved for EU
> deployment.** Not a marketing claim — every requirement maps to a
> code path with a test and an audit event.

The market argument lives in `0041-GAP-ANALYSIS.md` §7. This roadmap
is the build order.

---

## Position before this roadmap

Already shipping (claude-scoped, in production):

| Layer | Role | Regulation |
|---|---|---|
| L10 path-gate | FS write protection | GDPR Art. 32 |
| L16 audit chain | hash-chained tamper-evident `audit.jsonl` | GDPR Art. 30 / 32 |
| L16 Phase 4 | per-user consent gate | GDPR Art. 6 / 7 |
| L19 disclosure card | one-time AI-nature statement | EU AI Act Art. 50 |
| L22 WorkerEngine | engine abstraction (Claude / Codex / OpenCode CLI) | EU AI Act Art. 14 |
| L29 delegation | engine confinement (hermetic tempdir, env allowlist) | GDPR Art. 32 |
| ADR-0007 | tenant axis with `data_residency.zone` / `allowed_engines` | EU AI Act Art. 14 |
| `engine_policy.py` | declarative routing with allow/deny per zone | EU AI Act Art. 14 |
| `compliance_zone_classifier.py` | PII-regex + persona-hint zone routing | GDPR Art. 5 |
| `core/compliance/atelier_compliance_reports` | EU AI Act Art. 50, GDPR Art. 30, chain attestation | EU AI Act + GDPR |
| `core/admin/.../compliance_reports.py` | console UI for compliance reports | — |
| `core/license/atelier_license` | local RS256 JWT verification, no phone-home | — |

What this roadmap adds: **four structural layers (L34–L37) + DSB
materials + E2E suite + the two OpenCode deployment profiles** (Ollama
default, self-hosted HTTP optional).

What this roadmap explicitly does **not** add: anything in §"Out of
claude scope" below.

---

## Out of claude scope (operator + external)

These are real ADR-0041 requirements that no amount of code can
discharge:

| Item | Owner | Trigger |
|---|---|---|
| DPO signature on DPIA | DPO | After L34–L37 ship + DSB material |
| External penetration test | Security firm | After L34–L37 ship |
| CISO + Geschäftsleitung sign-off | Org leadership | After pentest passes |
| 3-month production soak | Time | After sign-off |
| GPU hardware procurement | Operator | Before OpenCode-HTTP profile |
| Actual 7B model download (~15 GB) | Operator | Before OpenCode-HTTP profile |
| Mistral DPA contract | Legal | If Mistral is enabled |
| SIEM credentials + topology | Operator | If SIEM integration desired |
| Cloud security-group / iptables rules | Operator | L35 ships *intent*; operator enforces at perimeter |

This list is what the operator must own. Everything else below is
deliverable by claude.

---

## Milestone 0 — gap analysis (done)

- ✅ `docs/decisions/0041-GAP-ANALYSIS.md` written
- ✅ `docs/decisions/IMPLEMENTATION-ROADMAP.md` (this file) synced
- → operator review

---

## Milestone 1 — Layer 34: Data Classification + Flow Guard

**Status:** next

**Scope:**

- New module `operator/bridges/shared/data_classification.py` —
  4-stage `DataClassification` enum (PUBLIC / INTERNAL / CONFIDENTIAL
  / SECRET) and a `DataFlowGuard` that consults
  `tenant.atelier.yaml::spec.data_classification.matrix` to validate
  every engine-spawn call site.
- New audit events: `data_flow.approved` (INFO),
  `data_flow.blocked` (CRITICAL).
- Wire into `engine_switch.py` at every spawn site — fail-closed
  exception when the classification × engine_id pairing is denied.
- Default matrix:
  - `PUBLIC` → any allowed engine.
  - `INTERNAL` → any engine in `data_residency.allowed_engines`.
  - `CONFIDENTIAL` → only engines tagged `compliance: local-only` in
    the registry.
  - `SECRET` → only engines tagged `compliance: local-only` AND
    `network_egress: none`.
- Tests: `test_data_classification.py` + `test_data_flow_guard.py`.
- ADR: `docs/decisions/0042-L34-data-classification.md`.
- Ref doc: `docs/claude-ref/layer-34-data-classification.md`.
- CLAUDE.md section update with must-NOT list.

**Must NOT do:**
- Don't fold classification into `compliance_zone` — orthogonal.
- Don't make `data_flow.blocked` advisory; hard refusal.
- Don't put the prompt text in `details`; classification + reason only.

**Acceptance:** `pytest operator/bridges/shared/test_data_flow_guard.py`
passes; running `claude` with `data_classification=SECRET` against a
non-local engine raises and audit-logs `data_flow.blocked`.

---

## Milestone 2 — Layer 35: Network Egress Lockdown + EU_PRODUCTION preset

**Status:** after M1

**Scope:**

- New module `operator/bridges/shared/egress_gate.py` —
  network-level analogue to L10 path-gate. Consults
  `<tenant>/global/egress_policy.json` (mode-0600, L10-protected)
  and refuses outbound connections to forbidden hosts.
- Two preset templates shipped under
  `operator/bundle/config-templates/`:
  - `tenant.atelier.eu-production-ollama.yaml` — default.
    `allowed_engines: [opencode_ollama]`, `forbid_engines: [claude,
    codex, opencode_anthropic, opencode_openai]`,
    `data_residency.zone: EU`, egress allowlist locked to the local
    Ollama socket.
  - `tenant.atelier.eu-production-http.yaml` — opt-in self-hosted
    OpenCode HTTP profile. `allowed_engines: [opencode_http]`,
    egress allowlist locked to `http://opencode-llm:8000`.
- New engine registration `opencode_ollama` (L22 OpenCodeEngine pinned
  to `--provider ollama`) and `opencode_http` (new
  `OpenCodeHttpEngine` wrapping the HTTP profile).
- New audit event: `egress.blocked` (CRITICAL).
- Boot self-test addition: `egress.preset_loaded`
  verifies the preset is internally consistent (no engine in
  `allowed_engines` references a forbidden host).
- Tests: `test_egress_gate.py` + `test_eu_production_preset.py`.
- ADR: `docs/decisions/0043-L35-egress-lockdown.md`.
- Ref doc: `docs/claude-ref/layer-35-egress-lockdown.md`.

**Must NOT do:**
- Don't rely on application-level engine policy alone; the egress
  gate is the structural defense (a misconfigured engine still has to
  go through the network).
- Don't allow EU_PRODUCTION with claude — the preset must fail loudly.
- Don't make egress decisions per-call without caching; rate it like
  L16 audit emission (best-effort metric, fail-closed decision).

**Acceptance:** with EU_PRODUCTION preset active, `curl
api.anthropic.com` from inside the adapter process fails and emits
`egress.blocked`.

---

## Milestone 3 — Layer 37: Audit-at-rest Encryption + 7y Retention

**Status:** parallel with M2 (independent)

**Scope:**

- Audit rotation: when `audit.jsonl` exceeds a configurable size or
  age, rotate to `audit.YYYY-MM-DD.jsonl`.
- Sealing: rotated segment is encrypted with AES-256-GCM
  (`age` recipient or `gpg` key — operator-chosen at install time)
  and chmod-444. Original is securely deleted post-seal.
- Chain link: the first entry of the new segment carries the prior
  segment's tail hash, so `voice-audit verify` still verifies
  end-to-end after sealing.
- New tenant config:
  ```yaml
  spec:
    audit:
      retention_years: 7
      encryption_at_rest:
        enabled: true
        recipient: "age1..."  # or gpg key id
      rotation:
        max_size_mb: 100
        max_age_days: 30
  ```
- New CLI: `voice-audit unseal <segment>` (operator-only) for legal
  hold / DPO inspection.
- New audit event: `audit.segment_sealed` (INFO), `audit.unseal_requested`
  (WARNING).
- Tests: `test_audit_rotation.py`, `test_audit_at_rest.py`,
  `test_audit_chain_across_seals.py`.
- ADR: `docs/decisions/0044-L37-audit-at-rest.md`.
- Ref doc: `docs/claude-ref/layer-37-audit-at-rest.md`.

**Must NOT do:**
- Don't break `voice-audit verify` — sealed segments must verify via
  documented unseal path.
- Don't lose the chain link across rotation boundaries.
- Don't store the encryption key alongside the sealed segments.

**Acceptance:** rotate + seal a segment; `voice-audit verify` still
passes end-to-end including sealed segments.

---

## Milestone 4 — Layer 36: DSGVO Art. 17 Erasure Orchestrator

**Status:** after M1–M3 (touches all other layers)

**Scope:**

- New CLI / endpoint: `atelier-erasure <user_id>` and
  console route `/admin/erasure/<user_id>`.
- Orchestrates per-layer erasure plan:
  1. L28 recall — `DELETE FROM turns WHERE user_id = ?`
  2. L33 artifacts — list unpinned, purge; pinned require operator
     confirmation.
  3. L7 skill-forge — purge user-scope skills referencing the user.
  4. L24 data-snapshot — purge if user_id appeared in snapshot
     metadata (snapshots are PII-redacted at creation, so this is
     rare).
  5. L16 audit chain — selective redaction. Replace `user_id` in
     prior events with `_redacted` + emit `erasure.applied` event
     referencing the prior `audit_id`. Re-sign the affected segment.
  6. L34 data_flow logs — same redaction protocol.
- Per-request file: `<tenant>/global/erasure/<request_id>.json`
  (intent + plan + per-layer progress + completion).
- New audit events: `erasure.requested` (WARNING),
  `erasure.applied` (INFO, per layer), `erasure.completed` (WARNING),
  `erasure.failed` (CRITICAL).
- Tests: `test_erasure_orchestrator.py` (cross-layer integration),
  `test_audit_redaction.py` (chain integrity after redaction).
- ADR: `docs/decisions/0045-L36-erasure-orchestrator.md`.
- Ref doc: `docs/claude-ref/layer-36-erasure.md`.

**Must NOT do:**
- Don't delete audit-chain entries — redact in place + emit
  `erasure.applied` referencing the prior `audit_id`.
- Don't run erasure across tenants.
- Don't make erasure asynchronous-best-effort — regulatory
  obligation; must report success or named failure per layer.

**Acceptance:** `atelier-erasure user_42` purges across all layers,
emits one `erasure.applied` per layer + one `erasure.completed`;
post-erasure `voice-audit verify` still passes (chain integrity
preserved).

---

## Milestone 5 — DSB material + privacy notice

**Status:** parallel with M1–M4

**Scope:**

- `docs/compliance/DSB-CHECKLIST.md` — concrete operator checklist
  (German), based on the gap analysis §7 list.
- `docs/compliance/DPIA-TEMPLATE.md` — DPIA template
  pre-populated with AtelierOS architecture facts (German + English).
- `docs/compliance/PRIVACY-NOTICE-TEMPLATE.md` — privacy notice
  template (German + English) the operator can customise per
  deployment.
- `docs/compliance/PENTEST-SCOPE.md` — pentest scope document for
  the external firm (so the operator can hand it over directly).
- `docs/compliance/COMPLIANCE-REPORT-GUIDE.md` — how to use the
  existing `atelier-compliance-reports` plugin + console route to
  generate monthly reports.

**Not deliverable here:** any of these signed by a DPO or counsel.
That is the operator's job.

---

## Milestone 6 — EU compliance E2E suite

**Status:** alongside each layer; consolidated last

**Scope:**

- `operator/bridges/shared/test_eu_compliance.py` — five canonical
  tests, each gated to fail loudly if any structural defense is
  weakened:
  1. `test_no_egress_to_anthropic` — EU_PRODUCTION preset blocks the
     network call.
  2. `test_classification_secret_only_local` — SECRET classification
     fails closed against any non-local engine.
  3. `test_audit_trail_complete` — every engine spawn produces an
     audit entry with the expected schema.
  4. `test_erasure_cross_layer` — `atelier-erasure user_42` purges
     across L7 / L24 / L28 / L33 and redacts L16.
  5. `test_sealed_audit_verifies` — sealed rotated segments still
     verify via `voice-audit verify`.
- Each test ships with the layer that makes it pass; M6 is the
  consolidation pass.

**Acceptance:** `pytest operator/bridges/shared/test_eu_compliance.py`
passes green on a fresh checkout.

---

## Honest timeline

V1 quoted 12 weeks. V2 estimate, based on the gap analysis:

| Milestone | Effort (claude sessions) | Wall-clock (operator-paced) |
|---|---|---|
| M0 gap + roadmap | done | done |
| M1 L34 | 1–2 | 1 day |
| M2 L35 + EU_PROD preset | 2–3 | 2 days |
| M3 L37 audit-at-rest | 2 | 1 day |
| M4 L36 erasure orchestrator | 3–4 | 2 days |
| M5 DSB material | 1–2 | 1 day |
| M6 E2E suite consolidation | 1 | 0.5 day |
| **Total claude-scoped** | ~12 sessions | ~1 work-week |
| External: pentest | — | 2–4 weeks |
| External: DPO/legal sign-off | — | 2–6 weeks |
| Soak | — | 12 weeks |

The claude-scoped delta is **~1 week of build-out**, not 12 weeks. The
12-week original number conflated build, organisational, and time-based
work.

---

## Decision log

- 2026-05-19 — operator chose "Ollama default + HTTP optional" for the
  OpenCode profile. Both presets ship under M2.
- 2026-05-19 — operator chose "Roadmap V2 → L34" delivery order.

---

## Continuous compliance (after rollout)

| Cadence | Owner | Action |
|---|---|---|
| Daily | adapter | `voice-audit verify` timer (already shipping) |
| Weekly | operator | review `audit.jsonl` for anomalies + `data_flow.blocked` events |
| Monthly | operator | generate compliance report via console route |
| Quarterly | operator | internal security audit |
| Annually | external firm | penetration test + DPIA review |
| On demand | DPO | `atelier-erasure` for Art. 17 requests |
