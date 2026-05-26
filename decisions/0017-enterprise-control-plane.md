# ADR-0017 — Enterprise Control Plane (sellable Edition)

**Status:** Proposed (2026-05-15)
**Components:**
- `core/license/` (NEW — Apache-2.0, in this repo)
- `atelier-enterprise` (NEW — separate repo, **proprietary**, distributed as signed `.atelier-pkg`)
- Phases 14.3–14.6 extending `core/admin/` (Apache-2.0)
- Phases I2 / I3 extending `core/console/` (Apache-2.0)
- `atelier-cloud-ops` (separate repo, Ops-only, K8s + Helm + Stripe)
**Sister-ADRs:** 0014 (admin-ui), 0015 (console-self-service-ui), 0007 (multi-tenant gateway), 0010 (operator-observability)

---

## Context

ADR-0014 (`atelier-admin`) and ADR-0015 (`atelier-console`) shipped
the operator-tier and owner-tier web UIs of AtelierOS. Both are
under Apache-2.0 and ride on the existing gateway ASGI app. Today
the project carries:

- 13 functional UI sections in `atelier-console` (Owner-Self-Service)
- 2 of ~7 planned phases in `atelier-admin` (Operator-Tier auth +
  read-only viewers, ADR-0014 Phases 14.1 + 14.2)
- The full compliance backbone: hash-chained audit, consent gate,
  disclosure card, path-gate hook, engine-policy, zone-residency,
  per-tenant quota, OIDC/SCIM, signed `.atelier-pkg` distribution
  format (ADR-0007 Phase 5), Helm chart + container (ADR-0007 Phase
  4)

What is **missing** to position AtelierOS as a sellable
Enterprise product (per the marketing strategy of 2026-05-15):

1. A **commercial monetisation surface** — a payment trigger that
   activates when an organisation crosses a size threshold, without
   blocking the open-source community.
2. A **compliance-report generator** — one-click PDF artefacts
   (EU AI Act Art. 50 Evidence, GDPR Art. 30 Records of Processing,
   Audit-Chain Integrity Attestation) that compliance officers can
   hand to auditors. Today only raw chain inspection exists.
3. A **set of Enterprise-only features** that are valuable enough
   for €15–25k/year procurement decisions but NOT part of the
   open-core promise: cross-tenant audit-search, SLA dashboards,
   WORM archival, SSO setup wizard, premium support integration.
4. A **hosted SaaS path** — `atelier.cloud` — for operators who
   don't want to self-host. Same code, different deployment model.

The Apache-2.0 + CLA §3 (Relicense-Right) baseline established in
the **Licensing baseline** section of `CLAUDE.md` permits all of
this **without changing the kernel licence**: new components either
ship under Apache-2.0 in this repo, or live in a separate proprietary
repo authored by the Maintainer. The CLA §3 right is preserved for
true emergencies (hyperscaler-clone defence); routine commercial
launch does not need it.

---

## Decision

Ship the **Enterprise Control Plane** as a layered extension of
`atelier-admin` + `atelier-console`, not as a UI rewrite or
framework swap. Four building blocks, deployable independently:

### Block 1 — License-Gate (`core/license/`)

A new plugin in this repo, **Apache-2.0**, mounted onto the Gateway
ASGI app at `/v1/license/*`. Validates RS256-signed JWT license
keys against a pinned public key. Responsibilities:

- Read `<atelier_home>/global/license/license.jwt` at boot
- Validate signature, expiry, and structural claims
  (`customer_id`, `tier`, `employee_count_max`, `valid_until`,
  `seats`, `feature_flags`)
- Expose `/v1/license/status` (GET, unauthenticated except for
  customer-id redaction) for both `atelier-admin` and
  `atelier-console` to read
- Implement 30-day grace period after expiry: mutations refuse,
  GET endpoints stay open (the "no operator-lockout" invariant)
