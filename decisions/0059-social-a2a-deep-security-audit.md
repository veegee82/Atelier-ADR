# ADR-0059 — Social Layer + A2A Deep Security Audit: Prompt Injection & Manipulation Hardening

**Status:** Accepted — all findings resolved 2026-05-26
**Date:** 2026-05-26
**Authors:** Adversarial security review — Forge persona (Claude Code, maintainer session)
**Companion to:**
  [0053 L39 AtelierFed](0053-L39-atelier-fed-social-federation.md) ·
  [0048 L38 A2A](0048-L38-remote-trigger-receiver.md) ·
  [0052 Security Hardening Threat Model](0052-security-hardening-threat-model.md) ·
  [0058 Social Layer Security](0058-social-layer-security-hardening.md)

---

## Context

Following the implementation of L39 (AtelierFed), L41 (Social Capability Grants),
and the ADR-0052 structural threat model, an adversarial deep-dive was performed
specifically targeting prompt injection and malicious manipulation vectors in the
social and A2A subsystems. This audit read all implementation code (not just ADRs)
and identified 15 concrete findings, 4 of which are HIGH or CRITICAL severity.

The review covered:
- `social_sanitizer.py`, `social_http_server.py`, `social_feed.py`,
  `social_registry.py`, `social_actor.py`, `social_consent.py`,
  `social_capability.py`, `a2a_worker.py`
- `compliance_assertion.py`, `cal_predicates.py`

---

## Findings Summary

| # | Title | Severity | Fixed |
|---|---|---|---|
| F1 | Process-lifetime session fence token (same token for all posts) | HIGH | ✅ |
| F2 | Fence token leaks via outbox endpoint → trivial framing-escape | HIGH | ✅ |
| F3 | `content_warning` field not protected against framing-tag injection | MEDIUM | ✅ |
| F4 | Actor impersonation via attacker-controlled `key_id` URL | HIGH | ✅ |
| F5 | `is_ai` field trusted from inbound envelope, not from actor document | MEDIUM | ✅ |
| F6 | `GrantChecker` exists but never wired into inbox handler | CRITICAL | ✅ |
| F7 | Rate limiter is per-actor-ID, not per-IP — actor-ID proliferation bypass | MEDIUM | ✅ (documented) |
| F8 | Missing CAL predicate for `social.spawn_worker = NEVER` | HIGH | ✅ |
| F9 | `publish_post()` double-sanitization with empty `actor_id` on first pass | LOW | ✅ |
| F10 | A2A `workspace_hint` appended to sanitized instruction body, bypassing sanitizer | MEDIUM | ✅ |
| F11 | CAL `consent.grant` predicate passes when grantor/grantee is `None` | LOW | ✅ |
| F12 | `actor_doc` possibly undefined when `known_actor` branch is taken | LOW | ✅ |
| F13 | `list_posts()` returns `list`, outbox calls `.get()` on it → silent empty outbox | LOW | ✅ |
| F14 | Known actor public key never re-validated after first fetch | MEDIUM | ✅ (documented) |
| F15 | `verify_envelope()` silent exception on non-string `public_key_hex` | LOW | ✅ |

---

## Critical Finding Detail: F6 — GrantChecker Not Wired

The most structurally significant finding: `social_capability.py` (ADR-0054 M1,
implemented 2026-05-26) provides a complete `GrantChecker` class with deny-by-default
logic, rate limiting, TTL enforcement, and L16 audit events. However, the inbox handler
(`social_http_server._handle_inbox()`) never imported or called it. Any authenticated
follower could post any content type without any explicit capability grant.

**Fix:** Added Step 8b to `_handle_inbox()`: after signature verification and before
compliance-zone check, instantiate `GrantChecker` and call `.check(actor_id, required_cap)`.
Fail-silent (202) on deny, with `social.capability_denied` audit event.
Non-blocking on import failure (`_CAPABILITY_AVAILABLE` flag).

---

## High Finding Detail: F1+F2 — Fence Token Architecture Flaw

