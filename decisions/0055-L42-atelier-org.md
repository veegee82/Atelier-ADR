# ADR-0055 — Layer 42: AtelierOrg — Organisation Actors

**Status:** Draft  
**Date:** 2026-05-25  
**Layer:** L42  
**Depends on:** L16 (audit chain), L18 (roles), L34 (data classification), L36 (erasure), L38 (A2A), L39 (AtelierFed), L40 (AtelierSpace), L41 (capability grants)

---

## Context

Layers 39–41 establish a social graph of personal actors, five publishing domains
per user, and a granular capability-grant system for followers. Together they cover
the individual case well. The missing primitive is the **organisation**.

On every professional social network — LinkedIn, Mastodon, Bluesky — individuals
and organisations are distinct entity types with distinct governance needs:

- Multiple people act under a single shared identity.
- Staff come and go; the organisation's identity must survive turnover.
- Accountability must remain per-person internally while presenting a unified
  actor externally.
- Business-to-business relationships require organisation-level capability grants,
  not just person-to-person ones.
- A single compromised administrator account must not be able to capture the
  organisation's cryptographic identity.

Without a first-class organisation type, teams register fake personal accounts and
share credentials. This destroys accountability (GDPR Art. 5(1)(f)) and creates
an unresolvable audit ambiguity: which human triggered which action?

---

## Decision Drivers

1. **Shared identity, individual accountability** — external parties see one org
   actor; internally every action is traceable to a named member.
2. **Key survivability** — the org's signing key must survive any single member
   leaving, including all admins simultaneously if a key-rotation quorum is reached.
3. **Deny-by-default** — consistent with L16 and L41; new org followers start
   with zero capabilities.
4. **M-of-N governance** — high-stakes operations (key rotation, broad capability
   grants, org dissolution) require co-authorisation by multiple admins.
5. **B2B capability grants** — organisations must be able to grant capabilities
   to other organisations, not just individual actors.
6. **Agent affiliation** — deployed agents/personas must be cryptographically
   verifiable as belonging to an org.
7. **Audit-first** — every membership change, key operation, and capability grant
   enters the L16 hash chain before any state change.
8. **GDPR controller mapping** — the org config must name a responsible natural
   person (GDPR Art. 5(1)(f) accountability).

---

## The Organisation Actor

### ActivityPub Identity

An org actor is an ActivityPub actor with `type: "Organization"`. Its `actor.json`
lives at `https://<instance>/@<org-handle>/` and carries:

```json
{
  "type": "Organization",
  "id": "https://atelier.sh/@acme-corp",
  "preferredUsername": "acme-corp",
  "name": "ACME Corporation",
  "summary": "...",
  "url": "https://acme-corp.example",
  "publicKey": {
    "id": "https://atelier.sh/@acme-corp#main-key",
    "owner": "https://atelier.sh/@acme-corp",
    "publicKeyPem": "-----BEGIN PUBLIC KEY-----\n..."
  },
  "verifiedDomain": "acme-corp.example",
  "affiliatedActors": [
    "https://atelier.sh/@acme-assistant",
    "https://atelier.sh/@acme-data-agent"
  ]
}
```

`verifiedDomain` is set after the domain verification flow (see below).
`affiliatedActors` lists all org-deployed agents/personas.

### Domain Verification

To set `verifiedDomain`, the org owner places a DNS TXT record or a
`/.well-known/atelier-org.json` file on their web domain containing the org
actor's `id`. The verification is checked at claim time and re-checked on
every outbound post signing. A verified org displays a badge in the UI.

### Org Store

Each org lives under the tenant home:

```
<atelier_home>/tenants/<tid>/orgs/<org_handle>/
  actor.json          — public actor document (served by federation layer)
  keypair.json        — private signing key (mode 0600)
  config.json         — org config including responsible_party_id, policy
  members.json        — member roles (mode 0600)
  grants/             — outbound capability grants issued by this org
  audit.jsonl         — org-scoped audit sub-chain (links into L16 main chain)
```

---

## Key Management — Threshold Signing

This is the critical security primitive distinguishing org actors from personal actors.

### Problem

A personal actor's key is owned by one person. An org key shared as a secret
among N admins has N attack surfaces. If a single admin's device is compromised,
the org's entire cryptographic identity is at risk.

### Solution: Delegated Signing Keys with Admin Co-Authorisation

The org maintains a **long-lived root key** and issues short-lived
**Delegated Signing Keys (DSK)** for day-to-day operations:

```
Root Key (master identity)
  ├── never used to sign posts directly
  ├── sharded via Shamir's Secret Sharing across all Owners
  └── used only to sign DSK-issuance certificates

Delegated Signing Key (DSK)
  ├── issued by m-of-n Owners co-signing a DSK certificate
  ├── TTL: configurable (default 7 days)
  ├── signs all routine org posts, domain-related ops
  └── revocable by any Owner (single signature to invalidate)
```

