# ADR-0061 — EU AI Act Certification Gap Implementation:
# Content Marking · L40 Incident Tracker · Operator Declaration Gate · Annex IV Generator

**Status:** Accepted — implemented 2026-05-26
**Date:** 2026-05-26
**Authors:** Forge persona (Claude Code, maintainer session)
**Type:** Multi-component — Layer 40 + Three modules + CLI tooling
**Implements:** ADR-0057 §Component 1–4 (architectural spec → concrete implementation)
**Depends on:** ADR-0057 (gap analysis), ADR-0056 (compliance manifest),
ADR-0060 (ISO 42001 / NIST RMF), L16 (audit chain), L19 (disclosure),
L33 (artifact memory), L38 (A2A), self_test.py (boot gate)
**Note on numbering:** ADR-0057 §Component 2 labelled the incident tracker
"Layer 39". L39 is already assigned to AtelierFed (ADR-0053). This ADR
assigns the Incident Tracker to **Layer 40**.

---

## Context

ADR-0057 identified four architectural gaps that block a formal EU AI Act
self-assessment and any downstream BSI-Grundschutz or Notified Body audit:

| Gap | Article | ADR-0057 component |
|---|---|---|
| No machine-readable AI-content marking | Art. 50 §4 | Component 1 |
| No incident tracking or reporting mechanism | Art. 73 | Component 2 |
| No boot-time operator obligation gate | Art. 28–30 | Component 3 |
| No Annex IV technical documentation generator | Art. 43 + Annex IV | Component 4 |

ADR-0057 specifies *what* to build. This ADR specifies *how* — module paths,
data formats, integration points with existing layers, boot-time wiring, CLI
surface, and layer invariants.

The EU AI Act entered full applicability on 2026-08-02. The v1.0.0 release
target is 2026-06-21. All four components must reach M1 quality (testable,
audit-emitting) before release; PDF output and certpack export (M5–M6) may
follow in v1.1.

---

## Component 1 — Content Marking (Art. 50 §4)

### Regulation

Art. 50 §4 requires that AI-generated content (text, images, audio, video)
be marked in a machine-readable format so that it is detectable as
AI-generated. The marking must survive "reasonable" post-processing.

AtelierOS's existing L19 disclosure (one-time human-readable bot-disclosure
card per user) satisfies Art. 50 §1–3 (user-facing disclosure). §4 adds a
*machine-readable layer* for downstream tools, archival systems, and
regulators.

### Module

```
operator/bridges/shared/content_marking.py
```

### Provenance Envelope Format

Every AI-generated artifact (file or text response) receives a sidecar
in the following canonical JSON format:

```json
{
  "format":          "atelier-provenance/1.0",
  "regulation_ref":  "EU-AI-Act-Art-50-4",
  "ai_system_id":    "AtelierOS/{semver}",
  "model_hint":      "{engine_family}",
  "session_hash":    "{sha256[:16] of bridge:chat_key}",
  "generated_at":    "{ISO-8601}",
  "content_sha256":  "{sha256 of artifact bytes at time of writing}",
  "tenant_id":       "{tenant_id}"
}
```

Field invariants:
- `session_hash` is a 16-char prefix of SHA-256 of the raw `chat_key` —
  pseudonymous, not reversible to PII.
- `model_hint` is the engine family string ("claude", "codex", "opencode"),
  never the full model ID or API key material.
- `content_sha256` is computed on the file *after* writing (PostToolUse hook
  reads the file).

### Integration Points

**For file artifacts (Write / Edit / NotebookEdit):**

A PostToolUse hook (`operator/voice/hooks/content_marking_hook.py`) fires
after any successful Write/Edit/NotebookEdit. It:
1. Reads the written file and computes `content_sha256`.
2. Writes `{filepath}.atelier-provenance.json` alongside the artifact.
3. Registers the provenance sidecar with L33 (`artifact_register`) using
   MIME type `application/vnd.atelier.provenance+json`.
4. Emits `content_marking.sidecar_written` to the L16 audit chain (allow-list:
   `artifact_id`, `session_hash`, `tenant_id` — **never** file path or content).

The hook is registered in `operator/voice/hooks/hooks.json` under
`PostToolUse` alongside the existing `audit_hook`.

**For text responses (chat output):**

The adapter appends a structured HTTP response header when serving via REST:

```
X-Atelier-AI-Generated: 1
X-Atelier-System-ID: AtelierOS/{semver}
X-Atelier-Regulation: EU-AI-Act-Art-50-4
```

