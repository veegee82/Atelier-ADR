# ADR-0036 — Go-to-Market Strategy: AWP Workflow Generation as Killer Feature

**Status:** Accepted
**Date:** 2026-05-17
**Author:** Maintainer
**Depends on:** ADR-0032 (AWPKG format), ADR-0017 (Enterprise Control Plane),
ADR-0007 (multi-tenant), ADR-0011 (workflow plugins)
**Companion to:** `docs/business/open-core-boundary.md`

---

## Context

AtelierOS has a capability that no competing platform offers today: a user
describes a task in natural language and receives a **persistent, serialised,
engine-agnostic AWP DAG** — a workflow that can be versioned, shared, and
installed like a program. This document names that capability, explains why it
is a structural market advantage, and defines the monetisation architecture
that translates it into revenue.

### The competitive landscape — summary

| Category | Representative platforms | Workflow generation | Portability |
|---|---|---|---|
| No-code/low-code | Zapier, Make, n8n | NL assistant → flat sequence, GUI-only | Proprietary JSON; platform-locked |
| Orchestration engines | Airflow, Dagster, Prefect, Temporal | None; developer writes code | Git-portable but infra-dependent |
| LLM agent frameworks | LangGraph, CrewAI, Autogen, Dify, Flowise | Runtime DAG (LangGraph Plan-Execute) — ephemeral | None; DAG disappears after execution |
| Scientific registries | WorkflowHub (RO-Crate) | None | Open format, domain-specific (research) |

**The gap:** No existing system produces a serialised, semantically described,
engine-agnostic DAG that can be installed, shared, and versioned like a
software package. AtelierOS fills this gap.

---

## Decision

### 1. Positioning — "Workflows as Programs"

The single positioning statement:

> *AtelierOS is the first platform where you describe a workflow in natural
> language and receive an installable, portable, auditable program — not a
> screenshot, not a cloud-locked configuration, but an artifact.*

Analogy anchors for marketing copy:

| Reference | The "as X" framing |
|---|---|
| Docker | What Docker did for containers, AWP does for workflows |
| npm | What npm did for JavaScript packages, the AWP registry does for automations |
| GitHub Actions | What Actions did for CI/CD pipelines, AtelierOS does for AI-agent pipelines |

These anchors target the developer and platform-engineering audience that
makes buy/build decisions in target organisations.

---

### 2. Why the NL-generation layer belongs in the open-source core

The natural language → AWP DAG generation is an orchestration layer that
calls a user-supplied LLM API. The code is open source. It does not work
without an LLM API key — that is the natural paywall. This mirrors the
Git/GitHub split:

- **Git** (the tool) is fully open source.
- **GitHub** (the service) earns revenue on hosting, private repos, and
  enterprise features — not by withholding Git.

Keeping generation open is load-bearing for adoption:

1. Developers who self-host with their own API key can generate freely.
2. The format and runtime become the de-facto standard faster.
3. The network effect accrues to the registry, not to the generator code.

**Constraint:** The generator code and AWP format specification remain in
the Apache-2.0 open-core. Commercial value is captured in the layers
described in §3.

---

### 3. Three-tier revenue architecture

```
┌──────────────────────────────────────────────────────────────────┐
│  OPEN CORE  (Apache-2.0)                                         │
│  AWP format spec · Runtime/executor · NL generator · Basic CLI   │
│  Public workflow registry (read) · Self-hosted path · Bridges    │
└───────────────────────────┬──────────────────────────────────────┘
                            │ LLM API key required for generation
                            │ (natural paywall for cloud service)
┌───────────────────────────▼──────────────────────────────────────┐
│  ATELIER PRO  (BSL / FSL — proprietary SaaS)                     │
│  Hosted NL generation (no own API key needed)                    │
│  Private workflow registry + team collaboration                  │
│  Workflow versioning dashboard + diff viewer                     │
│  Webhook triggers + scheduling for hosted workflows              │
│  Priority execution queue                                        │
│  Target: ~€49/month per team                                     │
└───────────────────────────┬──────────────────────────────────────┘
                            │
┌───────────────────────────▼──────────────────────────────────────┐
│  ATELIER ENTERPRISE  (separate repo, commercial license)         │
│  Multi-tenant management + cross-tenant admin dashboard          │
│  EU AI Act Art. 50 / GDPR Art. 30 compliance reports            │
│  Audit-chain attestation (Sigstore, RFC 3161 timestamping)       │
│  SSO (SAML 2.0, Azure AD, Okta)                                  │
│  On-premise deployment + SLA                                     │
│  BYOK encryption at rest                                         │
│  Marketplace storefront (search, billing, payouts, moderation)   │
│  Target: custom contract, regulated industries                   │
└──────────────────────────────────────────────────────────────────┘
```

#### Marketplace revenue model

The `.atelier-pkg` format (ADR-0032) is open. The marketplace storefront is
Enterprise/Cloud. Authors publish free or paid packages; the platform takes a
30 % revenue share on commercial packages. Authors can always self-distribute
outside the marketplace — this prevents vendor lock-in perception and is a
deliberate trust signal.

---

### 4. Target segments — by priority

#### Segment 1 — System integrators (highest near-term revenue)

Companies that build automations for clients (SAP consultants, Salesforce
partners, IT service firms). They currently rebuild the same workflow
structures for every client engagement.

**Pain:** Each delivery is hand-crafted. No reuse story. No "install this
automation package on the next client."

**AWP value:** Build once as an `.atelier-pkg`. Install on each client
tenant. Version and update centrally. Bill per-package or per-tenant.

**Entry motion:** Pro tier for the agency; Enterprise for clients who want
on-premise.

#### Segment 2 — SaaS vendors embedding automation

Software products that want to offer workflow templates to their customers
(ISVs, vertical SaaS). They need a workflow format their customers can
install without leaving the host product.

