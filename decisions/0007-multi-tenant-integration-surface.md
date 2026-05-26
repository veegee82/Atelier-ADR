# ADR-0007 — Multi-Tenant Integration Surface for Third-Party Adoption

**Status:** Proposed
**Date:** 2026-05-10
**Builds on:** ADR-0001 (AWP adoption), ADR-0004 (Compliance zones), ADR-0005 (AWP standards-only), ADR-0006 (DAG walker)

## Context

AtelierOS today is structurally a **single-operator system**:

* `~/.atelier/` (formerly `~/.claudeOS/`) is one user's home tree.
* The bridge whitelist + PIN + roles model assumes one human owner per
  channel.
* Audit chain, quota, consent and disclosure stores are scoped per
  `(channel, chat_key, uid)` — no concept of *organisation* above that.
* Deployment is local `systemd` user units; there is no container image,
  no Helm chart, no documented multi-instance topology.
* The only integration surfaces a third party can use today are the four
  bridges (Discord / Slack / Telegram / WhatsApp) and the email bridge.
  There is no REST / gRPC / webhook API for programmatic embedding.
* Skills, personas and forge-tools are repo-local; sharing them across
  organisations means hand-copying YAML/JSON files.

This is fine for a private operator and for the workshop / family-chat
patterns Layer 16–21 were designed around. It blocks four concrete
classes of third-party demand:

1. **B2B SaaS embedding** — a SaaS vendor wants to expose AtelierOS
   personas inside their own product (per-customer-account). They need
   a programmatic API, not a chat bridge, plus tenant isolation guarantees.
2. **Self-hosted enterprise** — a regulated company wants AtelierOS in
   their own VPC with their own SSO, their own LLM endpoint (covered by
   ADR-0004), and their own audit-log retention policy. They need a
   container image and an OIDC/SAML identity-provider integration.
3. **Skill / persona marketplace** — third parties want to publish
   reusable skills, personas and forge-tools as signed packages others
   can install. Today: no package format, no signing, no registry.
4. **Observability / billing integration** — operators want
   Prometheus/OTel metrics + per-tenant cost/usage data exported to
   their existing tooling, not a hand-rolled audit-log scraper.

The maintainer's framing call (2026-05-10): the previous AWP integration
work (ADRs 0001 → 0006) clarified the **execution-and-orchestration**
axis. ADR-0007 addresses the orthogonal **tenancy-and-integration**
axis — *who* talks to AtelierOS, *how*, and with *what isolation*.

## Decision

AtelierOS adopts a **Gateway architecture** that places a thin
multi-tenant layer in front of the existing Body / Brain / Engines
stack. The Gateway owns tenancy, identity, API surface and packaging;
the existing AtelierOS stack owns persona / skill / tool execution
unchanged.

| Layer | Owner | Responsibility |
|---|---|---|
| **Gateway** (new) | `atelier-gateway` (new module) | Tenant model, OIDC/SAML/SSO, REST + gRPC + webhook API, rate-limits per tenant, per-tenant audit-chain isolation, billing-hooks, metrics-export |
| **Body** | AtelierOS | Bridges, Voice/TTS, Personas, Sandbox, Audit, Layers 1–22 (unchanged) |
| **Standards** | AWP schemas | Workflow / agent identity / capability / 5D budget / R17 (consumed declarative-only, see ADR-0005) |
| **Brain (optional)** | DAG walker (ADR-0006) | Walks AWP YAML through engine layer |
| **Engines** | `WorkerEngine` adapters | Claude Code / Codex CLI / Gemini CLI / Ollama / vLLM (ADR-0001 + ADR-0002) |

### Architectural rule

> **The Gateway is the only third-party-facing surface.** Direct access
> to bridges / adapter / forge / skill-forge / engine-registry from
> outside the operator host is forbidden. Existing private-operator
> deployments (single-tenant, no Gateway) continue to work unchanged
> — the Gateway is **opt-in**, gated on a `gateway.enabled = true`
> flag in `service.env`. Single-tenant operators never pay the
> Gateway's complexity tax.

### What AWP contributes

AWP's standards layer is **already** the right declarative shape for
multi-tenant integration:

* `agent.awp.yaml` — persona descriptors a tenant can publish
* `workflow.awp.yaml` — workflows a tenant can submit
* Capability declarations — what a persona may touch
* R17 output contract — uniform result shape across tenants
* 5D budget — per-tenant cost / wall-clock / token enforcement

The Gateway therefore reuses AWP's schemas as its public input shape.
**No new orchestration spec is invented.** This honours ADR-0005's
"standards only, no runtime" rule: AWP-runtime stays out, AWP-schemas
stay in.

### What is genuinely new

