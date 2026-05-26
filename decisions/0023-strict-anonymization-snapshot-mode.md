# ADR-0023 — Strict-Anonymisation Snapshot Mode (Layer 32)

**Status:** Proposed
**Date:** 2026-05-16
**Companion to:** ADR-0012 (Layer 24 — Large-Data Snapshot Layer),
ADR-0007 Phase 3.3 (data-residency zone gate), Layer 10 (path-gate
on `data_policy.yaml`).
**Implements:** Three sub-phases, all in one focused commit:
32.1 anonymiser module + policy fields, 32.2 wiring into
`data_register` / `data_snapshot`, 32.3 audit events + tests.

---

## Context

Layer 24 (ADR-0012) ships a four-pillar privacy stack for the
snapshot pipeline:

1. PII detection cascade (headers → value regex → optional Presidio)
2. Six redaction strategies per PII class (`drop` / `redact` /
   `pseudonymize` / `mask_partial` / `aggregate_only` / `hash`)
3. Per-field `x-sensitive: true` schema marker → hash before disk
4. Snapshot token-cap (4 000 tokens default) with degradation to
   schema-only on oversize

This is enough for the common case where the operator has curated
column-level policy. **Three residual leakage vectors remain that
no Layer-24 control structurally closes:**

| Vector | Concrete leak |
|---|---|
| **Sample rows of un-classified columns** | A `notes` free-text column the operator forgot to tag goes through raw into `sample[]`. |
| **Quantiles / distinct counts on small data** | `P05 / P95` on 20 rows = effectively min/max → reveals outliers. `distinct: 1` on a `user_id` column tells you the exact set. |
| **Column-name semantics** | Headers like `patient_diagnosis` or `customer_email` leak schema-level intent even when values are redacted. |

Plus a **default-permissive posture** — `strict_mode: false` is the
out-of-box default. An operator who hasn't written a `data_policy.yaml`
gets the most permissive behaviour.

Layer 30 (ADR-0022) made forge engine-agnostic. That means
**Codex CLI, OpenCode + Ollama, and any future worker engine can
now call `data_register` on the same snapshot pipeline**. The
attack surface for these residual vectors is correspondingly wider —
a worker engine the operator trusts less (community model, local
quantised LLM, future Hermes-shaped agent) shouldn't be able to
coax raw data into its context just because the snapshot pipeline
was designed around Claude.

**User-stated requirement** (2026-05-16, voice-note):
> "kann man das noch anonymisieren? Bau mal was ein, was das noch
> anonymisiert und strukturell verankert ist, sodass wirklich
> keine privaten Daten im Language Model landen."

Translation: structurally guarantee that no private data lands
in the LLM, regardless of operator config diligence or worker
engine choice.

---

## Decision

**Introduce a tenant-level "strict-anonymisation" mode that, when
active, structurally restricts every snapshot payload to a
zero-value projection** — schema-shape only, no sample rows, no
quantiles, no distinct counts below a k-anonymity threshold, no
extremes. Plus a **fail-closed post-scan** that walks the entire
LLM-bound payload and rejects it if any leaf-string matches a
known PII regex.

This is **opt-in per tenant** (default OFF — Layer-24 behaviour
unchanged for existing operators) and **enforced server-side** —
no agent-controllable flag, no per-call widening path.

### New `data_policy.yaml` fields

```yaml
spec:
  strict_anonymization: false           # default OFF (backward compat)
  k_anonymity_threshold: 5              # distinct < k → reported as "<k"
  rowcount_laplace_scale: 1.0           # Laplace noise on rowcount when strict
  reject_on_pii_leak: true              # post-scan: fail-closed on regex match
```

All four ride through the existing path-gate-protected
`data_policy.yaml` — only the operator can flip them, the LLM
cannot rewrite the file (Layer 10 v2 hardening).

### Strict-mode projection contract

When `strict_anonymization: true`, the snapshot payload returned
to the LLM is **strictly limited** to:

```jsonc
{
  "file": {
    "format": "csv",
    "size_b": 5234567890,
    "rowcount_approx": 1000847,       // Laplace-noised; rowcount_exact=False
    "rowcount_exact": false             // structurally false in strict mode
  },
  "schema": [
    {"name": "col_name", "type": "string", "pii_class": "email"}
    // No cardinality, no description.
  ],
  "sample": [],                         // ALWAYS empty in strict mode
  "stats": {
    "col_name": {
      "type_class": "categorical",     // "numeric" | "categorical" | "boolean"
      "nulls_class": "some" | "none" | "many",  // bucket, not count
      "distinct_class": "unique" | "low" | "medium" | "high"
      // No nulls (count), no p05/p50/p95, no distinct (count), no top
    }
  },
  "strict": true,                       // structural marker
  "anonymised": true
}
```

