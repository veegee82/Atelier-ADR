# ADR-0056 — Living Compliance Manifest

**Status:** Accepted — M1–M6 implemented 2026-05-26  
**Date:** 2026-05-26  
**Authors:** Architectural design — Forge persona (Claude Code, maintainer session)  
**Type:** Governance Primitive (cross-cutting — no runtime layer number)  
**Depends on:** L10 (path-gate), L16 (audit chain), L20 (quota), L37 (audit-at-rest)  
**Related:** CLAUDE.md compliance baseline, `docs/claude-ref/layer-security.md`

---

## Context

AtelierOS is built against EU AI Act 2026 + GDPR as a **structural design
constraint** (see CLAUDE.md compliance baseline). This constraint is currently
enforced through:

- Hardcoded invariants in layer implementations (L10, L16, L19, L34, L36, …)
- Rules documented in CLAUDE.md and `docs/claude-ref/`
- ADRs recording the original design decisions

This approach has a **temporal blind spot**: regulations are living documents.

The EU AI Act has phased applicability (Articles 5 and 50 applied August 2024;
full GPAI obligations August 2025; remaining provisions August 2026). It is
amended by Delegated Acts. GDPR receives new guidance from the EDPB. National
supervisory authorities issue binding decisions that narrow interpretation.

When a regulation changes today, there is no systematic mechanism to answer:

1. **Which layers are affected** by the change?
2. **Which open PRs violate** the updated requirement?
3. **How can a compliance officer** contribute a rule update without touching
   Python or JavaScript?
4. **When was each rule last reviewed** against the current legal text?

The current answer is: manual review by the maintainer, on an ad-hoc basis.
This is insufficient for a product positioned on EU AI Act compliance as a
market differentiator (see `business_model.md`).

---

## Decision Drivers

1. **Regulatory velocity** — Laws change faster than release cycles. Compliance
   rules must be updatable without a code release.
2. **Separation of concerns** — "What the law requires" and "how we implement
   it" must be separately versionable, reviewable, and auditable.
3. **Non-coder contribution** — Lawyers and compliance officers must be able to
   propose rule changes via a structured YAML file without reading Python.
4. **CI enforcement** — Every PR that touches compliance-relevant code must be
   automatically checked against the current manifest.
5. **Manifest integrity** — The manifest itself is a potential attack surface.
   A compromised manifest must not weaken compliance guarantees.
6. **No false confidence** — Runtime LLM validation is explicitly out of scope.
   Compliance enforcement remains structurally in the layers; the manifest
   drives development-time and CI-time gates only.
7. **Auditability** — Manifest version, signature verification result, and CI
   check outcomes enter the L16 hash chain on `bridge.sh doctor`.

---

## Decision: Compliance Manifest + CI Gate

### What the manifest is NOT

The manifest is **not** a runtime oracle. It does not gate individual requests.
It does not replace L10, L16, L19, or any other structural enforcement layer.
An LLM reading the manifest at request-time and declaring "this request is
compliant" is **explicitly rejected** as a design pattern for two reasons:

- LLM output is probabilistic; EU AI Act compliance is binary.
- A manifest manipulated by an attacker could cause the LLM to approve
  non-compliant behavior, defeating all structural safeguards.

Compliance enforcement **remains deterministic and structural**. The manifest
is a **development-time and CI-time governance layer** only.

### Directory layout

```
compliance/
├── eu-ai-act.yaml          # EU AI Act 2026 rules
├── gdpr.yaml               # GDPR rules
├── manifest.sig            # Detached GPG signature (maintainer key)
├── manifest-version.txt    # Semantic version of the manifest bundle
└── CHANGELOG.md            # Which legal change triggered which manifest update
```

All files under `compliance/` are English (repo language rule). `CHANGELOG.md`
entries reference the official OJ (Official Journal of the EU) publication and
amendment date.

### Rule schema (`eu-ai-act.yaml`)

