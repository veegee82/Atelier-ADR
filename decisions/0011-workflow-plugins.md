# ADR-0011 — Workflow Plugins (Multi-Trigger + Import/Export)

**Status:** Proposed
**Date:** 2026-05-11
**Companion to:** ADR-0001 (AWP adoption), ADR-0005 (AWP standards-only),
ADR-0006 (DAG walker), ADR-0007 (multi-tenant), ADR-0007 Phase 5
(.atelier-pkg packaging), ADR-0009 (declarative tenant surface)
**Implements:** none yet (this ADR is the design contract; phased
implementation plan lands as `0011-implementation-plan.md` after sign-off)

## Context

AtelierOS' AWP-Walker (ADR-0006) executes a `workflow.awp.yaml` DAG
node-by-node, with engine-policy gating, R17 validation, audit trail.
Today the trigger is always synchronous and operator-initiated —
`atelier_gateway.cli walker run <file>`.

Two adjacent capabilities are missing:

1. **No periodic execution.** Many useful workflows are periodic by
   nature: morning news briefings, weekly compliance summaries, daily
   inbox triage, monthly KPI rollups. Operators today wire host-level
   cron — bypassing tenant policy, engine allowlist, budget gate,
   audit chain.

2. **No in-chat invocation.** A workflow defined in YAML is also a
   *natural sub-routine* for an agent conversation: "summarise this
   week's incidents", "build the Friday deck". Today this requires
   either copy-pasting the workflow body or a code patch. The walker
   exists but has no first-class slash-command surface.

The conceptual frame that unifies both: a workflow is a **plugin** —
a portable, self-contained artefact a tenant ships, imports, exports,
and triggers from multiple sources. Schedule is one trigger source.
Slash-command is another. Gateway REST API (already exists) is a
third. Same DAG, same walker, same audit pipeline — only the trigger
differs.

## Decision

A unified **workflow-plugin surface** with three trigger sources, a
portable file format, and CLI-driven import/export.

### A) Workflow as a portable plugin

A workflow plugin is **one YAML file**, self-contained enough to be
mailed, gisted, dropped into a Git repo, or bundled into an
`.atelier-pkg`. Existing AWP schema unchanged; AtelierOS adds an
optional `triggers:` block as plugin-overlay metadata that AWP-aware
consumers without a scheduler simply ignore (the AWP-standards-only
line from ADR-0005 holds).

```yaml
# Self-contained workflow plugin
apiVersion: awp/v1
kind: Workflow
metadata:
  id: news-briefing
  version: 1.2.0
  description: Daily news briefing — tech + AI + EU regulation
  author: silvio@example.com

spec:
  # ---------- AWP-standard fields (unchanged) ----------
  agents:
    - id: researcher
      persona: research
      engine_id: claude_code
  nodes:
    - id: fetch_news
      agent: researcher
      input: "Search the last 24h for: ${args.topics}"
    - id: summarise
      agent: researcher
      depends_on: [fetch_news]
      input: "Summarise the headlines for a ${args.audience_level} reader"

  # ---------- AtelierOS plugin overlay (optional) ----------
  args:
    topics:
      type: string
      default: "tech,ai,eu-ai-act"
      description: comma-separated topic list
    audience_level:
      type: enum[novice,intermediate,expert]
      default: intermediate

  triggers:
    schedule:
      cron: "0 9 * * 1-5"
      timezone: Europe/Berlin
    slash:
      command: /news
      aliases: [/briefing]
      visibility: owner|admin
    api:
      enabled: true   # the existing Gateway POST /runs path

  deliver_to:
    bridge: telegram
    chat_id: "${tenant.default_owner_chat}"
    format: voice
    fallback:
      kind: email
      address: "${tenant.owner_email}"

  acts_as:
    on_behalf_of: "${trigger.invoker_uid}"   # filled per trigger source
```

The `${trigger.invoker_uid}` / `${args.*}` / `${tenant.*}` shape is a
documented substitution surface — resolved at trigger-time by the
plugin loader, not by AWP. The Walker sees a fully-resolved DAG.

### B) Three trigger sources, one execution path

| Source | Invoker identity | Args | Output destination |
|---|---|---|---|
| **schedule** | `acts_as.on_behalf_of` or `service_account` (declared in plugin) | `args.*` defaults from plugin | `deliver_to` block from plugin |
| **slash** (`/news topics=tech`) | The chatter's platform uid | Defaults overridden by command-line key=value pairs | Triggering chat (default) or plugin's `deliver_to` |
| **api** (`POST /v1/tenants/<tid>/workflows/<id>/runs`) | Bearer-token tenant | Request body JSON | Webhook + SSE (existing Gateway pattern) |