For Discord / bridge channels where HTTP headers are unavailable, the
provenance is embedded in the session artifact manifest (L33) keyed on
the turn's message ID. No inline text marker is injected into the reply
body — L19 already handles the human-readable disclosure obligation.

### Must NOT (Component 1)

- Must NOT put file path, content snippet, or model ID in audit `details`.
- Must NOT embed provenance markers *inside* file content (sidecar-only).
- Must NOT re-hash on provenance read (hash is locked at write time).
- Must NOT skip sidecar for partial writes (Bash redirect, tee, sed -i) —
  path-gate (L10) blocks those writes entirely; no marking needed for
  blocked operations.
- Must NOT fail-open if L33 `artifact_register` is unavailable — log
  WARNING, write sidecar anyway (marking is independent of artifact memory).
- Must NOT `import anthropic`.

---

## Component 2 — Layer 40: Incident Tracker (Art. 73)

### Regulation

Art. 73 requires providers and deployers of AI systems to report *serious
incidents* to the relevant national market surveillance authority without
undue delay and within 15 working days of becoming aware. A serious incident
is one that directly or indirectly leads to, or may lead to, harm to health,
safety, or fundamental rights of persons.

### Layer Assignment

**Layer 40** — distinct from L39 (AtelierFed / ADR-0053). Layer 40 has no
social-federation responsibilities; it is a pure incident lifecycle manager.

### Module

```
operator/bridges/shared/incident_tracker.py
```

### Storage

```
<tenant>/global/incidents/<YYYY-MM-DD>-<incident_id>.json   (mode 0600)
<tenant>/global/incidents/.index.jsonl                      (mode 0600, append-only)
```

Incident file schema:

```json
{
  "incident_id":      "{uuid4}",
  "detected_at":      "{ISO-8601}",
  "status":           "DETECTED|CLASSIFIED|UNDER_INVESTIGATION|RESOLVED|REPORTED",
  "severity":         "LOW|MEDIUM|HIGH|CRITICAL",
  "trigger_event":    "{L16 event class that triggered auto-detection, or 'manual'}",
  "tenant_id":        "{tenant_id}",
  "channel":          "{channel}",
  "session_hash":     "{sha256[:16] of chat_key, if applicable}",
  "description":      "{free text, operator-supplied or auto-generated summary}",
  "art73_reportable": true,
  "reported_at":      null,
  "reported_to":      null,
  "timeline":         [{"at": "...", "status": "...", "note": "..."}]
}
```

Field invariants:
- `description` is operator-supplied summary — **never** contains user message
  content, transcript text, or raw prompt/response.
- `session_hash` is pseudonymous (16-char SHA-256 prefix), never raw chat_key.
- `art73_reportable` is set by the classifier, not by the triggering hook.

### Auto-Detection

`IncidentTracker.auto_detect()` is registered as a subscriber to the L16
audit event stream. The following L16 events trigger auto-detection:

| L16 event class | Trigger condition | Auto-severity |
|---|---|---|
| `audit.chain_break` | any | CRITICAL |
| `path_gate.self_test_failed` | any | CRITICAL |
| `A2A.request_rejected` | ≥10 in 60 min from same origin | HIGH |
| `consent.bypass_attempt` | any | HIGH |
| `erasure.failed` | any | HIGH |
| `data_flow.blocked` | ≥5 in 10 min, same chat_key | MEDIUM |
| `egress.blocked` | ≥20 in 60 min | MEDIUM |

`consent.bypass_attempt` is a new L16 event class added by this ADR — emitted
when any code path reaches the consent gate with a user who has not granted
consent and the request is not a `/consent` command itself.

### CLI

```
atelier-incident list   [--status <status>] [--severity <sev>] [--limit N]
atelier-incident show   <incident_id>
atelier-incident classify <incident_id> --severity <sev> [--art73]
atelier-incident resolve  <incident_id> [--note "..."]
atelier-incident report   <incident_id> --to <authority-email>
atelier-incident export   [--format json|csv] [--since YYYY-MM-DD]
```

`atelier-incident report` generates an Art. 73 notification template in the
format expected by the German BSI (NIS2 / AI Act portal). It emits
`incident.reported` to the audit chain and sets `reported_at`/`reported_to`
on the incident file. The template is written to `./outputs/` for Discord
attachment.

### Discord Command

`/report-incident <brief description>` — creates a MEDIUM incident, severity
`manual`. Requires `admin` role or higher (L18).

### Audit Events

`incident.registered`, `incident.classified`, `incident.resolved`,
`incident.reported`

Audit allow-list: `incident_id`, `severity`, `status`, `trigger_event`,
`art73_reportable`, `tenant_id`, `channel`, `session_hash` —
**never** description text, user content, or authority contact details.