Normal posts and low-stakes grants are signed by the active DSK. The DSK
certificate is embedded in each signed document so verifiers can trace
`post ← DSK ← DSK-cert ← root key` without contacting the org server.

### Operations Requiring Root Key Co-Authorisation (m-of-n)

| Operation | Minimum quorum |
|---|---|
| Issue new DSK | m-of-n Owners (configurable; default 2-of-3) |
| Rotate root key | m-of-n Owners |
| Dissolve org | All Owners |
| Grant `agent.invoke.*` or `forge.exec.*` to external org | n-of-m Admins (default 2-of-3) |
| Grant `data.CONFIDENTIAL.read` to external actor | n-of-m Admins |
| Revoke all grants for an org | 1 Owner (single-signature emergency revoke) |

Pending co-authorisation requests are stored in `config.json::pending_approvals`
and shown in the Org Admin UI as an approval queue.

### Member Offboarding

When a member leaves:

1. Remove from `members.json` → `[L16 audit: org.member_removed]`
2. If the member held an Owner shard: trigger DSK rotation with remaining Owners.
3. The org's public identity is unaffected; the root key is not exposed.

---

## Membership Model

### Roles

| Role | Permissions |
|---|---|
| **Owner** | All operations; holds a root-key shard; can add/remove Owners |
| **Admin** | Issue/revoke grants (subject to M-of-N for broad grants); deploy/remove agents; manage domains |
| **Editor** | Post as org; respond to followers; cannot grant capabilities |
| **Agent** | Machine actor affiliated with the org; can act on behalf of org within its own capability scope |

Roles are stored in `members.json`:

```json
{
  "members": [
    {
      "actor_id": "@alice@atelier.sh",
      "role": "owner",
      "joined_at": 1779732855,
      "key_shard_index": 1
    },
    {
      "actor_id": "@bob@atelier.sh",
      "role": "admin",
      "joined_at": 1779800000,
      "key_shard_index": null
    }
  ]
}
```

### Delegated Authority and the Dual-Signature Audit Trail

When a member acts as the org, every resulting document carries:

- **Outer signature**: org's active DSK (verifiable by any external party)
- **Inner audit record**: member's personal `actor_id` in the L16 audit details

External parties verify the org signature only. Internal audit always records which
member acted. This satisfies both federation-portability and individual accountability.

---

## Agent Affiliation

An org-deployed agent is an ordinary personal actor with two additional fields
in its `actor.json`:

```json
{
  "attributedTo": "https://atelier.sh/@acme-corp",
  "endorsement": "<ed25519-signed endorsement document>"
}
```

The endorsement document is signed by the org's DSK and contains the agent's
actor ID, the org's actor ID, a scope (which capabilities the agent may exercise
on behalf of the org), and a TTL. Verifiers check:

1. Endorsement signature validates against the org's DSK.
2. DSK certificate traces back to the org's root public key.
3. TTL has not expired.

External followers of the org actor implicitly trust all affiliated agents within
the scope defined in the endorsement. The `affiliatedActors` list in `actor.json`
is informational; the cryptographic trust anchor is always the endorsement document.

Affiliation does not grant the agent expanded capability; it establishes identity
attribution. Capabilities are still governed by L41 grants.

---

## B2B Capability Grants

### Org-to-Org Grants

L41 grant documents work unchanged. When the `grantor_actor` is an org actor,
the grant is signed by the org's DSK. When the `grantee_actor` is an org actor,
the grant applies to all agents operating under that org's endorsement:

```
GrantChecker.check(grantee="@acme-data-agent@acme.atelier.sh")
  1. Resolve agent's attributedTo → "@acme-corp@acme.atelier.sh"
  2. Verify endorsement (DSK → root key)
  3. Look up grant where grantee_actor = "@acme-corp@acme.atelier.sh"
  4. Check that the requested capability is within the agent's endorsement scope
  5. Continue normal L41 conditions check (TTL, rate_limit, data_class_ceiling)
```

### B2B Grant Lifecycle

For grants between organisations, the approval queue is visible to both sides:

**Outbound (grantor side):**  
Admin proposes grant → enters pending_approvals → co-admins approve in UI →
on quorum: grant signed with DSK + audit event `org.grant_approved` →
grant delivered to partner org.

**Inbound (grantee side):**  
Grant received → stored in `grants/inbound/` → Admin sees it in Partner Grants UI →
can activate (makes the grant usable by affiliated agents) or reject.

