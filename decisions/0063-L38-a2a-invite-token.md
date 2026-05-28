# ADR-0063 — Layer 38 Extension: A2A Invite-Token Protocol

**Status:** Proposed — 2026-05-28
**Date:** 2026-05-28
**Authors:** Claude Code (maintainer session)
**Type:** Protocol extension + CLI tooling
**Extends:** [ADR-0048 (L38 RemoteTriggerReceiver + A2A TaskEnvelope)](0048-L38-remote-trigger-receiver.md)
**Milestone target:** L38 M4

---

## Context

ADR-0048 (L38 M1–M3) defines a cryptographically sound A2A TaskEnvelope
protocol. Setting up a new peer-to-peer connection today requires the
operator to:

1. Run `atelier-a2a pair <peer-id> <peer-url>` on instance A.
2. Receive a generated JSON endpoint-file from the CLI output.
3. Transmit that file out-of-band (secure email, `age`-encrypted blob, etc.)
   to the operator of instance B.
4. Instance B manually installs the file under `operator/cowork/remote_origins/`.
5. Instance B generates their own endpoint-file and sends it back.
6. Instance A installs that file under `operator/cowork/remote_endpoints/`.

Six manual steps, two files, two out-of-band transports, no built-in expiry,
no revocation. The gap was acknowledged in ADR-0048 under *Consequences
(Negative)*:

> *"Operators must manage HMAC key material out-of-band. L38 does not define
> a key-exchange protocol — that is operator responsibility."*

This ADR formalises the key-exchange protocol as a self-contained
**Invite-Token** that reduces the entire setup to two CLI commands and one
copy-paste.

### Design goals

| Goal | Requirement |
|---|---|
| Single artefact | All connection details in one URL-safe string |
| Expiry | Operator-defined TTL; token unusable after `exp` |
| Single-use | Optional; token can be one-shot to limit exposure |
| Revocability | Issuing instance can invalidate an unaccepted token |
| Bidirectionality | One exchange sets up both directions simultaneously |
| No new dependencies | stdlib only; no `import anthropic` |
| Audit-first | Every lifecycle event lands in the L16 hash chain |
| Dedicated keys | Tokens carry freshly generated keys, never the master key |

---

## Decision

Introduce the **A2A Invite-Token Protocol** as an extension of L38 M4:

* New module `operator/bridges/shared/a2a_invite.py` — token creation,
  serialization, deserialization, expiry and single-use enforcement.
* New module `operator/bridges/shared/a2a_invite_registry.py` — persistent
  invite store with revocation and TTL-keyed cleanup.
* Two new `atelier-a2a` subcommands: `invite` and `accept`.
* Three new audit events: `A2A.invite_created`, `A2A.invite_accepted`,
  `A2A.invite_revoked`.
* Optional `--respond` flag on `accept` for fully automated bidirectional
  setup in a single exchange.

---

## Token format

### Wire shape

```
atelier-a2a:v1:<base64url(payload_json)>.<base64url(hmac_sig)>
```

* **Prefix** `atelier-a2a:v1:` — human-readable type tag; allows a future
  `v2:` without format ambiguity.
* **Payload** — URL-safe base64 (no padding) of the compact JSON body
  (see below).
* **Signature** — URL-safe base64 of
  `HMAC-SHA256(invite_master_key, payload_bytes)`, where
  `invite_master_key` is a per-instance secret stored at
  `<global>/remote_trigger/invite_master_key` (mode 0600, generated on
  first use). This lets the issuing instance verify its own tokens before
  accepting a `revoke` command, without having to keep a full token
  copy on disk.

### Payload fields

Short keys reduce token length by ~30 % over full English names.

| Key | Type | Meaning |
|---|---|---|
| `v` | int | Protocol version (currently `1`) |
| `iid` | str | Issuing instance UUID (`instance_id.json`) |
| `oid` | str | `origin_id` the accepting instance should register |
| `url` | str | Base URL of the issuing instance |
| `rp` | str | Receive path (default `/v1/a2a/receive` if absent) |
| `hk` | str | Hex-encoded HMAC key (32 bytes) — for signing TaskEnvelopes *to* the issuer |
| `rk` | str | Hex-encoded recv key (32 bytes) — for verifying ResponseEnvelopes *from* the issuer |
| `pa` | list[str] | `allowed_personas` the accepting instance may invoke |
| `mt` | int | `max_ttl_s` cap (seconds) |
| `iat` | float | Unix timestamp: issued at |
| `exp` | float\|null | Unix timestamp: expires at (`null` = no expiry) |
| `su` | bool | Single-use flag |
| `ikey` | str | 16-hex-char prefix of the token's own signature — used as the invite's stable ID in the registry |
| `lbl` | str\|null | Human-readable label ("For Klaus at SaaS Co.") |

