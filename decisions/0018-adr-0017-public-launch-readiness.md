# ADR-0018 — ADR-0017 Public-Launch Readiness

**Status:** Proposed (2026-05-15)
**Predecessor:** ADR-0017 (Enterprise Control Plane) — structurally
complete as of 2026-05-15.
**Sister-ADRs:** 0007 (multi-tenant gateway), 0014 (atelier-admin),
0015 (atelier-console).
**Owner of decision:** Silvio Jurk (Maintainer / future GmbH-Geschäftsführer).

---

## Context

ADR-0017 (Enterprise Control Plane) reached **structural completion**
on 2026-05-15 across three repositories:

- `github.com/veegee82/AtelierOS` — open-core Apache-2.0 (Phases II,
  III, IV shipped + UI wiring in admin).
- `github.com/veegee82/Atelier-Enterprise` — proprietary overlay
  with one feature shipped (scheduled-reports, 20/20 tests green).
- `github.com/veegee82/Atelier-Cloud` — ops layer with Helm chart +
  Stripe webhook + tenant-provisioner (16/16 tests green).

The architecture is complete. The product is not yet sellable. A
ten-item gap separates "structurally sound" from "first paying
customer". This ADR organises that gap into five disjoint workstreams
with explicit owners, sequencing, acceptance criteria and risk
analysis, so each workstream can be picked up in its own session
without re-litigating priorities.

The TL;DR of the gap, before being grouped:

1. Repo-Rename `Atelier-Enterprice` → `Atelier-Enterprise` (typo fix). **DONE 2026-05-15.**
2. Legal review of `EULA.md` + `LICENSE` in the Enterprise repo.
3. RS256 Maintainer keypair generation + `pubkey.pem` commit.
4. Phase I — `atelier-admin` 14.3–14.6 (Cross-Tenant-Search +
   aggregated Compliance Dashboards + Metrics-UI-Embedding).
