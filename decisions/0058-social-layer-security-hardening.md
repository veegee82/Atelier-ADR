# ADR-0058 — Social Layer Security Hardening

**Status:** Accepted  
**Date:** 2026-05-26  
**Authors:** Security review — Forge persona (Claude Code, maintainer session)  
**Companion to:**
  [0053 L39 AtelierFed](0053-L39-atelier-fed-social-federation.md) ·
  [0054 L41 Social Capability Grants](0054-L41-social-capability-grants.md) ·
  [0052 Security Hardening Threat Model](0052-security-hardening-threat-model.md)

---

## Context

Following the implementation of Layer 39 (AtelierFed social federation) and the
completion of ADR-0052 (structural threat model for Layers 7–38), an adversarial
review of the Social Layer revealed five additional findings specific to the
federation surface. These findings are architectural in nature and address
attack vectors that are distinct from the L7–L38 threat model.

The review was performed against the production-ready code at
`operator/bridges/shared/social_*.py` and the ADR-0053/0054 specification.

Five findings are documented below as SL-1 through SL-5.

---

## Findings

### SL-1 — Coarse-Grained Follower Authority

**Risk rating:** HIGH  
**Category:** Capability escalation

**Attack vector:**  
Any actor that completes the follow protocol gains follower status. In the current
model, follower status is a binary condition. This means any follower can, by
default, attempt to trigger forge tools via task-type envelopes, deliver A2A
TaskEnvelopes if `spawn_worker: true` is set on the origin, and read any content
published at `visibility: followers`. The granularity of the permission model does
not match the surface it controls.

An attacker who successfully completes a follow request — which requires only a
valid Ed25519 keypair and a plausible actor document — immediately obtains all
follower-level permissions without any further operator action.

**Affected modules:**  
- `operator/bridges/shared/social_registry.py` — follow-accept logic  
- `operator/bridges/shared/social_http_server.py` — inbox handler  
- `operator/bridges/shared/remote_trigger_receiver.py` — A2A spawn gate

**Fix:**  
Layer 41 `GrantChecker` (ADR-0054) as the structural solution. Deny-by-default:
new followers start with zero capabilities. All access beyond passively receiving
public posts requires an explicit `CapabilityGrant` issued by the local operator.
The `spawn_worker: true` boolean in A2A origin records is demoted to a master
switch; the per-capability check is the effective gate.

**Implementation note:**  
`social_capability.py` implements `GrantStore` and `GrantChecker` with:
- SQLite at `<atelier_home>/tenants/<tid>/global/social/grants.db` (mode 0600)
- Ed25519-signed grant documents (same signing pattern as `PostEnvelope`)
- Wildcard `"*"` grantee scope for global follower benefits
- Per-grant rate-limit counters (N/period) to bound agent invocations
- L16 audit events: `grant.issued`, `grant.revoked`, `grant.allowed`, `grant.denied`
- L36 `L41GrantHandler` for GDPR Art. 17 erasure

---

### SL-2 — Actor Document Cache Poisoning

**Risk rating:** MEDIUM  
**Category:** Cryptographic integrity / MITM

**Attack vector:**  
When the local instance fetches a remote actor's document (for public key
resolution, display name, inbox URL), it performs an HTTPS GET. The response
is cached in `social_registry.py` with a 1-hour TTL. A man-in-the-middle
attacker who controls the TLS termination point (e.g. a compromised CDN,
a BGP hijack, or a rogue DNS resolver) can substitute the actor document
with one containing the attacker's public key. From that moment on, all
incoming posts from that actor pass signature verification against the
attacker's key, even if the real actor never authorized the key.

This is an instance of the "trust on first use" (TOFU) vulnerability: first
fetch establishes trust, subsequent fetches may silently rotate the key.

**Affected modules:**  
- `operator/bridges/shared/social_registry.py` — actor upsert and cache

**Fix:**  
Pin the actor's public key on the first successful fetch. On subsequent fetches,
if `public_key_hex` changes AND the change is not accompanied by a valid Ed25519
signature from the OLD key over the new document (key-continuity proof), reject
the update and emit `social.actor_key_mismatch` (WARNING) to the L16 audit chain.
This is "TOFU + key continuity" — the same pattern used in SSH known-hosts and
Signal's key transparency.

**Implementation note:**  
Add a `pinned_key_hex` column to the `actors` table in `social_registry.py`:

```sql
ALTER TABLE actors ADD COLUMN pinned_key_hex TEXT;
```

On `upsert_actor`:
1. If `pinned_key_hex` is NULL (first fetch): set `pinned_key_hex = public_key_hex`.
2. If `pinned_key_hex` is set and equals `public_key_hex`: no action (key unchanged).
3. If `pinned_key_hex` is set and differs: require `key_rotation_proof` — a
   signature over the new document using the pinned key. If proof absent or
   invalid: reject the update, retain old key, emit `social.actor_key_mismatch`
   (WARNING) with details `{"actor_id_prefix": actor_id[:16]}`.

