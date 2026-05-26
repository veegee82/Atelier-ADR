# ADR-0041 Gap Analysis — what already ships, what is missing, what is out of scope

**Status:** Companion to [0041-EU-Compliance-Atelier-OpenCode.md](0041-EU-Compliance-Atelier-OpenCode.md)
**Date:** 2026-05-19
**Audience:** maintainer (operator), DPO, legal review

This document maps every requirement of ADR-0041 against the **current**
AtelierOS codebase. It exists because the ADR was authored without
reference to the layer architecture that the project has accumulated
since Phase 5 (ADR-0004) and the L22 / L29 engine refactors. Following
ADR-0041 literally would create parallel structures (a second
`LLMEngine` hierarchy, a second compliance logger, a second
forbidden-endpoint list). This document is the corrective pass.

---

## 1. Executive summary

| Bucket | Count | Notes |
|---|---|---|
| **Already shipped** — production code path | 9 | Engine abstraction, audit chain, compliance reports, console, license, zone classifier, engine policy, consent gate, disclosure card |
| **Structurally missing — claude-scoped** | 4 | New layers L34–L37 |
| **Documentation / governance — claude-scoped** | 3 | DSB checklist, DPIA template, privacy notice |
| **Out of claude scope** | 6 | DPO signature, external pentest, 3-month soak, GPU procurement, Mistral DPA, real SIEM integration |

**Conclusion:** ADR-0041 is **substantially deliverable as a structural
extension** of the existing stack. It is **not** a greenfield project,
and treating it as one would damage the load-bearing invariants the
codebase already enforces (hash-chained audit, engine policy, tenant
scope, path-gate). The right framing is: *four new layers + governance
docs + an E2E suite, all wired into the existing infrastructure.*

---

## 2. Requirement → reality mapping

Each row: what ADR-0041 asks for, where it already lives, what the gap
is. "Path" is anchored at the repository root.

### 2.1 LLM engine abstraction (ADR § Phase 1.1)

| ADR ask | Reality | Gap |
|---|---|---|
| `LLMEngine` ABC | `operator/bridges/shared/worker_engine.py` (L22) — `WorkerEngine` Protocol with `spawn()` / `cancel()` / `inject()` | None. **Do not create a second hierarchy.** |
| `OpenCodeEngine` class | Same file — `OpenCodeEngine` already implemented (provider-agnostic: Claude / OpenAI / Ollama) | The ADR's `OpenCodeEngine` is a third-party HTTP server at `localhost:8000`. The AtelierOS `OpenCodeEngine` wraps the `opencode` CLI. **These are not the same thing.** Decide which one we mean (see §4). |
| `AnthropicEngine` (dev-only) | `ClaudeCodeEngine` (production, full feature set) | The ADR's framing ("Anthropic only for dev") is inverted from reality. The project's L29 delegation explicitly puts `claude` at the OS layer and other engines as workers. |
| Engine factory + selection | `engine_registry.py` + `engine_switch.py` (deterministic dispatch via `engine_id`) | None. |
| Routing policy | `engine_policy.py` — `default_engine`, `fallback_chain`, `compliance_zones` with `allow_engines` / `deny_engines` | None. The ADR's `routing.default` / `routing.fallback` / `routing.forbidden_for_sensitive_code` already map 1:1. |

**Verdict:** L22 + `engine_policy` cover Phase 1.1 fully. The only
remaining decision is *what does "opencode" mean to us* (CLI worker vs.
self-hosted HTTP server — see §4).

### 2.2 Data flow guard (ADR § Phase 2.1)

| ADR ask | Reality | Gap |
|---|---|---|
| Endpoint allow/deny list | `engine_policy.json::compliance_zones.<zone>.{allow_engines, deny_engines}` | Operates on **engine_id**, not **HTTP endpoint**. An engine_id like `claude` resolves through the `claude` CLI binary, which then calls `api.anthropic.com`. The endpoint is one layer removed from the policy. |
| Data classification (PUBLIC/INTERNAL/CONFIDENTIAL/SECRET) | `compliance_zone_classifier.py` — `general` / `code_only` / `personal_data` / `external_facing` | **Orthogonal axes.** The classifier reasons about *content-type-presence* (PII regex). The ADR reasons about *sensitivity-grade*. Both are valid and we need both. |
| `validate_endpoint()` raises `SecurityException` | Engine policy returns an allowed-list; the adapter's `engine_switch` picks an allowed engine or falls through to `default_chain` | Soft routing, not hard refusal. There is no equivalent of "fail closed and audit-log a violation." |
| Audit-log violation events | `engine.policy_skipped` and `engine.canary_failed` exist | Missing: `data_flow.blocked` for a refused classification → endpoint pairing. |

