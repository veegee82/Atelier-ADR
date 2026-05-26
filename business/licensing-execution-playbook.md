# AtelierOS — Licensing Execution Playbook

**Status:** Operational guide — do this, in this order
**Owner:** Silvio Jurk
**Last reviewed:** 2026-05-26

This document is a step-by-step execution guide: where to register, what to file,
what to publish, and how to collect the first license payment. No strategy — only actions.

---

## Phase 0 — Legal Entity Check (Week 1, Day 1)

Before any enterprise contract can be signed, the selling entity must be clearly established.
Enterprise procurement teams will not wire €15,000+ to a private person.

### What you likely need

| Situation | Entity | Why |
|---|---|---|
| Already have a registered Gewerbe (Einzelunternehmer) | Sufficient for Pro licenses; marginal for Enterprise | No liability separation; enterprise buyers may push back |
| No registered business yet | Register Einzelunternehmen first (1 day), then upgrade to UG or GmbH when first Enterprise deal is in sight | Fastest to Pro; manageable until €75k+ revenue |
| Enterprise deal imminent | UG (haftungsbeschränkt) — €1 Stammkapital minimum, ~€1,000–1,500 setup cost including notary | Looks professional; limits personal liability |

### Action items

- [ ] **Confirm your current Rechtsform.** If you have a registered Gewerbe, you can start
  selling Pro licenses immediately under that entity.
- [ ] **Open a dedicated business bank account** (if not already done) — Kontist, Qonto,
  or Deutsche Bank Business. Required for clean invoicing and VAT.
- [ ] **Register for VAT (Umsatzsteuer)** at your local Finanzamt if not already done.
  B2B software licenses in the EU are subject to reverse charge (§13b UStG for cross-border),
  but you still need a DE VAT ID (USt-IdNr.) for enterprise invoices.
- [ ] **Decision point:** Do you have an existing GmbH/UG? → skip. No entity? → register
  Einzelunternehmen at your local Gewerbeamt (costs €20–50, done in one day online in most
  German cities). Upgrade to UG when first Enterprise deal appears.

**Where:** gewerbe-anmeldung.de (most cities), or your city's Gewerbeamt portal.
**Cost:** €20–50.
**Time:** 1 day.

---

## Phase 1 — Trademark Registration (Week 1–2)

The "AtelierOS" name must be registered before any public commercial launch.
Without a trademark, a competitor or a cloud provider can adopt the name the moment you
go public.

### EUIPO Trademark (EU-wide, covers all 27 member states)

- **Platform:** euipo.europa.eu → "Apply for a trade mark" → eSearch
- **First step:** Run a free search at `euipo.europa.eu/eSearch` to confirm "AtelierOS"
  is not already registered. Search class 42 (software as a service, computer software design).
- **Nice Classification classes to file:**
  - Class 42: Computer software design and development; software as a service (SaaS);
    platform as a service (PaaS); AI agent platform services; software licensing
  - Class 35: Business software consulting (optional, add later)
- **Filing:** euipo.europa.eu → "Apply online" → Fast Track (if no custom elements)
- **Cost:** €850 for one class, €50 per additional class. Pay by credit card online.
- **Time to grant:** 5–7 months (Fast Track), but you have priority from day of filing.
  Filing date = protected date. File today.
- **You do NOT need a lawyer for this.** The EUIPO online wizard is sufficient for a
  word mark ("AtelierOS"). A lawyer adds value for a figurative mark (logo) or if you
  expect opposition.

### German DPMA as backup (optional, but faster)

- **Platform:** dpma.de → "Marken" → Online-Anmeldung
- **Cost:** €290 for one class + €100 per additional class
- **Time:** 3–4 months, Germany only
- **Recommendation:** File EUIPO. If you want faster German protection as insurance,
  file DPMA simultaneously (different office, different territory).

### Action items

- [ ] Search "AtelierOS" on euipo.europa.eu/eSearch — confirm no conflict
- [ ] File EUIPO word mark: Class 42 (+ Class 35 if budget allows)
- [ ] Save the filing receipt — the application number is your proof of priority date

**Cost:** €850–€950.
**Time to file:** 2 hours.

---

## Phase 2 — License Documents (Week 2–3)

You need legal documents before you can sell. Three documents are required:

