# ADR-0047 — Hosted-Mode Tenant Console + BYOK Key Management

**Status:** Implemented — M1 + M2 + M3 + M4 done (2026-05-22)
**Owner:** Silvio Jurk (Maintainer)
**Sister-ADRs:** 0007 (multi-tenant), 0017 (enterprise control plane),
0034 (operator bundle), 0037 (console relaunch), 0038 (remote deploy),
0044 (L37 audit-at-rest), 0046 (v1 release readiness)

---

## Context

ADR-0037 relaunches the console as a React/Vite SPA served co-located with
the instance (`127.0.0.1:8765`). This works for **self-hosted deployments**
where the operator has shell access to the machine.

In **hosted mode** (Atelier SaaS — one container per tenant on shared
infrastructure) two structural gaps emerge:

1. **No remote console access.** `127.0.0.1:8765` is only reachable from
   the container itself. Tenants have no shell access. They have no way to
   configure their instance after provisioning.

2. **No self-service key entry.** The platform needs tenant-supplied API
   keys (Anthropic, OpenAI, STT providers). In self-hosted mode the
   operator writes keys to the vault on disk. In hosted mode there is
   no disk access — keys must come from a browser UI. The management plane
   (Atelier's servers) **must never see plaintext secret values** (GDPR Art.
   32, principle of minimal exposure, EU AI Act Art. 14 confidentiality).

This ADR decides the architecture that closes both gaps while preserving
all existing security invariants (L16 vault, L10 path-gate, L34–L35
classification/egress, L36 erasure).

---

## Decision

Introduce a **two-plane architecture** for hosted mode:

| Plane | What runs there | Who operates it |
|---|---|---|
| **Control plane** | Central console SPA + Management API | Atelier (shared infrastructure) |
| **Data plane** | Per-tenant AtelierOS instance + Instance Agent | Atelier (isolated per tenant) |

The console UI is hosted at `console.atelierOS.io` (or operator-custom
domain). It communicates with the **Management API** (control plane). The
Management API has a secure channel to each tenant's **Instance Agent**
(data plane). The agent applies configuration changes locally on the
instance.

**BYOK key entry uses client-side encryption:** plaintext key values
are encrypted in the browser with the instance's RSA-OAEP public key
before they leave the client. The management plane stores only the
ciphertext and never holds a decryption key.

---

## Architecture

### Component map

```
Browser (tenant)
   │  OIDC login (ADR-0007 OIDC provider)
   │  HTTPS
   ▼
Control Plane
┌─────────────────────────────────────────────────┐
│  console.atelierOS.io                           │
│  React SPA (ADR-0037 web-next, reused)          │
│           │  REST / WebSocket                   │
│  ┌────────▼──────────────────────────────────┐  │
│  │  Management API  (api.atelierOS.io)        │  │
│  │  • tenant registry                        │  │
│  │  • instance health proxy                  │  │
│  │  • encrypted-blob store (ciphertext only) │  │
│  │  • config-push channel (gRPC or SSE)      │  │
│  └────────┬──────────────────────────────────┘  │
└───────────┼─────────────────────────────────────┘
            │  mTLS  (per-tenant client cert)
            ▼
Data Plane (isolated container, per tenant)
┌─────────────────────────────────────────────────┐
│  Instance Agent  (new: operator/agent/)          │
│  • registers with Management API at boot        │
│  • exposes health + metrics endpoint            │
│  • receives config-push events                  │
│  • decrypts BYOK blobs → writes to L16 vault   │
│  • NEVER forwards decrypted values out          │
│                                                 │
│  AtelierOS core (unchanged)                     │
│  L16 vault → bwrap env → worker engines         │
└─────────────────────────────────────────────────┘
```

### BYOK key-entry flow