**All three sources go through the SAME `RunDispatcher.submit()`
path** — the engine-policy gate, zone-residency gate, rate-limit,
budget, audit chain all fire unchanged. Trigger source is recorded
in `workflow.triggered.source` so a forensic review sees the same
DAG run with the same gates regardless of how it was kicked off.

### C) Scheduler daemon per tenant

`atelier-scheduler@<tid>.service` — single long-running process per
tenant. Reads `<tenant>/workflows/*.yaml` on boot AND on mtime
change. Builds an internal cron-table from each workflow's
`triggers.schedule` block (if present); fires walker runs at the
declared time.

Each fired run:
1. Resolves `acts_as` to an identity. `on_behalf_of: <uid>` is
   re-checked at fire time (TOCTOU defence — auto-pause with
   `workflow.disabled_role_loss` if the user lost the role).
2. Submits the workflow through `RunDispatcher.submit()` with
   trigger source `schedule`.
3. On terminal state, writes final-text + R17 artefacts to the
   configured outbox.
4. Audits `workflow.triggered { source: schedule }` →
   `workflow.delivered` (or `workflow.delivery_failed`).

`on_overrun: skip | queue | parallel` policy on the
`triggers.schedule` block decides what happens when a previous run
is still active. Default `skip` with `workflow.skipped_overrun`
audit.

### D) Slash-command dispatch

A workflow with `triggers.slash.command: /news` registers a custom
slash-command in the bridge dispatcher (same loader as the custom
slash-commands from ADR-0009, but the action is "run this workflow"
instead of "pin this persona for one turn"). On invocation:

1. The bridge dispatcher parses `/news topics=tech,ai` into
   `{topics: "tech,ai"}`.
2. Argument values are validated against the plugin's `spec.args`
   schema; unknown args fail with a usage hint; missing required
   args (no default) fail with a request for the missing field.
3. The dispatcher submits the run via `RunDispatcher.submit()` with
   trigger source `slash`, `invoker_uid` = chatter's platform uid,
   `args.*` resolved from the parsed command line.
4. Visibility gate (`owner | admin | member | all`) decides whether
   the chatter may run this workflow. Mirrors the role contract from
   Layer 18.
5. Output streams back into the triggering chat by default (the
   plugin's `deliver_to` is overridable for cron, NOT for slash —
   slash always replies in the triggering chat unless explicitly
   directed elsewhere with `/news --deliver-to=email`).

The chat sees a normal bridge reply, with the run-id mentioned in
a footer so a follow-up `/run-status <run-id>` works.

### E) Import / export — first-class CLI

The single-file format is the **portable atom**. Three CLI verbs
on top:

```bash
# Export a workflow plugin (sanitised — no secret_refs, no bridge tokens)
atelier_gateway.cli workflow export <tenant> <workflow-id> > news-briefing.awp.yaml

# Import a workflow plugin into a tenant
atelier_gateway.cli workflow import <tenant> ./news-briefing.awp.yaml

# Validate a workflow plugin without importing (schema + DAG + R17)
atelier_gateway.cli workflow validate ./news-briefing.awp.yaml

# List installed workflows in a tenant
atelier_gateway.cli workflow list <tenant>
```

**Export sanitisation is structural, not optional.** The exporter
walks the workflow tree and:
- Replaces resolved `secret_ref:` values with the literal
  `${tenant.secrets.<key>}` placeholder.
- Strips `tenant.*` substitutions of their resolved values (the
  placeholder name survives, the runtime value does not).
- Strips bearer-tokens from any `deliver_to.webhook.*` block.
- Audits `workflow.exported` with workflow-id + sanitisation
  count.

An exported workflow can be safely committed to a public repo, mailed
to a colleague, or attached to a forum post — the structural
guarantee is *no secrets land in the export*.

**Import validates fail-closed.** Schema violations, unknown args,
cron-syntax errors, references to non-existing personas — all reject
at import time, not at first fire. The import audits as
`workflow.imported { from_hash: <sha256-of-input>, workflow_id, version }`.

### F) Bundle distribution via `.atelier-pkg` (ADR-0007 Phase 5)

Multiple related workflows ship as one signed bundle:

```
acme-briefings-v1.0.0.atelier-pkg/
└── payload/
    ├── workflows/
    │   ├── morning-news.awp.yaml
    │   ├── weekly-kpi.awp.yaml
    │   └── monthly-board.awp.yaml
    ├── personas/
    │   ├── researcher.json
    │   └── briefing-author.json
    └── skills/
        └── acme-briefing-style/SKILL.md
```