**Pain:** Today they build custom automation features from scratch or bundle
Zapier/n8n, which means platform lock-in and maintenance overhead.

**AWP value:** Embed the AtelierOS runtime; ship workflow templates as
`.atelier-pkg`. Customers install and customise without leaving the product.

**Entry motion:** OEM/embed licence; Enterprise tier.

#### Segment 3 — Enterprise IT — internal workflow libraries

Large organisations that want to standardise, version, and audit their
internal automation portfolio.

**Pain:** Workflows scattered across Zapier accounts, n8n instances, Python
scripts, and one-off Bash jobs. No central inventory, no audit trail.

**AWP value:** Centralised registry, hash-chained audit log, compliance
reports (GDPR, EU AI Act), role-based access.

**Entry motion:** Enterprise tier; EU AI Act compliance as primary hook for
regulated industries.

#### Segment 4 — Developer community (top-of-funnel)

Engineers who want to automate their own workflows or build on AWP. They
self-host, contribute packages to the public registry, and become the word-
of-mouth vector.

**Entry motion:** Entirely free; open-core self-hosted. Conversion path:
Pro when team size or private registry needs emerge.

---

### 5. Go-to-market phases

#### Phase 1 — Developer adoption (now → ~12 months)

**Goal:** Establish AWP as the de-facto portable workflow format. 500+
packages in the public registry.

Actions:
- Launch `public.awp.dev` — public read/write registry, free for all.
- Open-source the NL generator and AWP runtime under Apache-2.0.
- Publish comparison benchmark: "AWP vs n8n JSON vs LangGraph — portability
  and reuse test" (technical blog post, HN launch).
- Discord community + GitHub Discussions for workflow authors.
- Ship the `atelier install <package>` CLI that makes installation feel like
  `npm install` or `docker pull`.

Success metric: 500 public packages, 1,000 GitHub stars, 50 active
community contributors.

#### Phase 2 — Pro monetisation (6–18 months)

**Goal:** First 50 paying Pro teams. Validate pricing. Establish conversion
funnel from self-hosted → Cloud.

Actions:
- Launch Atelier Cloud (hosted Pro tier, €49/team/month).
- Private registry as the primary conversion driver (the moment a team wants
  to share workflows internally without making them public, they upgrade).
- NL generation via Cloud API — no own API key needed — as the second
  conversion driver.
- Case study: one system integrator, one SaaS-embed, one enterprise IT team.

Success metric: 50 paying Pro teams, 3 published case studies, MRR €2,500+.

#### Phase 3 — Enterprise (12–30 months)

**Goal:** 3–5 anchor Enterprise customers. Activate the Enterprise repo.

Actions:
- Identify 3 prospects from Phase 2 with compliance requirements (banking,
  healthcare, public sector).
- Position EU AI Act Art. 30 + GDPR compliance report bundle as primary hook
  (structural requirement, not a nice-to-have, in force from August 2026).
- Ship on-premise deployment package (Helm chart, ADR-0007 Phase 4).
- Trademark registration (EUIPO + DPMA) to protect the "AtelierOS" and
  "AWP" brand once mindshare is established.

Success metric: 3 signed Enterprise contracts, ARR €150K+.

---

### 6. The network effect moat

The generator code is open. The format is open. The moat is the **registry**.

Every package published to the public registry:
- Makes the platform more useful for the next user (more ready-made workflows).
- Creates a switching cost (workflows reference each other as dependencies).
- Generates semantic data that improves future NL generation quality
  (aggregate schema patterns, not user content).

This is the npm/Docker Hub dynamic: the protocol is open, but the registry
accumulates value that cannot be forked away. A fork of the AtelierOS runtime
cannot take the registry's content.

---

### 7. Licensing alignment

This strategy aligns with `docs/business/open-core-boundary.md`:

- AWP format spec + generator + runtime → Apache-2.0 (open-core).
- Pro hosted service → BSL/FSL (use CLA §3 relicense right when activating).
- Enterprise Edition → proprietary commercial licence (separate repo, already
  in `atelier-enterprise`).
- Marketplace storefront → Enterprise/Cloud bundle.

The Stage-3 optionality (CLA §3 → BSL/FSL for future open-core releases) is
preserved for the event of a hyperscaler clone threat. It is insurance, not
the plan.

---

### 8. Must NOT do

- Don't gate the NL generator behind a paywall — the natural API-key paywall
  is sufficient; an additional code gate destroys developer adoption.
- Don't make the AWP format proprietary — no standard adoption without open
  spec.
- Don't use "AtelierOS Cloud" as the name for the on-premise Enterprise
  product — Cloud and Enterprise are distinct tiers with distinct entry
  motions.
- Don't build the marketplace storefront into the open-core repo — ref
  `open-core-boundary.md` §Enterprise.
- Don't skip trademark registration past Phase 2 end — once mindshare
  establishes the name, unregistered marks are hard to defend.
- Don't auto-save aggregate workflow data that contains user-identifiable
  content — registry analytics must be aggregate + anonymised (ADR-0023
  strict-anonymisation applies).

---

## Consequences

- The public registry (`public.awp.dev`) is a new infrastructure dependency
  requiring its own operational runbook.
- Pro and Enterprise tiers require a billing system (Stripe or equivalent)
  before Phase 2 launch.
- The `atelier install` CLI must be shipped as part of the AWPKG work
  (ADR-0032) before Phase 1 can claim a complete "npm-like" experience.
- EU AI Act Art. 30 compliance reports (ADR-0017 Phase II) become a Phase 3
  sales asset and must be production-ready before Enterprise outreach begins
  (August 2026 enforcement date).
- Trademark filing is a non-code action; it is the Maintainer's responsibility
  and does not block any engineering phase.
