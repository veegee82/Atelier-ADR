# ADR-0048 — Layer 38: RemoteTriggerReceiver + A2A TaskEnvelope Protocol

**Status:** Accepted
**Date:** 2026-05-22
**Companion to:** [0043 L35](0043-L35-egress-lockdown.md),
[0044 L37](0044-L37-audit-at-rest.md),
[0045 L36](0045-L36-erasure-orchestrator.md),
[0047](0047-hosted-tenant-console-byok.md)

---

## Context

AtelierOS Layer 29 (Delegation) enables an *outbound* spawn: the local
orchestrator persona triggers a WorkerEngine (L22) on behalf of a chat.
There is no *inbound* path — no way for a remote agent or cloud scheduler
to ask a local AtelierOS instance to execute a task, get a signed result,
and have that exchange anchored in the L16 audit chain.

The gap matters in three concrete scenarios:

1. **Hosted console** (`cloud.atelier.eu`) dispatches a nightly task
   (compliance scan, recall-index rebuild) to the operator's on-premise
   instance.
2. **Peer instances** — two AtelierOS nodes owned by the same operator
   delegate sub-tasks to each other.
3. **CI/CD or monitoring agents** trigger predefined L22 workers without a
   human chat-turn.

The design tension this ADR resolves:

* **Cryptographic provenance** — the receiving instance must be able to
  prove, after the fact, that a specific signed origin requested a specific
  task, and that the result returned matched the declared output schema.
* **Replay + injection** — a network attacker must not be able to replay
  a captured `TaskEnvelope` or forge an instruction by manipulating the
  transport.
* **Audit integrity** — every A2A exchange must land in the L16 hash
  chain before any engine-spawn side-effect occurs; a failed chain-write
  must block the spawn.
* **Scope containment** — the remote origin must not be able to exceed the
  same permissions a locally configured persona would have. L10 path-gate,
  L34 data-classification, and L35 egress limits apply without exception.

**Relationship to existing layers:**

| Layer | Role in L38 |
|---|---|
| L10 path-gate | Still the first FS-write defence; L38 spawns via L22 only |
| L16 hash chain | Anchor for every A2A audit event |
| L22 WorkerEngine | Execution substrate (M2+) |
| L29 delegation | Pattern reused; L38 is the inbound mirror |
| L34 data classification | Per-engine locality + egress matrix applied at spawn |
| L35 egress lockdown | `allowed_hosts` gate on origin check |

---

## Decision

Introduce **Layer 38** as a separate structural layer:

* New module `operator/bridges/shared/remote_trigger_receiver.py`.
* `TaskEnvelope` — the signed inbound request struct.
* `ResponseEnvelope` — the signed outbound result struct.
* `RemoteTriggerReceiver` — validates, audits, and (M2+) spawns.
* `OriginRegistry` — operator-managed list of trusted origins with their
  HMAC keys.
* Five new audit events on the L16 chain under the `A2A` namespace.

---

### TaskEnvelope shape

```python
@dataclass
class TaskEnvelope:
    task_id:       str    # UUID4, primary key for idempotency check
    nonce:         str    # cryptographic random (≥ 128 bit, hex)
    issued_at:     float  # Unix timestamp (float); valid window ±300 s
    origin_id:     str    # registered origin name, e.g. "cloud.atelier.eu"
    instruction:   str    # task text forwarded to WorkerEngine (M2+)
    result_schema: dict   # JSONSchema whitelist applied to worker output
    ttl_s:         int    # max engine wall-time (seconds); enforced at spawn
    signature:     str    # HMAC-SHA256(key, canonical_payload); hex-encoded
```

**Canonical payload** for HMAC computation (deterministic JSON, sorted keys,
no trailing whitespace):

```
HMAC-SHA256(
  key=origin_key,
  msg=json.dumps(
    {k: v for k, v in envelope.items() if k != "signature"},
    sort_keys=True, separators=(",", ":"), ensure_ascii=True
  ).encode()
)
```

The `instruction` field is included in the HMAC payload but MUST NOT appear
in any audit `details` (allow-list enforced).

### ResponseEnvelope shape