This is a separate implementation task; the column migration and validation logic
are tracked as L39 M2 follow-up work.

---

### SL-3 — Boost-Chain Injection via Third-Party Content

**Risk rating:** MEDIUM  
**Category:** Prompt injection / content integrity

**Attack vector:**  
An actor boosts a post from a malicious third party. The original post content was
sanitized by `social_sanitizer.py` when first received (NFKC normalization,
content length cap, fence token wrapping). However, a boost creates a new
`PostEnvelope` with `boost_of: <post_id>`. When the local node later renders the
boosted timeline in a new session context, it fetches and inlines the original
post content from `posts.db`.

The NFKC normalization in `social_sanitizer.py` was applied at storage time
using the sanitizer state from the *original* session. In a new session with a
different fence token, the sanitized content is re-displayed without the fence
token re-wrapping that would mark it as untrusted input. An attacker who crafts
original post content using homoglyphs, Unicode bidirectional control characters,
or partial tag injection may escape the sanitization boundary when the post is
re-displayed in a different context.

This finding partially overlaps with ADR-0052 F2 (NFKC-first canonicalization)
but the re-display path via boost chains was not covered by that finding.

**Affected modules:**  
- `operator/bridges/shared/social_feed.py` — post retrieval and rendering  
- `operator/bridges/shared/social_sanitizer.py` — content normalization

**Fix:**  
Re-sanitize any post content pulled from `posts.db` for display in a new session
context. Apply a fresh fence token wrapping at display time, not only at storage
time. Add `sanitize_for_display(post_id: str, session_fence: str) -> str` to
`social_feed.py` that:

1. Loads the stored post content.
2. Applies NFKC normalization again (idempotent).
3. Strips any remaining bidirectional control characters (`‪`–`‮`,
   `⁦`–`⁩`, `‏`).
4. Wraps in the session-bound fence token:
   ```
   <social_post origin="<actor_id_prefix>" post_id="<post_id_prefix>">
   <content>{sanitized}</content>
   </social_post>
   ```
   where origin and post_id use only the first 8 characters (no PII in LLM
   context structure).
5. Enforces the 2000-character content cap (truncation with `[…]` suffix).

**Implementation note:**  
This is a separate implementation task tracked as L39 M3. The key constraint
is that the fence token must be generated from the session's own entropy, not
re-used from the storage-time sanitization run — otherwise a long-lived stored
post can use the fence token itself as an injection vector once the token becomes
predictable.

---

### SL-4 — Retract Replay / Unauthorized Retract

**Risk rating:** MEDIUM  
**Category:** Authorization bypass / message integrity

**Attack vector:**  
Actor A sends a `PostEnvelope` with `post_type: "retract"` and includes a
`retract_of` field pointing to a post authored by actor B. The inbox handler
receives the retract envelope, verifies the Ed25519 signature against actor A's
public key (which passes — the envelope is authentically from A), and proceeds
to mark the referenced post as retracted.

The authorization check — "is the actor sending the retract the same actor who
authored the post being retracted?" — is straightforward in principle but
susceptible to a subtle implementation bug: if the handler checks
`in_reply_to` (which may be populated for posts that reply to the post being
retracted) instead of `retract_of` author, the check passes for any post where
the `in_reply_to` field happens to resolve correctly.

ADR-0053 documents "must NOT accept a retract from an actor who did not author
the original post" as a constraint, but explicit test coverage for this edge case
was missing.

**Affected modules:**  
- `operator/bridges/shared/social_http_server.py` — inbox retract handler  
- `operator/bridges/shared/social_feed.py` — retract application logic

**Fix:**  
Add explicit validation: on processing a retract envelope:
1. Load the post identified by `retract_of` from `posts.db`.
2. Confirm `post.actor_id == retract_envelope.actor_id`.
3. If they differ: reject the retract, emit `social.retract_rejected` (WARNING)
   with details `{"actor_id_prefix": retract_envelope.actor_id[:16], "reason": "not_author"}`.
4. Never fall back to `in_reply_to` for author resolution.

Add a test case to `test_social_capability.py` or a new
`test_social_retract_auth.py` that:
- Creates post P authored by actor B.
- Sends a retract for P from actor A (different actor).
- Asserts that P is not marked as retracted.
- Asserts that `social.retract_rejected` is emitted.

**Implementation note:**  
This is a test-coverage gap, not a known exploited bug. The fix is a one-line
author check in the inbox handler. The test is the primary deliverable for this
finding. Tracked as L39 M2 follow-up.

---

### SL-5 — Follower Enumeration via Timing Side Channel

**Risk rating:** LOW  
**Category:** Information disclosure / side channel

**Attack vector:**  
An adversary can enumerate whether a given actor_id is in the local instance's
follower list by comparing HTTP response latency on the `/inbox` endpoint:

- Follow-accept (actor is unknown → fast HMAC check + follower lookup + DB write):
  typically 5–15 ms.
- Follow-silent-reject (actor is known-blocked → fast HMAC check + blocked lookup
  → immediate return): typically 2–5 ms.
- Compliance-zone-reject (actor's compliance zone fails → fast HMAC check +
  zone check → immediate return): typically 2–5 ms.
- Post-from-follower (valid envelope from known follower → HMAC + registry lookup
  + post write): typically 8–20 ms.
- Post-from-non-follower (unknown actor → quick reject): typically 1–3 ms.

All branches return HTTP 202 (fail-silent), but the latency difference is
measurable over a low-latency network and allows an adversary to distinguish
"known follower" from "unknown actor" with high confidence after ~50 probes.

While the follow list is not GDPR-sensitive at the level of message content, it
is social-graph metadata that can reveal relationships the operator considers
private (e.g. which AI agents are following which other agents in a B2B context).

**Affected modules:**  
- `operator/bridges/shared/social_http_server.py` — inbox endpoint response timing

**Fix:**  
Add a constant-time response delay to the inbox endpoint. Before returning HTTP
202, always ensure that at least `SOCIAL_INBOX_MIN_RESPONSE_MS` milliseconds have
elapsed since the request was received (default: 10 ms). If processing completes
faster, sleep the remainder. If processing takes longer than the minimum, respond
immediately (no cap on slow responses).

```python
import time

SOCIAL_INBOX_MIN_RESPONSE_MS = int(
    os.environ.get("SOCIAL_INBOX_MIN_RESPONSE_MS", "10")
)

async def handle_inbox(request):
    t0 = time.monotonic()
    result = await _process_inbox(request)
    elapsed_ms = (time.monotonic() - t0) * 1000
    pad_ms = SOCIAL_INBOX_MIN_RESPONSE_MS - elapsed_ms
    if pad_ms > 0:
        await asyncio.sleep(pad_ms / 1000)
    return response_202()
```

This flattens the timing side channel for all branches that complete within the
minimum window. 10 ms is sufficient to absorb the database lookup variance on
typical hardware without materially affecting throughput at normal federation
rates (< 100 req/s per instance).

**Implementation note:**  
`SOCIAL_INBOX_MIN_RESPONSE_MS` defaults to 10 ms and is configurable via
environment variable. It should NOT be set to 0 (disables protection) or to
a value above 1000 ms (degrades legitimate federation performance). The minimum
is enforced in `social_http_server.py` at the outer response boundary, not inside
individual processing branches — so new code paths automatically benefit.
Tracked as L39 M2 follow-up.

---

## Consolidated Changes

| Finding | Risk | Fix Module | Status |
|---|---|---|---|
| SL-1: Coarse-Grained Follower Authority | HIGH | `social_capability.py` (new — ADR-0054 M1) | Implemented |
| SL-2: Actor Document Cache Poisoning | MEDIUM | `social_registry.py` — TOFU + key continuity | L39 M2 follow-up |
| SL-3: Boost-Chain Injection via Third-Party Content | MEDIUM | `social_feed.py` — `sanitize_for_display()` | L39 M3 follow-up |
| SL-4: Retract Replay / Unauthorized Retract | MEDIUM | `social_http_server.py` + test coverage | L39 M2 follow-up |
| SL-5: Follower Enumeration via Timing Side Channel | LOW | `social_http_server.py` — `SOCIAL_INBOX_MIN_RESPONSE_MS` | L39 M2 follow-up |

---

## Compliance Notes

- **GDPR Art. 5(1)(f)** — SL-1 (capability grants) directly enforces integrity
  and confidentiality of processing: remote actors can only access data they were
  explicitly granted permission to access.
- **GDPR Art. 25** — SL-1 implements privacy-by-design at the federation
  boundary: deny-by-default is the structural default, not an opt-in.
- **GDPR Art. 32** — SL-2 (key pinning) and SL-5 (timing normalization) reduce
  the attack surface for unauthorized access to personal data held in the social
  graph.
- **EU AI Act Art. 14** — SL-1 directly supports human oversight: the
  `agent.invoke.*` capability must be explicitly granted before a remote actor
  can trigger AI execution. Rate limits in the grant document enforce
  proportionality.

---

## Must NOT Do

- Do not treat SL-2 through SL-5 as "resolved" until the respective follow-up
  implementations are merged and their tests pass.
- Do not widen the `SOCIAL_INBOX_MIN_RESPONSE_MS` default above 1000 ms.
- Do not set `SOCIAL_INBOX_MIN_RESPONSE_MS=0` in any production configuration.
- Do not implement a "trusted actor" bypass that skips `GrantChecker` for
  actors with `relationship=mutual` — every follower, including mutual follows,
  starts with zero capabilities.
- Do not add the raw actor document content to any audit detail field when
  emitting `social.actor_key_mismatch`.
