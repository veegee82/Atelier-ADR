# ADR-001: EU-AI-Act-conformant Open-Code Engine Design

**Status:** Superseded — substantive content implemented via ADR-0042 (L34
Data Classification), ADR-0043 (L35 Egress Lockdown + EU_PRODUCTION
presets), ADR-0044 (L37 Audit-at-rest Encryption + Retention), ADR-0045
(L36 GDPR Art. 17 Erasure Orchestrator). This file is retained as the
historical design intent plus a curated list of four items from the
original proposal that did **not** roll into 0042–0045 and remain open.

**Original status:** Proposed (Revised), 2026-05-20
**Superseded:** 2026-05-21
**Supersedes:** `0041-EU-Compliance-Atelier-OpenCode.md`
**Cross-references:** ADR-0007 (multi-tenant axis), ADR-0017 (control
plane / compliance reports), ADR-0042, ADR-0043, ADR-0044, ADR-0045
**Companion files:** Both companion files (`opencode_audit_architecture.md`,
`opencode_audit_weaknesses.md`) have been deleted; git history at commit
`e60582f7437` preserves them. The four genuinely open items are documented
below.

---

## Why this ADR is marked Superseded

The original ADR-001 (dated 2026-05-20) re-litigated, in a single
2 318-line drop, the compliance design that was already sharded into
ADRs 0042 / 0043 / 0044 / 0045 and shipped as load-bearing layers
(L16, L22, L34, L35, L36, L37) over the preceding weeks. A clean-room
re-read of ADR-001 against the live repository revealed three classes
of statement:

1. **Already implemented in AtelierOS via a different mechanism**
   (e.g. immutable audit log, GDPR right-to-erasure, human override,
   exception handling, third-party integration auditing).
2. **Not applicable to AtelierOS's actual domain** (deterministic
   floating-point reproducibility, demographic-parity gates,
   1 M decisions/day query performance — AtelierOS orchestrates LLM
   subprocesses, not Annex-III ML decisioning systems).
3. **Genuinely open** — see § "Remaining open items" below.

To avoid silent drift between an aspirational ADR and the shipped
architecture, the substantive proposal is closed out here and replaced
by an explicit mapping table plus the four open items.

---

## Mapping: ADR-001 v2.0 proposals → current AtelierOS layers

