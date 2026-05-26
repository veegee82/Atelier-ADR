# ADR-0009 — Declarative Tenant Surface

**Status:** Proposed
**Date:** 2026-05-11
**Companion to:** ADR-0007 (multi-tenant axis), ADR-0008 (state out of repo)
**Implements:** none yet (this ADR is the design contract; phased
implementation plan lands as `0009-implementation-plan.md` after sign-off)

## Context

Today an operator configures a tenant by SSH-ing onto the host and
editing JSON / YAML files in place:

- `<tenant>/cowork/personas/<name>.json` — agent role
- `<tenant>/global/tenant.atelier.yaml` — policy
- `<atelier_home>/bridges/<channel>/settings.json` — channel + whitelist
- (hardcoded) slash-command list in `bridges/shared/js/in_chat_commands.js`

Three structural problems surface as soon as the deployment grows beyond
a single hand-edited host:

1. **No audit trail for configuration drift.** A compliance officer
   reviewing the audit chain sees every runtime event but cannot tell
   when a policy was loosened, by whom, or with what justification.
   `git blame` on the operator's host doesn't help — the production
   files were edited locally.

2. **Boilerplate explosion at scale.** A customer running 8 support
   personas (language × region) duplicates ~80 % of every persona JSON.
   A change to the shared "always disclose retention period" instruction
   requires 8 edits.

3. **Slash-command surface is locked.** A bank that wants `/ticket <id>`
   as a one-liner shortcut to its ticketing persona needs a daemon-side
   code patch today. The same daemon update cadence covers every other
   bridge channel — the change ripples through review-cycles that have
   nothing to do with the customer's wish.

The result: operators either (a) treat the production host as a
sacred-text artefact and resist all change, or (b) treat it as
disposable and rebuild from a script — at which point the script
*is* the declarative configuration, just informally.

ADR-0007 (multi-tenant) already isolates per-tenant configuration into
`<atelier_home>/tenants/<tid>/`. What's missing is the **declarative
input surface** — a sanctioned way for a customer to express "this is
my tenant's intended state" outside the production host.

## Decision

Three orthogonal but co-designed mechanisms that share one pattern:
*the customer drops a file, the runtime reads it, the audit chain
records the change*.

### A) GitOps apply-loop

A tenant declares a Git source in its `tenant.atelier.yaml`:

```yaml
apiVersion: atelier/v1
kind: Tenant
metadata:
  id: acme
spec:
  gitops:
    repo: git@github.com:acme/atelier-config.git
    ref: main
    paths: [personas/, slash-commands/, branding.yaml]
    apply_interval_s: 300
    drift_policy: fail_closed   # fail_closed | overwrite | warn
```

A per-tenant systemd timer `atelier-gitops-apply@<tid>.timer` runs
`atelier_gateway.cli tenant apply <tid>` every `apply_interval_s`. The
apply command:

1. `git pull` (or `git clone` on first run) into a per-tenant cache
   at `<atelier_home>/tenants/<tid>/global/gitops/cache/`.
2. Resolves the `paths` allowlist against the cache.
3. Diffs the cache against the live tenant directories (`cowork/personas/`,
   `cowork/slash-commands/`, etc.).
4. Applies missing-or-changed files atomically.
5. Emits `gitops.applied` per applied file (commit-sha + path) and one
   summary `gitops.cycle_completed` (counts + duration).

**Drift handling.** If the live file's mtime is newer than the apply
record AND the content differs from cache:

- `drift_policy: fail_closed` (default) — abort the cycle, emit
  `gitops.drift_detected` with file-path + diff-stat; operator must
  reconcile via `tenant apply --reconcile` (drift → cache as PR) or
  `--force-overwrite`.
- `drift_policy: warn` — log + apply anyway.
- `drift_policy: overwrite` — apply silently (for fully Git-managed
  shops).

**Authentication.** The repo URL may carry a `secret_ref:` for an SSH
deploy-key in the tenant's secret vault. HTTPS + GitHub-App-token is
supported via the same `secret_ref`.

### B) Custom slash-commands as config

Tenant drops `<tenant>/cowork/slash-commands/<name>.yaml`:

```yaml
trigger: /ticket
aliases: [/tkt, /case]
description: Look up a support ticket by ID
persona: support-tier2          # which persona handles the call
prompt_prefix: |
  Look up ticket and summarise status:
visibility: owner               # owner | admin | member | all
audit_event: tenant.slashcmd_invoked
```

The bridge dispatcher (`bridges/shared/js/in_chat_commands.js`) reads
`<tenant>/cowork/slash-commands/*.yaml` at boot AND on `inotify`-style
mtime check per inbound message (mirrors the existing `settings.json`
hot-reload pattern). Hardcoded slash-commands (`/help`, `/cancel`,
`/btw`, `/voice-*`, `/consent`, `/role`, `/audit`, `/quota`, …) always
win on collision — custom commands can extend, never override.

A custom command resolves to: pin the named persona for this single
turn → prepend `prompt_prefix` to the user's text → run the bridge
turn as normal. Quota, role, audit gates all fire unchanged.

### C) Persona inheritance

Persona JSON gains an `extends:` field:

