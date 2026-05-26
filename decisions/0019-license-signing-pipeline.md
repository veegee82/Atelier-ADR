# ADR-0019 — License-Signing & Distribution Pipeline

**Status:** Proposed (2026-05-15)
**Predecessor:** ADR-0017 (Enterprise Control Plane), ADR-0018
(Public-Launch Readiness) — specifically Workstream D.3 (Cloud
pre-launch features).
**Sister-ADRs:** 0007 (multi-tenant gateway), 0014 (atelier-admin),
0015 (atelier-console).
**Owner of decision:** Silvio Jurk (Maintainer / future GmbH-
Geschäftsführer).

---

## Context

ADR-0017 Phase III shipped a **structurally sound license-gate**:
RS256 JWT, embedded pinned public key, 30-day grace state machine,
metadata-only audit emission, fail-silent mount pattern.

ADR-0018 §A.2 covers the one-shot maintainer action that pins the
public key and produces the first evaluation license.

**Neither of these solves the core commercial flow:**

> *Customer pays via Stripe → license is signed → license is
> delivered → customer installs it → audit-chain records the
> activation.*

Today every step from "Stripe webhook fires" to "customer holds a
working `license.jwt`" is manual:

1. `atelier-cloud/billing/webhook_handler.py` receives the
   `checkout.session.completed` event.
2. The handler calls `ops/tenant_provisioner.py`, which calls
   `POST /v1/admin/tenants` on the gateway. A tenant exists.
3. **Gap.** No code path generates a signed JWT. No code path
   delivers it. No code path correlates the Stripe subscription
   to a license-validity window.

The maintainer would today have to:
- be paged on every signup,
- run `atelier-license issue` manually on the offline signing box,
- copy the JWT bytes to an email (with the private-key host trust
  boundary intact, this means no automated tooling),
- send the email to the customer,
- hope the customer's email provider doesn't strip the attachment.

That workflow does not scale beyond ~5 customers/week and is not
acceptable for any customer-facing claim of "self-service signup."
It also creates a single-point-of-failure: a maintainer holiday =
new customers cannot activate.

This ADR designs the pipeline that closes this gap while
**preserving the structural lock that the maintainer private key
never reaches an internet-connected host**.

---

## Decision

Build the License-Signing & Distribution pipeline as **three
loosely-coupled stages connected by a durable queue**, with the
private key isolated to an **air-gapped signing host** that polls
the queue over a one-way channel. Customer-facing delivery happens
via two parallel channels (email + customer-portal download) so a
failed email never strands a paying customer.

The pipeline ships in **three phases of increasing automation**:

| Phase | Signing trigger | Maintainer touch | Throughput | Target |
|---|---|---|---|---|
| **1 — Manual-with-Notify** | Maintainer runs CLI on offline box | Per-license | ~5 / week | Week 1–2 post-launch |
| **2 — Operator-Review-UI** | Maintainer clicks "sign" in air-gapped web UI | Per-license, < 60 s each | ~50 / week | Week 3–8 post-launch |
| **3 — Hardware-Token Auto-Sign** | YubiKey PIV + rule-based auto-approval | Exception-only | ~unlimited | Month 3+ |

**Each phase must be production-shippable on its own.** The
pipeline never moves backwards; Phase 2 is layered on top of Phase
1's persistence, Phase 3 on top of Phase 2's review UI.

The pipeline lives in two repositories:

| Component | Repo | License |
|---|---|---|
| Webhook receiver, queue producer, mail/portal delivery | `Atelier-Cloud` | Apache-2.0 |
| Tier→Flag map, license-spec schema, public-key embedding | `AtelierOS` (open-core) | Apache-2.0 |
| Air-gapped signing host tooling (queue consumer + signer) | new repo `Atelier-Signer` | All Rights Reserved (proprietary) |

`Atelier-Signer` is a **new, separate repository** because its
threat model is fundamentally different: it touches the private
key, and its codebase must be small enough that a single
maintainer can read every line before each release. Pulling it
into either of the existing repos would dilute that property.

---

## Architecture