**Verdict:** This is the **load-bearing gap**. ADR-0041's `DataFlowGuard`
maps to a **new orthogonal layer** that combines (a) data
classification, (b) network-endpoint awareness, and (c) fail-closed
refusal with audit. → **L34 (Data Classification + Flow Guard)** below.

### 2.3 Compliance / audit logging (ADR § Phase 2.2)

| ADR ask | Reality | Gap |
|---|---|---|
| Per-execution audit entry with prompt-hash, length, duration | `audit.jsonl` already records every engine spawn with hash-chained integrity | The ADR's schema lacks the hash-chain link; ours has it. **Do not weaken our schema** to match the ADR. |
| 7-year retention | Daily TTL sweep, 7-day default for session scope | Retention is **session-scoped**, not **audit-scoped**. `audit.jsonl` is currently append-only forever — we need an explicit retention policy + rotation. |
| Audit immutability | Hash-chain link + `voice-audit verify` daily timer + mode-0600 | Read-only `chmod 444` after rotation: missing. Encryption-at-rest: missing. |
| Compliance report generation | `core/compliance/atelier_compliance_reports` (EU AI Act Art. 50, GDPR Art. 30, chain attestation) + `core/admin/atelier_admin/routes/compliance_reports.py` (console UI) | None for the canonical reports. The ADR's monthly-PDF-export is one *consumer* of these reports, not a new system. |
| SIEM integration | Filebeat sidecar suggested in ADR docker-compose | Not in repo; this is an operator deployment concern, not a code concern. |

**Verdict:** Most of Phase 2.2 is **already shipping**. The honest gaps
are (a) rotation + retention policy, (b) encryption-at-rest for rotated
files, (c) chmod-444 after rotation. → **L37 (Audit-at-rest +
Retention)** below.

### 2.4 Enterprise configuration (ADR § Phase 3.1)

| ADR ask | Reality | Gap |
|---|---|---|
| `enterprise/atelier-compliance-config.yaml` | `operator/bundle/config-templates/tenant.atelier.yaml` (ADR-0007 axis: `(task, session, project, user, tenant_id)`) | Naming differs; the layout is sound. The ADR's flat yaml conflates *tenant-scoped* (data_residency, allowed_engines) and *cluster-scoped* (legal docs, encryption-at-rest) — our model already separates these correctly. |
| `mode: EU_PRODUCTION` | No preset profile | Missing: a shipped `tenant.atelier.eu-production.yaml` template with `allowed_engines=[opencode]`, `forbid_engines=[claude, anthropic, openai]`, `data_residency.zone=EU`, hard egress lockdown. → **L35 (EU_PRODUCTION preset)**. |
| Audit retention 7 years | Session TTL only | See §2.3. |
| Forbidden engines list (anthropic, openai) | Engine policy allows per-zone deny, but no shipped EU-preset uses it | Same. → **L35**. |
| Classification → allowed-endpoints matrix | Not present | New mechanism. → **L34**. |
| Encryption: AES-256 at rest, TLS 1.3 in transit | TLS handled by gateway/ingress (operator concern), AES-256 missing | → **L37**. |
| Approval chain (DPO / Legal / Security / CISO sign-off) | Not a software concern | Out of claude scope. Belongs in the governance docs. |

### 2.5 DSB / DPIA / privacy notice (ADR § Phase 3.2 + 3 + 4.2)

| ADR ask | Reality | Gap |
|---|---|---|
| `docs/DSB-CHECKLIST.md` | Not present | New markdown. Claude-scoped. |
| `docs/DPIA-TEMPLATE.md` | Not present | New markdown. Claude-scoped (but signature/legal review is not). |
| Privacy notice (German + English) | Not present | New markdown. Claude-scoped for structure; legal review is not. |
| Pentest checklist | Not present | New markdown. Claude-scoped for the *checklist*; the *actual pentest* is not. |

### 2.6 Right to deletion (ADR § Phase 4.1)

| ADR ask | Reality | Gap |
|---|---|---|
| GDPR Art. 17 erasure | `/forget` exists for L28 (recall.db) and L33 (artifact pre-warn) | Single-purpose, not cross-layer. A real Art. 17 request must purge: L7 skill-forge user-scope skills, L28 recall, L33 artifacts, L16 audit (selective redaction, not deletion, to preserve the chain), L24 data-snapshot policy. → **L36 (DSGVO Erasure Orchestrator)**. |
| Audit-log redaction for deleted users | Audit chain is append-only by design (tamper-evidence) | Tension with Art. 17. Resolution: insert an `erasure.applied` event referencing the redacted prior event by `audit_id`, replace the `user_id` field in the original with `_redacted`, and re-sign the chain segment. **This is non-trivial and load-bearing.** → **L36**. |

