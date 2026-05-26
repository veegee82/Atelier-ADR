# ADR-0044 — Layer 37: Audit-at-rest Encryption + Retention

**Status:** Accepted
**Date:** 2026-05-19
**Companion to:** [0041-EU-Compliance-Atelier-OpenCode.md](0041-EU-Compliance-Atelier-OpenCode.md), [0041-GAP-ANALYSIS.md](0041-GAP-ANALYSIS.md), [0042-L34-data-classification.md](0042-L34-data-classification.md), [0043-L35-egress-lockdown.md](0043-L35-egress-lockdown.md)

---

## Context

ADR-0041 § Phase 2.2 + Phase 3.1 require encryption-at-rest for the
audit chain and a 7-year retention floor. The gap analysis identified
three concrete gaps:

* No rotation policy — `audit.jsonl` grows forever.
* No encryption-at-rest — mode-0600 is the only protection.
* No documented retention — the daily TTL sweep handles session
  scope, not audit chain.

Existing pieces this builds on:

* L16 hash chain — `audit.jsonl` with `prev_hash` + `hash` linked
  entries, daily `voice-audit verify` timer.
* `forge.security_events.write_event` — atomic append with
  flock + fsync; the chain semantics this ADR must preserve.

A subtle constraint: the L16 hash chain is *intra-file*. Rotating
the file breaks chain verification across the boundary unless the
new live file's first entry carries the rotated tail-hash as its
`prev_hash`.

## Decision

Introduce **Layer 37** as a separate structural layer:

* New module `operator/bridges/shared/audit_sealer.py`.
* Three policy dataclasses: `RotationPolicy`, `EncryptionConfig`,
  `RetentionPolicy`, composed into `AuditPolicy`.
* `should_rotate(audit_path, policy) -> RotationDecision` —
  size / age threshold check.
* `rotate_and_seal(audit_path, policy, audit_writer, sealer) ->
  RotationResult` — atomic rotation that writes a
  `audit.rotation_link` event as the first entry of the new live
  file, preserving the chain link across the segment boundary.
  Optional RFC 3161 external timestamping hook (opt-in, non-fatal).
* `enforce_retention(audit_dir, policy, audit_writer) -> [paths]` —
  deletes sealed segments older than `retention_years`, audit-first.
* `unseal_to_temp(sealed_path, ...) -> plaintext_path` — operator /
  DPO inspection path, audit-before-decrypt.
* New tenant config sub-key `tenant.atelier.yaml::spec.audit`.
* Eight audit events on the L16 chain (see § Audit contract).

### Lifecycle of a segment

```
                       size or age threshold
                                │
            ┌──────────┐  rotate│  ┌──────────────────────┐
            │ live     ├────────┼──▶ rotated (plaintext)  │
            │ audit.jsonl       │  │ audit.<stamp>.jsonl  │
            └──────────┘        │  └──────────┬───────────┘
                                │             │ encrypt
                                │             ▼
                                │  ┌─────────────────────────┐
                                │  │ sealed                  │
                                │  │ audit.<stamp>.jsonl.age │
                                │  └──────────┬──────────────┘
                                │             │ retention_years
                                │             ▼
                                │  ┌──────────────┐
                                │  │ retired      │
                                │  │ (file removed)│
                                │  └──────────────┘
                                │
                                ▼
                  audit.rotation_link
                  prev_hash = rotated.last_hash
```

### Chain-link continuity

The first entry of the new live file is always:

```json
{
  "ts": <rotation time>,
  "event_type": "audit.rotation_link",
  "severity": "INFO",
  "details": {"rotated_segment": "audit.<stamp>.jsonl"},
  "prev_hash": "<rotated tail hash>",
  "hash": "<sha256[:16] of (prev_hash + canonical(rec))>"
}
```

Cross-segment verification: walk segments chronologically, verify
each via `forge.security_events.verify_chain`, then assert
`sealed[i].last_hash == sealed[i+1].first_prev_hash`.

The chain semantics are preserved across rotation without needing
to change `write_event`.

### Sealing

Pipes plaintext through an operator-chosen binary:

* `age -r <recipient>` (default — modern, simple, recipient-based).
* `gpg --batch --yes --encrypt -r <recipient>` (back-compat).
* Operator can monkeypatch the sealer for tests / custom HSM
  setups.

Post-seal:

1. Sealed file written atomically (`.tmp` then `os.replace`).
2. Sealed file chmod-444.
3. Plaintext rotated file overwritten with zeros (best-effort,
   first 4 KB) then unlinked.
4. `audit.segment_sealed` (INFO) emitted into the new live chain.
5. If `tsa_enabled=true`: RFC 3161 timestamp token requested (see
   § External timestamping). Non-fatal — `audit.tsa_request_failed`
   (WARNING) on error; seal is not rolled back.

### Retention

`enforce_retention(audit_dir, policy)`:

1. List sealed segments older than `retention_years` (365.25 d/year).
2. For each: emit `audit.segment_retired` (INFO) into the new live
   chain *before* removal.