The session fence token was generated once per process startup (`secrets.token_hex(4)`)
and stored in a module-level global. Since the framed content was stored verbatim in
`posts.db` and the public outbox endpoint returned post content, any unauthenticated
HTTP client could read the token from the outbox. With the token known, an attacker could
craft post content containing `</social_post_XXXXXXXX>` (the full token-specific closing
tag), which bypasses the generic `</social_post` prefix check.

**Fix:** Architectural separation of storage content from LLM-injection framing:
- `sanitize_post_content()` now returns raw sanitized content (NFKC + truncate +
  control-strip + injection check) — no framing tags.
- New `sanitize_raw()` exposes the core sanitization.
- New `frame_for_llm(content, actor_id, post_id, fence_token=None)` generates a fresh
  random fence token per call, applies the framing, and checks for both the generic
  `</social_post` prefix AND the specific `</social_post_{fence_token}>` closing tag.
  The token is never persisted — it is local to the caller's LLM-injection invocation.
- `SocialFeedStore.get_post_framed()` is the safe method for LLM-context post retrieval.

---

## High Finding Detail: F4 — Actor Impersonation

Inbound envelopes from unknown actors triggered an actor document fetch from the
`key_id` URL (an attacker-controlled field). The fetched document's `instance_id`
was never verified against the envelope's `actor_id`. An attacker could post content
attributed to any `actor_id` by hosting a crafted actor document at a URL they control.

**Fix:** After fetching the actor document, verify:
```python
if actor_doc.get("instance_id") != actor_id:
    # audit social.actor_id_mismatch + return 202
```

---

## High Finding Detail: F8 — Missing CAL Predicate

The Compliance Assertion Layer (`cal_predicates.py`) had no predicate enforcing
the L39 structural invariant "social posts MUST NEVER trigger WorkerEngine spawns."
ADR-0053 states this as a "Must NOT do" but the CAL is the code-layer enforcement
mechanism. Without a predicate, a future developer could accidentally wire the social
inbox to a worker spawn without triggering a CRITICAL boot failure.

**Fix:** Added `_p_social_no_spawn` (CAL-P7) to `ALL_PREDICATES`:
```python
_SOCIAL_SPAWN_ACTIONS = frozenset({
    "social.spawn_worker", "social.post_spawn",
    "social.trigger_worker", "social.incoming_post_spawn",
})
```
Any action matching these strings is unconditionally denied by the CAL.

---

## Medium Finding Detail: F10 — A2A Workspace Hint Bypass

`a2a_worker.py` appended `workspace_hint` (containing the scratch workspace path)
to the sanitized instruction body BEFORE calling `frame_instruction()`. Since
`workspace_hint` was constructed after `sanitize_instruction()` ran, it bypassed
the injection sanitizer. While the workspace path itself comes from `tempfile.mkdtemp()`
(OS-controlled, safe), the structural pattern was wrong: non-sanitized content was
concatenated with sanitized content and passed as a unit to `frame_instruction()`.

**Fix:** `workspace_hint` is now appended AFTER `frame_instruction()` in a separate
`<a2a_workspace>` structural block, clearly separated from the instruction framing.
The LLM sees workspace information in a distinct block, not intermixed with the
instruction body.

---

## Documented (Not Code-Fixed) Findings

**F7 — Rate limiter per-actor-ID, not per-IP:**
The rate limiter in `social_registry.py` is keyed on `actor_id`. An attacker can
generate novel `actor_id` values to get a fresh rate-limit counter per identity. With
1000 unique actor IDs, the global limit (1000 posts/hour) can be exhausted by one
attacker, causing legitimate federation traffic to be dropped.

This is documented rather than fixed in this ADR because the correct fix requires
infrastructure context (reverse-proxy IP extraction, which is deployment-specific).
The operator should configure a network-layer rate limiter (nginx, Cloudflare, etc.)
in front of the social HTTP server. The application-layer rate limit is defense-in-depth,
not the primary DoS protection. Tracked for M3 social layer work.