```python
@dataclass
class ResponseEnvelope:
    task_id:    str    # mirrors TaskEnvelope.task_id
    origin_id:  str    # mirrors TaskEnvelope.origin_id
    issued_at:  float  # local timestamp at response creation
    status:     str    # "ok" | "filtered" | "rejected" | "timeout"
    data:       dict   # worker output, post-filter (M2+); {} in M1
    signature:  str    # HMAC-SHA256 of canonical response payload; hex
```

Response signing uses a **receiver key** (separate from the origin key)
so the cloud-side verifier can authenticate the result without needing
the origin's secret.

### OriginRegistry

```python
# operator/cowork/remote_origins/<origin_id>.json  (mode 0600)
{
  "origin_id":  "cloud.atelier.eu",
  "hmac_key":   "<hex-encoded secret>",   # origin signs TaskEnvelope
  "recv_key":   "<hex-encoded secret>",   # receiver signs ResponseEnvelope
  "enabled":    true,
  "max_ttl_s":  300,                       # floor on ttl_s; envelope may lower
  "allowed_personas": ["assistant"]        # which personas may be spawned
}
```

Keys are read from the registry file per call (no in-process cache) to
ensure hot-rotation takes effect immediately. Mode 0600 is enforced at
load time; a world-readable key file is a CRITICAL self-test failure.

### Validation sequence (M1)

1. **Schema** — `TaskEnvelope` all fields present and typed.
2. **Origin** — `origin_id` in `OriginRegistry`; entry `enabled: true`.
3. **Time window** — `abs(now() - issued_at) ≤ 300 s`; rejects stale/future.
4. **Nonce** — `nonce` not seen before (TTL-keyed nonce store, 600 s window).
5. **TTL cap** — `ttl_s ≤ origin.max_ttl_s`.
6. **Signature** — HMAC-SHA256 recomputed over canonical payload; constant-time
   compare (`hmac.compare_digest`); reject on mismatch.
7. **Audit** — `A2A.envelope_received` MUST be written to L16 chain BEFORE
   returning any response. If the write fails, the whole request is rejected.

Steps 1–6 fail silently to the caller (all errors collapse to
`A2A.request_rejected` + `status: "rejected"` ResponseEnvelope). The
distinction is in the audit event's `reason` field — never in the HTTP body.

### Nonce store

```python
# in-memory dict, TTL-keyed: nonce → expiry_ts
# size capped at 10 000 entries; LRU eviction beyond cap
# persisted to <global>/remote_trigger/nonces.msgpack on shutdown (optional)
```

A replayed `TaskEnvelope` with the same `task_id + nonce` is rejected with
`A2A.request_rejected` + `reason: "replay"` regardless of time window.

### Audit contract

Five event types on the L16 hash chain:

| Event | Severity | When |
|---|---|---|
| `A2A.envelope_received` | INFO | After signature verification passes; BEFORE spawn |
| `A2A.engine_spawned` | INFO | After WorkerEngine spawn succeeds (M2+) |
| `A2A.result_filtered` | INFO | After schema filter applied to worker output (M2+) |
| `A2A.response_signed` | INFO | Immediately before returning ResponseEnvelope |
| `A2A.request_rejected` | WARNING | On any validation or runtime failure |

Allow-list keys in audit `details`:
`task_id`, `origin_id`, `persona`, `channel`, `chat_key`, `reason`,
`nonce_prefix` (first 8 hex chars only), `status`, `filter_pass_count`,
`filter_reject_count`, `engine_id`, `ttl_s`, `duration_ms`.

**Never** in `details`: `instruction`, `result_schema`, `data`, `signature`,
`hmac_key`, `recv_key`, full nonce, `issued_at` (timestamp only via
`duration_ms`), any worker output.

### Result filter (M2)

`result_schema` is a JSONSchema document. The worker output is validated
against it before inclusion in `ResponseEnvelope.data`. Fields not
declared in the schema are silently stripped. If no fields pass the filter,
`status` is `"filtered"` and `data` is `{}`. This is the structural defence
against a worker returning PII or oversized data to an unauthenticated
remote caller.

---

## Milestones

### M1 — Signature stack + audit anchoring (DONE)

