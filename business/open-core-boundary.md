# AtelierOS — Open-Core Boundary

**Status:** Active
**Owner:** Silvio Jurk
**Last reviewed:** 2026-05-26

This document defines the **commercial boundary** of AtelierOS: which
capabilities belong in the Apache-2.0 open-source repository and which
are reserved for paid offerings. Engineers contributing to AtelierOS
consult this list when deciding where new work belongs.

The boundary is a load-bearing strategic asset. Crossing it
accidentally — building an "Enterprise" feature into the open-core
repo because "it's just a small thing" — destroys the commercial
upside without warning. Hence: explicit list, reviewed before merging
anything in the right-hand column.

---

## Three principles

1. **Core = the runtime; Enterprise = operating it at scale.** The
   open-core covers everything a competent operator needs to *run*
   AtelierOS for themselves or their team. Everything that makes
   *running it for thousands of users in a regulated industry*
   easier belongs in the Enterprise tier.

2. **Hosted Cloud is orthogonal.** "AtelierOS Cloud" is a separate
   product — the open-core software, run by the Maintainer, sold as
   convenience. It's not gated by features but by hosting +
   reliability + support. Even pure-OSS users can use Cloud.

3. **Marketplace is platform economics.** Phase 5's `.atelier-pkg`
   format is open. The *marketplace storefront* (search, billing,
   payouts to skill authors, ratings, abuse moderation) is
   Enterprise/Cloud. Authors can always self-distribute packages
   outside the marketplace.

---

## The Boundary

### Open-Core (Apache-2.0, lives in `github.com/.../claudeOS`)

| Capability | Why open |
|---|---|
| AtelierOS Gateway (REST API, runs, webhooks, SSE) | The fundamental runtime |
| Per-tenant audit chain + hash verification | Auditability is a *baseline* trust property, not a premium feature |
| Forge tools + Skill-Forge + sandboxing (bwrap, path-gate) | Without these AtelierOS isn't AtelierOS |
| Voice / WhatsApp / Telegram / Discord / Slack / Email bridges | Adoption-driving surface; can't lock these |
| Cowork personas + routing | Same |
| Dialectical reasoning + LDD instrumentation | Differentiating ideas — they should spread |
| OIDC bearer-token auth (single issuer per tenant) | Baseline auth; no IdP integration goes beyond this in OSS |
| SCIM 2.0 stub (Users only, no Groups, no PATCH-bulk) | Provisioning *primitive* — full IdP integration is Enterprise |
| Single-tenant + multi-tenant axis (ADR-0007 Phase 1) | Multi-tenancy is structural; not a paywall |
| Per-tenant Prometheus metrics endpoint (Phase 6) | Observability is baseline trust |
| `.atelier-pkg` package format + signing (Phase 5) | Distribution standard; can't be proprietary or no one adopts |
| Container + Helm chart (Phase 4) | Deployment artefact, not a feature |
| Roles, consent gate, quota, proposal stack, disclosure card | Multi-user safety primitives — adoption-critical |
| All current `_default`-tenant single-operator features | Self-evident — single-operator path stays fully free |

### Enterprise Edition (separate repo, commercial license, paid)

| Capability | Why commercial |
|---|---|
| **SSO beyond a single OIDC issuer** — SAML 2.0 IdP integration, Azure AD specific connector, Okta-specific connector, federation chains | Enterprise procurement requirement; clear paid tier |
| **Compliance reports** — SOC-2, HIPAA, ISO-27001, GDPR Article-30 mapping; auto-generated quarterly reports from the audit chain | Concrete deliverable enterprises pay for |
| **Multi-region replication** — active/active or active/passive across availability zones, with conflict-free audit-chain merge | High-cost engineering, narrow audience, real ops value |
| **Cross-tenant admin dashboard** — single pane of glass over N tenants; usage, billing-ready exports, anomaly alerts | Operator-of-operators tooling |
| **Aggregated org-metrics** — Prometheus rollups across tenants on a separate endpoint with separate auth (the open-core endpoint stays per-tenant only) | Larger orgs need it; small orgs don't |
| **Marketplace storefront** — search, billing, payouts, ratings, moderation, license enforcement on `.atelier-pkg` distribution | Platform economics; revenue share with skill authors |
| **Premium support tier** — SLAs, dedicated Slack/email channel, 24×7 on-call for incidents, named TAM | Operational service, not a feature |
| **Hot-reload of policy without restart** — for tenants that can't tolerate a Gateway restart on `tenant.atelier.yaml` change | Enterprise-grade operational quality |
| **Custom audit-chain export connectors** — Splunk, Datadog, Elastic, OpenSearch | Integration with existing enterprise observability investments |
| **Audit-chain attestation** — third-party notary integration (Sigstore, RFC 3161 timestamping) for non-repudiation | Regulated industries (banking, healthcare) need this |
| **Bring-your-own-key (BYOK) encryption at rest** — operator-managed KMS for tenant data | Compliance |
| **PII redaction in audit chain** — automatic PII scrubbing with replayable redaction tokens | GDPR / privacy-regulated tenants |
| **Premium personas / Premium skills** — operator-curated + supported sets (e.g. a hardened "finance-compliance" persona) | Productized expertise |

### BSL-1.1 Component — Compute Plugin (`core/compute/`)