### 2.1 Commercial License Agreement (Pro + Enterprise)

This is the contract a buyer signs when purchasing a Pro or Enterprise license.
It must cover: grant of rights, restrictions, support SLA terms, payment, warranty
disclaimer, liability cap, governing law (German law, Hamburg or Berlin courts).

**Fastest path (no lawyer yet):**
Use an adapted version of a standard software license template, then have a lawyer
review before your first Enterprise deal (not before your first Pro deal).

Template sources:
- **IT-Recht Kanzlei** (it-recht-kanzlei.de): German IT law firm offering
  ready-made software license templates for ~€150–300. Suitable for Pro tier immediately.
- **Docracy / Common Paper**: English-language templates, usable for international buyers.
- **IFRAU / Bitkom Muster-SLA**: German trade association template for SLA clauses.

**Key clauses to include:**
- License scope: self-hosted, named instance count, no sublicensing
- Permitted use: production, internal and customer-facing deployments
- Support SLA: response time commitment, escalation path, exclusions
- Liability cap: limited to 12 months of license fees paid
- Governing law: Germany, Hamburg courts (or Berlin if preferred)
- Termination: annual renewal, 60-day notice for non-renewal
- Audit rights: licensor may audit compliance once per year on 30-day notice

### 2.2 OEM License Agreement

Separate document for cloud providers. Key additional clauses vs. Pro/Enterprise:
- White-label rights (grant scope: can they remove the "AtelierOS" name?)
- Attribution requirements if not white-label
- Revenue share reporting obligations (if revenue-share model chosen)
- Source code audit rights (read-only access to Enterprise repo)
- Sublicensing: the OEM can grant end-customers the right to use the deployed software
- No sublicensing of the license itself (end-customers cannot resell)

**For OEM agreements, use a lawyer.** These are €50k–€500k deals. A one-time
legal review of the OEM template (~€500–1,500) is the right investment before
the first signature.

**Where to find a startup-savvy German IT lawyer:**
- LEGAL REVOLUTION (legalrevolution.de) — startup-focused, flat fees
- Legalbase (legalbase.de) — Berlin startup law firm, fixed-fee packages
- SCWP Schindhelm — Hamburg, specialised in software IP and licensing

### 2.3 Data Processing Agreement (DPA, DSGVO Art. 28)

Every Enterprise and OEM customer will require a DPA before signing.
The DPA governs how AtelierOS (as a software vendor providing support access)
may process personal data.

**Important:** In the pure license model, AtelierOS does NOT process personal data
on behalf of the customer — the customer self-hosts. This significantly simplifies
the DPA. The main scope: support access to logs, remote debugging sessions.

Use the **Bitkom DPA template** (available free at bitkom.org → "Muster-AV-Vertrag").
Adapt for software vendor support use case. One-time legal review recommended.

### Action items

- [ ] Purchase IT-Recht Kanzlei software license template (Pro tier — use immediately)
- [ ] Adapt template: add AtelierOS-specific clauses (support SLA, governing law)
- [ ] Download Bitkom DPA template; adapt for self-hosted support use case
- [ ] Book one-hour consultation with startup IT lawyer before first Enterprise deal
  (not urgent for Pro tier — Pro buyers rarely demand custom legal review)
- [ ] Draft OEM template outline; send to lawyer when first OEM prospect appears

**Cost:** €150–300 (Pro template) + €500–1,500 (OEM lawyer review).
**Time:** 1 week.

---

## Phase 3 — Technical License Infrastructure (Week 2–3, parallel)

The JWT license gate is already implemented (ADR-0017). What is missing is the
**issuance and payment workflow**.

### 3.1 License JWT Issuance

The AtelierOS license server issues RS256 JWTs containing capability flags
(`compute`, `compute_fabric`, `enterprise`). This infrastructure already exists.

**For Year 1 (manual issuance):**
You do not need an automated license server. Process:
1. Customer pays (invoice or Stripe)
2. You generate the JWT manually using the existing `atelier-license-issue` tooling
3. You send the JWT to the customer by email
4. Customer places it at `<atelier_home>/global/license/license.jwt`

This is sufficient for 3–20 customers. Automate only when volume demands it.