3. chmod 0o644 + unlink. Continue on per-file failure.

### Operator unseal

`voice-audit unseal <segment>` (DPO / legal hold). Behavior:

1. Resolve sealer kind from filename suffix (`.age` / `.gpg`) or
   explicit kwarg.
2. **Emit `audit.unseal_requested` (WARNING) into the live chain
   BEFORE decryption.** This is the audit-of-the-audit — a DPO
   request for a specific past day is itself audited regardless
   of what happens to the plaintext.
3. Decrypt into a tmpdir-bound file with mode 0600. Caller is
   responsible for removing the plaintext when inspection is done.

### Tenant configuration

```yaml
spec:
  audit:
    retention_years: 7
    encryption_at_rest:
      enabled: true
      recipient: "age1xyz..."
      sealer_cmd: age           # age | gpg
      # RFC 3161 external timestamping (opt-in, non-fatal):
      tsa_enabled: false
      tsa_url: ""               # e.g. "http://tsa.example.com/tsr"
      tsa_hash_algo: sha256     # currently only sha256
    rotation:
      max_size_mb: 100
      max_age_days: 30
```

Defaults: `retention_years=7`, `encryption.enabled=false`,
`tsa_enabled=false`, `max_size_mb=100`, `max_age_days=30`.
EU_PRODUCTION presets carry this block forward-declared (M2) but
ignored by current code; M3 (this ADR) activates the enforcement.

### RFC 3161 external timestamping (opt-in)

**Problem it solves.** L16 + L37 sealing provide *internal*
tamper-evidence. An external auditor cannot independently verify
that a sealed segment from 2026-Q1 was not fabricated in 2026-Q4.
RFC 3161 TSA (Time-Stamp Authority) responses provide a
cryptographically-signed third-party attestation that the sealed
segment's SHA-256 hash existed at a specific time — independent of
the AtelierOS operator.

**How it works.**

1. After a successful seal, `rotate_and_seal` computes the SHA-256
   of the sealed file.
2. Builds a minimal RFC 3161 `TimeStampReq` (59 bytes DER) via
   `_build_tsa_request(hash_bytes)` — pure stdlib, no crypto
   dependency, hand-crafted DER encoding.
3. POSTs to `tsa_url` (`Content-Type: application/timestamp-query`).
4. Writes the raw `TimeStampResp` as
   `<sealed>.{age,gpg}.tsr` (chmod 444, alongside the sealed file).
5. Emits `audit.segment_timestamped` (INFO) with `sealed_segment`,
   `tsa_url`, and `timestamp_token_path`.
6. On any failure (network, HTTP ≠ 200, timeout): emits
   `audit.tsa_request_failed` (WARNING), sets
   `RotationResult.timestamp_token_path = None`, does **not** raise.
   The seal is final regardless.

**Verification.** Operators verify `.tsr` files offline using
standard RFC 3161 tooling (e.g. `openssl ts -verify`). AtelierOS
does not ship a built-in TSA verifier in M3+.

**TSA choice.** Any RFC 3161-compliant TSA works. Common choices
for regulated EU deployments: DigiCert TSA, GlobalSign TSA, or a
self-hosted Bouncy Castle / OpenSSL TSA. Free public TSAs (FreeTSA)
are acceptable for non-critical use. The operator is responsible for
the TSA's trustworthiness.

**Anti-scope.** Does not require Merkle-tree migration. Does not
require blockchain. Does not change L16 chain semantics. One TSA
call per sealed segment is the entire surface.

### Audit contract

Eight event types on the L16 hash chain:

| Event | Severity | When |
|---|---|---|
| `audit.rotation_link` | INFO | First entry of every fresh live file after rotation |
| `audit.rotation_started` | INFO | Reserved for future async pipeline (not emitted by M3) |
| `audit.rotation_failed` | CRITICAL | Sealer non-zero exit; raised + audited |
| `audit.segment_sealed` | INFO | Successful seal post-rotation |
| `audit.segment_timestamped` | INFO | RFC 3161 TSA token written (opt-in) |
| `audit.tsa_request_failed` | WARNING | TSA call failed; seal stands, non-fatal |
| `audit.segment_retired` | INFO | Pre-removal in `enforce_retention` |
| `audit.unseal_requested` | WARNING | Pre-decrypt in `unseal_to_temp` |

Allow-list: `rotated_segment`, `sealed_segment`, `sealer_cmd`,
`rotated_size_bytes`, `age_days`, `reason`, `requester`,
`tsa_url`, `timestamp_token_path`. Never the audit content, never
the encryption key, never TSA response body, never the inspector's
identity beyond the free-form `requester` string the caller supplies.

## Consequences

### Positive

* 7-year retention floor is now a declared, configurable policy.
* Encryption-at-rest is a single-stage pipe to operator-chosen
  binary; no new Python crypto dependency.
* Hash-chain integrity is preserved across rotation via
  `audit.rotation_link` events.