### Must NOT (Component 2)

- Must NOT put `description` or free text in audit `details`.
- Must NOT put session_hash (even prefix) in audit `details` for REPORTED
  events (contact info is pseudonymous enough at the incident level).
- Must NOT auto-report to external authority (operator action only via CLI).
- Must NOT make `incident.registered` advisory — a write failure must
  propagate as CRITICAL to the boot-level health metric.
- Must NOT store incident files world-readable (mode 0600, flock on write).
- Must NOT `import anthropic`.

---

## Component 3 — Operator Declaration Gate (Art. 28–30)

### Regulation

Art. 28–30 impose concrete obligations on *deployers* (operators in
AtelierOS terminology): use the system as intended, implement human oversight,
notify the provider of serious incidents (Art. 73), and — if the deployment
is High-Risk — maintain logs and register with the EU AI Act database.

AtelierOS is Limited-Risk, but the gate is designed to trigger
*reclassification* if the operator declares Annex III functions, prompting
a different self-assessment path.

### Module

```
operator/bridges/shared/operator_declaration.py
```

### Declaration File

```
<atelier_home>/global/operator_declaration.json   (mode 0600)
```

Schema:

```json
{
  "version":         "1.0",
  "declared_at":     "{ISO-8601}",
  "atelier_version": "{semver at time of signing}",
  "operator": {
    "name":          "{legal entity name}",
    "email":         "{DPO or responsible contact}",
    "jurisdiction":  "{ISO 3166-1 alpha-2, e.g. DE}"
  },
  "deployment": {
    "intended_use":  "{free text, ≤500 chars}",
    "forbidden_use": "{free text, ≤500 chars}",
    "user_population": "{general_public|employees|professionals|researchers}"
  },
  "risk_classification": "limited_risk|high_risk",
  "annex_iii_functions": [],
  "human_oversight_measures": ["{...}", "{...}"],
  "incident_authority": {
    "name":          "BSI",
    "url":           "https://www.bsi.bund.de/...",
    "contact_email": "{...}"
  },
  "declaration_hash": "{sha256 of canonical JSON without this field}"
}
```

`declaration_hash` is the SHA-256 of the JSON with this field set to `null`
and keys sorted alphabetically — a lightweight tamper indicator.

If `annex_iii_functions` is non-empty, `risk_classification` MUST be
`high_risk`. The gate enforces this: a declaration with Annex III functions
and `risk_classification: limited_risk` is rejected.

### Boot-Time Gate

`operator_declaration.py` exposes `check_declaration(atelier_home)` which
`self_test.py` calls as a new **CRITICAL** probe. Failure semantics follow
the existing self-test table:

| Condition | Outcome |
|---|---|
| File missing | CRITICAL → adapter refuses to start |
| File mode not 0600 | CRITICAL → adapter refuses to start |
| `declaration_hash` mismatch | CRITICAL → adapter refuses to start |
| Version mismatch (major bump) | WARNING → adapter starts, surfaces in `doctor` |
| `annex_iii_functions` non-empty + `limited_risk` | CRITICAL → adapter refuses to start |

`operator.declaration_accepted` is emitted to the L16 audit chain on each
successful boot (not on every health check — only on adapter start).
`operator.declaration_missing` is emitted before the CRITICAL halt.

### Wizard CLI

```
atelier-operator-declare
```

Interactive wizard (no external dependencies, pure stdlib):
1. Prompts for operator fields.
2. Asks: "Will this deployment perform any of the following functions?" (lists
   Annex III categories). If yes → sets `risk_classification: high_risk`.
3. Generates canonical JSON, writes to declaration path with mode 0600.
4. Prints SHA-256 for operator records.
5. Emits `operator.declaration_created` to audit chain.

Re-declaration is required (gate rejects the old file) when:
- The installed AtelierOS major version changes.
- `annex_iii_functions` list changes (new deployment scope).

### Must NOT (Component 3)

- Must NOT make the boot gate WARNING-only (must be CRITICAL — fail-closed).
- Must NOT store operator email, DPO contact, or incident authority URL in
  audit `details`.
- Must NOT accept a declaration signed for a different `atelier_version` major.
- Must NOT auto-generate the declaration without operator interaction
  (the wizard prompts; it never fills in deployment fields from config).
- Must NOT `import anthropic` from `operator_declaration.py`.

---

## Component 4 — Annex IV Generator CLI (Art. 43 + Annex IV)

### Regulation