| ADR-001 proposal | AtelierOS layer / ADR | Disposition |
|---|---|---|
| **Tier 0** Deterministic Compute (Docker + CPU-pin + Fixed-Point arithmetic) | — | **Not applicable** to LLM-subprocess pipeline; partially relevant to L25 Compute Worker — see open item #3. |
| **Tier 1** Merkle Tree audit log with periodic root commitments | **L16** linear hash chain + **L37** audit-at-rest sealing (ADR-0044) | **Deliberately not adopted.** L16 is load-bearing (`CLAUDE.md`: *"Don't write to audit.jsonl with a different schema/chain"*). Switching to a Merkle tree would break `voice-audit verify`, the L37 sealer, and every per-layer emitter (L10, L18, L20, L24, L33, L34, L35, L36). External root commitments are tractable as an L37 extension without touching L16 — see open item #1. |
| HSM-protected HMAC keys | L37 `age`/`gpg` sealer (ADR-0044) | **Partially covered.** L37 protects rotated segments; HSM for the *live* L16 secret would be an additive enterprise-tier option — see open item #2. |
| **Tier 2** Pre-Decision Risk Check (demographic parity blocks decision) | L11 Dialectic decision-points + L34 DataFlowGuard (ADR-0042) | **Domain mismatch.** AtelierOS does not make hire/reject/credit decisions. L11 / L34 gate engine selection and data-flow paths, not personal fairness. |
| Human Override (Art. 14) `DecisionWithHumanOverride` pattern | L18 Roles + L21 Proposals + L11 Dialectic | **Covered, different pattern.** The original claim "Human Override not implemented" does not hold for AtelierOS. |
| Exception handling pipeline (timeouts, missing features, fallback decisions) | `adapter.py` HTTP transient-error reset, stream-idle watchdog, engine-trust hardening | **Covered.** The original claim "Exception Handling completely missing" does not hold for AtelierOS. |
| **Tier 3** "User can see own decision" query API | L28 Recall + L36 Erasure trail (ADR-0045) | **Partially covered** at the scope AtelierOS needs (per-chat recall, per-subject erasure trail). |
| Tier 3 auditor aggregate report | ADR-0017 Phase II `atelier-compliance-reports` + admin-console compliance routes | **Covered.** EU AI Act Art. 50 and GDPR Art. 30 reports + chain attestation are already shipped. |
| Anonymize-not-delete with content-hash recompute | L36 pseudonymous `subject_id` (regex-enforced) + L37 retention sweep (ADR-0045 + ADR-0044) | **Deliberately not adopted.** Recomputing `content_hash` after anonymization would break L16 chain integrity by construction. L36 resolves the same Art. 17 ↔ Art. 15 conflict via pseudonymity-from-inception, which preserves chain integrity. |
| 5–6 month implementation timeline | ADRs 0042–0045 already shipped | **Obsolete.** |
| Weakness #1 "Cryptographic chain too weak vs. insider DB tampering" | L16 + L37 + daily `voice-audit verify` timer | Partial. External timestamping would harden further — see open item #1. |
| Weakness #2 "Determinism not guaranteeable" | — | **Domain mismatch.** LLM output is non-deterministic by construction; reproducibility-as-bit-identity is not an AtelierOS goal. |
| Weakness #3 "Risk assessment too simplistic" | L11 Dialectic site `forge_creation` / `path_gate` | Different framing; not applicable to personal-fairness scope. |
| Weakness #4 "GDPR Right-to-Erasure vs. audit retention is unsolvable" | L36 + L37 (ADR-0045 + ADR-0044) | **Resolved for AtelierOS.** The claim of unsolvability does not hold given pseudonymous `subject_id` + retention sweep. |
| Weakness #5 "Human Override not designed" | L18 + L21 + L11 | **Resolved for AtelierOS.** |
| Weakness #6 "Exception Handling completely missing" | adapter HTTP reset + stream-idle + L22 WorkerEngine | **Resolved for AtelierOS.** |
| Weakness #7 "Query performance does not scale to 1 M decisions/day" | L28 FTS5 SQLite (sized for orchestration workload, not Annex-III decisioning) | **Not applicable** to current scope. |
| Weakness #8 "Model versioning unclear" | L22 `engine_id` + `core/delegate` subprocess identification | Different framing; partially covered. |
| Weakness #9 "Third-party integration not addressed" | L29 Delegation + L34 + L35 Egress Gate (ADR-0043) | **Covered.** |
| Weakness #10 "Testing realism" | `operator/bridges/run-all-tests.sh` + per-layer subprocess E2E (CLAUDE.md §LDD) | Covered through different methodology. |
| Weakness #11 "Why-different-from-internal-model-behavior" | L28 user-model context + L11 dialectic justifications | Different scope. |

---

## Remaining open items (genuinely not addressed by 0042–0045)

These are the four points from ADR-001 that survive scrutiny and warrant
follow-up. Each is small, additive, and does **not** require touching
L16 / L34 / L35 / L36 / L37 invariants.

### Open item #1 — External timestamping / root commitments for L37

**What.** L16 + L37 provide *internal* tamper-evidence (hash chain,
sealed rotated segments, daily verify). They do not provide *external*
attestation that a given audit segment existed at a given time.

**Why it matters.** A regulator or external auditor cannot, today,
independently prove that a sealed segment from 2026-Q1 was not
fabricated retroactively in 2026-Q4.

**Where it fits.** L37 extension. The sealer already emits
`audit.segment_sealed` events; the same hook can publish the segment
hash to an RFC 3161 Time-Stamping Authority (TSA) or OpenTimestamps.

**Anti-scope.** Does **not** require Merkle-tree migration. Does **not**
require blockchain. A single TSA call per sealed segment is sufficient.

### Open item #2 — HSM backend option for the live L16 HMAC secret

**What.** Today the per-tenant audit secret lives on the filesystem.
For enterprise deployments under strict insider-threat models, the
secret should be reachable via PKCS#11 or AWS KMS / GCP KMS without
ever materializing in process memory.