Install with the existing `package install` verb extended to honour
the `workflows/` payload subdir. ed25519-signed; the workflow
payload subdir gets the same trust gate as personas + skills.

### G) GitOps integration (ADR-0009)

Workflows live in the customer's `atelier-config` Git repo alongside
personas and slash-commands:

```
atelier-config/
├── workflows/
│   ├── morning-news.awp.yaml
│   └── friday-kpi.awp.yaml
├── personas/
│   └── briefing-author.json
└── slash-commands/
    └── tkt.yaml
```

The GitOps apply-loop reads `workflows/` and reconciles into the
tenant. Schedule edits in Git = audit trail in Git + audit chain
entry on apply. A `git revert` of a `cron` change rolls back the
schedule at the next apply cycle.

### H) Identity + consent

**Schedule trigger.** Either:
- `acts_as.on_behalf_of: <uid>` — runs as a specific user; user must
  have `owner` or `admin` role in the target chat. Re-checked at
  every fire (auto-pause on role loss).
- `acts_as.service_account: <name>` — operator-owned identity from
  `<tenant>/global/auth/service_accounts.json`. Service accounts may
  only deliver to chats where a Layer-19 disclosure card has fired
  at least once for *some* uid in the chat.

**Slash trigger.** Identity is the chatter's platform uid. No
re-resolution needed — the gate is the visibility field
(`owner|admin|member|all`).

**Push-into-chat baseline.** A scheduled run that delivers into a
chat the recipient never opted into is structurally rejected. The
Layer 19 disclosure card is the consent anchor — without it, the
delivery fails with `workflow.delivery_no_disclosure`.

### I) Outbox integration