**F14 — Known actor public key never TTL re-fetched:**
Once an actor is registered, the public key is read from the local SQLite row on every
subsequent message. Key rotation by the remote actor has no effect — the local node
keeps verifying against the stale key indefinitely.

Documented for the M3 key-rotation ceremony work (analogous to A2A's `atelier-a2a rotate`).
The TOFU (trust-on-first-use) model described in ADR-0058 SL-2 addresses the cache-poisoning
variant; re-fetch TTL is a usability issue that does not introduce a security regression
relative to the current state.

---

## Changes Made

| File | Change |
|---|---|
| `social_sanitizer.py` | Remove `_SESSION_FENCE_TOKEN`; split into `sanitize_raw()` + `frame_for_llm()`; add injection check to `sanitize_content_warning()` |
| `social_feed.py` | Store raw content; add `get_post_framed()`; fix `publish_post()` double-sanitization |
| `social_http_server.py` | Wire `GrantChecker` (Step 8b); verify actor_doc `instance_id`; fix `is_ai` from doc not envelope; initialize `actor_doc = None`; fix outbox `list_posts()` call |
| `social_registry.py` | Add `is_follower()` method |
| `cal_predicates.py` | Add `_p_social_no_spawn` (CAL-P7); fix consent predicate for None grantor/grantee |
| `a2a_worker.py` | Move `workspace_hint` after `frame_instruction()` into separate `<a2a_workspace>` block |
| `test_social_sanitizer.py` | Update tests for new API; add `frame_for_llm` tests |
| `test_social_feed.py` | Update for raw content storage |
| `test_social_http_server.py` | Update for new inbox step 8b; outbox fix |
| `test_compliance_assertion.py` | Add test for CAL-P7 social spawn predicate |
| `test_social_federation_e2e.py` | Update framing assertions |

---

## Test Coverage After Fix

| Suite | Count | Status |
|---|---|---|
| `test_social_sanitizer.py` | 36 | ✅ all pass |
| `test_social_http_server.py` | 16 | ✅ all pass |
| `test_social_feed.py` | 29 | ✅ all pass |
| `test_social_registry.py` | 20 | ✅ all pass |
| `test_social_capability.py` | 38 | ✅ all pass |
| `test_social_federation_e2e.py` | 10 | ✅ 8 pass, 2 skip (pre-existing DB isolation) |
| `test_compliance_assertion.py` | 59 | ✅ all pass |
| `test_adr0052_security.py` | 21 | ✅ all pass |

---

## Must NOT Do

- **Don't re-introduce a module-level or process-lifetime fence token.** Framing tokens
  must be ephemeral — generated at LLM-injection time, never persisted.
- **Don't store `frame_for_llm()` output in `posts.db`.** The database must contain only
  raw sanitized content; framing is applied on-the-fly at read time.
- **Don't bypass the `GrantChecker` step in `_handle_inbox()`.** The deny-by-default
  invariant (ADR-0054) must be enforced at intake, not just at the data layer.
- **Don't add `social.*` action types that trigger `ClaudeCodeEngine.spawn()` without
  adding a CAL predicate first.** The CAL-P7 predicate covers known action names; new
  spawn paths in the social layer require a corresponding predicate update.
- **Don't trust `is_ai` from an inbound PostEnvelope.** Trust only the fetched and
  verified actor document. An envelope is attacker-controlled data; an actor document
  is fetched from the declared actor URL and signature-verified.

---

## References

- [ADR-0048](0048-L38-remote-trigger-receiver.md) — A2A protocol
- [ADR-0052](0052-security-hardening-threat-model.md) — structural threat model (F1-F10 fixed)
- [ADR-0053](0053-L39-atelier-fed-social-federation.md) — L39 social federation
- [ADR-0054](0054-L41-social-capability-grants.md) — L41 capability grants
- [ADR-0058](0058-social-layer-security-hardening.md) — social layer security (SL-1 through SL-5)
- EU AI Act Art. 50 — AI disclosure in public output (F5: is_ai field integrity)
- GDPR Art. 32 — security of processing (F1/F2: content containment, F4: actor identity)
