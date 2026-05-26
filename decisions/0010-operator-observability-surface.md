# ADR-0010 — Operator Observability Surface

**Status:** Proposed
**Date:** 2026-05-11
**Companion to:** ADR-0007 Phase 6 (per-tenant Prometheus `/metrics`),
ADR-0009 (declarative tenant surface)
**Implements:** none yet (this ADR is the design contract; phased
implementation plan lands as `0010-implementation-plan.md` after sign-off)

## Context

ADR-0007 Phase 6 shipped per-tenant Prometheus exposition at
`GET /v1/tenants/{tid}/metrics`, plus two curated Grafana dashboards.
That solves the *runtime telemetry* axis. Two adjacent operator
concerns remain unaddressed:

1. **The audit chain stays operator-local.** SOC-2 / ISO-27001 /
   KRITIS / BaFin / EBA reviews expect audit material to land in
   the customer's central SIEM (Splunk, Sentinel, Datadog, Elastic,
   Loki). Today the path is "operator manually `scp`s the JSONL
   file"; no sanctioned forward exists. Phase 6 metrics are an
   aggregate read-projection — they don't carry the per-event
   detail a forensic SIEM query needs.

2. **AtelierOS speaks as "AtelierOS" everywhere.** Disclosure cards,
   voice TTS identity, default reply tone — all hardcoded to the
   project's own branding. A white-label reseller selling the
   platform as "ACME Assistant" must fork to retire the AtelierOS
   identifier from user-visible strings.

Both gaps share the same shape: the operator wants to *project* the
system's surface into their own environment without forking. Both
also share the same constraint: the projection must never bypass
the structural guarantees (audit chain integrity, EU-AI-Act
disclosure requirement, secret-vault redaction).

## Decision

Two small, scope-clean additions co-designed as one observability
surface:

### A) Audit-sink plugin surface

A tenant declares one or more sinks in `tenant.atelier.yaml`:

```yaml
apiVersion: atelier/v1
kind: Tenant
metadata:
  id: acme
spec:
  audit_sinks:
    - kind: syslog
      host: siem.acme.internal
      port: 6514
      protocol: tcp+tls
      facility: local0
      filter:
        min_severity: WARNING
    - kind: webhook
      url: https://splunk.acme.com/services/collector
      secret_ref: SPLUNK_HEC_TOKEN
      batch_size: 100
      flush_interval_s: 5
      retry:
        max_attempts: 5
        backoff: exponential
    - kind: jsonl_tail
      path: /var/log/atelier/<tenant>.jsonl
      rotate_size_mb: 100
```

**Sink implementations.** One small Python module per `kind` under
`core/gateway/sinks/`:

| Kind | What | ~LOC |
|---|---|---|
| `syslog` | RFC 5424 over TCP+TLS or UDP; structured-data fields per event | 150 |
| `webhook` | HTTP POST, HMAC-signed (mirrors existing webhook-dispatcher) | 200 |
| `jsonl_tail` | Append to a file outside `<atelier_home>` (for filebeat/promtail pickup) | 100 |
| `gcs` / `s3` | Object-storage append-batched (deferred to sub-phase) | 250 each |

Each sink subscribes to the tenant's `audit.jsonl` via the existing
hash-chain writer's tail-and-notify hook (Phase 6 already added the
tap point for `/metrics`).

**Failure semantics — load-bearing.** The audit-chain write is the
canonical operation; sink forward is **best-effort, non-blocking**.
Three rules:

1. A failing sink **never** delays or blocks the chain write.
2. Failed events land in `<tenant>/global/sinks/<sink-id>/dead-letter.jsonl`
   for later replay via `atelier_gateway.cli sink replay <tid> <sink-id>`.
3. If dead-letter file exceeds `dead_letter_cap_mb` (default 100 MB),
   the sink degrades: stops accepting new events, logs the degradation,
   emits `sink.degraded` into the chain. Audit chain integrity is
   not affected.

**Filtering — minimal, not policy-replacement.** Each sink may declare
`filter.min_severity` (`INFO` / `WARNING` / `CRITICAL`) and
`filter.event_prefix_allow` (e.g. `["gateway.", "consent."]`). The
filters reduce *forwarded* volume; they never alter what lands in the
canonical chain. A regulator who wants the full chain reads
`<tenant>/global/forge/audit.jsonl` directly.

