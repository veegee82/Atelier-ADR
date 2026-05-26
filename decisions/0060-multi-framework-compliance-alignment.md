# ADR-0060 — Multi-Framework Compliance Alignment: ISO 42001 + NIST AI RMF

**Status:** Accepted — M1–M6 implemented 2026-05-26  
**Date:** 2026-05-26  
**Authors:** Architectural design — Forge persona (Claude Code, maintainer session)  
**Type:** Governance Extension (no new runtime layer — extends ADR-0056 + ADR-0057)  
**Depends on:** ADR-0056 (compliance manifest), ADR-0057 (certification readiness),
ADR-0042 (L34 data classification), ADR-0043 (L35 egress), ADR-0045 (L36 erasure),
ADR-0044 (L37 audit-at-rest), L16 (audit chain), L19 (disclosure)  
**Related:** `docs/decisions/0057-eu-ai-act-certification-readiness.md`,
`docs/decisions/0056-compliance-manifest.md`, `business_model.md`,
`docs/decisions/ADR-0042-production-ready-roadmap.md`

---

## Context

A competitive analysis conducted May 2026 compared AtelierOS's compliance posture
against the leading AI governance platforms. The runtime enforcement layer
(L10, L16, L19, L34, L35, L36, L37, L38, L39) is structurally ahead of all
identified competitors: no comparable platform has disclosure, consent,
hash-chain audit, data-flow guards, erasure, and incident tracking
deterministically enforced in a single OS.

The analysis identified one commercial gap that the existing ADRs do not close:

> **ISO 42001 and NIST AI RMF are not explicitly mapped to AtelierOS layers,
> and no machine-readable evidence package can be generated for these frameworks.**

This matters for three concrete reasons:

