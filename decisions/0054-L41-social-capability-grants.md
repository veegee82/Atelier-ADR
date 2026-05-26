# ADR-0054 — Layer 41: Social Capability Grants

**Status:** Accepted — M1 implemented 2026-05-26  
**Date:** 2026-05-25  
**Layer:** L41  
**Depends on:** L16 (audit chain), L34 (data classification), L38 (A2A), L39 (AtelierFed), L40 (AtelierSpace)

---

## Context

Layer 39 (AtelierFed) introduces a social graph of actors. Layer 40 (AtelierSpace)
gives every user five publishing domains and a personal profile. Together they create
a surface where remote actors can interact with local resources: they read posts,
reply, boost, send A2A tasks, or trigger forge tools.

The current model has no capability layer between "is a follower" and "can do
anything a follower can do by default." That default set is too coarse:

- Publishing to a followers-only domain should not imply the right to invoke an agent.
- Being a trusted collaborator on one domain should not grant access to another.
- A2A task delivery should be opt-in per origin, not implicit for every follower.

Without explicit, granular grants the operator must choose between two bad options:
keep all content public (over-disclosure) or block all interaction (social graph is
inert). Neither is the right operating point.

---

## Decision Drivers

1. **Minimal scope** — GDPR Art. 5(1)(c) data minimisation applies to capabilities
   as much as to data. An actor should receive exactly the access needed, no more.
2. **Owner control** — the local user must be able to revoke any grant at any time
   with immediate effect, from a single UI surface.
3. **Deny-by-default** — consistent with L16 consent gate and L35 egress lockdown.
   New followers start with zero permissions.
4. **Federation-portable** — a grant must be verifiable by a remote instance without
   contacting the grantor's server on every request. Ed25519 signing (same pattern
   as L39 PostEnvelope) enables this.
5. **Audit-first** — every grant, revocation and capability check that results in a
   deny must be written to the L16 hash chain before any side effect.
6. **No ambient authority** — holding a signed grant proves nothing without the
   social graph confirming the grantee is still a follower. Unfollow = implicit
   revocation of all grants.

---

## The Capability Model

### Capability Identifiers

Capabilities are dot-separated strings in the format `<resource>.<action>`:

```
domain.<slug>.read          Read posts in a specific domain (incl. followers-only)
domain.<slug>.publish       Co-publish into a domain (collaborator)
domain.*.read               Read all domains
domain.*.publish            Publish into any domain

agent.invoke.<persona>      Send a task to a named persona
agent.invoke.*              Send a task to any persona

forge.exec.<tool>           Trigger a specific forge tool by name
forge.exec.*                Trigger any forge tool

social.graph.read           Read the local actor's following/follower list

data.PUBLIC.read            Access public-class data (L34)
data.INTERNAL.read          Access internal-class data (L34 ceiling)
data.CONFIDENTIAL.read      Access confidential-class data (L34 ceiling)

a2a.send                    Deliver a signed A2A TaskEnvelope (L38)
```

Wildcard `*` means "all instances of that resource at the time of the check."
Wildcards are explicit — they must be granted by name; there is no implicit
promotion from a narrow grant to a broad one.

### The Grant Document

A grant is a self-contained, signed JSON document:

```json
{
  "grant_id":       "grnt_<16-char-hex>",
  "schema_version": 1,
  "grantor_actor":  "@silvio@atelier.sh",
  "grantee_actor":  "@alice@other.instance",
  "capabilities":   ["domain.research.read", "agent.invoke.assistant"],
  "conditions": {
    "valid_until":        1800000000,
    "rate_limit":         "20/day",
    "data_class_ceiling": "INTERNAL"
  },
  "issued_at":  1779732855,
  "revoked_at": null,
  "signature":  "ed25519:<hex>"
}
```

The canonical signing payload is `json.dumps(grant_without_signature,
sort_keys=True, separators=(",", ":"), ensure_ascii=True)` — the same
pattern used in L39 PostEnvelope.

`grantee_actor` may be the literal string `"*"` to address all current followers.
A grant to `"*"` is re-evaluated against the follower list at check time; revoking
a follower implicitly removes the benefit of all `"*"` grants for that actor.

### Conditions

| Field | Semantics |
|---|---|
| `valid_until` | Unix timestamp; grant is silently invalid after this point |
| `rate_limit` | `"N/period"` where period ∈ `second, minute, hour, day`; enforced locally |
| `data_class_ceiling` | Caps the highest L34 classification the grantee may access, regardless of what the capability allows |