**Scope:** protocol definition and validation only; no WorkerEngine spawn.

* `TaskEnvelope`, `ResponseEnvelope` dataclasses.
* `OriginRegistry` loader (mode 0600 check, per-call read).
* `RemoteTriggerReceiver.receive(envelope_dict) -> ResponseEnvelope`.
* Steps 1–7 of the validation sequence.
* Nonce store (in-memory, size-capped).
* All five `A2A.*` audit events emitted (engine-related events fire as
  no-ops in M1 — `A2A.engine_spawned` with `status: "skipped_m1"`).
* `ResponseEnvelope.data` is always `{}` in M1 (no spawn).
* Test suite: 51 cases.

### Protocol v2 — bidirectional instance attestation (DONE, M1 follow-up)

The original M1 protocol carried only the origin authentication. v2 adds
two fields, signed into the same HMAC payload:

* `TaskEnvelope.sender_instance_id` (required) — caller's local UUID,
  signed into the HMAC payload. Allow-listed audit key.
* `ResponseEnvelope.instance_id` (required) — receiver's local UUID,
  signed into the receiver-key HMAC. Sender verifies and refuses on
  configured-pin mismatch.

New module: `operator/bridges/shared/instance_identity.py` — persists
the local UUID at `<atelier_home>/global/instance_id.json` (mode 0600)
on first call. Thread-safe, label-mutable, self-heals world-readable
mode. Test suite: 19 cases.

### M2 — WorkerEngine integration + result filter + injection defence (DONE)

**Scope:** real spawn + output filtering + structural prompt-injection
defence. Opt-in per origin via ``spawn_worker: true``; default is M1
fallback (no spawn, empty data).

* New module: `operator/bridges/shared/a2a_worker.py` — single
  `spawn_a2a_worker(...)` entry point used by the receiver.
* Sanitization (raises `InjectionAttempt` on trip):
  - byte cap (`MAX_INSTRUCTION_BYTES = 16 KB`)
  - NFKC normalization (homoglyph collapse)
  - control-char strip (preserves `\t \n`)
  - literal `</a2a_instruction>` (case + whitespace tolerant) → reject
  - empty after strip → reject
* Trust-rules system prompt declares the `<a2a_instruction>` block as
  the only authoritative source; out-of-block text is *not*
  instructions. Origin attrs HTML-escaped against attribute-break
  attacks.
* Persona resolved via `allowed_personas[0]`.
* `ttl_s` enforced as engine wall-time cap (`timeout=` kwarg).
* Engine-agnostic via `engine_factory` constructor hook — defaults to
  `ClaudeCodeEngine`, tests inject fakes.
* Result filter (`result_schema` properties whitelist).
* `A2A.engine_spawned` and `A2A.result_filtered` carry real counts +
  `persona` + `engine_id` + `duration_ms`.
* `ResponseEnvelope.data` populated with filtered output.
* Test suite: 31 worker cases + 12 receiver-with-spawn cases.

**L34 / L35 gates:** the receiver does NOT call `_check_compliance_or_fail()`
or `egress_gate.validate()` directly today — those gates are owned by the
adapter spawn path (`adapter.py::_compliance_cache`). Since A2A spawns go
through the same `ClaudeCodeEngine.spawn()` surface, any future
monkey-patch operators install on `_check_compliance_or_fail()` covers
A2A automatically. Direct callsite invocation is a follow-up if A2A spawns
bypass the adapter path (none currently do).

### M3 — Bidirectional sender + multi-origin + console route + E2E (DONE)

**Scope:** outbound A2A, fan-out to multiple peers, console viewer, real
HTTP E2E in both directions.

* New module: `operator/bridges/shared/remote_trigger_sender.py` —
  `RemoteTriggerSender.send()` builds + HMAC-signs a TaskEnvelope, POSTs
  it via stdlib `urllib`, verifies the response signature, audits each
  step. `RemoteEndpointRegistry` reads
  `operator/cowork/remote_endpoints/<id>.json` per call. Three new
  `A2A.*` events: `envelope_sent` · `response_received` ·
  `response_rejected`. Test suite: 20 cases including a real HTTP fake
  receiver.
