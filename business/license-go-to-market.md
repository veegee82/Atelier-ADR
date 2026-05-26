# AtelierOS — License-First Go-to-Market Strategy

**Status:** Strategic reference — license-only distribution model
**Owner:** Silvio Jurk
**Last reviewed:** 2026-05-26

This document defines AtelierOS's go-to-market strategy under a **pure licensing model**: no
managed hosting, no SaaS operation, no customer infrastructure to maintain. Revenue comes
exclusively from commercial licenses. The open-source core remains Apache-2.0 and free forever.

---

## 1. Core Principle: Zero Hosting, Pure License Margin

AtelierOS is the **platform**. Others operate it. We collect the license fee.

This is the Red Hat model applied to AI compliance infrastructure:
- Red Hat does not run Linux for you — it licenses you the right to run enterprise Linux
  with support, certification, and indemnification.
- AtelierOS does not run your AI agents — it licenses you the right to run a
  structurally EU-AI-Act-compliant agent platform, with commercial guarantees.

**Why this works for AtelierOS specifically:**

1. The open-source core is the adoption driver — free, no lock-in, developer-first.
2. Compliance is legally mandatory, not optional — regulated buyers *must* pay for the
   certification and support that comes with a commercial license.
3. CLA §3 (Relicense Right) gives the Maintainer leverage without forking the code —
   the same repository powers both the Apache-2.0 community and commercial licensees.
4. The EU AI Act enforcement window (August 2026) creates buyer urgency that no marketing
   campaign can manufacture — it is structurally external.

---

## 2. License Tiers

### Tier 0 — Open Source (Apache-2.0, free forever)

**Who:** Individual developers, researchers, open-source projects, evaluation

**Includes:**
- Full AtelierOS runtime (all 38 layers)
- All bridges (Discord, WhatsApp, Telegram, Slack, Email, Signal, Web)
- Forge + SkillForge + audit chain + consent gate + GDPR tools
- Docker image + Helm chart + setup wizard
- Community support (GitHub Issues, Discussions)

**Does NOT include:**
- Commercial support SLA
- Compliance report generation (SOC-2, EU AI Act documentary proof)
- SSO beyond single OIDC issuer
- Cross-tenant admin dashboard
- Compute Fabric (BSL-1.1, trial-only in OSS)
- Trademark license ("Powered by AtelierOS" badge)

**Goal:** Maximize adoption. Every developer running AtelierOS free is a future enterprise
reference customer or a future OEM licensee's end user.

---

### Tier 1 — Professional License (€2,400/year per instance)

**Who:** SMB teams of 5–50 people; internal deployments; dev agencies; compliance-aware
startups who self-host

**Trigger for upgrade from OSS:**
- Need for commercial support SLA (response within 4 h)
- Need for automated compliance reports for client deliverables
- Need for "Powered by AtelierOS" trademark usage in commercial product
- Need for full Compute Worker (Bayesian strategy, production use)

**Includes everything in Tier 0, plus:**
- Commercial support SLA (4 h response / next business day resolution)
- Automated compliance report generation (quarterly, EU AI Act + GDPR Art. 30)
- Full Compute Worker license (`compute` JWT flag)
- Trademark license: "Powered by AtelierOS" badge for customer-facing deployments
- Named legal indemnification contact (for regulatory inquiries)
- Access to security advisories 30 days before public disclosure

**Revenue model:** Annual subscription, invoiced, no metering. One license = one deployment
(any number of tenants on that deployment).

---

### Tier 2 — Enterprise License (€15,000–€80,000/year, negotiated)

**Who:** Organizations ≥50 employees or ≥€5 M revenue; regulated industries (healthcare,
finance, legal, public sector); any buyer needing formal compliance certification

**Trigger for upgrade from Pro:**
- Legal requirement for third-party audited compliance documentation
- Need for SSO (SAML 2.0, Azure AD, Okta)
- Multi-region or multi-tenant deployment at scale
- Named account manager + dedicated Slack channel
- Custom audit-chain export to SIEM (Splunk, Elastic, Datadog)
- BSI-Grundschutz or ISO-27001 procurement requirement

**Includes everything in Tier 1, plus:**
- Enterprise Edition repo access (SSO connectors, SIEM sink, cross-tenant dashboard)
- Full Compute Fabric license (`compute_fabric` JWT flag)
- Multi-region replication (active/active or active/passive)
- Custom audit-chain export connectors (Splunk HEC, Elastic Logstash, Datadog Agent)
- Dedicated named account manager
- 24×7 on-call for P1 incidents
- Audit-chain attestation via RFC 3161 TSA (L37)
- BYOK encryption at rest (operator-managed KMS)
- PII redaction in audit chain (automatic, replayable)
- Priority patch SLA (critical CVEs within 24 h)
- Logo on "Powered by AtelierOS" reference page
- Compliance report signed with Maintainer's qualified electronic signature (eIDAS)