| Component | What it does | Delegated-to |
|---|---|---|
| **Tenant model** | `tenant_id` as first-class scope above `(channel, chat_key, uid)` | Gateway (new) |
| **Identity adapter** | OIDC / SAML / SCIM provisioning, per-tenant IdP | Gateway (new) |
| **REST API** | `POST /v1/tenants/{tid}/runs`, `GET /v1/tenants/{tid}/runs/{run_id}`, webhook callbacks | Gateway (new) |
| **gRPC API** | streaming variant of REST, for high-throughput embedding | Gateway (new, Phase 3+) |
| **Package format** | `.atelier-pkg` — signed bundle of skills + personas + forge-tools | new module |
| **Container image** | OCI image bundling AtelierOS + Gateway + bwrap; documented Helm chart | new build pipeline |
| **Metrics export** | `/metrics` (Prometheus) and OTLP push for traces; per-tenant labels | new module |
| **Billing hooks** | event-stream `tenant.usage.recorded` (tokens / wall-clock / runs) | new module |

## Schema sketch — `tenant.atelier.yaml`

```yaml
apiVersion: atelier/v1
kind: Tenant
metadata:
  id: acme-corp
  display_name: ACME Corporation
  created_at: "2026-05-10T12:00:00Z"
spec:
  identity:
    provider: oidc
    issuer: https://idp.acme.com/
    audience: atelier-acme
    scim_url: https://idp.acme.com/scim/v2/
  data_residency:
    zone: eu-west
    allowed_engines: [azure_openai_eu, vllm_eu_west]
    forbid_engines: [claude_code, gpt4_us]
  budget:
    max_runs_per_day: 5000
    max_tokens_per_day: 10_000_000
    max_wall_clock_per_run_s: 300
  packages:
    installed:
      - acme/customer-support-persona@1.4.2
      - acme/internal-docs-skills@2.0.0
  audit:
    retention_days: 365
    export:
      kind: s3
      bucket: acme-atelier-audit
      kms_key: arn:aws:kms:eu-west-1:...
```

The schema reuses AWP's 5D-budget shape and ADR-0004's `compliance_zones`.
Nothing is invented that AWP or earlier ADRs already standardised.

## API sketch — `POST /v1/tenants/{tid}/runs`

```http
POST /v1/tenants/acme-corp/runs HTTP/1.1
Authorization: Bearer <jwt-from-tenant-idp>
Content-Type: application/yaml

apiVersion: atelier/v1
kind: Run
spec:
  persona: customer-support
  input: "Refund request for order #4711"
  webhook:
    url: https://acme.com/atelier/callback
    secret_ref: webhook-secret-1
  budget_override:
    max_wall_clock_s: 60
```

Response is a `run_id`. The webhook fires on `run.completed` /
`run.failed` / `run.budget_exceeded`. Streaming variant via
`GET /v1/tenants/{tid}/runs/{run_id}/events` (SSE) or gRPC.

## Phasing

| Phase | Scope | E2E gate |
|---|---|---|
| **0** | This ADR | doc review |
| **1** | Tenant model + per-tenant scope-root (`<atelier_home>/tenants/<tid>/`); existing single-tenant path = `tenants/_default/`; audit-chain split per tenant | E2E: two tenants run concurrently, audit chains verifiable independently |
| **2** | REST API (FastAPI), bearer-token auth (static keys first), webhook dispatch | E2E: `curl POST /v1/.../runs` → run completes → webhook hits a stub server |
| **3** | OIDC adapter + SCIM provisioning; `tenant.atelier.yaml` loader; per-tenant `engine_policy.json` (reuses ADR-0004) | E2E: Keycloak in CI, two tenants with disjoint engine allowlists |
| **4** | Container image (OCI) + minimal Helm chart; documented `bwrap` requirements for K8s (privileged-init or user-namespace) | E2E: kind cluster boots image, smoke test runs |
| **5** | `.atelier-pkg` format (sigstore-signed); CLI `atelier package install <pkg>`; persona/skill/tool registry shape | E2E: round-trip publish → install → run on second host |
| **6** | Prometheus `/metrics` + OTLP export; per-tenant labels; billing event-stream | E2E: scrape `/metrics`, assert per-tenant counters |
| **7** | gRPC API parity for streaming use cases; rate-limit per tenant; quota integration with Layer 20 | E2E: gRPC client streams 100 concurrent runs, quotas enforced |

Hard rule (carried from ADR-0001): **every phase ships per-subtask E2E
hitting real subprocesses**. Phase 4 (containers) and Phase 5 (packages)
specifically need real bwrap and real cosign in CI, not mocks.

## Non-Goals

* **Replacing the bridges.** Discord / Slack / Telegram / WhatsApp / Email
  bridges keep working unchanged. The Gateway is an *additional*
  surface, not a replacement. Single-operator deployments may run
  bridges only and never enable the Gateway.
* **Re-architecting the security envelope.** Layers 1–22 stay
  unchanged. The Gateway adds a tenant scope *above* them; it does not
  weaken or duplicate any existing gate.
* **Inventing a new orchestration protocol.** AWP schemas are the
  declarative input shape (consistent with ADR-0005). The Gateway
  validates against AWP, never replaces it.
* **Hosting AtelierOS-as-a-Service ourselves.** The Gateway is a
  capability for *operators* to host multi-tenant. Whether the
  maintainer runs a public SaaS is a separate business decision; this
  ADR enables it but doesn't commit to it.