This two-sided acknowledgement is optional for personal-actor grants but
**mandatory** for org-to-org grants — an org cannot be silently granted broad
capabilities by a third party without an admin acknowledging it.

---

## Integration Points

### L39 AtelierFed

Org actors participate in the same ActivityPub inbox/outbox as personal actors.
The federation layer adds one new step for inbound activities from org actors:
resolve the sender's `attributedTo` chain and verify any agent endorsement before
passing to the L41 GrantChecker. No protocol changes to ActivityPub.

### L41 Capability Grants

The GrantChecker gains an `OrgResolver` helper:

- If `grantee_actor` is an org actor → expand to all affiliated agents with
  valid endorsements at check time.
- If `grantor_actor` is an org actor → verify DSK signature chain instead of
  personal Ed25519 key.
- `data_class_ceiling` on a B2B grant applies to all agents under the grantee org.

### L38 A2A RemoteTriggerReceiver

Before honouring `spawn_worker: true` for an origin that maps to an org-affiliated
agent, the GrantChecker checks both the personal-agent grant path and the
org-level grant path. The more permissive path wins, subject to the endorsement
scope ceiling.

### L34 Data Classification

Org actors introduce a new matrix dimension: `actor_type: organization` may have
a stricter default ceiling than `actor_type: person` at operator discretion.
The `data_class_ceiling` in a B2B grant caps what any org-affiliated agent may
access, regardless of individual agent grants.

### L36 Erasure Orchestrator

A new `L42OrgHandler` is added to `real_handler_chain()`. On erasure of
`subject_id`:

- If the subject is an org member: remove from `members.json`; revoke DSK if
  the member held a key shard; trigger DSK rotation audit event.
- If the subject is an org owner (the sole owner): org enters a **suspended** state;
  no new outbound activity until a new Owner is designated.
- If the subject is the org itself: delete the full org store, revoke all
  outstanding grants, emit `org.dissolved` to the L16 chain before rmtree.

### L16 Audit Chain

New org-scoped audit events (all written before state change):

| Event | Details allow-list |
|---|---|
| `org.created` | `org_id`, `owner_actor_prefix` |
| `org.member_added` | `org_id`, `member_actor_prefix`, `role` |
| `org.member_removed` | `org_id`, `member_actor_prefix`, `role` |
| `org.dsk_issued` | `org_id`, `dsk_id_prefix`, `ttl_s`, `quorum_size` |
| `org.dsk_revoked` | `org_id`, `dsk_id_prefix` |
| `org.domain_verified` | `org_id`, `domain_hash` (SHA-256 of domain, not plaintext) |
| `org.agent_affiliated` | `org_id`, `agent_actor_prefix` |
| `org.agent_deaffiliated` | `org_id`, `agent_actor_prefix` |
| `org.grant_proposed` | `org_id`, `grant_id`, `capability_count`, `grantee_type` |
| `org.grant_approved` | `org_id`, `grant_id`, `approver_count` |
| `org.grant_rejected` | `org_id`, `grant_id` |
| `org.dissolved` | `org_id`, `member_count_at_dissolution` |

---

## GDPR Controller Mapping

Every org config carries:

```json
{
  "responsible_party": {
    "actor_id": "@alice@atelier.sh",
    "legal_name_hash": "<SHA-256 of legal name — never plaintext>",
    "declared_at": 1779732855
  }
}
```

`responsible_party.actor_id` is the natural person accountable under GDPR Art. 5(1)(f).
The legal name is stored only as a hash in the config; the plaintext is never in
the audit chain. For regulatory enquiries the operator resolves the hash against
the identity store.

On erasure of the responsible party, the org enters **suspended** state until
a new responsible party is declared — GDPR controller accountability cannot be
vacuous.

---

## UI Model (AtelierSpace — Org Mode)

```
AtelierSpace
  ├── Personal Profile
  ├── Domains
  ├── Grants
  └── My Organisations          ← new entry; switches context

Org Admin View (when org selected):

  ┌─────────────────────────────────────────────────┐
  │  ACME Corporation  ✓ acme-corp.example          │
  │  @acme-corp · 3 members · 2 agents deployed     │
  └─────────────────────────────────────────────────┘

  Tabs: Overview · Members · Agents · Grants · Partners · Audit

  Members tab:
    @alice (Owner)  @bob (Admin)  @carol (Editor)
    [+ Invite member]

  Agents tab:
    acme-assistant  · endorsed until 2026-08-01  [Renew] [Remove]
    acme-data-agent · endorsed until 2026-07-15  [Renew] [Remove]
    [+ Affiliate agent]

  Grants tab (same structure as personal L41 Grants tab, but grantor = org):
    Global (all followers of org):
      domain.blog.read  · no expiry             [Revoke]

    Per-actor:
      @alice@partner.org
        forge.exec.data-analysis  · 100/day · until 2027-01-01
                                        [Edit]  [Revoke]

  Partners tab (org-to-org grants):
    Inbound grants (received):
      From @partner-org@other.instance
        agent.invoke.assistant · 20/day  [Active] [Revoke]

    Outbound grants (issued to other orgs):
      To @acme-corp@client.instance
        data.PUBLIC.read  · no expiry    [Active] [Revoke]

  Pending Approvals (M-of-N queue):
    Bob proposed: grant forge.exec.* to @external-corp
    ✓ Alice approved  · ○ Carol pending   [Approve] [Reject]
```