The Compute Layer occupies a **third position** that is neither Apache-2.0 open-core
nor Enterprise-repo closed. Its source code is publicly readable and forkable, but
**production use requires a commercial license** (ADR-0031).

**Why BSL-1.1 and not Apache-2.0?**
The compute plugin (out-of-LLM-loop iteration worker + Compute Fabric) is the
strongest commercial differentiator AtelierOS has. Shipping it as Apache-2.0 would
let any operator run unlimited production workloads without a commercial relationship.
BSL-1.1 keeps the source auditable — a hard requirement for security-conscious
enterprise buyers who need to verify what runs on their data — while preserving a
revenue lever. After 4 years each version converts automatically to Apache-2.0.

**Access tiers:**

| Tier | License flag | What is allowed |
|---|---|---|
| **Trial / Free** (no JWT) | — | 500 iterations total · Grid + Random strategies only · 1 concurrent worker · non-production only |
| **Pro / Business** | `compute` | All strategies incl. Bayesian · full worker count · production use |
| **Enterprise** | `compute` + `compute_fabric` | Everything above + parallel workers · sharding · DataSourceAdapter (ADR-0026 Compute Fabric) |

**Enforcement — two layers:**

1. **Legal:** BSL-1.1 `Additional Use Grant` permits non-production use freely.
   Production deployment without a valid license key is a license violation —
   enforceable like any software contract (same model as HashiCorp, MariaDB,
   CockroachDB).

2. **Technical:** An RS256 JWT at `<atelier_home>/global/license/license.jwt`
   (mode 0600, validated locally — no phone-home) gates every `compute_run` call.
   Without a JWT carrying the `compute` flag, the system stays in trial mode
   regardless of what an operator does to the advisory trial counter.
   The Bayesian strategy and the entire Compute Fabric are unreachable without a
   valid JWT — deleting the trial state file does not change this.

**What stays open (Apache-2.0):** The MCP tool *interface* (`compute_run`,
`compute_status`, `compute_result`, `compute_abort`) and its audit schema are
documented in the open-core. The *implementation* (`core/compute/`) is BSL-1.1.

**Contributor rule:** Any new file added to `core/compute/` is automatically
BSL-1.1 by virtue of the directory-level `LICENSE` file. Do not add Apache-2.0
headers to files in that directory.

---

### Hosted Cloud (`atelier.cloud` — separate offering, may run open-core)

The Hosted Cloud product runs the open-core software with operational
guarantees:

- Managed Gateway hosting (no self-deploy)
- Managed Postgres / object storage / backups
- Automatic Tenant provisioning + DNS
- Free / Team / Business / Enterprise pricing tiers
- Enterprise tier on Cloud bundles the Enterprise Edition features above
- Pay-as-you-go billing tied to tenant-id (uses Phase-6 metrics under
  the hood)

Cloud is orthogonal to the open-core boundary — a Cloud customer on
the Free tier gets exactly the open-core. The value is convenience.

---

## When new work shows up — the check

For every new feature/PR, ask three questions:

1. **Does a single-operator deployment need this?**
   - Yes → open-core.
   - No → keep checking.

2. **Is this an extension of an existing open-core feature (more
   integrations, more reports, more replication targets) rather than
   a structurally new capability?**
   - Yes → Enterprise.
   - No → keep checking.

3. **Does this make operating-at-scale-in-regulated-industries
   easier?**
   - Yes → Enterprise.
   - No → open-core by default (we err toward openness).

---

## Drift control

The Open-Core Boundary is fragile under contribution pressure.
External contributors will, in good faith, build features that
"feel like they belong" but actually erode the commercial line.
Three mitigations:

1. **This document is part of the merge checklist.** Reviewers
   consulting `CONTRIBUTING.md` are pointed here before approving
   any feature larger than a bugfix.

2. **Architecture review for non-trivial PRs.** Anything that
   touches auth, IdP, audit aggregation, cross-tenant flow, or
   storage gets explicit boundary review.

3. **No "small" boundary violations.** If a PR introduces a SAML
   helper, an audit-export connector for Splunk, a cross-tenant
   admin dashboard widget — it is rejected with a pointer to this
   document, and the contributor is offered the option to land it
   in the Enterprise repo instead (after a commercial agreement, or
   under contractor terms).

---

## Trademark — the second perimeter

Apache § 6 already denies trademark grants. "AtelierOS" remains the
Maintainer's identifier. Future trademark registration (EUIPO, DPMA
for Germany) protects the *name* even if a fork takes the *code*
under Apache-2.0. A fork is welcome; a fork *marketed as AtelierOS*
is not.

Trademark registration is on the to-do list, not yet executed.

---

## Strategic context

This boundary supports a three-stage trajectory:

| Stage | When | Lever |
|---|---|---|
| **1. Mindshare** | Now → ~18 months | Apache-2.0 open-core; Enterprise repo dormant; Cloud in alpha |
| **2. Revenue** | 12–30 months | Cloud GA; Enterprise repo activated with 3–5 anchor customers |
| **3. Defensive** | 24+ months | If hyperscaler clone threat materialises: use CLA § 3 to relicense future open-core releases under BSL/FSL with 2-year Apache change date |

The Stage-1 commitment to Apache is genuine — Mindshare is the
phase-appropriate strategy. The Stage-3 optionality (CLA § 3) is the
insurance, not the plan.