* **Generic "anyone can call any tool" API.** The Gateway exposes
  *personas* as the unit of delegation. A tenant invokes a persona
  with input; tool calls happen *inside* the existing sandbox. No
  raw `POST /v1/tools/{name}` surface.

## Risks

| Risk | Mitigation |
|---|---|
| Gateway becomes a god-layer that re-implements bridge logic | Keep Gateway thin: tenant + auth + API + dispatch. Anything above that goes back into the existing layers. |
| Tenant isolation looks watertight but isn't (shared bwrap, shared MCP server, shared audit chain) | Per-tenant scope-root in Phase 1 is the structural cut. Per-subtask E2E in Phase 1 specifically asserts cross-tenant audit isolation. |
| Container image makes bwrap unusable (K8s sec policies block user-namespaces) | Phase 4 documents the `bwrap` requirement clearly; provides a sidecar pattern as fallback for restrictive clusters. |
| Package signing rolls out broken (operators install unsigned) | Sigstore + transparency log; CLI refuses install without verifiable signature unless `--unsafe-unsigned` is passed. |
| OIDC adapter fragments per IdP | Use a battle-tested library (Authlib / oauth2-proxy pattern); add per-IdP integration tests as community demand surfaces. |
| Webhook delivery becomes a reliability liability | At-least-once delivery with operator-configurable retry; webhook signing with shared secret (HMAC-SHA256). |
| Maintenance burden of N tenants × M engines × P packages | Per-tenant resource caps from day 1; soft-default conservative; metrics surface tenant cost so operator can spot runaway consumers. |

## Consequences

### Positive

* AtelierOS gains a **B2B integration story** without forking the
  single-operator path. Private operators see no change unless they
  flip the Gateway flag.
* Reuses everything the AWP-integration ADRs already standardised
  (engine policy, compliance zones, R17, 5D budget, schemas). No
  parallel orchestration concept.
* Per-tenant scope-root composes cleanly with the existing four-scope
  model (task / session / project / user). Tenants become a fifth
  scope above user; the resolver chain extends naturally.
* Marketplace + container image close the *adoption* gap that the
  current "clone the repo" install pattern leaves. Companies cannot
  evaluate AtelierOS in 30 minutes today; with Phase 4 they can.
* Observability + billing hooks turn AtelierOS from "operator-managed
  black box" into a system that fits existing FinOps / SRE practices.

### Negative

* Significant new surface: tenant model, REST API, OIDC, container
  pipeline, package format. Total ~12–16 weeks of focused work spread
  across 7 phases. Cannot ship in one go.
* Test matrix grows: per-tenant tests, per-IdP tests, per-cluster
  tests. Mitigated by phasing — each phase ships its own E2E and
  doesn't block on the next.
* The maintainer's mental load doubles: from "single-operator system"
  to "single-operator system + multi-tenant platform". The opt-in flag
  is the structural mitigation; documentation has to make the two
  modes clearly distinguishable.
* Phase 1 (per-tenant scope-root) is invasive: every path resolver
  (`paths.py`, `scope.py`, audit-chain writer) gets a tenant axis.
  This lands during the AtelierOS rebrand (Phase 2 of CLAUDE.md
  rebrand roadmap is currently in progress) — coordination with the
  rebrand is mandatory to avoid a churn collision.

## Sequencing relative to the rebrand

The AtelierOS rebrand (CLAUDE.md "Project rebrand" section) is at
**Phase 2 → Phase 3** transition. ADR-0007's Phase 1 (per-tenant
scope-root) MUST land **after** the rebrand reaches Phase 4 (on-disk
migration helper) — otherwise the migration helper would have to
juggle both *new path* and *new tenant axis* simultaneously, and that
combination is the "half-rename half-tenancy" failure mode the rebrand
warnings call out.

Order of operations:

1. Rebrand Phase 2 → Phase 4 complete (on-disk migration helper green).
2. ADR-0007 Phase 1 (tenant scope-root) lands; default tenant
   `_default` covers existing single-operator paths.
3. ADR-0007 Phase 2+ proceed in parallel with rebrand Phase 5+.

## How this connects to LDD

Each phase is one LDD outer-loop step. Phase 1 specifically converges
when:

```
loss(phase1) = 1 - (cross_tenant_audit_isolation_e2e_pass_rate
                    * single_tenant_legacy_path_pass_rate)
```

reaches 0 — both invariants hold simultaneously. The single-tenant
legacy path E2E is the regression gate that protects existing
operators from the multi-tenant refactor's blast radius.

## References

* ADR-0001 — AWP as Orchestration Layer
* ADR-0004 — Compliance Zones + Engine Policy (re-used by per-tenant policy)
* ADR-0005 — AWP standards-only (re-affirmed: schemas yes, runtime no)
* ADR-0006 — Standards-Only DAG Walker
* CLAUDE.md "Project rebrand" — sequencing constraint
* AWP Spec — declarative schemas the Gateway speaks
* `docs/for-organizations.md` § 7.4 — enterprise-adoption threshold (extended by this ADR)
