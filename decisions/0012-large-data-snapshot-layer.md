# ADR-0012 — Large-Data Snapshot Layer (PII-Aware Data Locality for Forge)

**Status:** Proposed
**Date:** 2026-05-11
**Companion to:** Layer 6 (Forge), Layer 10 (path-gate), Layer 16 v3
(secret-vault capability split), Compliance baseline (GDPR Art. 5, 32),
ADR-0007 Phase 3.3 (zone-residency)
**Implements:** none yet (this ADR is the design contract; phased
implementation plan lands as `0012-implementation-plan.md` after sign-off)

## Context

Forge (Layer 6) generates runtime Python tools that run sandboxed
under bwrap with `ro-bind` mounts for declared input paths. Today the
agent sees the FILE PATH as a string in the prompt but knows nothing
about the file's content shape. Two failure modes follow:

1. **The agent inflates context with raw data.** Asked to "analyse
   `/data/sales.csv`", a naive agent may `Read` the first 5000 lines
   into context to "understand the data". A 1 GB CSV burns the
   context window before any reasoning starts; a tool that *could*
   have answered the question with deterministic pandas/polars never
   gets forged.

2. **PII reaches the LLM unredacted.** Even when the agent does the
   right thing and forges a tool, an intermediate inspection
   (e.g. `head -20 sales.csv`) pipes real e-mail addresses,
   phone numbers, IBANs through the LLM context. The audit chain
   records what the LLM was asked, not the underlying file —
   regulator review of `audit.jsonl` won't catch this leak.

AWP-aware tooling solves this by passing only a **compact statistical
snapshot** of the data to the LLM and letting the agent *forge a tool
that operates on the full dataset deterministically*. AtelierOS has
all the parts (Forge for tools, bwrap for isolation, path-gate for
structural protection) except the snapshot layer itself.

The data-minimisation principle (GDPR Art. 5(1)(c)) and security-of-
processing (Art. 32) both point at the same answer: the LLM sees
**aggregated, redacted projections**; the **full data stays
sandbox-side**.

## Decision

A new **data-locality surface** layered on top of Forge, opt-in per
tool via two new schema keys (`x-data` + `x-snapshot`). The surface
has four components:

### A) Snapshot generator

When a tool's `input_schema` declares a field with `x-data:
large_dataset`, the runner does NOT pass the file content to the LLM.
Instead, before the LLM call that would forge or invoke the tool,
the snapshot generator inspects the file and produces a structured
projection:

```yaml
# Generated snapshot — what the LLM actually sees
file:
  path:     /data/sales.csv           # path reference only
  format:   csv
  size_b:   1_073_741_824
  rowcount: 9_842_117                 # noisy ±5 for small ranges
  encoding: utf-8

schema:
  - {name: order_id,      type: string,  pii: opaque_id}
  - {name: customer_email, type: string,  pii: email}
  - {name: amount_eur,    type: float,   pii: none}
  - {name: country,       type: string,  pii: none, cardinality: 27}
  - {name: order_date,    type: date,    pii: none}

sample:
  - {order_id: "***pseudo:A1B2***", customer_email: "<email>", amount_eur: 142.50, country: "DE",    order_date: "2026-04-22"}
  - {order_id: "***pseudo:C3D4***", customer_email: "<email>", amount_eur:  87.10, country: "AT",    order_date: "2026-04-22"}
  # ... 18 more, head + random + tail blend

stats:
  amount_eur:
    nulls:      0
    p05:        4.99            # P5 / P95 instead of true min/max
    p50:       42.00
    p95:      512.00
  country:
    distinct:  27               # via HyperLogLog (datasketch)
    top:       [DE, AT, CH]     # only when distinct < 50
  customer_email:
    distinct:  ~1_540_000       # PII column: count only, no values