Delivery goes through the existing outbox protocol:
- `format: chat` — text message in target chat.
- `format: voice` — text + voice-note attachment (audience profile
  resolved from the recipient's `profile.json`).
- `format: email` — uses the email bridge if configured.
- `format: api` — for `trigger.source: api`, delivery is just the
  Gateway response (webhook + SSE).

`fallback:` block: if primary delivery fails (chat archived, bridge
down), retry through the fallback channel. Failed deliveries land
in `<tenant>/workflows/dead-letter/`.

### J) Audit events

| Event | Severity | When |
|---|---|---|
| `workflow.imported` | INFO | `workflow import` CLI succeeded |
| `workflow.exported` | INFO | `workflow export` CLI run; carries sanitisation count |
| `workflow.registered` | INFO | Workflow appears in tenant tree (via GitOps, package install, or CLI import) |
| `workflow.triggered` | INFO | Walker run started; carries `source` (schedule \| slash \| api), `invoker_uid`, `run_id` |
| `workflow.delivered` | INFO | Output reached the outbox |
| `workflow.delivery_failed` | WARNING | Outbox write failed; retries / dead-letter triggered |
| `workflow.delivery_no_disclosure` | WARNING | Scheduled delivery blocked because target chat has no disclosure trail |
| `workflow.skipped_overrun` | WARNING | Previous run still active, skip policy applied |
| `workflow.disabled_role_loss` | WARNING | `on_behalf_of` user lost their role; workflow auto-paused |
| `workflow.malformed` | WARNING | YAML parse / schema error at load time |

## Consequences

### Positive

- **Workflows become first-class shareable plugins.** A YAML file is
  the unit of distribution; bundles are the curated set; GitOps is
  the per-tenant lifecycle. Same artefact, three distribution
  channels.
- **One DAG, many triggers.** The same news-briefing runs as a 09:00
  schedule for the CEO, as `/news` on demand for the analyst team,
  and as an API call from the company's mobile app. Tenant policy,
  budget, audit pipeline are identical across all three.
- **Compliance posture preserved.** All three triggers go through
  `RunDispatcher.submit()` — engine-policy / zone / quota / audit
  fire unchanged. Push-into-chat is gated on a Layer-19 disclosure
  trail.
- **Reuses existing infrastructure** — Walker (ADR-0006), Dispatcher
  (ADR-0007 Phase 2), `.atelier-pkg` packaging (Phase 5), GitOps
  apply-loop (ADR-0009), outbox protocol, audit chain. No new
  discipline.
- **Export sanitisation is structural** — `workflow export` strips
  secrets before they can land in a public repo. A regulator
  reviewing the customer's GitHub finds workflow logic, never
  credentials.

### Negative

- **One more daemon per tenant** (the scheduler) — modest cost
  (~20 MB RSS per Python process). Tenants with no schedules can
  skip the timer entirely.
- **Substitution surface is a small DSL.** `${trigger.invoker_uid}`,
  `${args.*}`, `${tenant.*}` introduce a templating layer the AWP
  standard doesn't have. Mitigated by scoping the substitution to
  the AtelierOS plugin overlay — the resolved DAG handed to the
  Walker is plain AWP.
- **Slash-command surface gains a per-tenant differential.** A
  workflow that registers `/news` in tenant A doesn't exist in
  tenant B. Mitigated by `/help` already separating bridge / tenant
  / workflow commands.
- **Cron-syntax mistakes** are caught at import / apply time. A
  malformed schedule never registers, but a *semantically* wrong
  schedule (`0 9 * * 6,7` when the operator meant weekdays) is a
  runtime concern. Mitigated by an explicit `next_fire_at` audit
  log on registration so the operator can sanity-check.

### Risks deliberately accepted

- A scheduled workflow that consumes the tenant's full token budget
  by 09:01 leaves nothing for the rest of the day. Mitigation:
  per-workflow `max_runs_per_week` cap + existing tenant
  `max_tokens_per_day` gate.
- A workflow plugin imported from an untrusted source could declare a
  schedule against a chat the operator administers. Mitigation:
  `workflow import` requires explicit `--confirm-triggers` when the
  YAML carries any `triggers:` block, with a per-trigger summary
  shown for the operator to approve.
- An `on_behalf_of` user who is removed from the chat leaves an
  orphan workflow. Auto-disabled with audit; operator-side cleanup
  CLI handles the rest.

## What this ADR deliberately does NOT include

- **Reactive triggers** ("fire when GitHub-issue X opens") — that's
  the lifecycle-event-bus space, deliberately deferred. Schedule
  is time-based; slash is user-initiated; api is operator-initiated.
  All three are *requested* runs, never *reactive* runs.
- **Workflow chaining via cron** ("when schedule A finishes, run
  schedule B"). That's what the DAG itself is for. If you need a
  chain, model it inside one workflow.
- **UI for cron editing or workflow authoring.** CLI + YAML only in
  v1. A future Web-UI may sit on top of the same surface.
- **Cross-tenant workflows** — each workflow lives in one tenant.
  Bundle distribution is the cross-tenant pattern.
- **Mutating workflow imports** (e.g. "import this and adjust the
  cron to my timezone automatically"). Import is verbatim. The
  customer's GitOps repo is the right place to fork-and-adjust.

## Implementation surface (sketch — full plan in 0011-implementation-plan.md)

| Sub-phase | Module | Scope |
|---|---|---|
| 11.1 | `atelier_gateway/workflow_loader.py` | YAML loader + substitution resolver + arg-schema validation |
| 11.2 | `atelier_gateway/workflow_registry.py` + CLI verbs `import` / `export` / `validate` / `list` | Tenant-scoped workflow store + sanitised export |
| 11.3 | `atelier_gateway/scheduler.py` + systemd unit `atelier-scheduler@.service` | Cron daemon |
| 11.4 | Slash-command loader extension in `in_chat_commands.js` + adapter dispatch | Slash trigger |
| 11.5 | RunDispatcher integration (`trigger.source` field, args resolution, outbox delivery, fallback) | Unified execution path |
| 11.6 | `.atelier-pkg` `workflows/` payload subdir handling | Bundle distribution |
| 11.7 | GitOps apply-loop integration (read `workflows/` from atelier-config repo) | Per-tenant lifecycle |
| 11.8 | Audit events + Prometheus metrics + Grafana dashboard panel | Observability |
| 11.9 | E2E: schedule fires + slash dispatches + sanitised export round-trips + closure |

## References

- ADR-0001 — AWP adoption.
- ADR-0005 — AWP standards-only (the boundary this ADR respects:
  workflow YAML is portable AWP; the `triggers:` block is an
  optional AtelierOS plugin overlay that AWP-only consumers
  ignore).
- ADR-0006 — DAG walker (the execution path all three triggers
  share).
- ADR-0007 Phase 2 — RunDispatcher + Gateway REST API (the
  submission funnel).
- ADR-0007 Phase 5 — `.atelier-pkg` (the bundle format extended to
  carry workflows).
- ADR-0007 Phase 6 — Prometheus `/metrics` (the surface where
  workflow throughput becomes visible).
- ADR-0009 — declarative tenant surface (the GitOps loop also
  applies workflow files).
- Layer 19 (bot-disclosure card) — the consent anchor for
  push-into-chat delivery.