* Fail-silent rejection: the receiver returns an UNSIGNED
  `status: "rejected"` envelope for any of validation steps 1–7. The
  sender accepts this as a legitimate rejection only when
  `signature == ""` AND `data == {}` — closes the only window where an
  unsigned response is honoured, and only when no content can leak.
* Stdlib HTTP wrapper: `operator/bridges/shared/a2a_http_server.py` —
  thin `ThreadingHTTPServer` around the receiver for instances that do
  not run the full FastAPI gateway (edge boxes, sidecar deployments,
  E2E tests). Routes: `POST /v1/a2a/receive` · `GET /healthz`. 64 KB
  inbound body cap.
* Bidirectional E2E: `test_a2a_bidirectional.py` spawns two real HTTP
  receivers on ephemeral 127.0.0.1 ports, generates key pairs, exchanges
  envelopes in both directions, asserts the receiver's `instance_id` in
  the response, asserts pin-mismatch rejection, asserts replay
  rejection, asserts injection-attempt rejection, asserts the audit
  chain never contains the instruction text. 10 cases.
* Console route: `core/console/atelier_console/routes/remote_trigger_log.py`
  — `GET /v1/console/remote-trigger/log` returns the last N A2A events
  grouped by peer, plus `/remote-trigger/origins` and
  `/remote-trigger/endpoints` (no secrets exposed). 12 helper-level
  tests.
* CLI tooling: `atelier-instance-id` and `atelier-a2a` (with
  subcommands: `pair`, `send`, `list-origins`, `list-endpoints`,
  `show-origin`, `show-endpoint`). Pair command writes a local origin
  file and prints the corresponding peer-side endpoint file for
  out-of-band transport.

### Protocol v3 — Binary attachments + scratch workspace (DONE)

A2A v3 extends the envelope schema to carry binary payloads in both
directions. New module: ``operator/bridges/shared/a2a_attachments.py``.

**TaskEnvelope shape (v3 additions):**

```python
@dataclass
class TaskEnvelope:
    # ... v2 fields ...
    attachments: list   # list of {name, mime, sha256, content_b64}
```

**ResponseEnvelope shape (v3 additions):**

```python
@dataclass
class ResponseEnvelope:
    # ... v2 fields ...
    attachments: list   # list of {name, mime, sha256, content_b64}
```

**Attachment shape:**

```python
@dataclass
class Attachment:
    name:        str   # sanitized: [A-Za-z0-9_-][A-Za-z0-9._-]{0,127}
    mime:        str   # media type, e.g. "image/png", "text/csv"
    sha256:      str   # hex digest of the raw bytes (signed into HMAC)
    content_b64: str   # base64-encoded payload
```

**Structural invariants:**

- ``MAX_ATTACHMENTS_COUNT = 16`` (per envelope).
- ``MAX_ATTACHMENTS_TOTAL_BYTES = 1 MiB`` (per envelope, raw bytes).
- ``MAX_ATTACHMENT_NAME_LEN = 128``.
- Name pattern excludes path separators, leading dots, ``..``.
- Receiver computes ``sha256(b64decode(content_b64))`` and rejects
  on mismatch (defence against in-flight modification).
- HTTP body cap raised to **4 MiB** to fit one max-cap attachment +
  base64 overhead + JSON wrapper.
- Audit allow-list extended with ``attachments_count``,
  ``attachments_total_bytes``, ``attachment_names`` (sanitized),
  ``attachment_sha_prefixes`` (16 hex chars). **Never** the full
  sha256, **never** the content.

**Worker workspace (M2+ flows):**

When ``spawn_worker=true``, the receiver now provisions a private
scratch tempdir (mode 0700) per spawn:

  ``<scratch>/in/``  — inbound attachments dropped here, digest-verified
  ``<scratch>/out/`` — worker writes outputs here

After the worker exits, the receiver harvests files matching
``result_schema.attachments_out_allowed`` (whitelist; if absent, no
attachments are returned). Each harvested file is re-validated through
the same caps and digest pipeline before being signed into the
ResponseEnvelope.

**Deterministic compute engine** (``a2a_compute_engine.py``):