**Where the JWT generation tooling lives:**
`operator/bridges/shared/license.py` or equivalent in ADR-0017 implementation.
Confirm the CLI command exists: `atelier-license-issue --tier enterprise --seats 500 --expires 2027-06-01`

### 3.2 Payment and Invoicing

**For Pro licenses (self-serve goal):**
- **Stripe Invoicing** (stripe.com) — create a product "AtelierOS Professional License",
  price €2,400/year, send invoice link to customer. No storefront needed yet.
- Alternatively: **Lexoffice** or **Sevdesk** for German-law invoicing with automatic
  DATEV export (important for tax compliance).

**For Enterprise and OEM (negotiated, not self-serve):**
- Manual invoice via Lexoffice/Sevdesk
- Wire transfer (SEPA) — standard for B2B in Germany
- Stripe does not handle €50k+ enterprise deals well; use bank wire

**Setup Stripe:**
- stripe.com → create account → verify identity (German ID + bank account)
- Create a product: "AtelierOS Professional License" — €2,400 per year
- Set up a payment link (no website needed): customers get a Stripe payment URL
- Enable automatic invoice PDF generation (required for VAT compliance)

### 3.3 License Tracking Spreadsheet (Year 1)

Until you have 20+ licenses, a simple spreadsheet is the right tool:

| Customer | Tier | Annual fee | Start date | Renewal date | JWT fingerprint | DPA signed |
|---|---|---|---|---|---|---|

Keep this in your encrypted storage, not in the repo.

### Action items

- [ ] Verify `atelier-license-issue` CLI works end-to-end: generate a test JWT,
  place it in a test deployment, confirm capability flags are enforced
- [ ] Create Stripe account; set up "AtelierOS Professional License" product + payment link
- [ ] Create Lexoffice or Sevdesk account for German invoicing
- [ ] Create license tracking spreadsheet

**Cost:** Stripe 0.5–2% per transaction (waived for wire transfers) + Lexoffice €9/month.
**Time:** 1 day.

---

## Phase 4 — Publishing (Week 3–4)

### 4.1 GitHub Repository

The repository already has Apache-2.0 `LICENSE`, `NOTICE`, `CLA.md`, `CONTRIBUTING.md`.
Add the following:

**`COMMERCIAL-LICENSE.md`** (new file, repo root):
```markdown
# Commercial Licensing

AtelierOS is open-source under Apache-2.0. For commercial use cases that require
a support SLA, compliance reports, or OEM rights, a commercial license is available.

## License tiers

| Tier | Annual fee | For whom |
|---|---|---|
| Professional | €2,400 | Teams and agencies deploying for customers |
| Enterprise | €15,000–€80,000 | Regulated industries, large organisations |
| OEM / White-Label | €50,000–€500,000 | Cloud providers embedding AtelierOS |

## How to purchase

Email: licensing@atelierOS.dev (or your actual contact address)

Include: your organisation name, expected deployment size, and which tier you need.
Response within 1 business day.
```

**Update `README.md`:** Add a "Commercial License" section pointing to
`COMMERCIAL-LICENSE.md`. Place it after the installation section, before Contributing.

### 4.2 Website — Pricing Page

The landing page in `web-next` needs a `/pricing` or `/licensing` route.

**Minimum viable pricing page (launch in 1 day):**
Three columns: Open Source (free), Professional (€2,400/yr), Enterprise (contact us).
Each column lists 5 key features included/excluded.
CTA: "Get a license" → email link or Stripe payment link for Pro.

**Do NOT build a full billing portal yet.** A mailto link for Pro and Enterprise is
sufficient until you have 5+ customers. Every hour building a billing portal
is an hour not spent on the first OEM deal.

### 4.3 docs/business/ visibility

The strategy documents in `docs/business/` are internal. They should NOT be published
as-is on the public website. The public-facing messaging is the pricing page, not the GTM doc.

However: the `COMMERCIAL-LICENSE.md` and the licensing section of `README.md` are public
and should be professional and complete.

### Action items

- [ ] Create `COMMERCIAL-LICENSE.md` at repo root
- [ ] Update `README.md` with "Commercial Licensing" section
- [ ] Add `/licensing` page to `web-next` (three-column pricing layout, no billing logic)
- [ ] Set up `licensing@atelierOS.dev` email alias (or use silvio@... until domain is ready)
- [ ] Publish atelierOS.dev (the landing page is already in `web-next` — needs standalone hosting)