- Free-Tier semantics: **no license file = unlimited Apache-2.0
  edition.** A user without `license.jwt` is the documented default;
  no feature blocks. The gate only fires when an installed license
  is *expired or revoked*.

Audit events: `license.activated`, `license.expired`,
`license.grace_started`, `license.violated`, `license.revoked`.

### Block 2 — Compliance-Report-Generator (Phase E3 of admin + console)

PDF generation lives in both `atelier-admin` and `atelier-console`
as new routes. Library: `weasyprint` (pure Python, no headless
Chrome). Initial three reports:

| Report | Source | Format |
|---|---|---|
| EU AI Act Art. 50 Evidence | `disclosure.*` + `consent.*` audit events, time-window filtered | PDF with chain-hash signature + per-event details |
| GDPR Art. 30 Records of Processing | Per-tenant rendering of every distinct `engine_id`, `compliance_zone`, `data_handle`, plus retention policies | PDF following the GDPR RoPA template |
| Audit-Chain Integrity Attestation | `voice-audit verify` run + signed hash of last + first event | PDF that a Wirtschaftsprüfer can stamp |

Templates live in `atelier-compliance-reports/templates/*.html`
with Jinja2 rendering. Per-tenant theming via `branding.yaml`.

License-gate: report generation is **free in the open-core**
(transparency MUST be free — a compliance feature behind a paywall
would be ethically wrong AND a regulatory red flag). The
*Enterprise plugin* adds value on top: bulk export, scheduled
generation, custom report layouts, archival to WORM storage.

### Block 3 — Enterprise Plugin (`atelier-enterprise`, separate repo)

A **separate, proprietary repository** authored by the Maintainer.
Distributed as a signed `.atelier-pkg` (ADR-0007 Phase 5). Apache-
core users never see this code; they get an "Upgrade to
Enterprise" hint in the UI when they bump into a gated feature.

Features:

- **Compliance Reports — Premium**: scheduled monthly generation,
  signed PDF with operator-key, custom template upload, multi-
  tenant batch
- **Cross-Tenant Audit-Search**: federated search across every
  tenant chain (operator must hold both per-tenant tokens AND the
  enterprise license)
- **SSO Setup Wizard UI**: visual editor for OIDC/SCIM
  configuration that ADR-0007 Phase 3.4–3.6 backed
- **Audit-Chain WORM Export**: S3-compatible cold-storage
  integration with hash-anchor verification on re-import
- **SLA Monitoring Dashboard**: uptime, latency p50/p95/p99 per
  tenant, alert thresholds
- **Support Integration**: ticket creation from inside the UI,
  attaches audit-chain snippets + recent run-IDs automatically
- **White-Label Branding UI**: web editor for `branding.yaml`
  (logo, colours, custom disclosure paragraph)

Licence: a custom commercial licence with a Docker-Desktop-style
size gate:

> *Free for personal use, organisations under 50 employees AND
> €5M annual revenue. Larger organisations require a commercial
> licence starting at €15,000/year per instance.*

### Block 4 — Cloud Operations (`atelier-cloud-ops`, separate repo)

Ops-only repository (Helm values, Stripe-webhook handlers,
customer-onboarding scripts). NOT a product the user installs —
the operational glue behind `atelier.cloud`. K8s manifests reuse
the Helm chart from ADR-0007 Phase 4 verbatim. EU-region hosting
on Hetzner / IONOS / Cloud-Sovereign-DE.

Tiers (already declared in marketing strategy 2026-05-15):

| Tier | Price | Limits |
|---|---|---|
| Starter | ~30 €/month | 1 tenant, 100 bots, 50k LLM-calls |
| Business | ~300 €/month | 5 tenants, 1000 bots, 500k calls, SSO |
| Enterprise | from 50k €/year | unlimited, SLA, CSM, attestation |

---

## Surface — phase breakdown