Estimated token size with 32-byte keys (64 hex chars each) and a typical
URL: **620–720 characters** — comfortably copy-pasteable, fits a QR code
at error-correction level M.

### Key generation invariant

`hk` and `rk` in the Invite-Token are **freshly generated per invite**
via `secrets.token_bytes(32)`. They are not derived from, and do not
replace, any existing origin or endpoint key. Accepting a token creates a
new, independent connection; revoking the token (or deleting the resulting
origin/endpoint files) has no effect on other connections.

---

## Invite registry

```
<atelier_home>/global/remote_trigger/invites.json  (mode 0600, flock on write)
```

Schema (append-only dict keyed by `ikey`):

```json
{
  "<ikey>": {
    "ikey":       "16hexchars",
    "oid":        "peer-instance-name",
    "lbl":        "For Klaus",
    "iat":        1748480000.0,
    "exp":        1749084800.0,
    "su":         false,
    "accepted":   false,
    "accepted_at": null,
    "revoked":    false,
    "revoked_at": null
  }
}
```

* On `invite` creation → entry written with `accepted: false, revoked: false`.
* On `accept` → entry updated: `accepted: true, accepted_at: <ts>`.
  Single-use tokens are marked as consumed at this point and rejected on
  any further `accept` call.
* On `revoke` → entry updated: `revoked: true, revoked_at: <ts>`.
  Revoked tokens cannot be accepted even within their expiry window.
* Cleanup: entries where `exp < now() - 86400` (one day past expiry) may
  be pruned by `atelier-a2a list-invites --clean`.

---

## Validation sequence on `accept`

1. **Prefix check** — token starts with `atelier-a2a:v1:`.
2. **Base64 decode** — payload and signature decodable without error.
3. **Schema** — all required fields present and correctly typed.
4. **Expiry** — `exp` is null OR `now() < exp`; otherwise reject with
   `reason: "expired"`.
5. **Registry lookup** — `ikey` present in local invite registry (if this
   instance is the issuer) or skip (remote token, issuer is elsewhere).
6. **Revocation** — `revoked: false` in registry; otherwise reject with
   `reason: "revoked"`.
7. **Single-use** — if `su: true` and `accepted: true` already, reject
   with `reason: "already_accepted"`.
8. **Signature** — if this instance is the issuer (`iid` matches local
   `instance_id`): verify HMAC over payload against `invite_master_key`.
   If the token is from a remote issuer: signature is informational only
   (the accepting instance has no `invite_master_key` for the remote);
   skip signature check and proceed.
9. **Conflict check** — if `remote_origins/<oid>.json` already exists:
   warn operator and prompt for confirmation (CLI `--overwrite` flag
   bypasses prompt).
10. **Audit** — `A2A.invite_accepted` written to L16 hash chain BEFORE
    any file is written. If chain write fails, reject.
11. **File writes** — write `remote_origins/<oid>.json` and
    `remote_endpoints/<oid>.json` atomically (write to `.tmp`, then
    `os.replace`). Both files get mode 0600.
12. **Registry update** — mark invite as `accepted: true`.

Steps 1–8 failures all surface as the same `"rejected"` message to the
CLI user; the audit chain carries the `reason`.

---

## CLI interface

### `atelier-a2a invite`

```
atelier-a2a invite [OPTIONS]

Options:
  --scope PERSONA[,PERSONA...]   allowed_personas in the token (default: assistant)
  --ttl DURATION                 token validity: e.g. 1h, 7d, 30d (default: 7d)
  --single-use                   token may be accepted exactly once
  --label TEXT                   human-readable label stored in registry
  --max-call-ttl SECONDS         max_ttl_s per TaskEnvelope (default: 300)
  --qr                           also render token as QR code in terminal;
                                 save PNG to ./outputs/atelier-invite-qr.png
  --json                         machine-readable output (token + ikey + exp)
```