Annex IV of the EU AI Act lists the technical documentation that High-Risk
AI systems must maintain. For Limited-Risk systems the same structure serves
as the canonical self-assessment package that a Notified Body or BSI auditor
would request. All six sections must be machine-generatable from live system
state.

### Module and CLI

```
operator/bridges/shared/annex_iv_generator.py
atelier-annex-iv   generate [--framework eu-ai-act|iso-42001|nist-ai-rmf]
                             [--format markdown|pdf]
                             [--out-dir PATH]
                   validate  (checks all required sources are present)
                   diff      --from <date> --to <date>  (changelog between two snapshots)
```

PDF output requires `pandoc` (checked at runtime; missing pandoc → graceful
fallback to Markdown with a WARNING). Output is written to `./outputs/` for
automatic Discord attachment.

### Six Annex IV Sections and Their Sources

| Section | Content | Primary Source |
|---|---|---|
| 1 | General description — purpose, version, intended use, forbidden use | `operator_declaration.json` |
| 2 | Architecture and data flows | ADR index (`docs/decisions/*.md` header scan) + `tenant.atelier.yaml` |
| 3 | Monitoring, functioning, control | Compliance manifest (`compliance/eu-ai-act.yaml`, ADR-0056) + self_test last result |
| 4 | Logging capabilities | L16 audit chain statistics (event counts, retention config from L37) |
| 5 | Changes to the system | `CHANGELOG.md` or `git log --oneline --since=<declared_at>` |
| 6 | Standards and frameworks applied | Compliance manifest `frameworks[]` list + ADR reference index |

Section 2 ADR scan strategy: `annex_iv_generator.py` reads the first 10 lines
of every `docs/decisions/*.md` to extract `Status`, `Type`, and `Depends on`
fields (no LLM involved). Accepted ADRs populate the architecture table;
Draft ADRs are footnoted as in-progress.

### Integration with ADR-0060

ADR-0060 extends the Annex IV generator with `--framework iso-42001` and
`--framework nist-ai-rmf` subcommands. Those subcommands read the
ISO 42001 and NIST AI RMF manifest files defined in ADR-0060.
`atelier-annex-iv` is the single CLI entry point for all three frameworks.

### Certification Package Export

```
atelier-certpack [--out-dir PATH]
```

Bundles into a single timestamped directory:

```
certpack-{YYYY-MM-DD}/
  README.md                   (cover page for the auditor)
  operator_declaration.json
  risk_classification.md      (RISK-CLASSIFICATION.md symlink copy)
  annex-iv-eu-ai-act.md
  annex-iv-iso-42001.md       (if iso-42001 manifest present)
  annex-iv-nist-ai-rmf.md    (if nist-ai-rmf manifest present)
  compliance_manifest.yaml
  compliance_manifest.yaml.sig (GPG signature, if signing configured)
  audit_summary.json          (chain statistics, not chain content)
  self_test_last.json         (last self-test result)
  incident_export.csv         (all RESOLVED/REPORTED incidents)
```

`atelier-certpack` calls `atelier-annex-iv validate` first; if any required
source is missing it exits 1 with a structured error list — no partial bundles.

Output lands in `./outputs/` for Discord auto-attach.

### Must NOT (Component 4)

- Must NOT include audit chain segment content in the package (summary stats
  only — chain content is operator-confidential and audit-at-rest encrypted
  per L37).
- Must NOT include `operator.email` or DPO contact in any file inside the
  bundle that is not `operator_declaration.json` (that file the operator
  chooses to share; the generator must not scatter PII across other files).
- Must NOT call the Annex IV generator from any hook or daemon (CLI-only,
  operator-initiated).
- Must NOT `import anthropic` from `annex_iv_generator.py`.
- Must NOT fail silently if `operator_declaration.json` is missing — exit 1
  with clear message directing operator to `atelier-operator-declare`.

---

## New L16 Audit Event Classes

This ADR introduces two new event classes to the L16 schema:

| Event class | Emitter | Severity | Audit allow-list |
|---|---|---|---|
| `content_marking.sidecar_written` | content_marking_hook | INFO | `artifact_id`, `session_hash`, `tenant_id` |
| `incident.registered` | incident_tracker | WARNING | `incident_id`, `severity`, `trigger_event`, `art73_reportable`, `tenant_id` |
| `incident.classified` | incident_tracker | INFO | `incident_id`, `severity`, `status` |
| `incident.resolved` | incident_tracker | INFO | `incident_id`, `status` |
| `incident.reported` | incident_tracker | WARNING | `incident_id`, `status`, `tenant_id` |
| `operator.declaration_accepted` | operator_declaration | INFO | `atelier_version`, `risk_classification`, `tenant_id` |
| `operator.declaration_missing` | operator_declaration | CRITICAL | `tenant_id` |
| `operator.declaration_created` | atelier-operator-declare CLI | INFO | `atelier_version`, `tenant_id` |
| `consent.bypass_attempt` | consent gate | WARNING | `channel`, `session_hash`, `tenant_id` |