Every numeric stat (`nulls` count, `p05/p50/p95`, `distinct` count,
top-N list) is **stripped at projection time**. The LLM sees only
the structural shape of the data — enough to write a Forge tool
that operates on the full data (which still works in the sandbox),
but nothing that can fingerprint individual records.

### k-Anonymity threshold

`distinct_class` is a four-bucket projection of the raw distinct
count:

| distinct | distinct_class |
|---|---|
| `< k_anonymity_threshold` | `unique` (sentinel — never reveals the actual N below k) |
| `[k, 100)` | `low` |
| `[100, 10 000)` | `medium` |
| `>= 10 000` | `high` |

`k_anonymity_threshold` default 5. Below k → bucket `unique`
regardless of true count. An attacker cannot distinguish "1
distinct value" from "4 distinct values" — both report as `unique`.

### Rowcount Laplace noise

`rowcount` → `rowcount + Laplace(0, rowcount_laplace_scale * rowcount)`
when strict. For small datasets (`rowcount < 100`) the existing
Layer-24 jitter already applies; strict mode layers Laplace on top
and **always** sets `rowcount_exact: false`. The exact count never
leaves the registry.

### Post-scan: fail-closed on PII leak

After projection, walk the entire payload (depth-bounded, depth 16)
and apply a curated PII regex set to every string leaf:

- Email shape (`...@...`)
- IBAN shape (`[A-Z]{2}\d{2}[A-Z0-9]{4,}`)
- Credit-card shape (Luhn-eligible 13–19 digit runs)
- E.164 phone shape (`\+\d{8,15}`)
- US SSN shape (`\d{3}-\d{2}-\d{4}`)
- DE Steuer-ID shape (`\d{11}`)

Match → REJECT the payload. Replace with `{"file": {...}, "schema": [],
"sample": [], "stats": {}, "anonymisation_rejected": true,
"reason": "post-scan-pii-leak"}` and emit
`data.anonymisation_rejected_pii_leak` (WARNING). The LLM gets
the file-existence + the rejection flag, nothing more.

`reject_on_pii_leak: false` downgrades the rejection to advisory:
the payload is preserved (with the offending leaves replaced by
`"<pii-redacted>"`) and the audit event fires anyway. Default is
**fail-closed** (true) — operators who want softer behaviour opt in
explicitly.

### Two new audit events

Registered in `forge/security_events.py::EVENT_SEVERITY`:

| Event | Severity | Details (allow-list) |
|---|---|---|
| `data.strict_anonymisation_applied` | INFO | `data_handle`, `columns`, `dropped_keys` (count) |
| `data.anonymisation_rejected_pii_leak` | WARNING | `data_handle`, `match_count`, `reason` |

Metadata only — no field names, no values, no regex hits in clear.
Mirror of L23 / L25 / L28 / L29 metadata-only rule.

### Default-deny posture

Default is `strict_anonymization: false` (backward-compat). But:

- New `data_policy.yaml` template generated by operator-CLI
  (future Phase 32.x) ships with `strict_anonymization: true`
  as the recommended default.
- Documentation in README + CLAUDE.md leads with **"enable for any
  tenant where worker engines outside Claude are reachable"** as
  the bright-line rule.

---

## Consequences

### Positive

- **Hard ceiling on snapshot leakage** that cannot be widened by
  any agent-controllable parameter. The LLM never sees raw values
  from the data even when the operator forgot to declare a column
  as PII.
- **Engine-agnostic safety floor**: Codex / OpenCode / Hermes
  workers inherit the same protection as Claude — operators who
  enable strict mode for delegate-heavy tenants get a uniform
  guarantee independent of which worker the LLM picks.
- **Layered with existing Layer-24 controls**: PII detection,
  redaction, pseudonymisation, token-cap all still run. Strict
  mode is the **last gate**, not a replacement.
- **GDPR Art. 5(1)(c) — data minimisation** is structurally
  satisfied when strict mode is on: the LLM sees the minimum
  needed to write a tool, never more.

### Negative / Tradeoffs

- **LLM cannot iteratively reason on data distribution** in strict
  mode. It sees `distinct_class: high` but not `47 832 distinct
  customer IDs`. A workflow that needs the exact quantile to pick
  a binning strategy must run that as a Forge tool first and feed
  the result back as plain text.
- **Post-scan is regex-based** — the six patterns above cover the
  common cases but a novel PII class (a private API token format,
  a custom medical-record-number format) might slip through. The
  per-tenant `column_overrides` from Layer 24 is the right escape
  hatch for that.
- **Laplace noise on rowcount** is **not differential privacy in
  the formal sense** (no global privacy budget, no composition
  bound). It is a single-shot perturbation that prevents exact
  record-count fingerprinting; an attacker who can call
  `data_snapshot` repeatedly will eventually average out the
  noise. The audit chain caps repeated calls per persona via
  Layer 20 quota; for genuine DP one would need a different
  primitive (Phase 32.5+, separate ADR).