| Phase | Block | Inhalt | Apache-2.0 / Proprietary | Est. effort (solo) |
|---|---|---|---|---|
| **I** | atelier-admin | 14.3 Cross-Tenant-Search-Backend, 14.4 Aggregierte Compliance-Dashboards, 14.5 Metrics-UI-Embedding, 14.6 Closure | Apache | 4–6 weeks |
| **II** | atelier-admin + atelier-console | Compliance-Report-Generator (3 PDF types) via weasyprint | Apache | 3–4 weeks |
| **III** | atelier-license | License-gate plugin (JWT validation, grace, audit events, UI page) | Apache | 2–3 weeks |
| **IV** | atelier-console | UX-Polish: Pico-CSS / Tailwind-CDN, dark-theme, mobile-responsive, i18n DE+EN, white-label-binding | Apache | 3–4 weeks |
| **V** | atelier-enterprise | Initial commercial-feature set (premium reports, cross-tenant-search-UI, SSO wizard, WORM export, SLA dashboard) | **Proprietary** | 4–6 weeks |
| **VI** | atelier-cloud-ops | K8s + Helm + Stripe + customer onboarding | Internal | 8–12 weeks |
| **VII** | marketing | Landing page, pricing page, whitepaper, demo video, HN-coordination | — | 2–3 weeks |

**MVP-saleable** = Phases I + II + III + V (subset) → ~13–19 weeks
solo. Phases IV, VI, VII parallelisable with additional capacity.

---

## Compliance posture (load-bearing)

Same backbone as ADR-0015 — Enterprise Control Plane never
weakens a single compliance mechanism:

| Mechanism | Enterprise plane preserves it by |
|---|---|
| EU AI Act Art. 50 (active disclosure) | Disclosure card stays structurally locked; commercial UI only READS disclosure state |
| GDPR Art. 6/7 (consent gate) | All consent paths route through `consent.py`; no UI shortcut |
| GDPR Art. 30 (RoPA / audit chain) | New PDF report IS the RoPA; chain integrity unchanged |
| GDPR Art. 32 (security of processing) | License-gate uses RS256 + pinned public key; no new password/secret material; path-gate covers `<atelier_home>/global/license/` automatically (new protected subtree) |
| EU AI Act Art. 14 (human oversight) | All Enterprise-only mutations emit `console.action_performed` / `admin.action_performed` events; nothing runs without a chain trace |
| EU AI Act Art. 14 / GDPR Art. 32 (zone residency) | `atelier.cloud` defaults `compliance_zone = "eu-central"`; zone-residency gate stays authoritative |

### License-gate audit events (registered in `EVENT_SEVERITY`)

```
license.activated      INFO      customer_id (hash), tier, valid_until, employee_count_max
license.expired        WARNING   customer_id (hash), expired_at
license.grace_started  WARNING   customer_id (hash), grace_ends_at
license.violated       WARNING   reason (curated: signature-invalid, expired, employee-count-claim-exceeded, seats-exceeded)
license.revoked        WARNING   customer_id (hash), reason
```

Audit-emitter validates against per-event allow-list at the
boundary; cleartext customer-id (the full UUID), license body,
or signing material NEVER appear in chain details. The
`test_license_audit_metadata_only` regression gate enforces it.

### License-gate path-gate protection (Layer 10 extension)

`<atelier_home>/global/license/license.jwt` and the public-key
file `<atelier_home>/global/license/pubkey.pem` are added to
the protected-path set in `operator/voice/hooks/path_gate.py`.
The LLM cannot Write/Edit/Bash-redirect into these files —
license-management is operator-only via the
`atelier-license-cli` tool.

---

## Consequences

### What this enables

- **Sustainable revenue without abandoning open-core principles.**
  Apache-2.0 community keeps every load-bearing feature
  (bridges, engines, forge, skill-forge, audit, voice, all 28
  layers). Premium UI features pay for ongoing development.
- **Acquisition-ready product.** A buyer (strategic, e.g. IONOS /
  OneTrust / a banking consortium) sees: open-source brand +
  paying customer book + recurring ARR + Relicense-Right via
  CLA §3 = standard SaaS-acquisition diligence shape.