```
1. Tenant opens console → "API Keys" page
2. Console: GET /api/v1/tenants/{tid}/byok-pubkey
   → Management API returns instance's RSA-OAEP-2048 public key (DER, cached)
3. Tenant types API key value into browser input (never sent in plaintext)
4. Browser: Web Crypto API (SubtleCrypto.encrypt, RSA-OAEP, SHA-256)
   → ciphertext ← encrypt(pubkey, plaintext_api_key)
5. Console: POST /api/v1/tenants/{tid}/secrets/{key_name}
   body: { ciphertext: "<base64>", algorithm: "RSA-OAEP-SHA256" }
6. Management API:
   a. validates tenant auth (OIDC JWT)
   b. stores ciphertext in secrets table (per tenant, per key_name)
   c. emits config-push event to Instance Agent
   d. emits audit event: secret.byok_updated { key_name, tenant_id }
      — NO ciphertext, NO plaintext in audit
7. Instance Agent receives push:
   a. fetches ciphertext from Management API (mTLS)
   b. decrypts with RSA-OAEP private key (stored in L16 vault, never leaves instance)
   c. writes plaintext to L16 vault (mode 0600)
   d. emits local audit event: vault.secret_rotated { key_name }
   e. signals adapter hot-reload
8. Console shows: "Key updated. Last rotated: <timestamp>."
   Last 4 chars of the plaintext are echoed back from the agent (optional, opt-in only).
```

### Instance Agent registration

```
Boot sequence (instance container):
1. Agent generates RSA-2048 keypair if not present in L16 vault
2. Agent POSTs to Management API:
   { tenant_id, pubkey_pem, instance_version, capabilities[] }
   Auth: pre-shared provisioning token (one-time, written into container env at deploy time)
3. Management API issues a per-instance mTLS client cert (short-lived, 30-day rotation)
4. Agent stores cert in L16 vault; provisioning token is deleted
5. All subsequent agent ↔ Management API traffic uses mTLS
```

### Console sections (hosted mode)

| Section | Content | Backend route |
|---|---|---|
| Overview | Instance health, version, uptime | proxy via agent |
| API Keys | BYOK entry (per key_name) | secrets store (ciphertext only) |
| Bridges | Active bridges, whitelist, rate-limit | config-push to agent |
| Voice | STT provider, voice profile fields | config-push to agent |
| Quota / Usage | Requests today/month, quota ceiling | Management API meter |
| GDPR | Consent status, erasure request (L36), `/forget` | proxy via agent |
| Audit | Last N audit events (filtered, no raw text) | proxy via agent |
| Settings | Personas, LDD layers (L14), engine policy | config-push to agent |

---

## Key management details

### Private key lifecycle

- Generated once at first boot inside the container.
- Stored at `<vault>/byok.key` (mode 0600, never world-readable).
- NEVER leaves the container — not in logs, not in the audit chain, not in
  any Management API call.
- Rotated via: agent generates new keypair → registers new pubkey with
  Management API → re-encrypts all stored secrets client-side (or
  Management API re-pushes all blobs for re-encryption trigger).

### Key-name namespace

```
anthropic_api_key
openai_api_key
stt_openai_api_key
stt_local_whisper_api_key   (reserved for future local provider auth)
custom_<slug>               (operator-defined, up to 32 chars, slug-safe)
```

Forbidden names: any string containing `audit`, `vault`, `path_gate`,
`policy`, `license` (collision with system internals).

### What the Management API stores (secrets table)

```sql
CREATE TABLE tenant_secrets (
    tenant_id   TEXT NOT NULL,
    key_name    TEXT NOT NULL,
    ciphertext  BLOB NOT NULL,          -- RSA-OAEP ciphertext, base64
    algorithm   TEXT NOT NULL DEFAULT 'RSA-OAEP-SHA256',
    updated_at  TIMESTAMP NOT NULL,
    updated_by  TEXT NOT NULL,          -- OIDC subject (hashed), not raw email
    PRIMARY KEY (tenant_id, key_name)
);
-- NEVER store plaintext. NEVER log ciphertext in application logs.
```

---

## Phased implementation

### M1 — Instance Agent scaffold (2 weeks)
- `operator/agent/` package: registration, health endpoint, mTLS setup.
- Keypair generation + L16 vault integration.
- Management API stubs: `/register`, `/health-proxy`.
- Audit events: `agent.registered`, `agent.heartbeat_missed`.
- E2E test: agent registers, Management API responds, mTLS verified.

### M2 — BYOK pipeline (2 weeks)
- Management API secrets table + `/byok-pubkey` + `/secrets/{key_name}`.
- Console "API Keys" page (Web Crypto encrypt, no plaintext in flight).
- Agent decrypt + vault write + hot-reload signal.
- Audit events: `secret.byok_updated`, `vault.secret_rotated`.
- E2E test: key entered in browser → decrypted in agent → readable from vault.