All new events follow the existing hash-chain link protocol (L16): each event
carries `prev_hash`, and a chain break on any new event class is CRITICAL.

---

## Hook Registration

`operator/voice/hooks/hooks.json` must be extended with:

```json
{
  "PostToolUse": [
    "...",
    "operator/voice/hooks/content_marking_hook.py"
  ]
}
```

`content_marking_hook.py` fires after Write, Edit, MultiEdit, NotebookEdit.
It MUST NOT fire after Bash (Bash file writes are blocked by L10 path-gate;
any that pass through are not AI-generated artifacts in the Annex IV sense).

---

## Self-Test Integration

`operator/bridges/shared/self_test.py` gains one new CRITICAL probe:

```python
# Operator Declaration Gate (ADR-0061 Component 3)
from .operator_declaration import check_declaration
result = check_declaration(atelier_home)
# result: (ok: bool, reason: str)
```

This runs in every self-test call path: boot, `bridge.sh doctor`, Docker
HEALTHCHECK. It does NOT fire in `--quick` mode (10 s budget) to avoid
disk I/O on every container health poll.

---

## Milestones

| ID | Deliverable | Effort | Depends |
|---|---|---|---|
| M1 | `content_marking.py` + hook + PostToolUse registration + INFO audit event | 4h | L33, L16 |
| M2 | `incident_tracker.py` + storage + auto-detect from L16 stream | 6h | L16 |
| M3 | `atelier-incident` CLI (list/show/classify/resolve) | 4h | M2 |
| M4 | `operator_declaration.py` + `atelier-operator-declare` wizard | 5h | self_test.py |
| M5 | Self-test CRITICAL probe for operator declaration | 2h | M4 |
| M6 | `annex_iv_generator.py` + `atelier-annex-iv generate` (Markdown) | 8h | ADR-0056, M4 |
| M7 | `atelier-certpack` export + `atelier-incident report` Art. 73 template | 5h | M3, M6 |
| M8 | `atelier-annex-iv` PDF output (pandoc) + ISO 42001 / NIST subcommands | 4h | M6, ADR-0060 |
| M9 | E2E test: full certpack generation on a live tenant | 4h | M7 |
| M10 | `consent.bypass_attempt` event + `/report-incident` Discord command | 3h | M2 |

**Total: ~45 engineer-hours**

Critical path for v1.0.0 release (2026-06-21): M1–M5 + M10 (testable,
audit-emitting, boot-gate enforced). M6–M9 may ship in v1.1.

---

## Invariants and Must-NOT Summary

- **Fail-closed on everything.** Missing declaration → CRITICAL. Chain break → CRITICAL.
  No "soft mode" env flag.
- **No PII in audit chain.** Session hash prefix only. No email, name, chat_key, content.
- **No `import anthropic`** in any module introduced by this ADR (CI AST lint).
- **No LLM in hot path.** Annex IV generator, incident classifier, and declaration
  checker are all deterministic — no subprocess claude calls.
- **Sidecar-only marking.** Provenance never embedded in file content.
- **Operator-initiated reporting.** `atelier-incident report` is never called
  automatically; it requires an operator CLI invocation.
- **Layer 40 is not social.** No dependency on L39 (AtelierFed), `social_*.py`,
  or federation protocols.

---

## References

- ADR-0057 — EU AI Act Certification Readiness: Gap Closure (architectural spec)
- ADR-0056 — Compliance Manifest (machine-readable compliance rules)
- ADR-0060 — Multi-Framework Compliance Alignment: ISO 42001 + NIST AI RMF
- ADR-0053 — Layer 39 AtelierFed (social federation — NOT the incident layer)
- ADR-0048 — Layer 38 Remote Trigger Receiver + A2A (audit-first invariant model)
- ADR-0044 — Layer 37 Audit-at-Rest Encryption (chain segment format)
- ADR-0045 — Layer 36 GDPR Art. 17 Erasure Orchestrator
- EU AI Act, Articles 28–30, 43, 50 §4, 73; Annex IV
- GDPR Articles 5, 30, 32
- `docs/claude-ref/self-test.md` (boot self-test protocol)
- `docs/claude-ref/layer-security.md` (L16 audit chain schema)