5. Console route-wiring of the compliance-report endpoints (admin
   has them, console doesn't).
6. SPA sections for compliance-reports in both Admin and Console
   (operator-facing UI buttons).
7. `.atelier-pkg` build pipeline for Enterprise distribution.
8. Additional Enterprise commercial features (Cross-Tenant
   Audit-Search, SSO Wizard, WORM Archive, SLA Dashboard, Support
   Integration, White-Label UI).
9. Additional Cloud features (`seed_plans.py`, customer
   onboarding/offboarding, DR setup, cost-reports, custom-domain
   support).
10. Marketing assets (landing page, pricing page, whitepaper, demo
    video, HN/Lobsters launch coordination).

---

## Decision

Organise the ten items into **five workstreams** with explicit owner
and parallelisation map. Each workstream has its own success
criteria; workstreams can advance independently except where
explicit dependencies exist.

| WS | Name | Owner | Effort (solo) | Items |
|---|---|---|---|---|
| **A** | One-shot operator actions | Operator | ~30 min total | 1, 3 |
| **B** | Legal review | External counsel | extern, ~2–4 weeks calendar | 2 |
| **C** | UI completion | Code | ~1 week | 5, 6, 7 |
| **D** | Feature completion (open + closed) | Code | ~12–16 weeks | 4, 8, 9 |
| **E** | Marketing & launch | Operator + Code | ~2–4 weeks | 10 |

**MVP-launchable threshold** = A complete + B accepted + C
complete + a curated subset of D + E phase-1 complete. Items in
D fan out beyond MVP and continue post-launch.

---

## Workstream A — Operator one-shots (~30 min)

### A.1 Repo rename `Atelier-Enterprice` → `Atelier-Enterprise` [DONE]

**Action:** GitHub UI → Settings → Rename. GitHub auto-creates a
redirect from the old URL; existing clones keep working.

**Post-rename cleanup** (Claude Code session, ~5 min):
- `CLAUDE.md` § "ADR-0017 Phase V + VI" → update repo URL
  references.
- `Atelier-Enterprise/README.md`, `EULA.md` → update self-references.
- `Atelier-Cloud/README.md` → update Enterprise repo URL.
- No code changes (the URL is documentation-only).

**Acceptance:** `gh repo view veegee82/Atelier-Enterprise` succeeds;
no stale "Enterprice" strings outside historical git log.

### A.2 RS256 Maintainer keypair

**Action:**

```bash
# On the Maintainer's offline signing machine, NOT this dev host:
cd ~/projects/claudeOS/core/license
python -m atelier_license.cli keygen \
    /secure/path/maintainer-priv.pem \
    /tmp/maintainer-pub.pem
cp /tmp/maintainer-pub.pem atelier_license/pubkey.pem
git add atelier_license/pubkey.pem
git commit -m "feat(atelier-license): commit pinned Maintainer public key (priv kept offline)"
git push
```

**Critical posture:**

- Private key NEVER touches a network-connected machine.
- Private key NEVER lands in any git repo (`.gitignore` of
  atelier-license already excludes `*.priv.pem` patterns).
- The pubkey IS committed — it's the structural trust anchor every
  AtelierOS deployment validates against.

**Acceptance:** `core/license/atelier_license/pubkey.pem`
exists in HEAD; running `atelier_license.verify_token` against a
freshly-issued license JWT succeeds.

### A.3 First evaluation license

Issue a 90-day evaluation license for the Maintainer's own
testing:

```bash
python -m atelier_license.cli issue \
    /secure/path/maintainer-priv.pem \
    --customer-id maintainer-eval-01 \
    --tier business \
    --employee-count-max 50 \
    --seats 5 \
    --days 90 \
    --flags compliance_reports_premium,sso_wizard \
    --out /tmp/eval-license.jwt
```

Optionally check it into a private gist or password manager for
re-installation.

**Why now:** without a real license, the enterprise plugin can't
be exercised end-to-end against the open-core deployment. This
unlocks D-stream testing.

---

## Workstream B — Legal review (~2–4 weeks calendar, extern)

### B.1 EULA + LICENSE legal review

**Documents requiring review (proprietary repo):**

- `Atelier-Enterprise/LICENSE` — "All Rights Reserved" placeholder
  + pointer to EULA.
- `Atelier-Enterprise/EULA.md` — Docker-Desktop-style
  size-gated commercial license draft (v0.1, marked
  `LEGAL REVIEW PENDING`).

**Review checklist for the lawyer** (German + EU software-licensing
qualification required):

- §1 Definitions — do "Small Organisation" thresholds (50 employees
  AND €5M revenue) hold up in court? Cross-border (DE / AT / CH)
  enforceability?
- §2/§3 Grant — is the no-cost grant for Small-Org Production-Use
  legally distinct from §3 commercial requirement for Large-Org
  Production-Use?
- §4 Permitted Modifications — would "bug-fix patches as PRs back
  to me" qualify as an implicit copyright assignment we should
  capture more cleanly?
- §5 Distribution — does the signed-package channel + per-license
  source-clone access need GDPR-compatible operator data
  collection?
- §11 Governing law / jurisdiction — is "seat of Licensor" too
  vague for international customers; should we specify a fixed
  city + court (e.g. AG/LG München)?
- §12 Self-attestation — is the implicit-acceptance-by-installation
  pattern enforceable in DE/EU consumer-law contexts?

**Acceptance:** Counsel signs off on a `v1.0` revision; the
`LEGAL REVIEW PENDING` banner at the top of `EULA.md` is removed;
the Enterprise repo's README updates to reflect the cleared status.

### B.2 GmbH-in-Gründung discussion

Parallel to the EULA review, discuss with counsel:

- Should the Maintainer (Silvio Jurk privately) be the Licensor
  in the EULA, or should a GmbH-in-Gründung be set up first?
- VAT / Umsatzsteuer obligations on Enterprise license fees
  (B2B EU vs international).
- Liability cap appropriateness given the compliance-critical
  nature of the product.
- Cross-EU customer-data handling under GDPR Art. 28 (data
  processor agreement template).

**Not blocking MVP** (the Maintainer can ship as a natural-person
Licensor for the first 1–3 paying customers), but should land
before the first €100k of contracted revenue.

---

## Workstream C — UI completion (~1 week solo)

### C.1 Console route-wiring (item #5; ~1 day)

Mirror the `atelier-admin/routes/compliance_reports.py` work in
`atelier-console`, but with **tenant-tier scoping** instead of
operator-tier cross-tenant access.

Concretely:

- Add `core/console/atelier_console/routes/compliance_reports.py`.
- Endpoints (tenant-tier session + CSRF for mutations):
  - `GET  /v1/console/compliance-reports/types`
  - `POST /v1/console/compliance-reports`
  - `GET  /v1/console/compliance-reports`
  - `GET  /v1/console/compliance-reports/{report_id}/download`
  - `DELETE /v1/console/compliance-reports/{report_id}`
- Path-traversal protection + session-bound tenant id (not URL-
  passed tid). The owner can only generate reports for their own
  tenant.
- Tests mirror `test_compliance_reports.py` from admin; ~16 cases.

**Acceptance:** Owner-tier session generates a PDF; download
works; admin and console produce byte-identical PDFs for the
same chain + period.

### C.2 SPA sections (item #6; ~1–2 days)

Both Admin and Console SPA need a new section to expose the
reports visually.

**Admin SPA** (vanilla JS in `atelier_admin/web/`):
- New sidebar entry "Compliance Reports".
- Section view: dropdown (report-type) + date-range picker +
  generate button → progress spinner → list of previously-generated
  reports with download + delete actions.
- Theme-aware (light/dark) via the design tokens locked in
  Phase IV.

**Console SPA** (vanilla JS in `atelier_console/web/`):
- Same section pattern, scoped to the owner's tenant.
- Add to `NAV` array in `app.js`.
- Drilldown view: PDF preview iframe + metadata block (anchor
  hash, chain intact yes/no, generated-at).

**Acceptance:** A user logged in via the dev-login flow clicks
"Generate" and gets a PDF back in <30 s. Visual review on
Chromium + Firefox in both themes + 360 px / 768 px / 1280 px
viewports.

### C.3 `.atelier-pkg` build pipeline (item #7; ~1–2 days)

The Enterprise repo's README mentions "distributed as signed
`.atelier-pkg`" but no build script exists yet.

**Deliverables:**

- `Atelier-Enterprise/scripts/build_pkg.sh` — packages the plugin
  (`atelier_enterprise/` tree + `requirements.txt` + `LICENSE` +
  `EULA.md`) into a `.atelier-pkg` archive per ADR-0007 Phase 5
  format.
- Signs the archive with the Maintainer's offline RS256 key
  (delegates to `atelier-license issue-package` or similar — needs
  a small new CLI subcommand in atelier-license).
- Verification step: re-extracts in a tmpdir, validates signature,
  runs the test suite from the extracted state.

**Acceptance:** `bash scripts/build_pkg.sh 0.1.0` produces
`Atelier-Enterprise-0.1.0.atelier-pkg` + `.sig` that an operator
can install via `atelier-license install-package` (which itself
is a new sub-command in `atelier-license/cli.py`).

---

## Workstream D — Feature completion

The largest workstream by effort. Splits into three sub-stacks:
open-core (D.1), Enterprise (D.2), Cloud (D.3). Each sub-stack
runs at its own cadence; none block MVP-launch but each unlocks
specific commercial tiers.

### D.1 atelier-admin Phase I, 14.3–14.6 (item #4; ~4–6 weeks)

Per ADR-0014's existing roadmap, four sub-phases remain:

- **14.3 — Cross-Tenant Search backend.** Federated FTS over
  every tenant chain the operator's token has access to. Backed
  by the existing `forge.security_events.iter_events` walker.
- **14.4 — Aggregated Compliance Dashboards.** A read-only
  operator view that surfaces per-tenant `disclosure.*` /
  `consent.*` counts, chain-intact status, license-tier
  distribution.
- **14.5 — Metrics-UI Embedding.** Iframe-embed of the Grafana
  dashboards documented in ADR-0007 Phase 6. Honours the
  operator-supplied `metrics_iframe_base_url` config.
- **14.6 — Closure.** Closure ADR + README updates + bump
  `atelier-admin` to v0.2.

**Acceptance:** 50+ new tests in atelier-admin; full admin test
suite > 220 green.

### D.2 Enterprise commercial features (item #8; ~6–8 weeks)

Implement the remaining commercial features from ADR-0017 Phase V's
roadmap. **Sequenced by sales value**:

1. **Cross-Tenant Audit-Search** (~1 week) — depends on D.1 14.3
   backend; thin UI overlay that adds operator-tier access via
   the enterprise plugin's `/v1/enterprise/audit-search` route.
2. **SSO Setup Wizard UI** (~1–2 weeks) — visual editor for
   the OIDC/SCIM YAML that ADR-0007 Phase 3.4–3.6 backs. The
   backend is already mature; this is pure SPA + form-validation
   work.
3. **WORM Audit Archive** (~1–2 weeks) — S3-compatible cold-
   storage exporter with chain-anchor verification on re-import.
   Hetzner Storage Box + AWS S3 + Backblaze B2 are the three
   targets (boto3 already in optional requirements).
4. **SLA Monitoring Dashboard** (~1 week) — per-tenant uptime +
   latency p50/p95/p99 surfaced via Prometheus query. Builds on
   ADR-0007 Phase 6 metrics.
5. **Premium Compliance-Report variants** (~1 week) — custom
   Jinja2 templates + signed PDFs (operator's RSA key, not the
   Maintainer's) + WORM archive integration.
6. **Support Integration** (~3–5 days) — ticket creation from
   inside the Admin UI, auto-attaching audit-chain snippets.
7. **White-Label Branding UI** (~3–5 days) — web editor for
   `branding.yaml` (logo, accent colours, custom disclosure
   paragraph). Honours the design-token contract from Phase IV.

Each feature gets its own per-feature commit + test suite;
Enterprise plugin's `/v1/enterprise/status` lists which
feature-flags are mounted.

### D.3 Cloud pre-launch features (item #9; ~4–6 weeks)

Sequenced by deployment-readiness:

1. **`seed_plans.py`** (~1 day) — one-shot script that pushes
   the `billing/plans.py` Tier definitions into Stripe via the
   public API. Idempotent (Stripe Idempotency-Key).
2. **`customer_onboarding.py`** (~1 week) — full signup flow:
   landing-page form → Stripe checkout link → checkout-completed
   webhook → tenant provisioning → operator-token email.
3. **`customer_offboarding.py`** (~1 week) — cancellation flow
   with data-export ZIP + 30-day soft-delete window per GDPR
   Art. 17.
4. **Status-page integration** (~1 week) — Atlassian Statuspage
   or self-hosted (BetterStack / Statuspage Open). Wires to
   Prometheus alertmanager.
5. **Per-region DR automation** (~1 week) — eu-west-3 standby
   replica + automated failover playbook.
6. **Cost-allocation reports** (~1 week) — per-tenant LLM-spend
   (from `gateway.run_*` events) → monthly billing breakdown.
7. **Custom-domain support** (~3–5 days) — per-tenant subdomain
   (`<tenant>.atelier.cloud`) via cert-manager + Ingress
   templating.

---

## Workstream E — Marketing & launch (~2–4 weeks)

### E.1 Landing page (~1 week)

Single-page site at `atelier.cloud` and `atelieros.eu`.

**Content structure:**

1. Hero: *"EU-compliant AI agent runtime. Self-hosted. EU AI
   Act + GDPR by construction."*
2. Three-tile feature row: Compliance Reports / Multi-Engine /
   Multi-Channel.
3. "How it works" — three-step diagram with the disclosure card,
   consent gate, and audit chain.
4. Pricing tier table (Starter / Business / Enterprise — match
   `billing/plans.py`).
5. Self-hosted CTA: "Apache-2.0 download from GitHub".
6. Compliance-officer pull-quote — link to whitepaper PDF.
7. Footer: GitHub repos + EULA + Privacy.

Vanilla HTML/CSS (no SPA); same design-token set as the Phase IV
console for visual consistency.

### E.2 Compliance whitepaper (~3–5 days)

8-page PDF: *"AtelierOS and the EU AI Act 2026 — A Compliance
Architecture Anatomy."*

Sections:

1. The four AI Act requirements that AI-agent vendors structurally
   ignore.
2. AtelierOS's structural answer to each: Disclosure card, consent
   gate, audit chain, engine-policy gate.
3. The GDPR overlay: per-tenant chain, secret vault, data
   minimisation in voice + memory layers.
4. Auditor's view: what a Wirtschaftsprüfer sees when handed an
   AtelierOS deployment + the three baseline PDF reports.
5. Where we go beyond: scheduled reports + WORM archive +
   cross-tenant audit search in the Enterprise edition.

Use the same `weasyprint` toolchain as the in-product PDF
generators so the brand is consistent.

### E.3 Demo video (~2–3 days)

60-second screencast:

1. Operator types `/join` in a Telegram chat → disclosure card
   appears with bot identity + opt-in commands.
2. User types `/consent on` → granted, audit event lands.
3. Bot does real work (sees a prompt, runs a Forge tool, replies).
4. Operator runs `voice-audit verify` → green chain.
5. Operator clicks "Generate AI Act report" in the Admin UI →
   PDF opens → green integrity banner + hash anchor.

Cuts to slides naming each AI Act article being demonstrated.

### E.4 HN / Lobsters launch (~1 day execution)

**Title:** *"Show HN: AtelierOS — self-hosted AI agent runtime
with EU AI Act compliance built in"*

**Submit slot:** Tuesday 8–10 AM PT (per HN best-time analysis).

**Pre-launch checklist** (the night before):

- Landing page live + HTTPS.
- GitHub README has prominent "Why AtelierOS" section + screenshots.
- 60-second demo video uploaded to YouTube (unlisted), embedded
  on landing page.
- HN account has enough karma to avoid auto-flagging.
- Three "first comment" replies pre-drafted (one defensive, one
  technical, one "where can I try it").

**Acceptance:** front-page reach for ≥ 4 h; ≥ 100 GitHub stars
within 24 h; ≥ 5 inbound enterprise enquiries within a week.

### E.5 Awesome-list submissions (~1 day)

Submit to:
- `awesome-llm-tools`
- `awesome-mcp`
- `awesome-selfhosted`
- `awesome-selfhosted-ai`
- `awesome-agents`
- `awesome-eu-tech` (if it exists; otherwise create)

---

## Sequencing & critical path

```
┌─────────────────────────────────────────────────────────────────────┐
│                          PARALLEL TRACKS                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  A.1+A.2 (30min)──┐                                                 │
│                   │                                                 │
│  B (extern, 2-4w)─┼─→  E.2 whitepaper draft                         │
│                   │    (can start; needs B done before public)      │
│                   │                                                 │
│  C.1+C.2 (1w)────┐│                                                 │
│  C.3 (1-2d)──────┤│                                                 │
│                  ▼▼                                                 │
│  ┌── MVP-launchable threshold ──┐                                   │
│  │  A done + C done + B accepted │                                  │
│  └──────────────┬───────────────┘                                   │
│                 ▼                                                   │
│  E.1 landing (1w) ──┐                                               │
│  E.3 demo (2-3d) ───┤── E.4 HN launch                               │
│  E.5 awesome (1d) ──┘                                               │
│                                                                     │
│  D.1 admin Phase I (4-6w) ─── runs in parallel,                     │
│  D.2 enterprise feats (6-8w)   continues post-launch                │
│  D.3 cloud feats (4-6w)        as feature drops                     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Critical path to MVP-launch** (8–10 weeks elapsed):

1. Day 1: A complete (30 min Maintainer-side).
2. Day 1: Engage legal counsel; B clock starts.
3. Week 1: C complete (1 week solo coding).
4. Week 2–4: B in flight; E.1 + E.2 + E.3 ready in parallel.
5. Week 4: B accepted → EULA finalised → repo public-mirror-ready.
6. Week 4–5: E.4 + E.5 launch.
7. Week 5+: D streams continue; first sales conversations begin.

**Acceleration levers** (if budget allows hiring):

- Frontend dev (E.1 + C.2 SPA work) — cuts ~1 week off critical path.
- Backend dev (D.1) — runs in full parallel, doesn't shorten MVP-CP.
- Legal expedite (B) — typical 2 weeks → 5 business days for ~3×
  hourly rate.

---

## Risks & mitigation

| # | Risk | Probability | Impact | Mitigation |
|---|---|---|---|---|
| R1 | EULA found unenforceable in DE/EU consumer law | Medium | High (no commercial path) | Engage counsel early; preserve `tier=free` as the structural product Apache-2.0 fallback |
| R2 | RS256 private key lost / leaked | Low | Critical | Hardware token + offline backup; CLA §3 lets us roll a new pubkey if needed |
| R3 | Hyperscaler clone after public launch | Medium | High (revenue dilution) | CLA §3 + open-core split + size-gated EULA already mitigate; BSL-relicense remains the nuclear option |
| R4 | HN launch flops (< 50 stars, no inbound) | Medium | Medium (delayed sales) | Pre-warm via LinkedIn DACH outreach in parallel; soft-launch path is direct enterprise sales |
| R5 | Phase I Cross-Tenant-Search blocks SSO-wizard launch (D.2 #2) | Medium | Low | Decouple: SSO wizard ships against the existing ADR-0007 Phase 3.4–3.6 backend; doesn't actually depend on D.1 14.3 |
| R6 | Stripe webhook signing secret rotation breaks production | Low | High | Document rotation playbook in cloud-ops README before launch; alert-on-401 in monitoring |
| R7 | Compliance-officer finds an audit-chain corner case the PDF doesn't surface | Medium | Medium | Beta-test the three baseline reports with 2-3 friendly Mittelstand contacts before E.4 |
| R8 | First customer wants a feature in D.2 that isn't shipped yet | High | Low | The size-gated EULA's "Small Org" free tier buys time; commercial customers get a roadmap + ETA |

---

## What this ADR explicitly does NOT do

- **Doesn't pick the GmbH-vs-natural-person path for the Licensor.**
  That's a finance + tax decision the operator makes with counsel.
- **Doesn't commit to a specific E.4 launch date.** Timing depends
  on B (legal review) which has external dependency.
- **Doesn't try to fully enumerate D.2 (Enterprise features).**
  The seven sub-items are the ADR-0017 baseline; market feedback
  may reorder them.
- **Doesn't replace ADR-0017.** ADR-0018 is the launch-readiness
  layer on top of the already-shipped 0017 structure.

---

## Acceptance — when ADR-0018 is "Done"

ADR-0018 transitions from *Proposed* → *Accepted* the moment:

- The first paying enterprise customer signs a commercial license.
- OR the first €1,000 of self-service Stripe revenue is collected.
- OR three independent enterprise evaluations are in flight.

After any of those, ADR-0018 supersedes itself with ADR-0019
(post-launch operations) which covers SLA tracking, customer-
success workflows, and roadmap-driven feature prioritisation
based on actual signal from the field.

---

## References

- ADR-0017 — Enterprise Control Plane (the predecessor; structural
  layer this ADR builds atop).
- ADR-0014 — atelier-admin (Phase I work lives in this ADR's
  Phase 14.3–14.6 roadmap).
- ADR-0015 — atelier-console (C.1 + C.2 extend this).
- ADR-0007 Phase 5 — `.atelier-pkg` signed package format (C.3
  consumes).
- ADR-0007 Phase 6 — Prometheus metrics (D.2 SLA dashboard
  consumes).
- `Atelier-Enterprise/EULA.md` — legal review target (B.1).
- `Atelier-Cloud/billing/plans.py` — tier definitions (D.3.1 seeds
  to Stripe).
- Marketing strategy conversation 2026-05-15 — Open-Core decision,
  Docker-Desktop license model, three-phase Go-to-Market plan
  (E synthesises).