**Pricing bands:**
| Deployment size | Annual fee |
|---|---|
| ≤500 active users | €15,000 |
| 501–2,000 active users | €35,000 |
| 2,001–10,000 active users | €60,000 |
| >10,000 active users | €80,000 + negotiated |

---

### Tier 3 — OEM / White-Label License (€50,000–€500,000/year, negotiated)

**Who:** Cloud providers, managed-service providers, system integrators who embed AtelierOS
into their own product and resell access to their customers

**Includes everything in Tier 2, plus:**
- Full white-label rights (remove "AtelierOS" branding entirely, or "Powered by" attribution)
- Source-code audit rights (read access to Enterprise repo for security review)
- Joint go-to-market (reference story, press release, co-marketing)
- Dedicated engineering liaison for integration questions
- Early access to new layers and features (30-day preview)
- Custom `allowed_engines` and `data_residency` defaults pre-configured for the OEM's
  cloud infrastructure
- SLA: 99.9% uptime guarantee on the license server (JWT validation endpoint)

**Revenue model options:**
| Structure | Best fit |
|---|---|
| Flat annual fee | Small/mid EU sovereign cloud with predictable tenant count |
| Revenue share (3–8 % of generated AI platform MRR) | Large provider with proven distribution |
| Per-active-tenant (€X/month reported monthly) | Managed-service providers with variable volume |

---

## 3. Buyer Personas

### Persona A — The Compliance-Driven CTO (Enterprise Direct)

**Profile:** CTO or VP Engineering at a 100–1000 person company in a regulated industry
(healthcare, legal, insurance, public sector). Has received a board mandate to demonstrate
EU AI Act compliance before August 2026. Does not want to build; wants to buy a solution
that passes a regulatory audit.

**Pain:** "Our legal team says we need documented AI decision trails. Our vendor (OpenAI/
Anthropic/Microsoft) doesn't produce these. We need something auditable."

**AtelierOS pitch:** "Your audit trail is cryptographically hash-chained and tamper-evident
from the first message. Compliance reports in the format regulators expect are generated
automatically. No retroactive bolt-on — this is structural."

**Sales cycle:** 4–12 weeks. Procurement-heavy. Needs: security questionnaire, DPA, pentest
report, named reference customer in same industry.

**Decision trigger:** Receiving a first regulatory inquiry OR a competitor announcing a
compliant AI deployment.

---

### Persona B — The Sovereign-Cloud Product Manager (OEM Channel)

**Profile:** Product manager or VP Product at a European cloud provider (Hetzner, IONOS,
OVHcloud, Scaleway). Has a mandate to add an "AI" product to the portfolio before a
competitor does. Does not have the engineering resources to build a compliance-first AI
agent stack in time.

**Pain:** "We can offer GPU compute, but our customers are asking for a managed AI agent
platform. We have 6 months and 3 engineers."

**AtelierOS pitch:** "You don't need 6 months and 3 engineers. License our stack, put your
brand on it, and ship next month. We handle the compliance story. You handle the customer."

**Sales cycle:** 6–16 weeks (procurement + technical integration). Integration effort is
documented and scoped to 2–4 weeks of the provider's engineering time.

**Decision trigger:** A competitor offering an AI platform, OR the first enterprise customer
asking for an EU AI Act-compliant offering the provider cannot deliver.

---

### Persona C — The Self-Hosting Engineering Lead (Pro License)

**Profile:** Engineering lead at a 10–80 person dev agency, SaaS startup, or internal IT
department. Already running AtelierOS on free tier; team is using it daily. Needs commercial
support for a customer-facing deployment.

**Pain:** "We're building a product on top of AtelierOS for our client. We need a support
SLA to put in our contract and a compliance report to give the client's legal team."

**AtelierOS pitch:** "Pro license gives you the support SLA, the compliance report, and the
right to ship 'Powered by AtelierOS' to your client. One invoice, all done."

**Sales cycle:** 1–3 weeks. Often self-serve or low-touch email. Triggered by first
client contract requirement.

---

## 4. Three Go-to-Market Motions

### Motion 1 — Community-Led (OSS → Pro conversion)

**Channel:** GitHub, developer communities, Hacker News, developer conferences

**Funnel:**
1. Developer discovers AtelierOS via GitHub, a blog post, or word-of-mouth
2. Runs the free tier — self-hosted, zero friction
3. Builds something real (internal tool, client project, prototype)
4. Hits a commercial requirement (client contract, support SLA, compliance report)
5. Purchases Pro license (self-serve via invoice, no sales call required)

