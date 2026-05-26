# AtelierOS — Legal Liability & Risk Mitigation

**Status:** Living document
**Owner:** Silvio Jurk
**Last updated:** 2026-05-26

This document summarises the legal exposure a cloud operator of AtelierOS faces
and the mitigation layers that reduce or cap that exposure. It is input for AGB
drafting, insurance procurement, and architecture decisions — not a substitute
for qualified legal advice.

---

## How users could sue

### 1. Breach of contract (BGB §280)
If a published SLA (e.g. "99.9% uptime") is missed and data is lost as a result,
customers have a contractual damages claim. The exposure scales with the value of
the lost data and the customer's consequential losses.

### 2. GDPR Art. 82 — the highest-risk vector
Any data subject (end user, not just the B2B customer) has a **direct claim for
material and immaterial damages** against the operator if a GDPR violation caused
harm. This right cannot be fully contracted away in B2B terms. Courts increasingly
award non-trivial sums even for loss of control over personal data without proven
financial loss.

### 3. Tort liability / negligence
Demonstrable gross negligence — no backups, known vulnerabilities left unpatched,
no encryption — creates liability outside of contract. "State of the art" security
measures (documented) shift the burden significantly.

### 4. EU AI Act (watch)
Conversational AI systems are not yet classified as high-risk under Annex III.
If that classification changes, stricter product-liability rules apply automatically.
Track regulatory updates; ADR-0057 (compliance monitoring) is the live signal.

---

## Mitigation layers

### Layer 1 — Contractual (mandatory before first paying customer)

**Terms of Service / AGB must include:**

- **"AS IS" disclaimer** — software delivered without warranty of fitness for purpose
- **Liability cap** — operator liability limited to subscription fees paid in the
  preceding 12 months; no liability for indirect loss, lost profit, or data loss
  where the customer failed to maintain independent backups
- **Exclusion of consequential damages** — to the extent permitted by applicable law
- **Force majeure** — IaaS provider outages (Hetzner), DDoS, infrastructure events
  outside operator control
- **Governing law and jurisdiction** — Germany / Berlin; avoids forum shopping

**Example liability cap clause:**
> *"Die Haftung des Anbieters ist der Höhe nach auf den Betrag begrenzt, den der
> Nutzer in den zwölf Monaten vor dem schadensauslösenden Ereignis tatsächlich
> gezahlt hat. Eine Haftung für mittelbare Schäden, entgangenen Gewinn oder
> Datenverlust ist — soweit gesetzlich zulässig — ausgeschlossen."*

**Data Processing Agreement (AVV/DPA) per GDPR Art. 28:**
Required for every B2B customer who processes personal data via AtelierOS.
Must specify: processing purpose, security measures, sub-processors (Hetzner),
deletion obligations, audit rights. Without a signed AVV the operator shares
liability for the customer's GDPR violations.

### Layer 2 — Technical (document as "state of the art")

Documented technical measures reduce gross-negligence exposure and are required
evidence in supervisory authority investigations:

| Measure | AtelierOS status |
|---|---|
| Audit chain + hash verification | Structural (L16) |
| Encryption at rest | Operator deployment responsibility (document in runbook) |
| Encryption in transit (TLS) | Caddy handles this in production stack |
| Access logging | Audit chain covers this |
| Incident response process | ADR-0057 + systemd timer |
| Backup schedule | Must be documented per deployment |
| Sub-processor AVV (Hetzner) | Available in Hetzner customer portal — sign it |

### Layer 3 — Insurance

- **Cyber insurance** — covers notification costs, regulatory fines (partial),
  legal fees, and third-party claims arising from data breaches. Providers:
  Hiscox, Markel, Getsafe Business. Typical cost: €500–2,000/year at startup
  revenue levels.
- **Professional indemnity / IT liability (Berufshaftpflicht)** — covers damages
  from software defects and professional errors. Often bundled with cyber cover.

Neither policy replaces contractual caps, but they fund the defence when a claim
arrives.

### Layer 4 — Organisational

- **Data Protection Officer (DSB):** Mandatory once 20+ people regularly process
  personal data. External DSB: €150–400/month. Required before scaling.
- **Sub-processor list:** Maintain a current list of all sub-processors
  (Hetzner, any email provider, any analytics tool). Customers have the right to
  object; the AVV must allow notification of changes.
- **Breach notification procedure:** GDPR Art. 33 requires notification to the
  supervisory authority within 72 hours of becoming aware of a breach.
  Document who makes that call and how.

---

## Priority action list

| Action | When | Cost estimate |
|---|---|---|
| IT/data-protection lawyer → AGB + AVV template | Before first paying customer | €1,500–3,000 one-off |
| Sign Hetzner AVV | Now — available in customer portal | Free |
| Obtain cyber + IT liability insurance | Before public launch | €500–2,000/year |
| Document backup schedule + encryption runbook | Now | Internal effort |
| External DSB | When team or customer base grows | €150–400/month |

---

## What this document is not

This is a founder's working reference, not legal advice. The actual AGB, AVV,
and insurance contracts must be reviewed and signed off by a qualified lawyer
admitted in the relevant jurisdiction (Germany for the primary legal entity).