### M3 — Console hosted-mode sections (3 weeks)
- All 8 console sections wired to Management API proxied routes.
- Config-push for bridges, voice, settings via agent.
- GDPR section: consent toggle, erasure request form (calls L36 orchestrator).
- Feature flag: `ATELIER_HOSTED_MODE=true` (env var, set by deploy tooling).

### M4 — Hardening + production gate (1 week)
- Cert rotation automation (30-day mTLS certs via ACME or internal CA).
- Rate-limit on BYOK endpoint (max 10 updates/minute/tenant).
- Management API: ciphertext size cap (4 KB), key_name allowlist enforced.
- Penetration test checklist: no plaintext in transit, no key material in logs.
- `bridge.sh doctor` extended: agent heartbeat check → CRITICAL if missed > 5 min.

---

## Compliance

| Requirement | Mechanism |
|---|---|
| GDPR Art. 32 (security of processing) | Client-side encryption; management plane stores ciphertext only |
| GDPR Art. 5 (data minimisation) | key_name in audit only, never key value; no plaintext transit |
| GDPR Art. 36 (DPO consultation) | L36 erasure orchestrator handles ErasureRequest for tenant secrets |
| EU AI Act Art. 14 (human oversight) | Console gives operator full visibility + control over instance config |
| L16 vault invariant | Decrypted values written only to vault; never in LLM context |
| L10 path-gate | Agent writes to vault via L16 API, not direct FS write |
| L35 egress | Management API hostname added to `allowed_hosts` in hosted preset |
| L34 classification | BYOK secrets = CONFIDENTIAL; data plane locality = local |

---

## Must NOT do

- **Must NOT** store or log plaintext API key values anywhere outside the
  instance vault — not in Management API DB, not in application logs,
  not in the audit chain.
- **Must NOT** transmit ciphertext in audit `details` (ciphertext is still
  sensitive metadata; audit records only `key_name` + `updated_at`).
- **Must NOT** accept `custom_<slug>` key names containing system-reserved
  substrings (`audit`, `vault`, `policy`, `license`, `path_gate`).
- **Must NOT** allow the Management API to initiate decryption — decryption
  happens exclusively inside the data-plane instance agent.
- **Must NOT** share the RSA private key across tenants or across instance
  restarts without explicit key rotation.
- **Must NOT** make `ATELIER_HOSTED_MODE=true` silently change compliance
  behavior — all L34–L36 invariants apply identically in hosted and
  self-hosted modes.
- **Must NOT** proxy raw audit log content through the Management API —
  the console audit section shows filtered, structured events only.
- **Must NOT** reuse the provisioning token after registration; it MUST be
  deleted from the container env after the mTLS cert is issued.
- **Must NOT** `import anthropic` from `operator/agent/`.

---

## Alternatives considered

### A: Per-tenant subdomain + direct TLS termination
Expose each instance's console port publicly at `<tid>.console.atelierOS.io`.
Simple but requires per-tenant TLS cert provisioning, complex ingress routing,
and gives no central management view across tenants. BYOK key entry would still
need the same encryption scheme. Rejected: operational complexity outweighs
simplicity gain.

### B: SSH tunnel on demand
Tenant opens a temporary SSH tunnel to their container. Avoids the Management
API entirely. Rejected: requires tenant to install SSH tooling, is not
browser-native, and does not solve the BYOK use case.

### C: Push-only config via signed S3 / object store
Tenant uploads encrypted config blobs to a shared object store; agent pulls
on a schedule. Simpler than a Management API but has high latency (polling
interval), no real-time health visibility, and no WebSocket/SSE push path.
Rejected: poor UX for a console that should feel live.

---

## References

- ADR-0007: Multi-tenant model, OIDC auth, `validate_tenant_id()`
- ADR-0016 / L16: Secret vault, bwrap env injection
- ADR-0017: Enterprise Control Plane (admin-UI patterns reused)
- ADR-0037: Console relaunch (web-next React SPA reused for hosted console)
- ADR-0038: Remote deploy / Tailscale (data-plane isolation patterns)
- ADR-0044: L37 audit-at-rest (sealed segments referenced by console audit section)
- ADR-0045: L36 erasure orchestrator (GDPR erasure for tenant secrets)
- [Web Crypto API — SubtleCrypto.encrypt](https://www.w3.org/TR/WebCryptoAPI/#dfn-SubtleCrypto-method-encrypt)
- [RFC 8017 — RSAES-OAEP](https://datatracker.ietf.org/doc/html/rfc8017#section-7.1)