```

The generator is implemented as a deterministic Python module
(`atelier_data/snapshot.py`); supports CSV / TSV / Parquet / JSON /
JSONL out of the box via DuckDB (read-only mode — no SQL execution
exposed to the LLM, just schema + stats extraction). Format
detection by magic-byte + extension.

### B) PII detection + redaction pipeline

A three-layer detection cascade runs across both schema and sample:

1. **Header heuristics.** Column-name patterns: `email`, `mail` →
   email; `phone`, `tel`, `mobile` → phone; `iban`, `bic` → bank;
   `name`, `firstname`, `surname` → name; `dob`, `birthdate` →
   date_of_birth; `ssn`, `ahv`, `steuer_id` → national_id;
   `address`, `street`, `zip`, `postcode` → address.

2. **Value regex.** Same six-class regex set already in use for
   compliance-zone classification (ADR-0007 Phase 3): e-mail,
   +49/+1/+44 phone, IBAN, credit card, US SSN, CH AHV, DE
   Steuer-ID. Run against the first 1000 rows.

3. **Optional Presidio backend.** When
   `<atelier_home>/global/data_policy.yaml` declares
   `pii_backend: presidio`, the operator-installed Presidio NLP
   models add Named-Entity-Recognition for the harder cases
   (person names, locations, organisations). Off by default —
   Presidio brings ~150 MB of model deps; the regex+header pass
   covers ~85% of the operationally-important PII classes.

Per detected PII class, the redaction strategy is configurable:

| Strategy | Effect on sample | Effect on stats |
|---|---|---|
| `drop` | Column missing from sample entirely | Column missing from stats |
| `redact` (default) | Literal `<email>` / `<phone>` etc. | Count + distinct only, no values |
| `pseudonymize` | `***pseudo:A1B2***` consistent per value | Distinct count preserved (Faker-deterministic) |
| `mask_partial` | `j****@***.com` | Count + distinct only |
| `aggregate_only` | Column missing from sample | Count + distinct only |
| `hash` | SHA-256 of value, first 8 chars | Count + distinct |

**Pseudonymisation determinism** uses a per-tenant secret seed from
the vault (`secret_ref: PSEUDO_SEED`). Same tenant → same value →
same pseudo across invocations. Cross-tenant → different pseudos
(no linkage attack).

Operator-side overrides via `<atelier_home>/global/data_policy.yaml`:

```yaml
apiVersion: atelier/v1
kind: DataPolicy
spec:
  pii_backend: regex+headers   # regex+headers | presidio
  default_strategy: redact
  column_overrides:
    customer_email: pseudonymize
    iban: drop
    zip_code: aggregate_only
  noise:
    rowcount_jitter:    "±5_when_lt_100"
    distinct_jitter:    "±3_when_lt_50"
    extremes:           "p05_p95"   # min/max never raw
```

### C) Forge schema extension

Forge tools declare data-locality intent on the input field:

```yaml
input_schema:
  properties:
    sales_data:
      type: string
      x-bind: ro                    # existing — read-only sandbox bind
      x-data: large_dataset         # NEW — triggers snapshot pipeline
      x-snapshot:                   # NEW — per-tool snapshot tuning
        rows: 20
        rows_strategy: head+random+tail
        stats: [nulls, p05, p50, p95, distinct, top]
        pii_overrides:
          customer_email: pseudonymize