---

## Alternatives Considered

### A. Use L18 roles directly for organisations

Rejected. L18 roles are local (operator console) and designed for a single user
identity. They have no concept of a shared actor, no key management, and no
federation-portability. Extending them would break the clean separation between
local access control and federated identity.

### B. Multi-key personal actors (one actor, many key pairs)

Rejected. ActivityPub allows only one canonical key per actor. Adding key-rotation
is possible, but shared-key semantics (any one of N keys can sign) would require
custom ActivityPub extensions with no ecosystem support. The Org-actor-with-DSK
model is self-contained.

### C. Relay actor (a bot that reposts on behalf of members)

Rejected. A relay has no cryptographic binding between the relay's posts and the
originating person. Audit accountability is lost. This is exactly the "fake personal
account + shared password" antipattern this ADR eliminates.

### D. ActivityPub Groups (FEP-1b12)

Noted. AtelierOrg is intentionally compatible with the ActivityPub Groups
specification (FEP-1b12) but does not depend on it. The org actor behaves like a
Group actor for external followers (boosting member posts into the org's outbox)
while adding the key-management and capability-grant layers on top. Forward
migration to a native FEP-1b12 Group is straightforward.

---

## Compliance Notes

- **GDPR Art. 5(1)(f)** — the responsible-party mapping ensures there is always an
  accountable natural person behind an org actor. Vacuous accountability is blocked
  by the suspended-state mechanism.
- **GDPR Art. 17** — L42OrgHandler in the L36 chain handles member and org erasure
  with correct state transitions before any data deletion.
- **GDPR Art. 28** — org-to-org B2B grants where one org processes data on behalf
  of another constitute a data-processing-agreement relationship; the two-sided
  grant acknowledgement provides the technical evidence of the arrangement.
- **EU AI Act Art. 14** — `agent.invoke.*` grants require M-of-N admin approval,
  ensuring a human quorum authorises every broad AI-execution grant. The
  endorsement scope ceiling prevents affiliated agents from exceeding their
  sanctioned authority.
- **Audit invariant** — all `org.*` events enter the L16 hash chain BEFORE the
  corresponding state change. Audit details carry only hashed or prefix-truncated
  identifiers, never free-form text.

---

## Open Questions

1. **Key shard storage** — where are Owner key shards stored? Options:
   (a) local encrypted file per Owner (`~/.config/claude-voice/org_shards/<org_id>.enc`),
   (b) hardware token (FIDO2/YubiKey), (c) cloud KMS per Owner. M1 uses (a);
   (b) is the M3 target.

2. **DSK co-authorisation transport** — how do multiple Owners co-sign a DSK
   issuance? Options: (a) out-of-band (each Owner signs locally, signatures
   assembled by the requestor), (b) real-time approval queue in the console.
   M1 uses (a); (b) is the M2 target.

3. **Follower inheritance** — when a user follows an org actor, do they
   automatically follow affiliated agents? Recommendation: no — affiliation is
   identity attribution, not subscription. Users must explicitly follow individual
   agents. This keeps the social graph intentional.

4. **Sub-organisations** — can an org have child orgs (departments, subsidiaries)?
   The model supports it via endorsement chains, but the UI and governance rules
   for hierarchical orgs are out of scope for L42 M1.

5. **Org discovery** — how do users find orgs on remote instances? ActivityPub
   WebFinger works for known handles. A local org directory page (public, opt-in)
   is a potential M2 feature.

---

## Milestones (indicative, not committed)

| Milestone | Scope |
|---|---|
| **M1** | Org store layout, personal DSK (single-owner, no threshold), org actor federation, agent affiliation + endorsement, L16 audit events, L36 handler, L42OrgHandler |
| **M2** | Multi-owner threshold DSK, M-of-N approval queue in console, org Grants tab (per-actor + global), domain verification |
| **M3** | B2B org-to-org grants with two-sided acknowledgement, Partners tab, hardware-token shard storage (FIDO2) |
| **M4** | Sub-organisation hierarchy, ActivityPub Groups (FEP-1b12) alignment, public org discovery directory |