**Where it fits.** `atelier-enterprise` plugin. Add a
`spec.audit.signing_key` block to the tenant schema with URIs like
`pkcs11://...` or `awskms://...`; the L16 emitter resolves the key
through the configured backend.

**Anti-scope.** Apache-core L16 keeps its current filesystem-secret
path as the default. This is enterprise-tier opt-in.

### Open item #3 — Container-image-hash provenance for L25 Compute Worker

**What.** The Tier 0 idea (Docker image hash as cryptographic proof of
the code that produced an output) is not applicable to LLM subprocesses
but **is** applicable to L25 `compute_run` strategies (grid, random,
bayesian). A regulated tenant running L25 should be able to prove
"this hyperparameter sweep was executed by image `sha256:abc…`".

**Where it fits.** L25 worker hardening. The compute worker already
takes a strategy + bounded resource budget; adding image-hash capture
to its audit event is mechanical.

**Anti-scope.** Fixed-point arithmetic and CPU-pinning from ADR-001
Tier 0 are deferred — they only matter if a tenant claims bit-identical
reproducibility of compute output, which AtelierOS does not today.

### Open item #4 — Fairness / bias monitoring (parked)

**What.** ADR-001 Tier 2 demographic-parity gates and subgroup-analysis
dashboards are not applicable to AtelierOS in its current scope
(coding-agent orchestration).

**When it un-parks.** When *any* of the following occur:

1. AtelierOS ships an Annex-III workflow variant of its own; **or**
2. A tenant deploys AtelierOS as a component of an Annex-III system
   (e.g. automated CV screening, credit-scoring assistant, or any
   workflow meeting EU AI Act Art. 6 + Annex III criteria). EU AI Act
   Art. 25–28 cascade the provider's obligations to the deployer —
   AtelierOS becomes a high-risk AI component the moment a tenant uses
   it that way, regardless of AtelierOS's own scope.

Until either condition is met: documented as out-of-scope, not as a gap.
Operators running regulated workflows must disclose this to the
AtelierOS maintainer so the appropriate layer can be activated.

---

## What is deliberately not adopted (and why)

These items from ADR-001 are explicit non-goals for AtelierOS, recorded
here so future readers do not re-propose them without engaging the
trade-off:

- **Merkle-tree audit as L16 replacement.** Linear hash chain + L37
  sealing was chosen for tooling simplicity (every emitter is one append
  + one HMAC) and to keep `voice-audit verify` a single-pass linear
  scan. The marginal security benefit of a Merkle tree at AtelierOS's
  event volumes does not justify breaking the per-layer emitter
  contract. See ADR-0044 for the L37 lifecycle that closes the same
  attack surface differently.
- **Hash-recompute-on-anonymization.** Would break L16 chain integrity.
  L36 (ADR-0045) achieves Art. 17 compliance through pseudonymous
  `subject_id` enforced at the regex level, which never requires
  rewriting historical events.
- **Tier-2 personal-fairness gates.** Out of scope for an orchestration
  platform; see open item #4 for the re-entry condition.
- **5–6 month parallel implementation track.** ADRs 0042–0045 are
  shipped; the work already happened.

---

## Historical reference

The original 950-line ADR-001 v2.0 text is preserved in the git
history at commit `e60582f7437` (`docs: Add ADR-001 Open Code EU AI
Act Compliance Architecture`, 2026-05-20). Both companion files
(`opencode_audit_architecture.md`, `opencode_audit_weaknesses.md`)
have been deleted from the working tree (2026-05-21); git history at
the same commit preserves them.

---

## Open follow-ups for this ADR

- [x] Companion files deleted from working tree (2026-05-21).
- [x] Open item #1 (RFC 3161 TSA) implemented in L37 `audit_sealer.py`
      (2026-05-21) — opt-in via `spec.audit.encryption_at_rest.tsa_enabled`.
- [ ] Open issue / ADR draft for open item #2 (HSM-backed L16 secret
      in `atelier-enterprise`).
- [ ] Open issue / ADR draft for open item #3 (image-hash provenance
      for L25).
- [ ] Decide whether to also mark `0041-EU-Compliance-Atelier-OpenCode.md`
      as Superseded with a pointer here (currently both files exist
      with overlapping scope).