A WorkerEngine implementation that runs a fixed CSV-summary pipeline
deterministically (no LLM). Currently:

  ``csv_summary`` — for each numeric column in ``in/*.csv``, compute
  count/mean/stdev/min/max; render matplotlib histogram of the first
  numeric column.

This proves the path is engine-agnostic and supports trustless cheap
compute as well as full-LLM workers.

**LIVE compute E2E** (``test_a2a_e2e_compute.py``):

Two real HTTP receivers paired via real HMAC keys. Cloud sends a
~800 KB slice of the Spotify charts dataset (real CSV from
``/home/shumway/projects/datasets/archive/charts_songs_daily.csv``) as
an attachment. Local invokes ``DeterministicComputeEngine``, harvests
``summary.json`` + ``histogram.png`` from the scratch dir, signs them
into the response, returns to Cloud. Cloud verifies signature, pin,
and per-attachment digest. Both directions work. Proof artifacts
written to ``/tmp/atelier_a2a_e2e_proof/`` for visual inspection.

### M4 (future) — Persistence + key rotation tooling

Out of scope for the current milestone but next on the roadmap:

* Persistent nonce store (SQLite or Redis) to survive a restart inside
  the 600 s nonce window.
* `atelier-a2a rotate <origin_id>` CLI: atomically generates a new key
  pair, writes new registry file, emits `A2A.key_rotated` audit event.
* Cross-instance key-exchange protocol (today: operator-managed
  out-of-band — signed email, age-encrypted file, etc.).

---

## Consequences

### Positive

* Any A2A exchange is **cryptographically bound** to the originating
  key, the exact instruction hash (HMAC includes `instruction`), and
  the result — making post-hoc repudiation impossible without breaking
  HMAC.
* **Replay-proof** — nonce + time window together ensure a captured
  envelope cannot be reused.
* **Scope-contained** — a remote origin cannot exceed the privileges of
  `allowed_personas`; L10/L34/L35 gates apply identically to local spawns.
* **Audit-first** — chain write is a prerequisite for any response;
  a failed write blocks the request, not the other way around.
* **Fail-silent** — validation errors return an identical `"rejected"`
  response; the distinction lives only in the audit chain, preventing
  oracle attacks.

### Negative / trade-offs

* Operators must manage HMAC key material out-of-band (secure key
  exchange with the remote origin). L38 does not define a key-exchange
  protocol — that is operator responsibility.
* Nonce store is in-memory by default. A process restart inside the
  600 s nonce window could theoretically allow a replay. The optional
  msgpack persistence mitigates this; a Redis/DB backend is a follow-up.
* `result_schema` validation requires the origin to correctly declare its
  expected output. A too-permissive schema weakens the filter. Operator
  responsibility; documentation covers this.

### Out of scope

* **HTTP transport / webhook endpoint** — L38 is a protocol and validation
  layer. Transport (HTTP, gRPC, message queue) is operator-chosen and
  wired at the adapter level.
* **Async / queued execution** — `RemoteTriggerReceiver.receive()` is
  synchronous in M1 and M2. A job-queue abstraction is a follow-up.
* **Multi-tenant cross-dispatch** — one `RemoteTriggerReceiver` per tenant.
  ADR-0007 silo applies; cross-tenant dispatch is out of scope.
* **Key rotation protocol** — operators rotate keys by replacing the
  registry file; L38 picks up the new key on the next call. Rotation
  tooling is a follow-up.

---

## Compliance mapping

| Requirement | Source | This ADR delivers |
|---|---|---|
| Provenance of automated processing | GDPR Art. 30 | `A2A.*` events in L16 hash chain, immutable |
| Access control for remote callers | GDPR Art. 32 | `OriginRegistry`, HMAC verification, `allowed_personas` |
| Data minimisation on result path | GDPR Art. 5(1)(c) | `result_schema` filter strips undeclared fields |
| Tamper-evident cross-instance trace | EU AI Act Art. 14 | L16 anchor before any spawn; audit-first invariant |
| Compliance zone enforcement | ADR-0007 | L34 + L35 gates unchanged at spawn site |

---

## Must NOT do

* Don't skip HMAC verification for any origin, including `localhost` or
  `127.0.0.1` origins — origin identity is cryptographic, not network-based.
* Don't put `instruction`, worker output, or `signature` in any audit
  `details` field — allow-list enforces.
* Don't make `A2A.request_rejected` advisory; it is WARNING and the
  caller receives only the opaque `"rejected"` status.
* Don't cache the `OriginRegistry` across calls — hot key-rotation requires
  per-call reads.
* Don't make a world-readable registry file fail-open; it is a CRITICAL
  self-test failure.
* Don't allow `result_schema: {}` (empty schema) to pass all fields through
  unchecked — an empty schema MUST produce `data: {}` (zero-field filter).
* Don't spawn a WorkerEngine if the L16 audit write fails (audit-first
  invariant).
* Don't bypass L10, L34, or L35 for A2A spawns — same gates as local spawns.
* Don't `import anthropic` (CI AST lint).

---

## Validation

Test surface, after M1+M2+M3:

| Suite | Count |
|---|---|
| `test_instance_identity.py` | 19 |
| `test_remote_trigger_receiver.py` (M1) | 51 |
| `test_remote_trigger_receiver_m2.py` (M2 spawn paths) | 12 |
| `test_remote_trigger_sender.py` | 20 |
| `test_a2a_worker.py` (sanitization + framing + spawn) | 31 |
| `test_a2a_bidirectional.py` (real HTTP E2E, both directions) | 10 |
| `test_a2a_attachments.py` (v3 attachment validation) | 24 |
| `test_a2a_e2e_compute.py` (LIVE compute E2E, real CSV → real PNG) | 2 |
| `core/console/tests/test_remote_trigger_log.py` (console route helpers) | 12 |
| **Total** | **181** |

Run all at once via `bash operator/bridges/run-all-tests.sh`.

Coverage targets (M1 superset, kept current):

  - `TaskEnvelope` shape validation (all fields present/typed, extras ignored).
  - `OriginRegistry` (known origin, unknown origin, `enabled: false`,
    mode-check on key file, per-call re-read).
  - Time-window acceptance (in-window, expired, future).
  - Nonce store (fresh nonce accepted; same nonce rejected; nonce expiry;
    cap + LRU eviction).
  - TTL cap (`ttl_s ≤ max_ttl_s` enforced; exceeding cap rejected).
  - Signature (correct HMAC accepted; wrong key rejected; bit-flip rejected;
    constant-time compare used — no timing side-channel).
  - Audit-first invariant (chain write fail → request rejected, no spawn).
  - Audit allow-list (smuggled keys stripped; `instruction` not present).
  - Fail-silent: all error paths return identical `"rejected"` ResponseEnvelope.
  - `A2A.envelope_received` emitted before any response on success path.
  - `A2A.request_rejected` emitted on every failure path with `reason`.
  - M1 no-op: `A2A.engine_spawned` emitted with `status: "skipped_m1"`.
  - Response signing (HMAC over canonical response payload, receiver key).
  - CI lint (no `import anthropic`).

---

## Future work (M4+)

M1, M2, M3 all complete. Open items for later milestones:

* **Persistent nonce store** (M4): optional SQLite / Redis backend to
  survive a process restart within the 600 s nonce window. Today an
  in-memory store + 600 s window is acceptable for the threat model:
  receivers that restart frequently can already raise the window via
  config without changing the protocol.
* **Key-rotation tooling** (M4): `atelier-a2a rotate <origin_id>` CLI
  generates a new key pair, writes both sides atomically, emits
  `A2A.key_rotated` audit event. Coordination protocol with the peer
  is operator-managed today (out-of-band).
* **Cloud-side reference client as standalone package** (M4): factor
  out the `remote_trigger_sender.py` + minimal stdlib deps into a
  PyPI-distributable package so non-AtelierOS callers (CI/CD agents,
  monitoring stacks) can dispatch envelopes without vendoring the
  whole repo.
* **Direct L34/L35 callsite enforcement** (M4): today the gates
  inherit through the adapter spawn path; an explicit call inside
  `_spawn_and_filter()` would close the door on alternative spawn
  paths that bypass the adapter monkey-patch hook.