```yaml
version: "1.0.0"
regulation: "EU AI Act 2026"
oj_reference: "OJ L 2024/1689"
effective_date: "2026-08-02"
last_reviewed: "2026-05-26"

rules:
  - id: "eua.art50.disclosure"
    article: "Art. 50 — Transparency obligations for deployers of AI systems"
    paragraph: "1"
    summary: >
      Deployers of AI systems that interact directly with natural persons must
      ensure those persons are informed they are interacting with an AI system.
    severity: critical
    implemented_by:
      - layer: L19
        file: "operator/bridges/shared/disclosure.py"
        test: "tests/test_l19_disclosure.py"
    invariants:
      - "disclosure card fires exactly once per (channel, chat, uid)"
      - "disclosure cannot be disabled by operator config or env var"
      - "card length must be <= 1500 chars"
    forbidden_patterns:
      - "auto-admit without prior disclosure"
      - "trusted-observer bypass"
      - "compliance-off mode via any flag"
    last_verified: "2026-05-26"
    notes: >
      The /join command is the canonical trigger. Any new entry point that
      introduces a user to the system must call the same disclosure path.

  - id: "eua.art50.bot_nature"
    article: "Art. 50"
    paragraph: "2"
    summary: >
      AI-generated text published to the public must be machine-readable
      marked as AI-generated.
    severity: critical
    implemented_by:
      - layer: L19
        file: "operator/bridges/shared/disclosure.py"
        test: "tests/test_l19_disclosure.py"
    invariants:
      - "every outbound message carries provenance metadata in adapter envelope"
    forbidden_patterns:
      - "stripping provenance header from outbound message"
    last_verified: "2026-05-26"

  - id: "gdpr.art6.consent"
    article: "GDPR Art. 6 + Art. 7 — Lawfulness of processing"
    summary: >
      Processing of personal data requires a lawful basis. For AtelierOS,
      the chosen basis is explicit consent (Art. 6(1)(a)) per user session.
    severity: critical
    implemented_by:
      - layer: L16
        file: "operator/bridges/shared/consent.py"
        test: "tests/test_l16_consent.py"
    invariants:
      - "consent gate is deny-by-default"
      - "consent is per-uid, TTL-capped"
      - "no shortcut grants consent on another user's behalf"
    forbidden_patterns:
      - "auto-grant consent"
      - "owner grants consent for subordinate user"
    last_verified: "2026-05-26"
```

The schema is intentionally human-readable first. A compliance officer can
edit YAML; they cannot be expected to read Python test fixtures.

### Manifest versioning and pinning

The manifest bundle carries a semantic version in `manifest-version.txt`:
`MAJOR.MINOR.PATCH`

- `PATCH` — editorial fix, no behavioral change
- `MINOR` — new rule added or existing rule tightened (backward compatible)
- `MAJOR` — rule removed or relaxed (requires ADR and maintainer sign-off)

The **running system** pins a minimum manifest version in `tenant.atelier.yaml`:

```yaml
spec:
  compliance_manifest:
    min_version: "1.0.0"
    require_signature: true
    signer_fingerprint: "ABCD1234..."   # maintainer GPG key fingerprint
```

The server never pulls HEAD automatically. A manifest version upgrade is
an explicit operator action, audited on the L16 hash chain.

### Manifest integrity (GPG signing)

```bash
# Maintainer signs after every update
gpg --armor --detach-sign \
    --output compliance/manifest.sig \
    compliance/eu-ai-act.yaml compliance/gdpr.yaml

# Verification (CI + bridge.sh doctor)
gpg --verify compliance/manifest.sig \
    compliance/eu-ai-act.yaml compliance/gdpr.yaml
```

Verification failure is `CRITICAL` in `bridge.sh doctor` — same severity as
`voice-audit verify` exit-1. The system does not start with a tampered manifest.

### GitHub Action: `compliance-check.yml`

Triggers on every PR that touches any of:
- `operator/bridges/shared/`
- `operator/bridges/*/adapter.py`
- `operator/voice/hooks/`
- `compliance/`
- `CLAUDE.md`
- `docs/decisions/`

Steps:

1. **Verify manifest signature** (GPG) — fail-closed if invalid
2. **Detect affected layers** from changed file paths (deterministic mapping)
3. **Load relevant rules** for those layers from the manifest
4. **Invoke Haiku** with: rule block + PR diff + structured output schema
5. **Parse findings** — each finding has `rule_id`, `severity`, `rationale`
6. **Annotate PR** with inline comments for each finding
7. **Block merge** if any `severity: critical` finding exists

Haiku's role in step 4 is precisely bounded:

- It reasons about whether the diff **violates a stated invariant** or
  **matches a forbidden pattern** from the manifest.
- It does **not** make legal determinations.
- It does **not** replace human review — `severity: warning` findings require
  human sign-off but do not block merge.
- False positives (Haiku flags incorrectly) cost a human review. False
  negatives (Haiku misses something) are caught by the structural layers.
  This asymmetry makes LLM-assisted CI acceptable for review, where it would
  be unacceptable for runtime enforcement.

Structured output schema for Haiku:

```json
{
  "findings": [
    {
      "rule_id": "eua.art50.disclosure",
      "severity": "critical",
      "location": "operator/bridges/discord/adapter.py:L847",
      "rationale": "The new /guest entry point admits users without calling the disclosure path. Invariant 'disclosure fires exactly once per uid' is violated.",
      "suggestion": "Route /guest through disclosure.ensure_disclosed() before creating session."
    }
  ]
}
```

### `atelier-compliance-check` CLI

New command, integrated into `bridge.sh doctor`:

```bash
# Full check against all manifest rules
atelier-compliance-check

# Layer-scoped check
atelier-compliance-check --layer L19

# Against a specific manifest version (for regression testing)
atelier-compliance-check --manifest-version 0.9.0

# Output: JSON for CI pipeline consumption
atelier-compliance-check --json
```

Output example:

```
Compliance Manifest v1.0.0 (sig: VALID, signer: maintainer)
Regulation: EU AI Act 2026 · GDPR

  ✓ eua.art50.disclosure    L19  test: PASS  last-verified: 2026-05-26
  ✓ eua.art50.bot_nature    L19  test: PASS  last-verified: 2026-05-26
  ✓ gdpr.art6.consent       L16  test: PASS  last-verified: 2026-05-26
  ⚠ gdpr.art13.transparency L??  test: NOT FOUND — no test_ref registered
  ✓ gdpr.art17.erasure      L36  test: PASS  last-verified: 2026-05-26

Result: PASS with 1 warning
```

The `NOT FOUND` warning is intentional: a rule in the manifest with no
registered test is a gap, not a silent pass.

### Audit integration (L16)

`bridge.sh doctor` emits a `compliance.manifest_check` event to the L16 hash
chain:

```json
{
  "event": "compliance.manifest_check",
  "manifest_version": "1.0.0",
  "sig_valid": true,
  "rules_checked": 14,
  "rules_passed": 14,
  "rules_warned": 0,
  "rules_failed": 0
}
```

**Audit allow-list:** `manifest_version`, `sig_valid`, `rules_checked`,
`rules_passed`, `rules_warned`, `rules_failed`. Never: rule text, diff
content, Haiku output, file paths.

---

## Alternatives Considered

### A. Runtime compliance check per request (rejected)

Haiku validates every incoming request against the manifest at runtime.

**Rejected because:**
- LLM output is probabilistic; EU AI Act compliance requires certainty.
- Manifest manipulation by an attacker would weaken runtime decisions.
- 200–500ms per-request latency is unacceptable.
- No audit trail for individual LLM decisions.
- Creates false confidence: "Haiku approved it" ≠ legally compliant.

Structural layers (L10, L16, L19, L34) are the correct enforcement mechanism.
They are deterministic, zero-latency, and hardened. The manifest does not
replace them; it governs their evolution.

### B. Manifest as CLAUDE.md section (rejected)

Embed the compliance rules directly in CLAUDE.md.

**Rejected because:**
- CLAUDE.md is prose for Claude Code, not machine-readable for CI.
- No GPG integrity guarantee.
- No versioning independent of the repo.
- A compliance officer cannot contribute without understanding CLAUDE.md's
  role in the Claude Code toolchain.

CLAUDE.md retains its compliance baseline section as a human summary.
The manifest is the authoritative machine-readable source.

### C. External compliance service (rejected)

Subscribe to a third-party legal API that provides up-to-date EU AI Act rules.

**Rejected because:**
- Introduces external availability dependency for a security-critical check.
- Third-party service is outside the operator's control.
- Audit chain would reference an external source not preserved in the repo.
- Violates the principle that AtelierOS's compliance posture must be
  self-contained and verifiable without network access.

---

## Milestones

| Milestone | Scope | Status |
|---|---|---|
| **M1** | `compliance/` directory + `eu-ai-act.yaml` + `gdpr.yaml` scaffolding | Draft |
| **M2** | GPG signing workflow + `manifest-version.txt` | Draft |
| **M3** | `atelier-compliance-check` CLI + `bridge.sh doctor` integration | Draft |
| **M4** | `compliance-check.yml` GitHub Action (Haiku, PR annotation, block on critical) | Draft |
| **M5** | L16 audit event `compliance.manifest_check` | Draft |
| **M6** | `tenant.atelier.yaml::spec.compliance_manifest` version-pin + boot verification | Draft |

---

## Invariants (load-bearing)

**Must NOT do:**
- Use manifest rules as a runtime request-validation gate.
- Pull manifest from HEAD at runtime without version-pin and signature check.
- Accept `sig_valid: false` as a non-critical condition — it is always CRITICAL.
- Put Haiku's reasoning text in the L16 audit chain.
- Allow a compliance officer to weaken a `severity: critical` rule to `warning`
  without a corresponding ADR and maintainer GPG re-sign.
- Treat a rule with `test: NOT FOUND` as implicitly passing.
- Add rules that reference unimplemented layers (no `implemented_by` → warning,
  not pass).
- Remove the `forbidden_patterns` field from any rule without a MAJOR version bump.
- `import anthropic` from `compliance_check.py` (CI AST lint enforces).

---

## Open Questions

1. **Key rotation** — when the maintainer GPG key changes, what is the
   migration path for pinned `signer_fingerprint` values in all deployed
   tenants?
2. **Multi-jurisdiction** — as AtelierOS expands beyond EU, does the manifest
   grow `us-state-ai-laws.yaml`? Or is it always EU-scoped?
3. **Delegated Acts** — when the European Commission issues a Delegated Act
   that modifies an existing article, does that warrant a MAJOR or MINOR bump?
   Proposed: MINOR if additive, MAJOR if a previously required behavior is
   narrowed or removed.
4. **External verifier** — should the manifest be publishable at a well-known
   URL (e.g., `atelier.sh/.well-known/compliance-manifest`) so third parties
   can independently verify an operator's stated compliance posture?