Output (default):

```
Invite token (valid 7 days, single-use):

atelier-a2a:v1:eyJ2Ijox....<sig>

Share this token with the operator of the peer instance.
They run:  atelier-a2a accept <token>
```

Audit event `A2A.invite_created`:
`ikey`, `oid`, `lbl`, `exp`, `su`, `pa`, `channel`, `chat_key`
(no `hk`, `rk`, `url`, `iid` in details).

### `atelier-a2a accept`

```
atelier-a2a accept TOKEN [OPTIONS]

Options:
  --overwrite       overwrite existing origin/endpoint files without prompting
  --respond         after accepting, generate and print a return invite token
                    so the issuing instance can accept the reverse direction
  --respond-ttl DURATION  TTL for the return invite (default: 7d)
  --respond-scope PERSONA[,PERSONA...]  scope for the return invite (default: assistant)
  --dry-run         validate and print what would be written; do not write files
  --json            machine-readable output
```

Output (default):

```
[OK] Connection to "peer-instance-name" established.
     URL:              https://peer.example.com
     Allowed personas: assistant
     Expires:          2026-06-04 09:00 UTC
     Origin file:      operator/cowork/remote_origins/peer-instance-name.json
     Endpoint file:    operator/cowork/remote_endpoints/peer-instance-name.json

To test: atelier-a2a send peer-instance-name "ping"
```

With `--respond`:

```
[OK] Connection to "peer-instance-name" established.

Return token for peer (share back via secure channel):

atelier-a2a:v1:eyJ2Ijox....<sig>
```

### `atelier-a2a list-invites`

```
atelier-a2a list-invites [--clean] [--json]
```

Shows all issued invites with their status (pending / accepted / revoked /
expired). `--clean` prunes entries older than one day past expiry.

### `atelier-a2a revoke-invite`

```
atelier-a2a revoke-invite <ikey-or-label>
```

Marks the invite as revoked in the registry. Emits `A2A.invite_revoked`.
Has no effect on already-accepted invites (the resulting origin/endpoint
files remain; remove them with `atelier-a2a remove-origin <oid>` if
needed).

---

## Bidirectional setup — full example

```bash
# Instance A (cloud.atelier.eu) generates an invite:
atelier-a2a invite --scope assistant --ttl 7d --single-use --label "Local B"
→ atelier-a2a:v1:eyJ2Ijox...Token-A...<sig>

# Instance B accepts and generates a return invite:
atelier-a2a accept atelier-a2a:v1:eyJ2Ijox...Token-A...<sig> --respond
→ [OK] Connection to "cloud-instance" established.
  Return token for cloud:
  atelier-a2a:v1:eyJ2IjoyO...Token-B...<sig>

# Instance A accepts the return invite:
atelier-a2a accept atelier-a2a:v1:eyJ2IjoyO...Token-B...<sig>
→ [OK] Connection to "local-b-instance" established.
```

Two commands, two copy-pastes. Both directions operational.

---

## QR-Code support

When `--qr` is passed to `invite`:

1. Token is rendered as an ASCII QR code in the terminal using Python's
   `qrcode` package (stdlib-plus; added to the optional dependency group
   `[ops]` in `pyproject.toml`). If the package is absent, a warning is
   printed and the ASCII render is skipped — the token string is always
   printed regardless.
2. A PNG is saved to `./outputs/atelier-invite-qr.png` for Discord
   attachment delivery or printing.

Physical-meeting workflow: operator of instance A runs `invite --qr` on
their laptop; operator of instance B scans the QR code with their phone
and runs `atelier-a2a accept <paste>` on instance B's terminal.

---

## Audit contract

Three new events on the L16 hash chain (same audit module as existing
`A2A.*` events):

| Event | Severity | Allow-listed detail keys |
|---|---|---|
| `A2A.invite_created` | INFO | `ikey`, `oid`, `lbl`, `exp`, `su`, `pa`, `channel`, `chat_key` |
| `A2A.invite_accepted` | INFO | `ikey`, `oid`, `lbl`, `channel`, `chat_key`, `bidirectional` |
| `A2A.invite_revoked` | WARNING | `ikey`, `oid`, `lbl`, `channel`, `chat_key` |

**Never** in details: `hk`, `rk`, `url`, `iid`, `ikey` full value (only
16-hex-char prefix), token string, `invite_master_key`, any key material.