```jsonc
// base-support.json
{
  "name": "base-support",
  "permission_mode": "default",
  "allowed_tools": ["Read", "Grep", "mcp__crm__search"],
  "append_system": "Respond in the customer's language. Cite policy IDs verbatim.",
  "ldd_preset": "quick"
}

// support-de.json
{
  "name": "support-de",
  "extends": "base-support",
  "description": "First-line support in German",
  "append_system": "Antworte auf Deutsch, formell."
}
```

Resolver does deep-merge after base-load:

- Lists (`allowed_tools`, `routing_anchors`, `add_dirs`) — **union**.
- Dicts (`mcp_servers`) — **shallow merge** per top-level key.
- Scalars (`description`, `model`, `permission_mode`) — **override**.
- `append_system` — **concatenate** with newline separator (child
  appended after base).

Inheritance chain depth-capped at 4. Cycle detection at load time →
`persona.cycle_detected` audit event + fail-closed load.

The resolved persona is observable via `/whoami --resolved` (a new
flag on the existing slash-command) — operator sees the merged JSON
plus the inheritance chain in the response.

## Consequences

### Positive

- **Audit chain gains a config-layer.** A regulator reviewing
  `audit.jsonl` now sees both runtime events AND when configuration
  was applied, with the source-of-truth commit-sha. Combined with
  the customer's Git history, the full "what changed when by whom"
  question is answerable from two append-only sources.
- **80 % persona boilerplate disappears.** A customer running
  language × region matrices writes one base per language, one
  override per region.
- **Customer-specific slash-commands become a 5-line YAML.** No PR,
  no review cycle, no daemon redeploy.
- **AtelierOS becomes Kubernetes-shaped for deployment.** Customer's
  Git repo + a tenant-config-as-code workflow is the *same* pattern
  every infrastructure team already runs for k8s manifests.
- **All three mechanisms reuse existing infrastructure** — hot-reload
  (settings.json pattern), audit chain (forge.security_events.write_event),
  secret vault (atlr_ + OIDC tokens), per-tenant directory tree
  (ADR-0007). No new discipline.

### Negative

- **Git-credential management per tenant.** Deploy-keys or GitHub-
  App-tokens must land in the secret vault. The trust boundary is
  now "whoever can push to the customer's config repo can rewrite
  tenant policy" — operators must understand this and choose
  protected-branch + required-review accordingly.
- **GitOps cycle can collide with manual edits.** The drift-policy
  field is the explicit operator choice; default fail-closed catches
  silent divergence at the cost of one alert per drift.
- **Inheritance increases debug complexity.** "Where does this value
  come from?" gets harder. Mitigated by `/whoami --resolved` showing
  the chain.
- **Slash-command surface gains a per-tenant differential.** A user
  moving between tenants sees different commands. Mitigated by the
  `/help` output which already separates "bridge commands" from
  "tenant commands".

### Risks deliberately accepted

- Customer's Git repo becomes a privileged source. If their repo is
  compromised, tenant policy is compromised. This is the same trust
  model as Kubernetes + ArgoCD + customer's manifests; the AtelierOS
  contribution is making it possible, not making the customer's
  Git-security stronger.
- A misconfigured `apply_interval_s: 60` plus a noisy CI loop on the
  config repo could thrash the apply-loop. Mitigated by `min_interval`
  clamp at 60 s and a per-tenant rate-limit (5 cycles / min).

## What this ADR deliberately does NOT include

- **Visual diff / preview UI for the GitOps cycle.** Operator-side
  tooling; CLI-only here. A `tenant apply --dry-run` flag covers the
  preview need from the terminal.
- **Cross-tenant config sharing or a marketplace.** That's an own ADR
  (Marketplace). This one stays within a tenant.
- **Inheritance in scopes other than personas** (e.g. policy-extends-
  policy, slash-command-extends-slash-command). Personas are where
  the boilerplate is. Other extensions can be revisited if the
  pattern proves out.
- **Hooks / lifecycle event bus.** Originally considered as a third
  ADR sibling; deliberately deferred until a real customer use case
  surfaces, because reactive event bus would risk doubling AWP's
  workflow-orchestration scope (ADR-0005).

## Implementation surface (sketch — full plan in 0009-implementation-plan.md)

| Sub-phase | Module | Scope |
|---|---|---|
| 9.1 | `atelier_gateway/gitops.py` | apply loop + drift detection + CLI |
| 9.2 | `atelier_gateway/slashcmd_loader.py` + JS dispatcher patch | custom slash-commands |
| 9.3 | `bridges/shared/personas/inheritance.py` + resolver integration | persona `extends:` |
| 9.4 | systemd template `atelier-gitops-apply@.timer` + bridge.sh wiring | scheduler |
| 9.5 | Closure: docs, audit-event registration in `EVENT_SEVERITY`, E2E |

## References

- ADR-0007 — multi-tenant axis (the per-tenant directory tree this
  ADR populates).
- ADR-0008 — state out of repo (the `<atelier_home>` location all
  three mechanisms write into).
- ADR-0005 — AWP standards-only (the boundary this ADR respects:
  config = peripheral, workflow = AWP, execution = engine).