```

The tool body itself receives the **real path** as a `ro-bind` mount
unchanged — Pandas/Polars/DuckDB read the full data and compute the
real answer. The snapshot is the AGENT-FACING view; the SANDBOX-FACING
data stays unmodified.

### D) MCP surface

Two new tools on the forge MCP server:

| Tool | Purpose |
|---|---|
| `mcp__forge__data_register(path, format?, policy?)` | Registers a large dataset by path; returns a `data_handle` and the LLM-facing snapshot. The handle is used in subsequent tool inputs. |
| `mcp__forge__data_snapshot(data_handle, options?)` | Re-generates a snapshot with different options (more sample rows, different stats, different redaction) — bounded by the operator's data_policy. |

The handle pattern means an agent can register a dataset once, see
the snapshot, forge multiple tools that work against the same data,
and inspect intermediate results — all without the real data ever
crossing into the LLM context.

### E) Audit events

| Event | Severity | Carries |
|---|---|---|
| `data.registered` | INFO | `data_handle`, format, size_b, rowcount (noised) |
| `data.snapshot_generated` | INFO | `data_handle`, schema-shape (column count + PII-class counts) — **NEVER the snapshot content itself** |
| `data.pii_detected` | INFO | classes detected (e.g. `email:1, phone:2`) — class counts only |
| `data.tool_accessed` | INFO | `data_handle`, tool_name, ro-bind path |
| `data.policy_violated` | WARNING | attempt to register a dataset without a matching tenant policy zone |
| `data.snapshot_oversized` | WARNING | snapshot exceeds the operator-configured prompt-token cap (default 4000 tokens); pipeline falls back to schema-only |

Audit-event payloads carry **only metadata** — schema shape, class
counts, sizes. The snapshot itself is the LLM-facing artefact; it
never lands in the audit chain. This mirrors the L23 rule for
voice-transcribe (`result.text` never in the chain).

## Consequences

### Positive

- **Closes the data-locality gap for Forge.** A 1 GB CSV becomes a
  500-token prompt + a deterministic tool run; today it's either an
  inflated context window or silently leaked PII.
- **GDPR Art. 5 + Art. 32 alignment is structural, not policy-paper.**
  The snapshot pipeline minimises what reaches the LLM by
  construction; the audit chain records that the redaction happened.
- **No new sandbox needed.** Reuses bwrap, path-gate, ro-bind.
  Snapshot generator runs inside the same MCP-server process Forge
  already runs in.
- **Operator policy is centralised in one file.** A DPO reviewing
  `<atelier_home>/global/data_policy.yaml` sees the redaction
  strategy for every PII class in one place; per-tenant overrides
  through `tenant.atelier.yaml::data_residency.pii_overrides`
  (Phase 3 mechanism extended).
- **Pseudonymisation determinism enables joins** without exposing
  raw values. Agent forges a tool that joins `customer_id` from
  CSV A with `customer_id` from CSV B; both columns pseudonymise
  to the same tokens under the same tenant seed; joins work; raw
  values never enter the LLM.
- **The compliance-zone routing (ADR-0007 Phase 3.3) gets
  reinforced.** A tenant pinned to `eu-west` cannot have its data
  snapshot processed by an engine outside that zone — the snapshot
  pipeline runs adapter-side, after engine selection, so the zone
  gate fires first.

### Negative

- **PII detection is never perfect.** False negatives exist —
  a "notes" free-text column may contain unstructured PII the regex
  pass misses; Presidio's NER reduces but does not eliminate this.
  Mitigation: explicit operator override per column; `data_policy`
  fail-closed on unrecognised column types when `strict_mode: true`.
- **Linkage attacks remain possible.** A snapshot with `country:
  DE` + `dob: 1987-03-14` + `zip: 12345` is structurally redacted
  but jointly re-identifies a person. Mitigation: operator can
  declare `quasi_identifiers: [zip, dob, country]` to force
  `aggregate_only` on the combination.
- **DP-Light is not formal DP.** P5/P95 + count-jitter prevent
  obvious outlier leakage but offer no ε-bounded privacy
  guarantee. Operators with formal-DP requirements need to layer
  Opacus/PySyft on the sandbox-side tool output — not a snapshot-
  layer concern.
- **Snapshot generation has a wall-clock cost.** A 10 GB CSV's
  first-pass HyperLogLog over distinct-counts is O(n) — measured
  seconds, not milliseconds. Mitigation: snapshot cached in
  `<atelier_home>/global/data/snapshots/<handle>.json` keyed by
  file-content-hash; reused across MCP calls within the same task.
- **Format coverage starts narrow.** CSV / TSV / Parquet / JSON /
  JSONL only in v1. Excel, ORC, Avro, proprietary formats need
  follow-up work. Mitigation: explicit `Format not supported`
  rejection at register-time, not silent fall-through to "treat
  as text".

### Risks deliberately accepted

- A tool author who declares `x-data` but doesn't read the snapshot
  semantics may misuse the path. Mitigation: schema validator
  rejects tools that combine `x-data: large_dataset` with
  `x-redact: true` (the data-handle pattern is already the
  redaction; double-protection signals confused intent).
- An operator who sets every PII strategy to `drop` produces
  snapshots so minimal the LLM cannot generate useful tools.
  This is the operator's choice and surfaces fast in the first
  failed run.

## What this ADR deliberately does NOT include

- **A data-lake / storage layer.** The path `/data/sales.csv` is
  whatever the operator's filesystem says. AtelierOS does not own
  the storage substrate, only the locality + snapshot pipeline.
- **Cross-run dataset persistence.** A `data_handle` is task-scoped
  by default; the snapshot cache is best-effort within the same
  task. Long-lived dataset registration is an explicit operator
  concern, not a Forge feature.
- **Streaming / out-of-core compute APIs.** The forged tool itself
  uses whatever Pandas / Polars / DuckDB can do — we don't ship a
  custom streaming framework. The snapshot layer is read-only over
  the input; tool authors choose their own compute model.
- **Schema-evolution detection.** Re-registering a file with the
  same path but changed schema gets a fresh handle + a new
  snapshot. We do not track "this dataset's schema changed since
  last time".
- **A formal differential-privacy guarantee.** DP-Light only.
  Tenants with formal-DP needs apply standard frameworks
  (Opacus / PySyft) tool-side; this ADR does not pretend to
  replace them.

## Compliance-baseline interaction

The compliance-baseline section in `CLAUDE.md` lists structural
mechanisms that must survive any change. This ADR:

- **Strengthens GDPR Art. 5 (data minimisation):** snapshot is
  smaller than the source; redaction is structural.
- **Strengthens GDPR Art. 32 (security of processing):** raw data
  never leaves the bwrap sandbox; LLM context carries only the
  redacted projection.
- **Strengthens compliance-zone routing (ADR-0007 Phase 3.3):**
  snapshot generation runs adapter-side after engine selection;
  zone-mismatched data can be rejected before any LLM call.
- **Does not weaken any existing mechanism.** Audit chain, path-
  gate, secret-vault split, consent gate — all unchanged.

The audit-chain payload rule is restated explicitly: **snapshot
content never lands in audit events**, only the schema-shape +
class-count metadata. The L23 voice-transcribe contract
(metadata-only audit) generalises into a project-wide invariant:
*PII-bearing artefacts never enter the chain; only their
schema-shape + counters do.*

## Implementation surface (sketch — full plan in 0012-implementation-plan.md)

| Sub-phase | Module | Scope |
|---|---|---|
| 12.1 | `atelier_data/format_sniffer.py` + `atelier_data/snapshot_csv.py` + `_parquet.py` + `_json.py` | Format-specific snapshot generators (DuckDB read-only) |
| 12.2 | `atelier_data/pii_detector.py` (regex+headers) | Detection pipeline, no Presidio yet |
| 12.3 | `atelier_data/redactor.py` + `data_policy.yaml` loader | Strategy application + operator policy |
| 12.4 | Forge schema extension (`x-data`, `x-snapshot` keys) + validator | Tool-author surface |
| 12.5 | MCP tools `data_register` + `data_snapshot` | Agent-facing API |
| 12.6 | Pseudonymisation seed via secret-vault + Faker integration | Deterministic pseudo across calls |
| 12.7 | Presidio optional backend behind `pii_backend: presidio` flag | NER for harder PII classes |
| 12.8 | Audit events + Prometheus metrics + Grafana panel | Observability |
| 12.9 | E2E: 1 GB CSV with synthetic PII through full pipeline + closure |

## References

- Layer 6 (Forge) — the runtime tool generation this ADR layers on.
- Layer 10 (path-gate) — the structural FS-write protection the
  snapshot pipeline does not weaken.
- Layer 16 v3 (secret-vault) — the pseudonymisation seed lives here.
- L23 (voice-transcribe) — the metadata-only-audit precedent this
  ADR generalises.
- ADR-0007 Phase 3 — per-tenant policy; `tenant.atelier.yaml`
  carries the per-tenant PII overrides.
- Compliance baseline in `CLAUDE.md` — GDPR Art. 5 / Art. 32
  anchors.
- [Microsoft Presidio](https://github.com/microsoft/presidio) —
  optional NER backend.
- DuckDB read-only mode — used as the schema+stats engine.