`A2A.invite_accepted` carries `bidirectional: true` when `--respond` was
used, so audit consumers can distinguish one-shot from two-way setups.

---

## Security properties

### What the token reveals to an eavesdropper

The `hk` and `rk` keys are in the token payload in cleartext (encoded but
not encrypted). An eavesdropper who intercepts the token can:

- Set up a connection to the issuing instance as if they were the legitimate
  peer, using the embedded keys to sign valid TaskEnvelopes.
- Impersonate the issuing instance to someone else (using `rk` to sign
  fake ResponseEnvelopes).

**Mitigations:**

| Threat | Mitigation |
|---|---|
| Interception | Use `--ttl 1h --single-use` on untrusted channels; a narrow window limits the attack surface |
| Replay of accepted token | Single-use flag + registry; second accept is rejected |
| Token tamper | HMAC signature over full payload (only meaningful to issuer, but prevents silent bit-flips) |
| Long-lived exposure | `exp` field; expired tokens are rejected |
| Compromise after accept | Delete origin file (`atelier-a2a remove-origin <oid>`) — severs the connection |

**Recommendation documented in CLI output:** "Transmit over an encrypted
channel (Signal, age-encrypted file, or similar). For untrusted channels,
use `--ttl 1h --single-use`."

### What an attacker who has an accepted token can NOT do

* Exceed the `allowed_personas` scope declared in the token.
* Bypass Layer 10 path-gate, Layer 34 data-classification, or Layer 35
  egress-lockdown — all gates apply identically to A2A-spawned workers.
* Escalate TTL beyond `max_ttl_s` declared in the token.
* See the issuing instance's `invite_master_key` or any other connection's
  HMAC keys.

---

## Consequences

### Positive

* Setup reduced from six manual steps to two CLI commands + one copy-paste.
* Built-in expiry eliminates forgotten long-lived credentials.
* Single-use flag allows safe distribution over semi-trusted channels
  (chat, email) with a tight TTL.
* Dedicated keys per invite mean a compromised token does not affect
  other connections.
* Fully backward-compatible: existing origin/endpoint files and the `pair`
  workflow continue to work unchanged.
* Audit chain records the full invite lifecycle; an operator can trace
  exactly when a connection was established and by which token.

### Negative / trade-offs

* Token contains HMAC key material in cleartext (base64). Operators must
  understand that the token is a credential, not just an address.
* `invite_master_key` is a new per-instance secret that must survive
  instance migration (include in backup / `atelier-migrate` scope — M4
  follow-up).
* `qrcode` package is an optional new dependency. Installations that run
  `pip install atelier[ops]` get it; bare installs skip QR rendering.
* Single-use enforcement requires the registry to be consulted on every
  `accept` call. For remote tokens (issuer on a different machine), the
  accepting instance cannot enforce single-use on behalf of the issuer —
  single-use is enforced only by the issuing instance's registry. The
  accepting instance always writes the origin/endpoint files on a valid
  (non-expired) token; the issuer's registry tracks the consumed state.

### Out of scope

* **Encrypted token payload** — protecting the key material in transit
  is the operator's responsibility (encrypted channel). Adding symmetric
  encryption to the token itself would require a shared secret for the
  encryption, which is the same problem we are trying to solve.
* **Public-key cryptography for token signing** — would allow the
  accepting instance to verify the issuer's signature using a published
  public key (e.g., via a DNS TXT record). Adds real security but also
  real PKI complexity. Deferred to a future ADR.
* **DNS-based discovery** — publishing `_atelier._tcp.<domain>` TXT records
  so instances can auto-discover each other's base URL and reduce token
  size. Deferred.
* **Cross-tenant invite** — an invite is always scoped to the issuing
  tenant. Cross-tenant dispatch remains out of scope per ADR-0007.

---

## Implementation plan

### M4.1 — Core token library (`a2a_invite.py`)

* `InviteToken` dataclass (mirrors payload fields).
* `generate_invite(...)` → `InviteToken` + token string.
* `parse_invite(token_str)` → `InviteToken` (raises `InviteFormatError` on
  malformed input).
* `validate_invite(token, registry, now)` → validates expiry, revocation,
  single-use; returns `ValidationResult`.
* `invite_to_origin_dict(token)` → dict ready to write as origin JSON.
* `invite_to_endpoint_dict(token, local_url)` → dict ready to write as
  endpoint JSON.