All conditions are AND-ed. Missing fields mean "no restriction."

---

## Grant Lifecycle

```
Owner issues grant
      │
      ▼
   [L16 audit: grant.issued]
      │
      ├── Grant delivered to grantee out-of-band
      │   (signed JSON attached to a social post, or shared via
      │    a dedicated L39 "grant-envelope" post type — TBD in M2)
      │
Remote actor presents grant
      │
      ▼
   [L41 GrantChecker]
   1. Parse + verify Ed25519 signature against grantor's public key (L39 actor.json)
   2. Confirm grantee actor_id matches the presenting actor
   3. Confirm grantor is the local tenant (cross-tenant grants not accepted)
   4. Confirm grantee is still a follower (unfollow = grant invalid)
   5. Evaluate conditions: TTL, rate-limit counter, data_class_ceiling
   6. Confirm the requested action matches a granted capability
      │
      ├── ALLOW → proceed to L38/L39/Forge handler
      │           [L16 audit: grant.allowed (grant_id, cap, actor prefix)]
      │
      └── DENY  → fail-silent (no reason in HTTP response)
                  [L16 audit: grant.denied (reason in details)]
```

**Revocation:**

```
Owner revokes grant (by grant_id or "all grants for actor X")
      │
      ▼
   revoked_at = now() written to local grant store
   [L16 audit: grant.revoked]
      │
      ▼
   On next capability check the grant fails at step 1 (signature still
   valid but revoked_at is set in local store — remote presentation of
   the signed document is no longer accepted)
```

Revocation is **local-only** and takes effect immediately. There is no
federation-wide revocation broadcast (consistent with ActivityPub's
eventual-consistency model); remote caches of the grant document become
stale at the moment `revoked_at` is set.

---

## Scope Levels

Three addressing scopes for a grant:

| Scope | `grantee_actor` value | Use case |
|---|---|---|
| **Single actor** | `@alice@other.instance` | Trusted collaborator, specific delegate |
| **All followers** | `"*"` | Public follower benefit (e.g. domain.*.read) |
| **Actor group** | `"group:<tag>"` | Team with shared access, managed by owner |

Groups are local tags the owner applies to followers in the AtelierSpace UI
(e.g. "team", "trusted", "press"). Grant to `group:trusted` is evaluated by
expanding the group to its current members at check time.

---

## Integration Points

### L39 Social Inbox

Before delivering an incoming post that requires a capability (e.g. a post to a
followers-only domain, or a `post_type: "task"` envelope):

1. Extract `sender_actor` from the PostEnvelope.
2. Call `GrantChecker.check(grantor=local_tenant, grantee=sender_actor,
   capability="domain.<slug>.read")`.
3. Fail-silent on deny (HTTP 202 always, reason to audit chain only).

### L38 A2A RemoteTriggerReceiver

Before `spawn_worker: true` is honoured for an origin that maps to a social actor:

1. Call `GrantChecker.check(..., capability="a2a.send")`.
2. A missing grant means the envelope is accepted (202) but no worker is spawned.
3. This replaces the current per-origin `spawn_worker` boolean with a finer-grained
   check; the boolean remains as a master switch (if false, no grant can override it).

### L34 Data Classification

`data_class_ceiling` in the grant conditions becomes an additional constraint on
the L34 `DataClassification` matrix check. If a response would carry CONFIDENTIAL
data but the grant ceiling is INTERNAL, the response is downgraded or withheld.

### L36 Erasure Orchestrator

A new `L41GrantHandler` is added to the `real_handler_chain()`. On erasure of
`subject_id`:

- Delete all grants where `grantee_actor` maps to the subject.
- Delete all grants where `grantor_actor` maps to the subject (if the subject is
  the local tenant, this is a full grant store wipe).

---

## UI Model (AtelierSpace — Grants Tab)