* Unseal path is itself audited — a DPO/legal request creates an
  audit trail that survives the inspection.
* Two sealer choices (`age` + `gpg`) cover both modern and legacy
  operator preferences.

### Negative / trade-offs

* Encryption requires an external binary; missing `age` / `gpg`
  at runtime fails the sealing step (and emits CRITICAL). This is
  the right failure mode — silent fall-back to plaintext at rest
  would weaken the compliance claim.
* Plaintext overwrite is best-effort (first 4 KB zero-fill). A
  determined attacker with raw disk access isn't stopped; routine
  `cat` is.
* Rotation is timer-driven, not append-driven. There's a window
  between threshold crossing and the next timer tick where the
  live file is over budget. Acceptable: operators size the daily
  budget with headroom.
* Cross-segment verification needs the operator to provide the
  unseal identity at audit time — automated daily verify can only
  check the live segment.

### Out of scope

* Cross-segment verifier CLI extension (`voice-audit verify
  --include-sealed --identity <key>`) — separate small change.
* Real HSM integration; operator-chosen recipient string suffices.
* SIEM forwarding of sealed segments — operator's deployment.

## Compliance mapping

| Requirement | ADR-0041 reference | This ADR delivers |
|---|---|---|
| 7-year retention | § Phase 2.2 | `RetentionPolicy.retention_years` (default 7) |
| Encryption-at-rest | § Phase 3.1 | `EncryptionConfig.{enabled, recipient, sealer_cmd}` |
| Immutable audit-log | § Phase 2.2 | chmod 444 on sealed segments; chain-link continuity preserved |
| Right to deletion (Art. 17) | § Phase 4.1 | Foundation — L36 (next milestone) will redact entries; this ADR ensures sealed segments are inspectable by L36 |
| DPIA-mappable audit retention | § Phase 3.2 | Tenant-config policy + audit-trail of every sealing / retirement / unseal |

## Must NOT do

* Don't break `voice-audit verify` — sealed segments must verify
  via a documented unseal-and-verify path.
* Don't lose the chain link across rotation boundaries — the
  first entry of each new segment is the prior segment's
  tail-hash.
* Don't store the encryption key alongside the sealed segments.
* Don't auto-decrypt on read — operator-initiated only.
* Don't put audit content, file content, key material, or TSA
  response body in `details` — allow-list enforces.
* Don't make `audit.rotation_failed` advisory; the rotation raises
  and the failure is in the chain.
* Don't make `audit.tsa_request_failed` CRITICAL — TSA is
  non-fatal by design; the seal is final and stands regardless.
* Don't change the TSA DER structure without testing against a
  real RFC 3161 TSA endpoint — a malformed request silently
  returns HTTP 400 or a PKIStatusInfo REJECTION on some TSAs.
* Don't `import anthropic` (CI lint).

## Validation

* `python3 operator/bridges/shared/test_audit_sealer.py` — 40
  tests pass (1 skipped under `age` absence). Coverage: policy
  dataclasses (incl. TSA fields), tenant config loader (incl. TSA
  round-trip), rotation gate (size + age), tail-hash extraction,
  rotation preserving chain link, sealing with fake sealer
  (chmod 444, plaintext removal, audit emission), seal-failure
  path (CRITICAL emission + raise), TSA happy-path (mock TSA,
  `.tsr` written, chmod 444, `audit.segment_timestamped` event),
  TSA failure non-fatal (WARNING emitted, seal stands, no `.tsr`),
  TSA disabled default (no `.tsr`, no events), TSA called with
  correct args (url + hash_algo), DER structure (59 bytes,
  correct tags, hash embedded), retention (retain recent, retire
  old, audit-before-removal), segment listing (live file excluded),
  audit allow-list, no-anthropic CI lint.
* `EVENT_SEVERITY` table carries all eight event types.

## Milestone status

* **M3 (done 2026-05-19):** Module + tests + ADR + ref doc. Rotation,
  sealing (`age`/`gpg`), retention, operator unseal, chain-link
  continuity across rotation boundaries.
* **M3+ (done 2026-05-21):** RFC 3161 TSA opt-in (`tsa_enabled`).
  Closes ADR-001 open item #1 (external timestamping).
* **M3.5 (done):** `voice-audit verify --include-sealed` walks all
  sealed segments and verifies cross-segment chain links. `.tsr` files
  are reported; operator verifies offline with `openssl ts -verify`.
* **M3.6 (done):** `bridge.sh doctor` (via `self_test.py`) adds an
  `egress.*` + sealer-binary check group confirming encryption-at-rest
  binary availability for the configured `sealer_cmd`.
* **M3.7 (done):** `atelier-audit-rotate.timer` daily systemd unit
  (`operator/voice/scripts/systemd/atelier-audit-rotate.timer`) invokes
  `audit_rotate.py` for scheduled rotation + sealing.
* **M4 (L36 done):** Erasure orchestrator (`erasure_orchestrator.py` +
  `erasure_handlers.py`) provides cross-layer GDPR Art. 17 deletion.