**Key assets:**
- GitHub repository with exceptional documentation and setup experience
- "Verified by AtelierOS" badge as the visible upgrade trigger
- Compliance report generator: one click in the console, shows "requires Pro license"
- Blog: technical deep-dives on EU AI Act compliance that rank on Google

**Target metric:** GitHub Stars → Weekly Active Instances → Pro conversion rate (goal: 2%)

**Cost:** Near-zero. Engineering time on documentation and developer experience is the investment.

---

### Motion 2 — Enterprise Direct Sales

**Channel:** LinkedIn outreach, conference presence (EU AI Act / GDPR / RegTech events),
referrals from Tier-1 law firms and compliance consultancies

**Funnel:**
1. Enterprise learns about AtelierOS via content marketing or referral
2. Requests a technical demo (live audit-chain demonstration, compliance report preview)
3. Security questionnaire + DPA review (2–4 weeks)
4. Pilot deployment on their infrastructure (4-week trial on Enterprise license terms)
5. Annual contract signed

**Key assets:**
- Live demo environment (own Hetzner reference deployment)
- Pentest report (SySS or Cure53) — non-negotiable for procurement
- ISO-27001 or BSI-Grundschutz evidence (Q3 2026 target)
- Reference customer in at least one regulated vertical before first enterprise close
- One-pager: "What your legal team needs to see" (maps AtelierOS layers to EU AI Act articles)

**Target: 3 enterprise contracts by end of 2026**

Sales owner: Silvio Jurk (direct) until first €100k ARR, then first enterprise AE hire.

---

### Motion 3 — OEM Channel Sales

**Channel:** Direct outreach to EU cloud provider product teams; introductions via
investor network and Deep Tech / Silicon-Allee events

**Funnel:**
1. Identify 8–12 EU cloud providers with a missing AI compliance product
2. Direct outreach: product or VP-product level (not engineering, not marketing)
3. Technical alignment call (30 min): show the integration path, the compliance story,
   the OEM license terms
4. Letter of Intent within 4 weeks (urgency framing: August 2026 enforcement)
5. Integration sprint (2–4 weeks engineering effort on licensee side)
6. Public launch of their AI platform "Powered by AtelierOS"

**Urgency script:**
> "EU AI Act High-Risk enforcement begins August 2026 — nine weeks from today.
> Your customers will ask you for a compliant AI platform. Your competitors are already
> shopping for one. You have the customers. We have the only ready stack. The integration
> takes your team two weeks. Let's talk."

**Target: 1 signed OEM LOI before July 2026**

A single OEM deal covers more than a year of infrastructure cost and validates the model.

---

## 5. Messaging Architecture

### Master message (all audiences)

> "AtelierOS is the only AI agent platform where EU AI Act compliance is architecturally
> impossible to bypass — not a setting you can forget to turn on."

### By audience

**Developers (OSS):**
> "Run a full multi-channel AI agent OS on your own infrastructure. Apache-2.0.
> No API key required. GDPR-ready out of the box."

**Pro buyers (SMB/Agency):**
> "Everything your client's legal team needs — compliance reports, audit trail,
> support SLA — in one annual license. Ship AI products without the compliance headache."

**Enterprise buyers (CTO/CISO):**
> "The audit chain is hash-verified from message one. Compliance reports are
> auto-generated on demand. Your regulators will ask for proof — this is the proof."

**OEM buyers (Cloud Provider):**
> "Your customers need EU AI Act compliance. You need an AI product.
> License our stack, put your name on it, ship next month."

---

## 6. Revenue Model and Targets

### Year 1 (2026): Proof of Commercial Model

| Revenue source | Target | Notes |
|---|---|---|
| Pro licenses | 10 × €2,400 = €24,000 | Self-serve, community-driven |
| Enterprise licenses | 3 × avg. €25,000 = €75,000 | Direct sales, 4–12 week cycles |
| OEM licenses | 1 × €100,000 = €100,000 | One anchor deal; LOI before July 2026 |
| **Total Year 1** | **~€200,000 ARR** | |

### Year 2 (2027): Scale the OEM channel

| Revenue source | Target |
|---|---|
| Pro licenses | 40 × €2,400 = €96,000 |
| Enterprise licenses | 10 × avg. €35,000 = €350,000 |
| OEM licenses | 3 × avg. €150,000 = €450,000 |
| **Total Year 2** | **~€900,000 ARR** |

**Cost structure (license-only model):**
- No hosting costs (licensees operate their own infrastructure)
- No support engineering headcount until Year 2 (founder-led support in Year 1)
- Core costs: legal (DPA, license templates), security audit (pentest), conference presence
- Estimated Year 1 operating cost: €25,000–€40,000