* No `import anthropic`; stdlib only except optional `qrcode`.

### M4.2 — Invite registry (`a2a_invite_registry.py`)

* `InviteRegistry` — file-backed store at
  `<global>/remote_trigger/invites.json` (mode 0600, `flock` on write).
* `create(token)`, `mark_accepted(ikey)`, `revoke(ikey)`, `list()`,
  `cleanup(max_age_s)` methods.
* Thread-safe (single-writer lock).

### M4.3 — CLI integration (`atelier-a2a`)

* Add `invite`, `accept`, `list-invites`, `revoke-invite` subcommands.
* `accept --respond` generates and prints a return token.
* `invite --qr` renders ASCII QR + saves PNG to `./outputs/`.

### M4.4 — Audit events

* Three new event types registered in `forge/forge/security_events.py`.
* All three emitters in `a2a_invite.py` follow the existing
  `write_event` pattern.

### M4.5 — Test suite

Target: **≥ 60 cases** across three suites.

| Suite | Coverage |
|---|---|
| `test_a2a_invite.py` | Token generation, serialization, round-trip parse, expiry rejection, revocation rejection, single-use rejection, schema validation, key isolation (fresh keys per invite), HMAC verification, `invite_to_origin_dict` / `invite_to_endpoint_dict` shape |
| `test_a2a_invite_registry.py` | Create, mark_accepted (idempotent), revoke, list, cleanup, flock on concurrent write, mode-0600 check |
| `test_a2a_invite_cli.py` (integration) | `invite` happy path, `invite --qr` (with mocked qrcode), `accept` happy path, `accept --respond` prints return token, `accept` on expired token fails, `accept` on revoked token fails, `accept --dry-run` writes nothing, `list-invites`, `revoke-invite`, `accept` single-use second-call rejected |

Wired into `operator/bridges/run-all-tests.sh`.

---

## Compliance mapping

| Requirement | Source | This ADR delivers |
|---|---|---|
| Minimise long-lived credentials | GDPR Art. 32 | `exp` field; default TTL 7 days |
| Audit trail for access grants | GDPR Art. 30 | `A2A.invite_created/accepted/revoked` in L16 chain |
| Revocability of access | GDPR Art. 17 (access-right analogue) | `revoke-invite` + `remove-origin` |
| Key isolation | GDPR Art. 32 | Dedicated per-invite keys; master key never in token |
| Human-readable audit | EU AI Act Art. 14 | `lbl` field + registry tracks who was given which token |

---

## Must NOT do

* Don't reuse any existing origin or endpoint HMAC key as `hk` or `rk` in
  a generated token — keys must be freshly generated per invite via
  `secrets.token_bytes(32)`.
* Don't put `hk`, `rk`, `url`, `iid`, or the full token string in any
  audit event detail field.
* Don't skip the audit write before file creation in `accept`
  (audit-first invariant from ADR-0048 carries over).
* Don't make `qrcode` a hard dependency — its absence must degrade
  gracefully (warn + skip ASCII render; always print the token string).
* Don't allow an empty `exp` (i.e., `null`) without an explicit
  `--no-expiry` flag — the CLI default must set a TTL.
* Don't enforce single-use across instances without a shared registry —
  document clearly that single-use is enforced by the issuing instance's
  registry only.
* Don't add `invite_master_key` to any backup or migration path without
  treating it as a secret (mode 0600, never in audit chain, never in
  plaintext log).
* Don't `import anthropic` from any module in this extension.

---

## References

* ADR-0048 — L38 RemoteTriggerReceiver + A2A TaskEnvelope Protocol
* ADR-0007 — Multi-tenant axis (scope isolation)
* ADR-0062 — Console UX Overhaul (Agent Hub Peer Cards depend on invite flow)
* `operator/bridges/shared/a2a_invite.py` (M4, to be created)
* `operator/bridges/shared/a2a_invite_registry.py` (M4, to be created)
* `operator/bridges/shared/instance_identity.py` — issuer `iid`
* `operator/bridges/shared/remote_trigger_receiver.py` — validation patterns
  reused in `accept` sequence
* `<atelier_home>/global/remote_trigger/invites.json` — invite registry
* `<atelier_home>/global/remote_trigger/invite_master_key` — signing secret