### 2.7 OpenCode deployment (ADR § Phase 1.2)

| ADR ask | Reality | Gap |
|---|---|---|
| `Dockerfile.opencode` | Not present | Can be drafted, but **cannot be tested** without a GPU machine + ~15 GB model download. |
| `docker-compose.eu-compliance.yml` | Not present | Same. |
| 7B model download | `python3 -m opencode.download --model opencode-7b-instruct` | Out of claude scope — this is a real runtime operation on the operator's hardware. |
| Performance baseline | — | Out of claude scope — requires real hardware. |

### 2.8 Penetration test + production validation (ADR § Phase 4)

| ADR ask | Reality | Gap |
|---|---|---|
| Network-level egress tests | Some E2E tests touch real `claude` CLI | A dedicated `test_eu_compliance.py` that asserts "from inside the EU_PRODUCTION container, `curl api.anthropic.com` fails" is missing. → claude-scoped E2E suite. |
| External penetration test | — | **Out of claude scope.** Requires a third-party firm. |
| 3-month production soak | — | **Out of claude scope.** Wall-clock time. |
| DPO sign-off | — | **Out of claude scope.** Human/legal. |

---

## 3. Proposed new layers (claude-scoped delivery)

The ADR's intent maps cleanly onto **four new structural layers**, each
following the existing convention (CLAUDE.md `## Layer N — <name>`
section, ref doc under `docs/claude-ref/`, full ADR, tests, must-NOT
list, audit events).

### Layer 34 — Data Classification + Flow Guard

**Purpose:** orthogonal sensitivity axis (PUBLIC / INTERNAL /
CONFIDENTIAL / SECRET) layered on top of L22 engine selection and the
existing compliance-zone classifier. Fail-closed refusal when a
classification can not be matched to any allowed engine.

**Wires into:**
- L22 (`engine_switch`) — consults `data_classification` before
  selecting an engine.
- L16 (audit chain) — emits `data_flow.blocked` (CRITICAL) on refusal,
  `data_flow.approved` (INFO) otherwise.
- ADR-0007 tenant axis — `tenant.atelier.yaml::spec.data_classification`
  (new sub-key) declares the matrix.

**Storage:** none — pure policy applied at every engine-spawn callsite.

**Must NOT do:**
- Don't fold classification into `compliance_zone` — they are orthogonal
  (zone = content-type-presence; classification = sensitivity-grade).
- Don't make `data_flow.blocked` advisory; it is a hard refusal that
  surfaces an exception to the user.
- Don't put the prompt text in the audit `details` field; classification +
  reason only.

### Layer 35 — Network Egress Lockdown + EU_PRODUCTION preset

**Purpose:** pre-configured `tenant.atelier.yaml` preset
(`tenant.atelier.eu-production.yaml`) that locks the tenant into
EU-only engines and emits an egress block at the OS level if any
network call attempts to reach a forbidden endpoint.

**Wires into:**
- ADR-0007 (`data_residency.zone`, `allowed_engines`, `forbid_engines`)
- L10 (path-gate) — analogue at network level: `egress_gate`.
- L16 (audit chain) — emits `egress.blocked` on attempted forbidden
  call.

**Storage:** `<tenant>/global/egress_policy.json` (declarative;
hot-reloaded; mode-0600; protected by L10 path-gate).

**Must NOT do:**
- Don't rely on application-level engine policy alone — the egress
  gate is the structural defense (a misconfigured engine still has to
  go through the network).
- Don't allow "EU_PRODUCTION with claude" — the preset must fail
  loudly if `allowed_engines` contains anything that talks to
  `api.anthropic.com`.

### Layer 36 — DSGVO Art. 17 Erasure Orchestrator

**Purpose:** cross-layer right-to-deletion. Coordinates L7 skill-forge,
L28 recall, L33 artifacts, L24 data-snapshot, L16 audit (selective
redaction with chain re-signing).

**Wires into:** every layer that holds user data.

**Storage:** `<tenant>/global/erasure/<request_id>.json` (intent + plan
+ progress). One file per Art. 17 request.

**Must NOT do:**
- Don't delete audit-chain entries — redact in place + emit
  `erasure.applied` referencing the prior `audit_id`.
- Don't run erasure across tenants (the L29 confinement holds).
- Don't make erasure asynchronous-best-effort — it is a regulatory
  obligation and must report success or named failure per layer.