**Audit events** registered in `EVENT_SEVERITY`:

| Event | Severity | When |
|---|---|---|
| `sink.configured` | INFO | First write after a sink-config change |
| `sink.forwarded` | INFO | Throttled — once per `report_interval_s` (default 60 s), carries counts |
| `sink.forward_failed` | WARNING | One per dead-letter write |
| `sink.degraded` | CRITICAL | Dead-letter cap exceeded |
| `sink.replayed` | INFO | After `sink replay` CLI run |

The throttle on `sink.forwarded` is deliberate — per-event audit of
forwarding would saturate the chain. Operators query Prometheus
(`atelier_audit_sink_forwarded_total{sink_kind, tenant_id}`) for
fine-grained throughput.

### B) Per-tenant branding

Tenant drops `<tenant>/global/branding.yaml`:

```yaml
apiVersion: atelier/v1
kind: TenantBranding
spec:
  bot_name: ACME Assistant
  operator_name: ACME Corporation
  operator_url: https://acme.com/ai-assistant
  voice_persona_default: warm

  disclosure_card:
    # Phase-2 fields below are CUSTOMISABLE.
    # Phase-1 fields (AI-nature notice, opt-in/opt-out commands,
    #   audit-shown-at) are STRUCTURAL and cannot be overridden.
    extra_paragraph: |
      ACME stores conversation transcripts for 90 days
      for service quality review (Art. 6 GDPR — legitimate interest).
    contact_email: ai-support@acme.com
    privacy_policy_url: https://acme.com/privacy

  ui_strings:
    welcome:        "Hi, ich bin der ACME Assistant — wie kann ich helfen?"
    rate_limited:   "Du hast dein Tageskontingent erreicht."
    cancelled:      "Abgebrochen."
    btw_ack:        "📝 Notiz an den laufenden Task durchgereicht."
    consent_card:   "ACME-Hinweis: …"
```

**Layer integration:**

| Layer | Reads | How it's applied |
|---|---|---|
| Layer 19 (disclosure card) | `disclosure_card.*` | `extra_paragraph` appended AFTER the mandatory AI-nature paragraph; `contact_email` + `privacy_policy_url` shown in a footer |
| Layer 16 Phase 4 (consent) | `ui_strings.consent_card` | Renders the operator's text on first observer text drop |
| Voice TTS (`summarize.py`) | `bot_name`, `voice_persona_default` | Bot identifies as `bot_name` in TTS output; persona-tone falls back to `voice_persona_default` when chat has none pinned |
| Bridge replies | `ui_strings.*` | Per-channel ACKs lookup keyed by the canonical message-kind |

**Slash-commands stay canonical.** `/consent`, `/btw`, `/role`,
`/voice-on`, etc. keep their spelling. The customer's branding affects
*what the bot says*, never *what the customer types*. Rationale: the
slash-command surface is the support-and-docs contract; rebranding it
would force per-tenant docs forks.

**Disclosure-card structural lock.** Three fields are hardcoded and
cannot be overridden by `branding.yaml`:

1. The AI-nature statement ("This is an AI assistant operated by …").
2. The `/join` / `/pass` / `/leave` opt-in/opt-out commands.
3. The audit-shown timestamp footer.

A `branding.disclosure_card.replace_card: true` setting is **rejected
at load time** with `branding.violates_disclosure` audit. EU AI Act 2026
Art. 50 baseline must survive any branding edit.

**Audit events:**

| Event | Severity | When |
|---|---|---|
| `branding.loaded` | INFO | First read after a branding change (mtime-cached) |
| `branding.violates_disclosure` | WARNING | Load-time rejection of an attempt to suppress mandatory disclosure fields |
| `branding.malformed` | WARNING | YAML parse error or schema violation; falls back to AtelierOS defaults |

## Consequences

### Positive

- **SIEM integration is an operator-side file drop.** No code patch,
  no PR. The customer's existing log-aggregation pipeline gets full
  audit visibility within minutes of writing the sink declaration.
- **White-label resale becomes a config-only motion.** A managed-service
  provider running AtelierOS for three customers ships three
  `branding.yaml` files — no fork, no parallel codebase, no merge
  conflicts on the next AtelierOS upgrade.