- **Some workflows break.** A "give me a summary of the first 5
  rows" prompt returns an empty `sample` array. Operators who
  enable strict mode must understand this is the intended
  behaviour — the snapshot is for *tool generation*, not for
  *human-readable inspection*.

### What this layer explicitly does NOT do

- **No differential privacy in the formal sense.** As noted; would
  need a per-tenant privacy budget tracker.
- **No homomorphic encryption / secure aggregation.** Workers
  still operate on plaintext data inside their bwrap sandbox
  (which is the whole point of the data-locality pattern — the
  big data never leaves the sandbox, only aggregated results
  come back).
- **No column-name hashing.** Column names stay in clear so the
  LLM can write a tool that addresses them. Operators with
  column-name secrets (rare) should rename their columns at the
  data-source layer.

---

## Test posture

Per-subtask E2E suite (`operator/forge/tests/test_strict_anonymizer.py`,
~20 cases):

- Default OFF: snapshot payload unchanged (byte-identical to
  pre-Layer-32 behaviour).
- Strict ON: `sample: []`, `stats.col.p05` is None, `top` is None,
  `distinct` replaced by `distinct_class`.
- k-anonymity: `distinct: 3` with `k=5` → `distinct_class: "unique"`.
- k-anonymity: `distinct: 47832` → `distinct_class: "medium"`.
- Rowcount: `rowcount_exact` always False; `rowcount_approx`
  differs from raw count.
- Post-scan happy path: payload with no PII passes through.
- Post-scan PII leak (email leaks via stat top): payload replaced
  by rejection skeleton; `data.anonymisation_rejected_pii_leak`
  audited.
- Post-scan advisory mode (`reject_on_pii_leak: false`): payload
  preserved with leaves replaced by `<pii-redacted>`; audit fires
  anyway.
- Audit events metadata-only: regex-walk every emitted detail
  field for raw data leakage.
- Integration through `call_data_register`: strict mode in the
  loaded policy → snapshot returned to caller has strict shape.
- Integration through `call_data_snapshot`: re-snapshot under
  strict policy → same projection.

---

## What you, as Claude Code, must NOT do (Layer 32)

- **Don't ever weaken the strict-mode projection based on a per-call
  tool argument.** `data_register` / `data_snapshot` MUST NOT accept
  a `bypass_anonymisation: true` parameter. The whole point is
  operator-only control via the path-gate-protected policy file.
- **Don't put sample rows, raw stat values, or column descriptions
  into the strict-mode payload "for debugging."** The contract is
  zero-value projection; debugging happens via the sandbox-side
  tool execution, not via dumping data into the LLM context.
- **Don't add a `data.strict_anonymisation_skipped` audit event for
  every non-strict call.** That would saturate the chain on
  default-config tenants. Strict-mode events fire only when the
  mode is active.
- **Don't move the post-scan BEFORE the projection step.** The
  scan must walk the FINAL payload, not the raw snapshot — a
  redaction-strategy that leaks PII into a stat field would
  otherwise be missed.
- **Don't widen the post-scan regex set without ADR amendment.**
  The six patterns are deliberately curated. Adding "anything
  that looks like a number" would false-positive on every
  rowcount and break legitimate workflows.
- **Don't bypass the rejection on `data.anonymisation_rejected_pii_leak`
  by returning the original payload.** When `reject_on_pii_leak: true`
  is the operator's posture, the rejection IS the contract —
  swallowing the audit event and returning the leaky payload would
  defeat the entire structural guarantee.
- **Don't make the k-anonymity threshold per-column overridable.**
  A single tenant-wide threshold is the auditable signal. Per-column
  knobs fragment the guarantee into "well, except for these
  columns" exceptions that erode the user's mental model.
- **Don't write the actual rowcount alongside the noised
  `rowcount_approx`.** The exact rowcount stays in the registry's
  internal `rec.rowcount` field (operator-side, not exposed) so
  the sandbox-side Forge tool can use it via the handle. The
  LLM-bound payload only ever sees the noised value.
- **Don't allow `strict_anonymization: true` to silently disable
  if PyYAML is missing.** Strict-mode is a structural contract;
  fail-loud at policy-load time so the operator notices their
  env is broken, rather than running unprotected.

---

## References

- ADR-0012 — Large-Data Snapshot Layer (the Layer this extends)
- ADR-0007 Phase 3.3 — data-residency zone gate (analogous
  operator-only policy gate)
- ADR-0022 — engine-agnostic Forge + SkillForge (the reason this
  matters for non-Claude engines)
- Layer 10 v2 — path-gate on `data_policy.yaml`
- L23 (voice-transcribe) — metadata-only audit precedent
- GDPR Art. 5(1)(c) — data minimisation
- EU AI Act 2026 Art. 15 — robustness; structural anonymisation
  is part of what makes the system regulator-defensible against
  data-exposure threat models.