```
┌───────────────────────────────────────────┐         ┌─────────────────────────────────────────┐
│  Internet-facing tier (Atelier-Cloud K8s) │         │  Air-gapped signing host (offline)      │
│                                           │         │                                         │
│  ┌──────────────────────────┐             │         │   ┌──────────────────────────────┐      │
│  │ stripe-webhook-receiver  │ produces    │   poll  │   │ atelier-signer (queue consumer)│    │
│  │ (atelier_cloud.billing)  │────────────▶│ S3/MinIO│◀──│ + RS256 signer (PyJWT)         │    │
│  └──────────────────────────┘   sign-req  │ (read   │   │ + YubiKey PIV (phase 3)        │    │
│           │                                │  only)  │   └──────────────┬───────────────┘    │
│           ▼                                │         │                  │ writes signed JWT  │
│   ┌──────────────────┐                     │         │                  ▼                    │
│   │ tenant-provisioner│  (creates tenant)  │         │   ┌──────────────────────────────┐    │
│   │   POST /v1/admin/tenants               │         │   │ outbound queue (S3/MinIO,    │    │
│   └──────────────────┘                     │         │   │   write-only from signer)    │    │
│           │                                │  poll   │   └──────────────┬───────────────┘    │
│           ▼                                │   ◀─────│                  │                    │
│  ┌────────────────────────────┐            │  signed │                                       │
│  │ license-delivery-service   │   consumes │   JWT   │                                       │
│  │ ┌────────────────────────┐ │            │         │                                       │
│  │ │ mailgun-template-render│ │            │         │                                       │
│  │ │ + portal /v1/licenses/me│ │           │         │                                       │
│  │ └────────────────────────┘ │            │         │                                       │
│  └────────────────────────────┘            │         │                                       │
└───────────────────────────────────────────┘         └─────────────────────────────────────────┘
```

**The one-way property is the load-bearing structural rule.**
The signing host **polls** the inbound queue (sign-requests) and
**writes** to the outbound queue (signed JWTs). It never accepts
inbound network connections, never holds an outbound TCP session
open, never executes code shipped via the inbound queue. The
inbound items are bounded YAML/JSON documents that the signer
validates against a strict schema before passing to the signing
function.

The S3/MinIO buckets sit on a network the signing host can reach
(e.g. a separate VLAN), but the bucket credentials granted to the
signing host are **read-only on the inbound bucket** and
**write-only on the outbound bucket** — IAM enforced. A
compromised signing-host process cannot delete past sign-requests
nor read past sign-responses other than its own current item.

---

## Phase 1 — Manual-with-Notify (Week 1–2)

**Goal:** First paying customer can self-serve, but the
maintainer is in the loop for each signing.

### Components

| Component | Status today | Phase 1 delta |
|---|---|---|
| Stripe webhook receiver | exists (`atelier_cloud.billing`) | Add `invoice.payment_succeeded` handler that enqueues a sign-request |
| Sign-request queue | does not exist | New: S3 bucket `atelier-sign-requests` (write-only from cloud, read-only from signer) |
| Tier→Flag map | does not exist | New: `core/license/atelier_license/tier_flags.py` — single source of truth, used by both `cli.py::_cmd_issue` and the signer |
| Signing host tooling | does not exist | New repo `Atelier-Signer`: a single-file CLI `signer sign-pending` that downloads pending requests, opens each in `$EDITOR` for maintainer review, signs on Y, drops on N, uploads signed JWTs |
| Delivery — email | does not exist | New: Mailgun template + sender service, triggered by signer's outbound queue |
| Delivery — portal | does not exist | New: `GET /v1/licenses/me` on the gateway (auth: customer bearer token issued by Stripe-checkout flow), returns the latest signed JWT for the calling customer |
| Audit chain | partial (license.* events exist) | Add `cloud.license_requested`, `cloud.license_signed`, `cloud.license_delivered`, `cloud.license_delivery_failed` |

### Tier→Flag Map (open-core, ADR-0017 §III follow-up)

Hard-coded mapping in `core/license/atelier_license/tier_flags.py`:

```python
TIER_FLAGS: dict[str, frozenset[str]] = {
    "free": frozenset(),
    "pro": frozenset({"compliance_reports_premium"}),
    "business": frozenset({
        "compliance_reports_premium",
        "sso_wizard",
        "support_integration",
    }),
    "enterprise": frozenset({
        "compliance_reports_premium",
        "cross_tenant_search",
        "sso_wizard",
        "worm_archive",
        "sla_dashboard",
        "support_integration",
        "white_label_ui",
    }),
}
```

The signer **must reject** sign-requests whose `feature_flags` set
is not exactly `TIER_FLAGS[tier]`. The `cli.py::_cmd_issue` is
amended to refuse arbitrary `--flags` and instead derive them
from `--tier`. The structural property: **a tier name
deterministically determines the flag set, no exceptions**.