- **Audit-chain canonical write stays the source of truth.** Sinks
  are projections — a chain integrity break is detected by
  `voice-audit verify` regardless of sink state. The new surface
  adds no new failure mode for the chain itself.
- **Reuses existing patterns** — webhook-dispatcher (Phase 2.4),
  secret-vault (Layer 16 v3), audit-chain writer (Layer 16),
  mtime-cached config reload (Layer 14, Phase 6). No new discipline.
- **Bounded blast radius for branding misuse.** The disclosure-card
  structural lock means an operator who *would* suppress the AI-
  nature notice gets a CRITICAL audit event AND a config-load
  rejection.

### Negative

- **Sink latency can flood dead-letter.** A slow SIEM endpoint backs
  up dead-letter; the operator must monitor `sink.degraded` events.
  Mitigation: dead-letter cap is configurable; `sink replay` CLI is
  the catch-up path.
- **Branding strings need translation coverage.** A customer with
  bilingual users must populate `ui_strings` per language — a future
  iteration can add a `ui_strings.de` / `ui_strings.en` split, but
  v1 ships flat (single-locale per tenant).
- **The disclosure-card structural lock is rigid by design.** A
  customer with a genuinely different disclosure obligation (e.g.
  Brazilian LGPD has different wording) cannot replace the card
  outright — they must either accept the AtelierOS baseline + their
  `extra_paragraph`, OR negotiate an exception that lands as a new
  branding-schema field. The friction is intentional; it's the
  surface where EU AI Act compliance is enforced.

### Risks deliberately accepted

- A misconfigured webhook sink with an unreachable URL will pile
  dead-letter until the cap fires. The operator's monitoring is
  expected to catch the `sink.degraded` event; AtelierOS does not
  page anyone on the operator's behalf.
- Branding doesn't extend to the *Prometheus metric names* — those
  carry `atelier_` prefix everywhere. Renaming them would break
  Grafana dashboards across deployments and is out of scope.

## What this ADR deliberately does NOT include

- **Sink fan-out across tenants** (one cross-tenant aggregate sink for
  the gateway operator's own SOC). Phase 6's metric model already
  documents that cross-tenant aggregates need their own auth gate;
  applying the same here would re-open ADR-0007's cross-tenant-fusion
  question. Out of scope.
- **Sink-side filtering via complex expressions** (e.g. CEL, JSON-
  Path queries). The `min_severity` + `event_prefix_allow` are
  enough for SIEM-level routing; richer filtering belongs in the
  customer's SIEM, not in the AtelierOS sink.
- **Branding override for Prometheus exposition.** Metric names
  carry the project identifier by convention; cross-deployment
  dashboard portability outweighs per-tenant naming flexibility.
- **Custom-language disclosure cards** beyond DE/EN. Phase-1 ships
  the existing two-locale support; LGPD / CCPA / Asia-specific
  disclosure cards land if and when those markets surface.

## Implementation surface (sketch — full plan in 0010-implementation-plan.md)

| Sub-phase | Module | Scope |
|---|---|---|
| 10.1 | `atelier_gateway/audit_sink.py` + `sinks/jsonl_tail.py` | sink registry + simplest sink |
| 10.2 | `sinks/syslog.py`, `sinks/webhook.py` | the two SIEM-shape sinks |
| 10.3 | `<tenant>/global/sinks/dead-letter/` + `sink replay` CLI | retry path |
| 10.4 | `branding.py` loader + Layer 19 disclosure integration | branding read path |
| 10.5 | TTS + bridge-reply integration of `ui_strings` | branding write path |
| 10.6 | Closure: Grafana dashboard updates, docs, E2E |

## References

- ADR-0007 Phase 6 — per-tenant `/metrics` (the runtime-telemetry
  axis this ADR complements with audit-detail forwarding).
- ADR-0009 — declarative tenant surface (the GitOps loop also applies
  `branding.yaml` and `audit_sinks` blocks).
- Layer 16 v3 (secret vault) — `secret_ref:` resolution for sink
  credentials.
- Layer 19 (bot-disclosure card) — the structural lock on disclosure
  is enforced here.