**Website hosting for atelierOS.dev:**
- Fastest option: Vercel (vercel.com) — free for static/Next.js, custom domain, HTTPS automatic.
  Connect GitHub repo → `web-next` directory → deploy. Done in 30 minutes.
- Alternative: Cloudflare Pages — also free, also 30 minutes.

**Cost:** €0 (Vercel/Cloudflare free tier) + domain (~€10/year if not registered yet).
**Time:** 1 day for minimal pricing page.

---

## Phase 5 — OEM Outreach (Week 4–6)

### 5.1 Find the Right Contact

You need the **Head of Product** or **VP Product / VP AI** at the target cloud provider.
Do NOT contact: sales, support, partnerships (too slow), or engineering (wrong decision-maker).

**Where to find them:**
- LinkedIn: search "[company name] Head of Product AI" or "VP Product"
- company.com/about or company.com/team (often lists leadership)
- Conference speaker lists: CloudFest (Rust, Germany), EuroCloud Congress, T3n events

**Priority list:**
1. **Hetzner Cloud** — robot@hetzner.com is the general contact, but find product leadership
   on LinkedIn. Search: "Hetzner Cloud Product Manager" → direct InMail.
2. **IONOS (1&1 Internet SE)** — ionos.com/partner → "Become a partner" form as first contact;
   better: find "Director Product IONOS Cloud" on LinkedIn.
3. **OVHcloud** — ovhcloud.com/en/partner → Technology Partner program form; they have a
   formal partner program. Also: OVHcloud has a "Technology Partner" track specifically for
   ISVs — register there as first contact.
4. **Scaleway** — scaleway.com → press/partnership contact; smaller team, easier to reach CEO directly.
5. **T-Systems** — partner.t-systems.com → ISV Partner Program. Formal process; expect 4–8 weeks
   to reach the right person. Worth doing in parallel, not as first contact.

### 5.2 The Outreach Message (LinkedIn InMail or email)

Keep it under 150 words. No attachments on first contact.

```
Subject: EU AI Act compliance platform — white-label option for [Company]

Hi [Name],

Your customers will start asking for EU AI Act-compliant AI deployments from
August 2026 — that's 9 weeks away.

We've built AtelierOS: the only AI agent platform where compliance is
architecturally impossible to bypass (hash-chained audit log, consent gate,
disclosure card — structural, not configurable). It's open-source with a
white-label OEM license option.

The integration for a cloud provider takes two to four weeks of engineering time.
You'd have a compliant AI platform product before enforcement begins.

Would a 30-minute technical call make sense this week?

Best,
Silvio Jurk
AtelierOS — github.com/veegee82/AtelierOS
```

### 5.3 The 30-Minute Call Script

1. **(5 min)** Their situation: Do they have an AI product today? What are customers asking for?
2. **(10 min)** Live demo: Open the reference deployment. Show the audit chain growing in
   real time. Run `voice-audit verify`. Show a compliance report PDF.
3. **(5 min)** Integration path: "Your team integrates our Docker stack into your provisioning
   pipeline. We document every step. Two to four weeks."
4. **(5 min)** License terms: OEM flat fee or revenue share, depending on their preference.
5. **(5 min)** Next step: Letter of Intent → technical integration sprint → joint launch announcement.

### 5.4 The Letter of Intent (LOI)

An LOI is not a binding contract — it is a 1-page document that says:
"We intend to sign an OEM license agreement by [date]. The expected terms are [X]."

LOIs are fast to issue (no lawyers needed yet) and establish commitment. When a
cloud provider signs an LOI, you have a reference you can mention in other OEM conversations.

Template available at commonpaper.com (Cloud Service Agreement LOI — adapt for licensing).

### Action items

- [ ] Identify 3 LinkedIn contacts at Hetzner, IONOS, and OVHcloud (product level)
- [ ] Send first 3 outreach messages (use the template above, personalise 2 sentences)
- [ ] Register on OVHcloud Technology Partner portal (parallel, low-effort)
- [ ] Register on T-Systems ISV Partner portal (parallel, long lead time)
- [ ] Prepare live demo environment: confirm reference deployment is accessible + TLS working
- [ ] Prepare 3-slide deck (problem / AtelierOS solution / integration path) for the call