**Gross margin: ~85–90%** — the license-only model yields software-level margins.

---

## 7. The August 2026 Window

EU AI Act High-Risk enforcement begins **August 2026**. This is not a projected event —
it is a legal deadline that is already law.

Every regulated organisation in the EU that deploys an AI system affecting hiring, credit,
healthcare, law enforcement, or critical infrastructure must be able to demonstrate:
- Documented decision trails (Art. 14)
- Disclosure to users (Art. 50)
- Consent management (GDPR Art. 6/7)
- Data residency controls (Art. 14)

AtelierOS delivers all four architecturally. Competitors will bolt compliance on as a module
after the deadline has already passed. This creates a **12–18 month competitive window**
where AtelierOS is the only credible option for regulated buyers who need to move now.

**The window is not permanent.** After 2027, well-funded competitors will have invested in
compliance. The OEM and Enterprise deals that close in 2026 are the ones that compound —
licensees build their products on AtelierOS, creating switching costs that outlast the window.

---

## 8. Launch Sequencing

### Phase 1 — License Infrastructure (now → June 2026)

- [ ] Publish Pro and Enterprise license terms (plain-language PDF, reviewed by counsel)
- [ ] Set up license purchase flow (invoice-based; no SaaS billing required yet)
- [ ] Finalize DPA template for Enterprise buyers
- [ ] Publish SECURITY.md and responsible disclosure policy
- [ ] Commission pentest (SySS or Cure53) — results needed for Enterprise procurement

### Phase 2 — OEM Outreach (June 2026)

- [ ] Identify 10 EU cloud provider contacts (VP Product or Head of AI Product)
- [ ] Prepare OEM one-pager: "Two weeks to your EU AI Act-compliant AI platform"
- [ ] Send first 5 outreach messages with specific urgency framing (August 2026)
- [ ] Present at Deep Tech Night @ HHI (23.06.2026) — OEM pitch to investors who know EU cloud VCs

### Phase 3 — Enterprise Pipeline (July–September 2026)

- [ ] First reference customer in regulated vertical (healthcare or legal preferred)
- [ ] Compliance report PDF demo ready for prospect review
- [ ] "What your legal team needs to see" one-pager published
- [ ] BSI-Grundschutz first contact (Q3 2026)

### Phase 4 — Scale (Q4 2026)

- [ ] First OEM deal signed and deployed (target: one provider live)
- [ ] Pro license self-serve page live on atelierOS.dev
- [ ] First enterprise contract signed
- [ ] ARR report: is the model working?

---

## 9. What This Model Does NOT Require

Explicitly: none of the following is needed to execute this strategy.

- **No managed hosting.** Licensees self-host. The reference Hetzner instance is a demo
  and proof-of-life, not customer infrastructure.
- **No per-seat metering infrastructure.** Pro and Enterprise are flat annual fees, invoiced.
  No billing database, no usage tracking, no credit-card processor.
- **No customer support team.** Year 1 is founder-led. The support SLA is a commercial
  commitment, not a headcount commitment — it means the founder responds within 4 hours, not
  that a team does.
- **No marketing budget.** The OSS community is the growth engine. Conference presence
  (one or two events in 2026) and a good README are the only required marketing investments.
- **No sales team.** Year 1 enterprise deals are founder-direct. The pipeline is short
  (3 enterprise targets) and high-value enough to justify the founder's time.

---

## 10. Risks and Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| EU cloud providers build internally instead of licensing | Low (timeline too short) | Emphasize August 2026 urgency; offer LOI within 2 weeks |
| Enterprise buyers demand on-premise support engineer | Medium | Scope Pro support as remote-only in license terms; onsite is add-on |
| Competitor releases compliant stack before August 2026 | Low | Speed advantage + existing pentest/BSI process is not replicable in weeks |
| Pro license conversion rate too low from OSS | Medium | Compliance report generator as the primary upgrade trigger; monitor conversion monthly |
| Single OEM licensee dominates and demands exclusivity | Low | License terms explicitly non-exclusive; geographic exclusivity only as premium add-on |

---

## References

- [`open-core-boundary.md`](open-core-boundary.md) — What is and is not included in Apache-2.0
- [`ideas-expose.md`](ideas-expose.md) — Full ideation including OEM concept (Part 5)
- [`ip-protection-strategy.md`](ip-protection-strategy.md) — CLA §3, trademark, patent options
- [`legal-liability.md`](legal-liability.md) — Liability caps and indemnification model
- `docs/claude-ref/adr-0017.md` — Enterprise Edition technical scope
- `CLA.md` — §3 Relicense Right (the legal foundation for commercial licensing)