1. **Enterprise procurement gates.** Regulated buyers (financial services,
   healthcare, public sector) increasingly require ISO 42001 alignment as a
   procurement precondition. Credo AI (Fast Company #6 in Applied AI, 2026)
   and IBM watsonx.governance both advertise pre-configured ISO 42001 and
   NIST AI RMF packs. AtelierOS cannot respond to an RFP that asks for either.

2. **The implementation already satisfies both standards.** ISO 42001 Clause 8
   (Operations), Clause 9 (Performance Evaluation), and Clause 10 (Improvement)
   are directly addressed by the runtime layers. NIST AI RMF's GOVERN / MAP /
   MEASURE / MANAGE functions are mapped structurally through L16, L34, L35,
   L36, and the Operator Declaration Gate (ADR-0057 Component 3). The gap is
   not in what the system does — it is in how that is evidenced and communicated.

3. **Framework convergence is the enterprise entry point.** EU AI Act, GDPR,
   ISO 42001, and NIST AI RMF overlap substantially. A single cross-reference
   table that shows "this AtelierOS layer satisfies requirements from all four
   frameworks simultaneously" is the narrative that closes enterprise deals
   and differentiates from compliance tools that address frameworks in silos.

**What ADR-0056 and ADR-0057 already cover:**

ADR-0056 defines the living compliance manifest for EU AI Act + GDPR.
ADR-0057 closes the four remaining EU AI Act certification gaps (content
marking, incident tracking, operator declaration gate, Annex IV generator)
and defines the certification package CLI. These two ADRs are complete and
not reopened here.

**Scope of this ADR:**

Extend the compliance manifest and the certification package CLI to cover
ISO 42001 and NIST AI RMF, without changing any runtime layer, without
introducing new runtime dependencies, and without altering the L16 audit chain.

---

## Competitive Context

| Platform | EU AI Act | GDPR | ISO 42001 | NIST AI RMF | Runtime-enforced |
|---|---|---|---|---|---|
| **AtelierOS (before this ADR)** | ✅ structural | ✅ structural | ❌ unmapped | ❌ unmapped | ✅ best-in-class |
| Credo AI | ✅ policy pack | partial | ✅ policy pack | ✅ policy pack | ❌ overlay only |
| IBM watsonx.governance | ✅ pre-configured | ✅ | ✅ | ✅ | ❌ overlay only |
| Holistic AI | ✅ deep | partial | partial | partial | ❌ overlay only |
| Prediction Guard | partial | partial | ❌ | ❌ | ✅ API gateway |

**After this ADR:** AtelierOS is the only platform with both structural
runtime enforcement AND explicit multi-framework evidence generation.

---

## Decision Drivers

1. **Enterprise RFP response.** Without ISO 42001 and NIST AI RMF mapping,
   AtelierOS is invisible in regulated-industry procurement processes.
2. **No new runtime work.** The layers already satisfy the requirements.
   The only work needed is mapping, manifest YAML, and CLI extensions.
3. **Converged evidence.** A single certification package that satisfies
   EU AI Act + GDPR + ISO 42001 + NIST AI RMF reduces operator burden
   and is a concrete product differentiator.
4. **Complement, not replace, ADR-0056.** The manifest architecture from
   ADR-0056 is the right abstraction. This ADR adds two new framework files
   to the same `compliance/` directory using the same schema.
5. **Certification timeline.** ISO 42001 certification by an accredited body
   has a 12–24 month lead time. The mapping work must precede the engagement,
   not follow it.

---

## The Mapping Thesis

Both ISO 42001 and NIST AI RMF were designed to be framework-agnostic —
they specify *what* must be governed, not *how*. AtelierOS's layer model
provides the *how* through deterministic structural controls. The mapping
is therefore not approximate ("we do something like this") but precise
("Layer X is the implementation of Clause Y, enforced deterministically,
with test reference Z").

This precision is the commercial argument: AtelierOS is not a compliance
tool that helps operators track whether their AI systems are compliant.
It is an AI runtime that is structurally compliant, with the evidence
generated as a by-product of normal operation.

---

## Decision

### Component 1 — ISO 42001 Manifest (`compliance/iso-42001.yaml`)

ISO/IEC 42001:2023 is the international standard for AI Management Systems
(AIMS). Its clause structure mirrors ISO 9001 and 27001, making it familiar
to enterprise risk and compliance teams.

AtelierOS satisfies the operational clauses through existing layers. The
planning and support clauses require operator-side evidence (DPIA, roles,
training records) that the Operator Declaration Gate (ADR-0057 Component 3)
already gates on.

#### Clause Mapping

| ISO 42001 Clause | Requirement Summary | AtelierOS Implementation | Layer / ADR |
|---|---|---|---|
| 4.1 Understand the organisation | Internal/external context of AI use | `tenant.atelier.yaml` scope model | ADR-0007 |
| 4.2 Understand interested parties | Stakeholder requirements and constraints | Whitelist, roles, consent gate | L16/L18/L19 |
| 4.3 Determine the scope | AIMS boundary definition | `spec.operator_declaration.permitted_use` | ADR-0057 §C3 |
| 5.1 Leadership commitment | Management responsibility for AI governance | Operator Declaration Gate (boot CRITICAL) | ADR-0057 §C3 |
| 5.2 AI policy | Documented AI use policy | `docs/compliance/OPERATOR-OBLIGATIONS.md` | ADR-0057 |
| 6.1 Risk and opportunity | Risk assessment and treatment | L34 data classification matrix | ADR-0042 |
| 6.2 Objectives | Measurable AI management objectives | Compliance manifest rules + test pass/fail | ADR-0056 |
| 7.1 Resources | Adequate resources for AIMS | N/A (operator obligation) | — |
| 7.2 Competence | Personnel competence | N/A (operator obligation) | — |
| 7.3 Awareness | Staff awareness of AI risks | Disclosure card (L19), consent gate | L19/L16 |
| 7.4 Communication | Internal/external AI communication | Bot-disclosure, `/join` flow | L19 |
| 7.5 Documented information | Document control and records | Audit chain, manifest, ADRs | L16, L37, ADR-0056 |
| 8.1 Operational planning | Plan and control AI processes | Engine-policy allowlist, data-flow guard | ADR-0007/L34/L35 |
| 8.2 AI risk assessment | Documented assessment process | `compliance/eu-ai-act.yaml` severity table | ADR-0056 |
| 8.3 AI risk treatment | Controls for identified risks | L10, L16, L34, L35, L36, L37, L38 | All layer ADRs |
| 8.4 AI system impact assessment | DPIA-equivalent for AI systems | DPIA template, operator declaration | ADR-0057 §C3 |
| 9.1 Monitoring + measurement | Performance evaluation | L16 audit chain, incident tracker (L39) | L16, ADR-0057 §C2 |
| 9.2 Internal audit | Systematic internal audits | `atelier-compliance-check` CLI | ADR-0056 |
| 9.3 Management review | Periodic review of AIMS | `atelier-annex-iv export-package` output | ADR-0057 §C6 |
| 10.1 Nonconformity | Corrective action process | `atelier-incident` CLI, Art. 73 workflow | ADR-0057 §C2 |
| 10.2 Continual improvement | Ongoing improvement of AIMS | LDD outer loop, ADR versioning | CLAUDE.md |

**Operator-only clauses** (7.1, 7.2) are flagged in the manifest as
`implemented_by: operator` and generate `[OPERATOR: FILL IN]` placeholders
in the Statement of Applicability. They do not contribute to the
`atelier-compliance-check` pass/fail result.

#### YAML Schema Extension

The `iso-42001.yaml` manifest follows the same schema as `eu-ai-act.yaml`
(ADR-0056) with one additional field: `iso_clause`.

```yaml
version: "1.0.0"
regulation: "ISO/IEC 42001:2023"
standard_body: "ISO"
effective_date: "2023-12-18"
last_reviewed: "2026-05-26"

rules:
  - id: "iso42001.8.3.audit-chain"
    iso_clause: "8.3"
    article: "AI risk treatment — tamper-evident records"
    summary: >
      Controls implemented to treat identified AI risks must be
      documented and verifiable. The audit chain is the primary
      AtelierOS control record.
    severity: critical
    implemented_by:
      - layer: L16
        file: "operator/bridges/shared/audit.py"
        test: "tests/test_l16_audit_chain.py"
    invariants:
      - "audit chain is hash-linked — no entry can be deleted without breaking verification"
      - "voice-audit verify exits 0 on an intact chain"
      - "daily verify timer notifies bridge on chain break"
    forbidden_patterns:
      - "skipping hash-chain link for any event type"
      - "silence audit-verify exit-1"
    last_verified: "2026-05-26"
```

#### Statement of Applicability (SoA)

ISO 42001 certification requires a Statement of Applicability — a document
listing all clauses, whether they apply, how they are implemented, and
why any are excluded. The SoA is generated by:

```bash
atelier-annex-iv generate --framework iso-42001 \
    --output docs/compliance/ISO-42001-SoA.md
```

The generator produces a table with all 22 operational subclauses. Clauses
marked `implemented_by: operator` are included with `[OPERATOR: FILL IN]`
markers. The SoA is not a legally binding document until signed by the
operator's DPO or ISO 42001 Lead Implementer.

---

### Component 2 — NIST AI RMF Manifest (`compliance/nist-ai-rmf.yaml`)

The NIST AI Risk Management Framework (AI RMF 1.0, January 2023) organises
AI risk management into four functions: GOVERN, MAP, MEASURE, MANAGE.
It is non-prescriptive and widely adopted in US-regulated sectors (finance,
healthcare) and by European enterprises with US parent companies.

#### Function Mapping

**GOVERN — Policies, processes, procedures, and practices**

| GOVERN Subcategory | AtelierOS Implementation | Layer / ADR |
|---|---|---|
| GV-1.1 Policies for AI risk | Compliance manifest rules + forbidden patterns | ADR-0056 |
| GV-1.2 Accountability structures | Whitelist, role hierarchy, operator declaration | L18, ADR-0057 §C3 |
| GV-1.3 Organisational risk tolerance | Data classification matrix (PUBLIC→SECRET) | ADR-0042 |
| GV-2.1 Risk awareness and training | Disclosure card, consent gate | L19, L16 |
| GV-3.1 Feedback mechanisms | Incident tracker, `atelier-incident scan` | ADR-0057 §C2 |
| GV-4.1 Regulatory requirements | EU AI Act + GDPR manifest, boot checks | ADR-0056, ADR-0057 |
| GV-6.1 Policies for third-party AI | Engine-policy allowlist, egress lockdown | ADR-0007, ADR-0043 |

**MAP — Categorise AI risk context**

| MAP Subcategory | AtelierOS Implementation | Layer / ADR |
|---|---|---|
| MP-1.1 Intended use defined | `spec.operator_declaration.permitted_use` | ADR-0057 §C3 |
| MP-1.5 Organisational risk tolerance | L34 classification × engine matrix | ADR-0042 |
| MP-2.1 Scientific grounding | Risk classification statement (Limited Risk) | ADR-0057 §Risk |
| MP-2.3 AI system impacts | DPIA template, Annex IV §7 (human oversight) | ADR-0057 §C4 |
| MP-3.1 Risks of AI system | `compliance/eu-ai-act.yaml` severity table | ADR-0056 |
| MP-5.1 Likelihood and impact | Incident categories (6 structural triggers) | ADR-0057 §C2 |

**MEASURE — Analyse and assess AI risk**

| MEASURE Subcategory | AtelierOS Implementation | Layer / ADR |
|---|---|---|
| MS-1.1 Risk metrics | `rules_passed`/`rules_failed` in compliance check | ADR-0056 |
| MS-2.1 AI risk tracking | Hash-chain audit log, daily verify timer | L16, L37 |
| MS-2.2 Human oversight | L18 roles, L21 proposals, LDD-Toggle | L18/L21/L14 |
| MS-2.5 Testing and evaluation | `run-all-tests.sh`, E2E per-subtask gate | CLAUDE.md |
| MS-2.7 Effectiveness of controls | Boot self-test, `bridge.sh doctor` | self_test.py |
| MS-4.1 Deployment monitoring | Agent monitoring via L16 audit chain | L16 |

**MANAGE — Prioritise and address AI risk**

| MANAGE Subcategory | AtelierOS Implementation | Layer / ADR |
|---|---|---|
| MG-1.1 Risk treatment plans | L10 path-gate, L34 data flow, L35 egress | L10/L34/L35 |
| MG-2.2 Incident response | `atelier-incident` CLI, 15-day Art. 73 path | ADR-0057 §C2 |
| MG-2.4 Residual risk | L36 erasure, L37 retention, data residency | ADR-0045/ADR-0044 |
| MG-3.1 Corrective actions | `incident.status_changed` → `notified` flow | ADR-0057 §C2 |
| MG-4.1 Risk communication | Certification package, SoA, Annex IV | ADR-0057 §C6 |

#### YAML Schema

```yaml
version: "1.0.0"
regulation: "NIST AI RMF 1.0"
standard_body: "NIST"
publication_date: "2023-01-26"
last_reviewed: "2026-05-26"

rules:
  - id: "nist.govern.gv-4.1"
    nist_function: "GOVERN"
    nist_category: "GV-4.1"
    summary: >
      Organisational teams are committed to regulatory compliance
      (EU AI Act, GDPR, sector-specific requirements).
    severity: critical
    implemented_by:
      - layer: L16
        file: "operator/bridges/shared/audit.py"
        test: "tests/test_l16_audit_chain.py"
      - layer: L19
        file: "operator/bridges/shared/disclosure.py"
        test: "tests/test_l19_disclosure.py"
    cross_references:
      - framework: "EU AI Act"
        rule_id: "eua.art50.disclosure"
      - framework: "ISO 42001"
        rule_id: "iso42001.7.4.communication"
    invariants:
      - "compliance manifest signature verified at every boot"
      - "disclosure fires before any session processes a user message"
    forbidden_patterns:
      - "compliance-off mode via any env var or flag"
    last_verified: "2026-05-26"
```

The `cross_references` field is new to the NIST manifest schema. It enables
the cross-framework evidence table (Component 3).

---

### Component 3 — Cross-Framework Evidence Table

The most commercially differentiated output of this ADR is a machine-generated
table that shows, for each AtelierOS layer, which requirements from all four
frameworks it satisfies simultaneously.

This table is generated by:

```bash
atelier-annex-iv cross-reference \
    --frameworks eu-ai-act,gdpr,iso-42001,nist-ai-rmf \
    --output docs/compliance/CROSS-FRAMEWORK-MAP.md
```

Example output format:

```markdown
## AtelierOS Multi-Framework Compliance Map

| Layer | EU AI Act | GDPR | ISO 42001 | NIST AI RMF |
|---|---|---|---|---|
| L10 Path-Gate | — | Art. 32 (security) | 8.3 (risk treatment) | MG-1.1 |
| L16 Audit Chain | Art. 14 | Art. 30, 32 | 7.5, 9.1 | MS-2.1, MS-4.1 |
| L16 Consent Gate | — | Art. 6, 7 | 4.2, 7.3 | GV-2.1 |
| L19 Disclosure | Art. 50 §1 | — | 7.4 | GV-2.1 |
| L34 Data Classification | Art. 14 | Art. 5, 25 | 6.1, 8.1 | MP-1.5, MG-1.1 |
| L35 Egress Lockdown | Art. 14 | Art. 32 | 8.3 | GV-6.1, MG-1.1 |
| L36 Erasure | — | Art. 17 | 8.4, 10.1 | MG-2.4 |
| L37 Audit-at-Rest | — | Art. 32 | 7.5 | MS-2.1 |
| L38 A2A Attestation | — | Art. 32 | 8.1, 8.3 | MG-1.1 |
| L39 Incident Tracker | Art. 73 | Art. 33 | 10.1 | MG-2.2, MG-3.1 |
| Operator Declaration Gate | Art. 28–30 | Art. 28 | 5.1, 4.3 | GV-1.2, MP-1.1 |
```

This table is the enterprise sales artefact. A compliance officer receiving
the certification package can hand it to their procurement team as evidence
that AtelierOS is not a point solution for one regulation, but a structural
control layer across all major AI governance frameworks.

---

### Component 4 — CLI Extensions

#### `atelier-annex-iv generate --framework <name>`

```bash
# Statement of Applicability for ISO 42001
atelier-annex-iv generate --framework iso-42001 \
    --output docs/compliance/ISO-42001-SoA.md

# NIST AI RMF Profile (organisation profile format)
atelier-annex-iv generate --framework nist-ai-rmf \
    --output docs/compliance/NIST-AI-RMF-Profile.md

# Cross-framework evidence table
atelier-annex-iv cross-reference \
    --frameworks eu-ai-act,gdpr,iso-42001,nist-ai-rmf \
    --output docs/compliance/CROSS-FRAMEWORK-MAP.md
```

#### `atelier-compliance-check --framework <name>`

```bash
# Check only ISO 42001 rules
atelier-compliance-check --framework iso-42001

# Check all frameworks
atelier-compliance-check --all-frameworks
```

#### Certification Package Extension

The `atelier-annex-iv export-package` command (ADR-0057 Component 6) is
extended to include multi-framework artifacts:

```
atelier-cert-package-20260801/
├── README.md
├── ANNEX-IV.md                            # EU AI Act Annex IV (ADR-0057)
├── RISK-CLASSIFICATION.md                 # EU AI Act risk statement
├── compliance/
│   ├── eu-ai-act.yaml                     # EU AI Act manifest (signed)
│   ├── gdpr.yaml                          # GDPR manifest (signed)
│   ├── iso-42001.yaml                     # ISO 42001 manifest (signed)  ← new
│   ├── nist-ai-rmf.yaml                   # NIST AI RMF manifest (signed)  ← new
│   └── manifest.sig
├── iso-42001/                             ← new
│   └── ISO-42001-SoA.md                   # Statement of Applicability
├── nist-ai-rmf/                           ← new
│   └── NIST-AI-RMF-Profile.md             # Organisation Profile
├── CROSS-FRAMEWORK-MAP.md                 ← new
├── audit/
│   ├── compliance-check-report.json       # All frameworks
│   └── incidents-export.json
├── tests/
│   └── test-summary.txt
└── declarations/
    └── operator-declaration.yaml
```

The full package is GPG-signed as a whole (ADR-0057 Component 6 invariant).

---

### Component 5 — Compliance Check CI Integration

The GitHub Action `compliance-check.yml` (ADR-0056 M4) is extended to
check the new manifests on every PR touching ISO 42001 or NIST AI RMF
mapped files.

Haiku's structured output schema gains two new optional fields:

```json
{
  "findings": [
    {
      "rule_id": "iso42001.8.3.audit-chain",
      "severity": "critical",
      "framework": "ISO 42001",
      "location": "operator/bridges/shared/audit.py:L214",
      "rationale": "...",
      "suggestion": "..."
    }
  ]
}
```

The `framework` field enables per-framework CI reports when running in
`--json` mode.

---

## What This ADR Does NOT Address

### Provider Agnosticism

AtelierOS currently depends on Claude/Anthropic as its primary engine.
IBM watsonx.governance and Bifrost support 50+ providers. This is a real
commercial gap, particularly for EU-sovereign deployments that require
Mistral, Llama 4, or on-premise models.

**This is a separate ADR (proposed: ADR-0061).** Provider agnosticism
affects the L22 WorkerEngine protocol, L34 data classification matrix
(engine locality), and L35 egress gates. It is not a documentation gap —
it requires runtime changes and a new engine adapter framework. Conflating
it with the documentation-only work of this ADR would delay both.

### EU-only Hosting / Sovereign Mode

AtelierOS uses Anthropic's API (US-based). For customers in regulated sectors
requiring data not to leave EU jurisdiction (DORA, NIS2, healthcare), this
is a structural constraint. The solution is EU-hosted open-weight models
(Mistral, Llama 4) combined with the L35 egress lockdown preset.

**This is resolved by ADR-0061** (provider agnosticism + EU-sovereign model
support) rather than this ADR. The `tenant.atelier.eu-production-ollama.yaml`
preset in L35 already anticipates this deployment mode; it requires the
engine adapter work to complete it.

---

## Alternatives Considered

### A. Certify against ISO 42001 via an external compliance overlay tool (rejected)

Use Credo AI or Modulos as the ISO 42001 evidence layer, positioned as a
complement to AtelierOS's runtime enforcement.

**Rejected because:**
- Introduces a commercial dependency on a direct competitor.
- The evidence lives in a third-party system, not in AtelierOS's audit chain.
- Operators must manage two separate systems and reconcile them at audit time.
- AtelierOS already has all the underlying data; generating the evidence
  internally is a direct extension of ADR-0056's manifest architecture.

### B. Manually authored static ISO 42001 SoA document (rejected)

Write a one-time static `docs/compliance/ISO-42001-SoA.md` and maintain it
by hand.

**Rejected because:**
- A static document becomes stale the moment any mapped layer changes.
- Without a generator, the SoA cannot be validated as consistent with the
  running system.
- A static document does not integrate with `atelier-compliance-check` or CI.
- ADR-0057's Annex IV generator established the pattern: compliance documents
  are generated from authoritative sources, not authored from scratch.

### C. Map all frameworks into a single unified manifest file (rejected)

Instead of separate `iso-42001.yaml` and `nist-ai-rmf.yaml`, use one
`unified.yaml` with per-rule framework annotations.

**Rejected because:**
- Regulatory frameworks evolve at different rates. The EU AI Act is amended
  by Delegated Acts; ISO 42001 by TC 42 working groups; NIST AI RMF by NIST
  publications. Separate files mean independent versioning and signing.
- A compliance officer contributing an ISO 42001 rule update should not touch
  a file that also contains EU AI Act rules — the separation of concerns
  prevents accidental rule weakening across frameworks.
- GPG signing in ADR-0056 signs the files individually. A unified file would
  require signing a file where one framework's change invalidates the signature
  for all frameworks.

---

## Milestones

| Milestone | Scope | Status | Depends on |
|---|---|---|---|
| **M1** | `compliance/iso-42001.yaml` — full clause mapping, all 22 subclauses, YAML schema, GPG-signed | **Done** 2026-05-26 | ADR-0056 M1+M2 |
| **M2** | `compliance/nist-ai-rmf.yaml` — full function mapping (GOVERN/MAP/MEASURE/MANAGE), `cross_references` field, GPG-signed | **Done** 2026-05-26 | ADR-0056 M1+M2 |
| **M3** | `atelier-annex-iv generate --framework iso-42001` → ISO 42001 SoA | **Done** 2026-05-26 | M1, ADR-0057 M8 |
| **M4** | `atelier-annex-iv generate --framework nist-ai-rmf` → NIST AI RMF Profile | **Done** 2026-05-26 | M2, ADR-0057 M8 |
| **M5** | `atelier-annex-iv cross-reference` → multi-framework evidence table | **Done** 2026-05-26 | M1+M2 |
| **M6** | `atelier-compliance-check --framework iso-42001` + `--all-frameworks` CLI flag | **Done** 2026-05-26 | M1+M2, ADR-0056 M3 |
| **M7** | GitHub Action `compliance-check.yml` extended for ISO 42001 + NIST AI RMF files | Draft — depends on ADR-0056 M4 | M6, ADR-0056 M4 |
| **M8** | `atelier-annex-iv export-package` extended: `iso-42001/`, `nist-ai-rmf/`, `CROSS-FRAMEWORK-MAP.md` | **Done** 2026-05-26 | M3+M4+M5 |

**Dependency with ADR-0057:** M1 and M2 can start in parallel with ADR-0057's
M1+M2 (compliance scaffold). M3–M8 require ADR-0057 M8 (Annex IV generator
base) to be complete.

**Delivery target:** M1–M6 before first ISO 42001 certification engagement
(Q3 2026); M7–M8 before first enterprise RFP response requiring multi-framework
evidence (Q3–Q4 2026).

---

## Invariants (load-bearing)

**Must NOT do:**
- Use `iso-42001.yaml` or `nist-ai-rmf.yaml` as runtime request-validation gates.
  These manifests are development-time and CI-time governance layers only,
  exactly as defined in ADR-0056.
- Mark an ISO 42001 operator-only clause (7.1 Resources, 7.2 Competence) as
  `implemented_by: atelier` — these are operator obligations, not platform obligations.
- Weaken a `severity: critical` rule in any manifest without a MAJOR version bump,
  a companion ADR, and maintainer GPG re-sign.
- Put cross-framework mapping logic in runtime code (`import anthropic` or
  any runtime import from the mapping module — CI AST lint enforces).
- Ship a Statement of Applicability with remaining `[OPERATOR: FILL IN]`
  markers — `atelier-annex-iv validate` must exit 0 before commit.
- Treat the NIST AI RMF Profile as a certification document — NIST AI RMF
  is a voluntary framework; there is no NIST certification body. It is an
  evidence artefact, not a compliance claim.
- Assign a `last_verified` date to a rule without updating the corresponding
  layer test reference — stale verification dates are worse than absent ones.
- Merge `iso-42001.yaml` and `nist-ai-rmf.yaml` into a single file for
  convenience — framework separation is a deliberate design decision
  (see Alternatives §C).
- Add ISO 42001 or NIST AI RMF rules for layers that do not exist yet.
  The manifest must reflect implemented and tested reality, not aspirations.

---

## Open Questions

1. **ISO 42001 Annex A controls.** ISO 42001 Annex A lists 38 optional controls.
   Should the SoA generator also produce an Annex A applicability assessment?
   Proposed: yes, as a separate `--section annex-a` flag. Deferred to M3 scope
   clarification.

2. **NIST AI RMF Playbooks.** NIST has published AI RMF Playbooks for specific
   sectors (financial services, healthcare). Should `nist-ai-rmf.yaml` include
   playbook subcategories, or stay at the core function level?
   Proposed: core functions only in v1.0; playbook extensions in v1.1 with a
   `--profile financial-services` flag on the generator.

3. **Certification engagement timing.** ISO 42001 certification requires a
   Lead Implementer and a certification body (e.g., TÜV SÜD, BSI). Should
   AtelierOS pursue certification of the platform itself, or provide evidence
   that enables operators to certify their deployments?
   Proposed: enable operator certification first (lower cost, faster, more
   commercially valuable); pursue platform certification as enterprise tier
   feature under ADR-0017.

4. **Cross-framework `cross_references` in `iso-42001.yaml`.** Should the ISO
   manifest also include `cross_references` back to EU AI Act and NIST rules,
   or is one-directional cross-referencing from NIST sufficient?
   Proposed: bidirectional, but implemented in the generator (not duplicated
   in YAML) — the generator reads all manifests and builds the graph.

5. **Manifest versioning lock-step.** If `eu-ai-act.yaml` is at v1.2.0 and
   `iso-42001.yaml` is at v1.0.0, does `manifest-version.txt` reflect the
   lowest or the highest? Proposed: `manifest-version.txt` becomes a bundle
   version, and each framework file carries its own `version:` field. The
   bundle version bumps on any framework file change.