- **Compliance-officer-ready.** The PDF reports ARE the artefact
  every Mittelstand legal team needs for an AI-Act audit. Today
  no competitor ships them. The Enterprise plugin's *premium*
  variants (scheduled, signed, archived) lock in mid-market.
- **Hosted-cloud path.** `atelier.cloud` is the easy-button for
  operators allergic to Debian server administration. Same code
  as self-hosted, no fork.
- **Multi-region EU-hosting USP.** Stating "EU-Central by
  default, no US cloud" in the cloud tier is a hard
  differentiator versus Hermes / OpenClaw / Claude Agent SDK.

### What it does NOT do (deliberately)

- **No license-gate on the open-core kernel.** Bridges, engines,
  forge, skill-forge, audit-chain, voice, layers 1–28 are
  unconditionally Apache-2.0. No employee-count claim, no
  payment trigger, no telemetry phone-home.
- **No telemetry from open-core deployments.** The Apache edition
  never reports usage statistics to atelier.cloud. The license-
  gate is local-only; the JWT is validated against an embedded
  public key without a network call. Operators auditing the
  source see this immediately.
- **No paywall on transparency.** Compliance-Report-Generator
  ships in `atelier-admin` + `atelier-console` (Apache, free).
  The Enterprise plugin adds *premium* variants (scheduling,
  custom layouts, bulk batch); a regulator-defensible RoPA stays
  one click away for every operator.
- **No replicated UI surface.** The Enterprise plugin extends
  the existing admin + console SPAs via the gateway's plugin
  mount mechanism (ADR-0007 Phase 5). No second frontend codebase.
- **No build-step.** Vanilla-JS contract from ADR-0014 + 0015
  carries through. CSS-layer additions (Pico / Tailwind-CDN) and
  i18n JSON files only.
- **No JWT signing in this plane.** The license-gate verifies
  signatures against a pinned public key. Signing happens in the
  Maintainer's offline license-issuer process, outside any
  AtelierOS-deployment surface.
- **No automatic upgrade-nag.** The "Upgrade to Enterprise" hint
  is a single static link in the UI, no popups, no
  notifications, no email-capture forms. Apache users never
  experience the open-core as a nagware funnel.

---

## Architecture summary

```
core/
├── license/                          ← NEW (Apache)
│   ├── bootstrap.sh
│   ├── requirements.txt              ← PyJWT, cryptography
│   ├── atelier_license/
│   │   ├── __init__.py
│   │   ├── app.py                    ← /v1/license/status, /v1/license/activate
│   │   ├── verifier.py               ← JWT verify + claims validation
│   │   ├── grace.py                  ← grace-period state-machine
│   │   ├── audit.py                  ← 5 license.* event emitters
│   │   └── cli.py                    ← atelier-license-cli install/show/revoke
│   └── tests/
│
├── admin/                            ← extended (Apache, 0014)
│   └── atelier_admin/routes/
│       ├── compliance_reports.py     ← NEW (Phase II)
│       ├── cross_tenant_search.py    ← Phase 14.3
│       └── ...
│
└── console/                          ← extended (Apache, 0015)
    └── atelier_console/routes/
        ├── compliance_reports.py     ← NEW (Phase II)
        └── ...

# Separate repos:
atelier-enterprise/                   ← NEW (proprietary)
└── (commercial features, distributed as signed .atelier-pkg)

atelier-cloud-ops/                    ← NEW (internal, ops-only)
└── (K8s + Helm + Stripe glue for atelier.cloud)
```

### Wire-format summary (license-gate)

| Method | Path | Purpose |
|---|---|---|
| GET  | `/v1/license/status`        | Tier, expiry, employee_count_max, feature_flags (cleartext fields only — customer_id redacted) |
| GET  | `/v1/license/healthz`       | License-gate liveness (unauth) |
| POST | `/v1/license/activate`      | Install a license.jwt file (operator-CLI-mediated; web-UI calls require admin-token + CSRF + Re-Auth) |