```
AtelierSpace
  ├── Profile & Social
  ├── Domains
  └── Grants                    ← new tab

Grants tab layout:

  Global grants (applies to all followers):
  ┌────────────────────────────────────────┐
  │ domain.open-source.read  · no expiry   │
  │                               [Revoke] │
  └────────────────────────────────────────┘
  [+ Add global grant]

  Per-actor grants:
  ┌────────────────────────────────────────┐
  │ @alice@other.instance                  │
  │   domain.research.read  · 01.07.2026   │
  │   agent.invoke.assistant · 20/day      │
  │                    [Edit]  [Revoke all] │
  ├────────────────────────────────────────┤
  │ @bob@third.party                       │
  │   (no grants)              [+ Grant]   │
  └────────────────────────────────────────┘

  Grant templates (quick-add):
    📖 Reader          domain.*.read
    ✍️  Collaborator    domain.*.read + domain.*.publish
    🤖 Agent Delegate  agent.invoke.* + a2a.send
    🔐 Full Trust      *  (requires explicit confirmation)
```

The UI never shows the raw grant document or signature — only the
decoded capability list, conditions, and grant_id prefix. The full
document is available via an operator-only export endpoint.

---

## Alternatives Considered

### A. Role-based (owner/admin/member/observer à la L18)

Rejected. L18 roles are designed for the local operator console, not for
remote social actors. Roles are coarse (four levels) and cannot express
"read domain X but not domain Y" or "run forge tool Z at 10/day."

### B. Per-domain access tokens (opaque bearer tokens)

Rejected. Opaque tokens cannot be verified by a remote instance without
contacting the issuing server on every request. This creates a centralisation
bottleneck inconsistent with the AtelierFed decentralised design.

### C. Extend L39 PostEnvelope with an embedded capability claim

Rejected for the general case. An embedded claim in every post would bloat
the envelope, require re-issuance on every permission change, and blur the
boundary between identity (who you are) and capability (what you may do).
The separate grant document is cleaner. A lightweight reference (`grant_id`)
may optionally be attached to a PostEnvelope as a hint, but verification
always goes through the local grant store.

### D. ActivityPub capability extensions (OCAP, FEP-XXXX)

Noted. The design is intentionally compatible with emerging ActivityPub
object-capability proposals but does not depend on them reaching stable
status. The Ed25519 + JSON approach used here is a strict subset of what
OCAP would require, so forward migration is straightforward.

---

## Compliance Notes

- **GDPR Art. 5(1)(c)** — capability grants enforce data minimisation at the
  federation boundary. Remote actors receive only what they were explicitly granted.
- **GDPR Art. 7** — grant issuance is an affirmative act by the data subject
  (the local user), equivalent to consent. Revocation is always available with
  immediate effect.
- **GDPR Art. 17** — L41GrantHandler in the L36 erasure chain ensures grants are
  deleted when a subject requests erasure.
- **EU AI Act Art. 14** — agent.invoke.* capabilities constitute human oversight
  of which external actors may trigger AI execution. The rate-limit and TTL
  conditions enforce proportionality.
- **Audit invariant** — every grant.issued / grant.revoked / grant.denied event
  enters the L16 hash chain BEFORE the corresponding state change. The audit
  details field carries only `grant_id`, `capability`, and a 16-char prefix of
  `grantee_actor` — never free-form instruction text or response content.

---

## Open Questions

1. **Grant delivery protocol** — how does the grantor communicate the signed
   grant document to the grantee? Options: (a) attached to a social post with
   `post_type: "grant-offer"`, (b) direct HTTP push to grantee's inbox,
   (c) out-of-band (copy-paste). M1 can use (c) for simplicity; (a) is the
   target for M2.

2. **Remote grant presentation** — when a remote actor presents a grant at an
   L38 or L39 endpoint, does it attach the full grant document to every request
   (verbose but self-contained) or reference it by `grant_id` (compact but
   requires the receiver to have cached it)? Recommendation: full document on
   first use, `grant_id` reference on subsequent calls within the same session.

3. **Group management UI** — actor tagging ("trusted", "team") needs a UI surface.
   This could live in the Grants tab or in the Social tab's follower list.

4. **Grant negotiation** — can a remote actor request a capability it does not
   yet have? If yes, this implies a request/approve flow (similar to a follow
   request). Out of scope for L41 M1; noted for M2.

---

## Milestones (indicative, not committed)

| Milestone | Scope |
|---|---|
| **M1** | Local grant store (SQLite), GrantChecker, L16 audit events, L36 handler, out-of-band grant delivery |
| **M2** | AtelierSpace Grants UI tab, grant templates, group tags, L39 inbox integration |
| **M3** | L38 A2A integration, `data_class_ceiling` enforcement via L34, grant-offer post type |
| **M4** | Remote grant presentation (full doc + grant_id reference), ActivityPub OCAP alignment |