This closes the inconsistent-license risk identified in the
open-core audit (Maintainer could issue `pro`-tier with all
enterprise flags by accident).

### Sign-request envelope (the queue protocol)

```yaml
apiVersion: atelier-signer/v1
kind: SignRequest
metadata:
  request_id: "sr_<22-char-urlsafe>"
  stripe_event_id: "evt_..."         # idempotency + audit correlation
  enqueued_at: "2026-05-15T14:32:00Z"
spec:
  customer_id: "cus_..."             # Stripe customer
  tenant_id: "acme"
  tier: "business"
  valid_from: "2026-05-15T14:32:00Z"
  valid_until: "2027-05-15T14:32:00Z"
  employee_count_max: 50
  seats: 10
  email: "ops@acme.example"
  request_signature: "<hmac-sha256>"  # cloud→signer authentication
```

The signer verifies `request_signature` against a **separate
shared secret** (not the Stripe webhook secret) that lives only
on the cloud-side webhook receiver and on the signing host. The
signing host **never** accepts a request whose HMAC is invalid,
even if it was successfully placed in the bucket.

### Phase 1 acceptance criteria

- [ ] Tier→Flag map landed in open-core + tested.
- [ ] `cli.py::_cmd_issue` refuses arbitrary `--flags` (regression
  gate: test that `--tier pro --flags worm_archive` exits non-zero).
- [ ] Stripe webhook handler produces sign-requests on
  `checkout.session.completed` + `invoice.payment_succeeded`.
- [ ] `Atelier-Signer` repo created with a single-file CLI that
  reads pending requests, prompts maintainer, signs on Y.
- [ ] Signed JWT is delivered to customer via Mailgun (real send
  to a test mailbox, not stubbed).
- [ ] `GET /v1/licenses/me` returns the latest signed JWT for an
  authenticated customer.
- [ ] Four new audit events are registered in `EVENT_SEVERITY` and
  written through `forge.security_events.write_event`.
- [ ] End-to-end test: real Stripe test-mode subscription →
  webhook → request enqueued → signer (in CI mode with a
  test-only pubkey) signs → email + portal both deliver → tenant
  CLI accepts the JWT.

### Phase 1 effort: ~2 weeks solo

---

## Phase 2 — Operator-Review-UI (Week 3–8)

**Goal:** Maintainer reviews and signs from anywhere via a
web UI, but the private key still lives only on the air-gapped
host.

### Architecture delta

The signing host gains an HTTPS-over-Wireguard sidecar
(`atelier-signer-ui`) that exposes a minimal React-free SPA
where the maintainer:

- Sees pending sign-requests in a table, sorted by enqueue time.
- Clicks one to see the full request envelope + Stripe subscription
  details (fetched from the cloud via a read-only API).
- Approves → the signing host signs and uploads to the outbound
  queue. Rejects → the request is moved to a `rejected/` prefix
  with a reason.

The UI is reachable **only via a Wireguard tunnel** from the
maintainer's laptop (or phone). The tunnel terminates on a small
firewall in front of the signing host; the host itself stays
network-isolated.

### Phase 2 acceptance criteria

- [ ] Wireguard tunnel + UI deployed.
- [ ] Maintainer can approve a signing request from a phone.
- [ ] Average wall-clock from webhook to delivered JWT < 5 min
  during European business hours.
- [ ] Rejected requests notify the customer via email with the
  rejection reason (privacy-safe — operator-curated reason set:
  "payment-not-verified", "manual-review-required", etc.).

### Phase 2 effort: ~3–4 weeks solo

---

## Phase 3 — Hardware-Token Auto-Sign (Month 3+)

**Goal:** Signing happens autonomously when the request matches a
maintainer-approved rule set; only exceptions are paged.

### Architecture delta

The signing host gains an unattended-signer mode that:

- Consumes pending requests in a loop.
- Validates each against an **auto-approval ruleset** (e.g. "tier
  in {pro, business} AND employee_count_max ≤ 100 AND Stripe
  customer in good standing for ≥ 7 days").
- Auto-approved requests are signed via the YubiKey PIV slot
  (touch-required by default, configurable per-rule for
  hands-free signing during ramp).
- Non-matching requests are queued for the Phase 2 manual review.

The YubiKey holds the RS256 private key. The signer talks to it
via `python-ykman` (or pkcs11). The key never appears in process
memory in extractable form.

### Phase 3 acceptance criteria

- [ ] YubiKey PIV slot loaded with the RS256 private key (key
  generation happens **on the device**, not on a host — the only
  copy outside the device is on a printed paper backup in a safe).
- [ ] Auto-approval ruleset documented + version-controlled in
  the `Atelier-Signer` repo.
- [ ] Median wall-clock from webhook to delivered JWT < 30 s for
  auto-approved tier.
- [ ] All auto-approvals are auditable: each signed JWT carries a
  `meta.ruleset_version` claim that ties it to a git commit hash
  of the rule set in effect at signing time.

### Phase 3 effort: ~2–3 weeks solo (after Phase 2 is stable)

---

## Renewal & Revocation

### Renewal (subscription period extends)

Triggered by `invoice.payment_succeeded` on an existing
subscription. The webhook receiver enqueues a sign-request with a
new `valid_until` and a `prior_license_jti` field. The signed JWT
carries a `replaces_jti` claim. Customer-side `atelier-license
install` accepts the new license and emits `license.activated`
with `replaces: <old-jti>` in the details.

The old JWT remains valid until its natural `exp` — the customer
has the full grace period to swap, which avoids race conditions
during long-running operations.

### Revocation (subscription canceled, payment failed, fraud)

Per ADR-0017's must-NOT rule: **no phone-home, no online CRL**.
Revocation works by **issuing a short-validity replacement**:

- `customer.subscription.deleted` → enqueue a sign-request with
  `tier="free"` and `feature_flags=[]`. The customer's next
  `license.activated` shows `tier: free`, which structurally
  disables all premium features per the Tier→Flag map.
- `invoice.payment_failed` → enqueue a sign-request with the
  same tier but `valid_until = now + 14d` (dunning grace).

Premium features are disabled deterministically when the customer
either installs the new free-tier JWT or when the old commercial
JWT expires — whichever comes first. The Apache-core stays
unconditionally functional.

This sidesteps the entire "how do we revoke a JWT" problem class
(no CRL, no OCSP, no phone-home): we **replace the JWT** instead
of revoking it, and the natural-expiry path enforces the limit.

---

## Security Considerations

1. **Maintainer-Privkey isolation.** The private key never
   leaves the air-gapped signing host (Phase 1–2) or the
   YubiKey (Phase 3). The cloud-side has zero access. A full
   compromise of the cloud Kubernetes cluster cannot mint a
   single license.

2. **Sign-request authentication via separate HMAC.** Even if a
   Stripe webhook secret leaks, an attacker cannot inject
   fraudulent sign-requests; they would also need the
   `signer-hmac` secret which lives only on the webhook
   receiver and the signing host.

3. **Bucket IAM hardening.** Read-only inbound credentials,
   write-only outbound credentials. The signing host's IAM
   policy denies `s3:DeleteObject` on the inbound bucket. A
   compromised signer process cannot tamper with past requests.

4. **Outbound queue write-only-once.** Signed JWTs are uploaded
   to keyed paths `<request_id>/license.jwt` with a deny-overwrite
   bucket policy. A second signing attempt for the same request
   fails — preventing accidental double-issuance.

5. **Audit-chain anchoring.** Every signed JWT's
   `cloud.license_signed` event carries a sha256 of the JWT bytes.
   The cloud-side `cloud.license_delivered` event references the
   same sha256. An after-the-fact reconciliation script can prove
   "every signed JWT was delivered exactly once".

6. **Tier→Flag enforcement at sign time AND verify time.** The
   signing host rejects requests with off-tier flag sets. The
   open-core verifier additionally validates the flag set against
   the embedded Tier→Flag map at install time. A signing-host
   compromise that ignores the rejection still produces JWTs that
   the open-core refuses to verify — defence-in-depth.

7. **Mail delivery is best-effort, portal is canonical.** The
   delivered-email path is convenience. The truthful canonical
   source is `GET /v1/licenses/me` on the gateway. A failed email
   never strands a customer because the portal endpoint is always
   reachable with their Stripe-checkout-derived bearer token.

8. **Customer email is not the trust anchor.** The Stripe
   customer-id is. An email-address change in Stripe → the next
   license-renewal still goes to the correct portal endpoint
   (keyed on `cus_X`), not to the new email. Email is for
   convenience notification only.

---

## What this ADR explicitly does NOT do

- **No CRL, no OCSP, no phone-home.** Revocation works via
  short-validity replacement licenses.
- **No browser-side license-installation UX.** Customer copy-pastes
  the JWT from the portal or email into `atelier-license install`.
  A future ADR can layer a browser-managed credential helper on
  top.
- **No multi-tenant license sharing.** Each Stripe customer
  produces exactly one license JWT, bound to one tenant. Customers
  wanting to license multiple tenants need multiple subscriptions
  (or an enterprise volume license — separate workflow, separate
  ADR).
- **No reseller/partner workflow.** Resellers, MSPs, and white-
  label scenarios are out of scope for ADR-0019. The Phase-3
  ruleset is designed so resellers could later be added without
  re-architecting the signer.
- **No support for non-Stripe payment providers.** Adding
  Paddle, Lemon Squeezy, or invoice-based payment is additive —
  each adds a new webhook receiver that produces the same
  sign-request envelope.

---

## Sequencing & critical path

```
Phase 1
 ├─ Open-core: tier_flags.py + cli.py validator         [3 days]
 ├─ Open-core: cloud.* event-types in EVENT_SEVERITY    [1 day]
 ├─ Cloud: webhook handler emits sign-requests           [3 days]
 ├─ Atelier-Signer: minimal CLI signer (new repo)        [4 days]
 ├─ Cloud: Mailgun delivery service                      [2 days]
 ├─ Open-core: GET /v1/licenses/me                       [2 days]
 ├─ E2E test (real Stripe test mode)                     [2 days]
 └─ Maintainer key + ADR-0018 §A.2 (prerequisite!)       [1 day]
                                                        ──────────
                                                          ~18 days

Phase 2  (starts when Phase 1 is in production)
 ├─ Wireguard tunnel + firewall setup                    [3 days]
 ├─ atelier-signer-ui SPA                                [10 days]
 ├─ Read-only cloud API for Stripe context lookup        [2 days]
                                                        ──────────
                                                          ~15 days

Phase 3  (starts when Phase 2 has been stable for 4 weeks)
 ├─ YubiKey PIV provisioning + paper backup              [2 days]
 ├─ Unattended-signer mode + auto-approval ruleset       [7 days]
 ├─ Ruleset documentation + tests                        [3 days]
                                                        ──────────
                                                          ~12 days
```

**Critical path for first-paying-customer:** ADR-0018 §A.2 (pubkey
commit + eval license) → Phase 1 of this ADR. Total: ~3 weeks
solo from today.

---

## Acceptance — when ADR-0019 is "Done"

- [ ] Phase 1 acceptance criteria all met.
- [ ] First production customer signed up via Stripe live mode
  and successfully installed the auto-delivered license.
- [ ] No manual maintainer touch was needed in the happy path.
- [ ] An end-to-end audit-chain query can correlate one Stripe
  subscription → one tenant → one signed license → N audit events
  showing the customer using the licensed features.
- [ ] Phase 2 is at least planned (Wireguard host provisioned).

Phase 2 and Phase 3 graduate on their own acceptance criteria;
ADR-0019 "Done" means Phase 1 is shipped and the Phase 2 + 3
roadmap is committed.

---

## Risks & mitigation

| Risk | Mitigation |
|---|---|
| Signing host downtime delays signups | Phase 1 acceptance includes a manual fallback path ("maintainer runs CLI directly on the host"). Phase 2's review UI works on a phone. Phase 3 has unattended signing. |
| Stripe webhook delivery failures | Stripe retries for up to 3 days. Webhook receiver is idempotent via `stripe_event_id`. A missed webhook can be replayed manually from the Stripe dashboard. |
| Customer never receives email | Portal endpoint is canonical; customer-portal SPA (ADR-0015) gains a "download license" button that hits `GET /v1/licenses/me`. Email is convenience, not the system of record. |
| Signing-host key corruption | Paper backup of the YubiKey PIV slot (Phase 3); off-site cold-storage backup of the RS256 keypair (Phase 1–2). Recovery procedure is documented + drilled before Phase 1 GA. |
| Bucket-level credential leak | IAM permissions are scoped tightly (read-only inbound, write-only outbound, deny-delete, deny-overwrite). A leaked credential cannot reissue past licenses nor read other customers' requests. |
| Tier→Flag drift between open-core and signer | The `tier_flags.py` module is the single source of truth. The signer imports it. CI gate: a test in the open-core verifies that the signer's pinned commit of the module matches the open-core HEAD. |
| Customer disputes ("I never got my license") | The cloud-side audit chain records `license_signed` + `license_delivered` with sha256 of the JWT bytes. Reconciliation script proves delivery. Customer-portal endpoint is always reachable with the Stripe bearer token. |

---

## What you, as Claude Code, must NOT do

- **Don't run the signer on the cloud-facing infrastructure.**
  The whole structural lock depends on the signing host being
  air-gapped. Adding "just a fast path for testing" that signs
  inside the cloud cluster opens the very attack surface this
  ADR is designed to close.
- **Don't allow arbitrary `feature_flags` at sign time.** The
  Tier→Flag map is the contract. A sign-request whose flags do
  not match the tier MUST be rejected. The regression gate is
  `test_signer_rejects_off_tier_flag_set`.
- **Don't introduce phone-home revocation.** ADR-0017's must-NOT
  rule still holds: no CRL, no OCSP, no Maintainer-controlled
  online check. Revocation is short-validity replacement, period.
- **Don't bypass the HMAC on sign-requests.** Even if a request
  arrived in the inbound bucket, the signer MUST verify the
  per-request HMAC against its own copy of the shared secret.
  A Stripe-webhook compromise that puts a forged request in the
  bucket is then still unsigned.
- **Don't add the Stripe webhook secret to the signing host.**
  The signing host MUST NOT have the ability to validate Stripe
  webhooks itself; that's the receiver's job. Splitting the trust
  is intentional — the signer trusts the receiver's HMAC, not
  Stripe directly.
- **Don't send the license JWT to a free-text customer-provided
  email address from the signing host.** The signing host writes
  the signed JWT to the outbound bucket; the cloud-side delivery
  service reads it and dispatches via Mailgun. The signer never
  initiates outbound network connections.
- **Don't merge the `Atelier-Signer` repo into either of the
  existing repos.** Its small, audited-line-by-line property is
  the security argument. Folding it into 50 KLOC of cloud or
  open-core code defeats that.
- **Don't auto-renew a license on every successful Stripe
  charge.** Renewals happen on `invoice.payment_succeeded` for
  the subscription period boundary, not on every monthly charge
  within a period. Otherwise a customer paying monthly for a
  one-year license receives 12 license re-issuances per year,
  each with its own `cloud.license_signed` event, flooding the
  audit chain.
- **Don't change the JWT's `jti` claim semantics.** `jti` is a
  fresh UUID per signed JWT. `replaces_jti` is the renewal back-
  pointer. Reusing `jti` across renewals would break the
  reconciliation property (one signed event = one delivered JWT).
- **Don't expose the inbound or outbound bucket publicly.** Both
  buckets are private. The cloud-side webhook receiver has IAM
  write-access to the inbound bucket. The signing host has IAM
  read-access to the inbound bucket and write-access to the
  outbound bucket. The cloud-side delivery service has IAM
  read-access to the outbound bucket. No other identity (least
  of all `*`) gets any access.

---

## References

- ADR-0007 — Multi-tenant gateway (Phase 5 `.atelier-pkg`
  signing format — similar trust-isolation pattern)
- ADR-0017 — Enterprise Control Plane (Phase III License-Gate
  is the verifier; this ADR is its issuer counterpart)
- ADR-0018 — Public-Launch Readiness (Workstream A.2 is the
  prerequisite, Workstream D.3 is the parent slot for this ADR)
- `core/license/atelier_license/verifier.py` — the
  verifier this ADR's signer is designed to satisfy
- `core/license/atelier_license/cli.py::_cmd_issue` —
  the existing manual signer path, which becomes the Phase 1
  fallback
- `atelier-cloud/billing/webhook_handler.py` — the existing
  webhook receiver, extended in Phase 1
- RFC 7519 — JSON Web Token (the JWT format)
- RFC 7517 — JSON Web Key (the public-key embedding format)
- PKCS#11 / YubiKey PIV — Phase 3 hardware-key access

---

**Status transitions:**

- **Proposed → Accepted:** When the maintainer commits to Phase 1
  acceptance criteria as the next active workstream.
- **Accepted → In-Progress:** When Phase 1 sub-tasks are picked
  up in a session.
- **In-Progress → Phase-1-Shipped:** When the first production
  customer is provisioned via the pipeline.
- **Phase-1-Shipped → Done:** When Phase 1 has been stable for
  4 weeks AND Phase 2 architecture is committed.

The ADR stays open until "Done"; Phases 2 and 3 graduate on their
own acceptance criteria.