**Cost:** €0 (LinkedIn Free allows 5 InMails/month; Premium allows 50).
**Time:** 3 hours for outreach setup; 30 min per prospect call.

---

## Phase 6 — First License Sale (Week 4–8)

### The Pro license path (fastest first revenue)

The most likely first paying customer is someone already using AtelierOS free who
needs a compliance report for a client. Watch GitHub Issues, Discussions, and Discord
for signals:
- "Can I get a support contract?"
- "Does AtelierOS generate compliance documentation?"
- "Is there a paid version with SLA?"

When you see this signal, reply with a direct offer:
> "Yes — the Professional license at €2,400/year includes a 4h support SLA and
> automated EU AI Act compliance reports. I'll send you an invoice today."

Send the Stripe invoice link. When paid, generate the JWT and email it.
Total time from request to delivered license: 15 minutes.

### The Enterprise path (4–12 week cycle)

1. Prospect requests a demo → schedule within 48h
2. Demo call → send one-pager + pricing sheet within 24h
3. Prospect sends security questionnaire → answer within 5 business days
4. Prospect sends DPA for review → return signed DPA within 3 business days
5. Agree on 4-week pilot (use Enterprise license terms from day one)
6. Pilot sign-off → annual contract signed → first payment
7. Generate Enterprise JWT → email to customer IT contact

### Action items

- [ ] Monitor GitHub Discussions and Discord for upgrade signals
- [ ] Set up email template for Pro license response (15-minute turnaround goal)
- [ ] Create a one-pager PDF: "AtelierOS Enterprise — What Your Legal Team Needs to See"
  (maps each layer to the specific EU AI Act article it satisfies)
- [ ] Commission pentest (SySS: syss.de, Cure53: cure53.de) — quote request takes 1 day,
  test takes 2–3 weeks, report enables enterprise deals. Budget: €8,000–15,000.
  This is the single highest-ROI investment before the first Enterprise deal closes.

---

## Timeline Summary

| Week | Phase | Key output |
|---|---|---|
| 1 | Legal entity check, trademark filing | USt-IdNr confirmed, EUIPO application filed |
| 2 | License templates, Stripe setup | Pro license agreement ready, payment link live |
| 3 | JWT tooling verified, COMMERCIAL-LICENSE.md | Pro license can be issued end-to-end |
| 3–4 | README update, pricing page, atelierOS.dev live | Public licensing page live |
| 4 | First OEM outreach (3 messages) | 3 contacts reached |
| 5–6 | First OEM calls, Pro license monitoring | 1+ calls booked, first Pro signal |
| 6–8 | First payment received | Pro or OEM LOI — first commercial revenue |

---

## Cost Summary

| Item | Cost | When |
|---|---|---|
| EUIPO trademark (Class 42) | €850 | Week 1 — do it today |
| Gewerbeamt registration | €20–50 | Week 1 if not yet done |
| Business bank account | €0–10/month | Week 1 |
| IT-Recht Kanzlei license template | €150–300 | Week 2 |
| Lexoffice invoicing | €9/month | Week 2 |
| Lawyer (OEM template review) | €500–1,500 | Before first OEM signature |
| Pentest (SySS or Cure53) | €8,000–15,000 | Before first Enterprise close |
| Vercel hosting (atelierOS.dev) | €0 | Week 3–4 |
| LinkedIn Premium (optional) | €60/month | If InMail quota needed |
| **Total before first revenue** | **~€10,000–18,000** | Mostly the pentest |
| **Total excluding pentest** | **~€1,500–3,000** | Weeks 1–4 |

The pentest is optional for Pro licenses and required for Enterprise. If budget is
tight, start without it and commission it when the first Enterprise prospect appears.

---

## References

- [`license-go-to-market.md`](license-go-to-market.md) — Full GTM strategy with revenue targets
- [`open-core-boundary.md`](open-core-boundary.md) — What is and is not in the OSS tier
- [`ip-protection-strategy.md`](ip-protection-strategy.md) — CLA §3, trademark, patent detail
- `CLA.md` — §3 Relicense Right (legal foundation for OEM licensing)
- `docs/claude-ref/adr-0017.md` — Enterprise Edition technical scope + license JWT implementation