### Layer 37 — Audit-at-rest Encryption + 7y Retention

**Purpose:** encrypt rotated `audit.jsonl` segments with AES-256-GCM,
declare a 7-year retention floor, and chmod-444 rotated files. The
running chain stays writable; only rotated segments are sealed.

**Wires into:**
- L16 (audit chain) — rotation hook calls L37 sealer.
- ADR-0007 tenant axis — `tenant.atelier.yaml::spec.audit.{retention_years, encryption_at_rest}`.

**Storage:** `<tenant>/global/audit/sealed/<segment>.jsonl.age` (or
`.gpg`, key-management is operator-chosen).

**Must NOT do:**
- Don't break `voice-audit verify` — sealed segments must still verify
  via a documented unseal-and-verify path.
- Don't lose the chain link across rotation boundaries — the first
  entry of each new segment is the prior segment's tail-hash.

---

## 4. Open decision — what does "OpenCode" mean here?

ADR-0041 § Phase 1.2 ships a **GPU-hosted self-contained HTTP server**
that exposes `/v1/completions` on port 8000. AtelierOS L22 already
has an **OpenCode CLI wrapper** that drives the `opencode` binary
(provider-agnostic, supports Claude / OpenAI / Ollama).

These are not interchangeable:

|  | ADR's OpenCode | AtelierOS L22 `OpenCodeEngine` |
|---|---|---|
| Model location | Local on GPU | Wherever the configured provider is |
| API surface | OpenAI-compatible HTTP | `opencode` CLI stdin/stdout |
| EU compliance | Yes (model on-prem) | **Depends** on provider — `--provider ollama` is EU-safe, `--provider anthropic` is not |
| Cost | One-time GPU + electricity | Per-token if cloud provider |

**Recommendation:** the EU_PRODUCTION preset pins
`OpenCodeEngine --provider ollama` (local Ollama serving a 7B model);
the ADR's hypothetical separate HTTP server is *redundant* given L22.
This decision needs operator confirmation before L35 is written.

---

## 5. Out of claude scope (explicit list)

These are real ADR requirements but cannot be discharged by code I can
write. Listing them so they appear on the operator's punch list:

1. DPO signature on the DPIA.
2. Legal review of the privacy notice (German + English).
3. External penetration test by a qualified firm.
4. CISO + Geschäftsleitung sign-off on the deployment.
5. 3-month production soak without incident.
6. GPU hardware procurement + actual model download.
7. Mistral DPA (if Mistral is enabled as fallback).
8. SIEM credentials and topology (Splunk / Elastic).
9. Firewall rules at the network perimeter — claude can write the
   intent (`egress_policy.json`), but the actual `iptables` / cloud
   security-group rules belong to the operator's deployment.

---

## 6. Recommended delivery order

1. **This GAP-ANALYSIS** — done, this document. Operator review next.
2. **Roadmap V2** — rewrite `IMPLEMENTATION-ROADMAP.md` against this gap analysis.
3. **L34 (DataFlowGuard)** — smallest structural addition, isolatable, testable.
4. **L35 (EU_PRODUCTION preset)** — builds on L34 + ADR-0007.
5. **L37 (Audit-at-rest)** — independent; can be parallel with L35.
6. **L36 (Erasure orchestrator)** — last, because it touches all
   other layers.
7. **DSB material + E2E suite** — written alongside the layers, not after.

Each step ships its own ADR, ref doc, tests, audit events,
must-NOT list, and CLAUDE.md section — matching the convention every
existing layer already follows.

---

## 7. The market argument

The structural difference between AtelierOS post-L37 and any other
agent platform on the market is this:

- **L19** (one-time disclosure card) — EU AI Act Art. 50 *structurally*
  enforced, not policy-only.
- **L16 Phase 4** (per-user consent gate) — GDPR Art. 6 / 7
  *structurally* enforced.
- **L16** (hash-chained audit) — GDPR Art. 30 / 32 *tamper-evident*.
- **L33 + L36** (artifact memory with Art. 17 erasure) — right-to-be-forgotten
  *cross-layer*, not best-effort.
- **L34** (data classification + flow guard) — sensitivity grading
  *enforced at every spawn*.
- **L35** (EU_PRODUCTION preset + egress lockdown) — *physically*
  unable to reach non-EU endpoints.
- **L37** (audit at rest + 7y retention) — *cryptographically* sealed
  long-term audit.

This is the basis for "the first agent actually approved in the EU."
None of the listed items is a marketing claim — each is a code path
with a test and an audit event.