The Enterprise plugin extends both admin + console SPAs with
additional routes mounted under `/v1/admin/enterprise/*` and
`/v1/console/enterprise/*`. The mount only succeeds when
`/v1/license/status` returns `tier in {pro, business, enterprise}`;
otherwise the plugin self-disables silently and the UI shows
"Upgrade to Enterprise" placeholders.

---

## What you, as Claude Code, must NOT do (enterprise rules)

- **Don't put the license-gate logic into the open-core gateway.**
  It lives in a dedicated plugin so a fork that wants to remove
  it can do so with `rm -rf core/license/` and remain
  fully functional under Apache-2.0. Moving the gate into
  `atelier-gateway` would make the Apache edition depend on a
  commercial-defence mechanism — a contradiction.
- **Don't make the license-gate phone home.** Verification is
  local-only against the pinned public key. A network call would
  (a) break air-gapped deployments and (b) raise legitimate
  privacy concerns from open-core adopters.
- **Don't gate compliance-report generation on the license.**
  Transparency is a structural feature — every operator must be
  able to produce a regulator-defensible audit at zero cost.
  The Enterprise plugin extends the reports (premium templates,
  scheduling, archival), it does not gate them.
- **Don't put the Enterprise plugin's code into this repo.** It
  lives in a separate proprietary repository so the licence-
  boundary stays unambiguous. Apache-2.0 contributors cannot
  accidentally see or modify proprietary code; proprietary code
  cannot accidentally inherit Apache copyright via co-location.
- **Don't bypass the path-gate on `<atelier_home>/global/license/`.**
  License files are operator-only via `atelier-license-cli`.
  Any Write/Edit/Bash that would mutate the file from inside the
  LLM context is blocked structurally — same as the audit chain,
  forge workspace, policy.json (Layer 10 extension).
- **Don't add per-feature license checks scattered through the
  codebase.** The pattern is: the Enterprise plugin's mount
  decides at boot whether to register its routes. Once mounted,
  routes run without per-call license re-checks (the mount IS
  the check). Per-call checks would let a clever fork strip them
  one-by-one; the mount-time check is structurally honest.
- **Don't introduce telemetry into the Enterprise plugin without
  an opt-in setting in `branding.yaml` or `tenant.atelier.yaml`.**
  Even premium customers expect on-prem privacy. The cloud
  edition is a different deployment context where telemetry is
  acceptable — but the self-hosted Enterprise plugin must default
  to no-phone-home.
- **Don't ship `atelier-enterprise` as a Docker image with embedded
  source.** Distribution format is signed `.atelier-pkg` (ADR-0007
  Phase 5) installed atop the Apache base. The package contains
  the plugin's Python wheel + signed manifest; the Apache base
  image stays uncontaminated.
- **Don't ship the cloud Stripe-webhook signing secret in any
  customer-facing artefact.** It lives only in
  `atelier-cloud-ops/`, deployed as a Kubernetes Secret. The
  Helm chart from ADR-0007 Phase 4 must NOT inline it.

---

## References

- ADR-0007 — multi-tenant gateway (auth, OIDC, SCIM, .atelier-pkg, Helm)
- ADR-0010 — operator-observability surface (Phase 6 metrics)
- ADR-0014 — atelier-admin operator-UI (Apache, sibling)
- ADR-0015 — atelier-console owner-UI (Apache, sibling)
- `CLAUDE.md` § "Licensing baseline" — Apache-2.0 + CLA §3 contract
- `CLAUDE.md` § "Compliance baseline" — load-bearing structural posture
- `outputs/atelier-console-konzept.md` — original UI design document
- Marketing strategy conversation 2026-05-15 — Open-Core decision,
  Docker-Desktop license model, three-phase Go-to-Market plan
