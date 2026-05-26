# ADR-0031 — Compute Layer License Gate + BSL-1.1 Source License

**Status:** Accepted  
**Date:** 2026-05-17  
**Author:** Maintainer  
**Supersedes:** None  
**Amends:** ADR-0017 (Enterprise Control Plane — adds `compute` and `compute_fabric` feature flags)  
**Amends:** ADR-0013 (Compute Worker — adds license gate at MCP entry point)  
**Amends:** ADR-0026 (Compute Fabric — adds `compute_fabric` license gate)  
**Depends on:** ADR-0013 (Compute Worker), ADR-0017 (Phase III RS256 JWT), ADR-0026 (Compute Fabric)

---

## Context

### The Compute Layer is the strongest Enterprise differentiator

ADR-0013 introduced an out-of-LLM-loop iteration worker (grid / random / Bayesian
optimisation) and ADR-0026 extended it with the Compute Fabric (parallel workers,
sharding, DataSourceAdapter). Both features are structurally absent from every other
conversational-AI platform on the market: they make AtelierOS the first system that
can run thousands of hyperparameter iterations *without burning a single LLM token
per step* and produce a GDPR Art. 30 audit trail for every iteration.

### Current state: no access control

As of ADR-0026, the compute surface has a *tenant-config* opt-in
(`spec.compute.enabled: false` / `spec.compute.fabric_enabled: false`) but no
*license* gate. Any operator who installs `core/compute/` gets the full
feature regardless of their commercial relationship. This is a business-model problem:

- There is no differentiation between evaluation, development, and production use.
- There is no lever to protect Fabric (the enterprise-grade tier) from being used
  without a commercial agreement.
- There is no trial funnel that converts evaluating operators into paying customers.

### Requirements

1. **Source-available**: the compute plugin source must remain readable and forkable.
   Closed-source compute breaks the trust model for security-conscious enterprise
   operators who need to audit what runs on their data.
2. **Test-before-you-buy**: companies must be able to evaluate the feature without
   signing a contract or providing a credit card.
3. **Production use requires a license**: when an operator integrates the compute
   layer into a product or deploys it in a production environment, a commercial
   license is required.
4. **No phone-home**: license validation must be fully local (RS256 JWT against a
   pinned public key), consistent with ADR-0017 Phase III.
5. **Graceful degradation**: an expired license gives a 30-day grace period before
   access is revoked, consistent with the existing `atelier-license` grace machine.
6. **Fabric is enterprise-only**: the ADR-0026 Compute Fabric (parallel workers,
   sharding, oracle, datasource adapters) is gated at a higher tier than basic
   compute.

---

## Decision

### 1. Source License: Business Source License 1.1 (BSL-1.1)

`core/compute/` is re-licensed from Apache-2.0 to **BSL-1.1** with the
following parameters:

| BSL-1.1 Parameter | Value |
|---|---|
| **Licensor** | Maintainer (Silvio Jurk) |
| **Licensed Work** | AtelierOS Compute Plugin (`core/compute/`) |
| **Additional Use Grant** | Non-production use (testing, evaluation, internal development) is permitted without restriction. Production use requires a valid production license key obtained from the Licensor. |
| **Change Date** | Four years from the release date of each version |
| **Change License** | Apache License 2.0 |

**Rationale for BSL-1.1 over alternatives:**

| Alternative | Why rejected |
|---|---|
| SSPL | Too aggressive — requires open-sourcing the entire operator stack when run as a service; deters legitimate enterprise customers |
| Elastic License 2.0 | No automatic conversion to open source — community has no long-term guarantee |
| Closed source | Breaks the trust model; security-sensitive operators cannot audit compute code that runs on their data |
| Commons Clause | Legally ambiguous; poor ecosystem reputation |
| **BSL-1.1** | Source visible, evaluation free, production gated, automatic Apache-2.0 after 4 years — well-understood (HashiCorp, MariaDB, CockroachDB) |

The rest of AtelierOS (`operator/forge/`, `operator/voice/`, `core/license/`,
etc.) stays Apache-2.0. The compute plugin is the only BSL component.

### 2. Two New License Feature Flags (ADR-0017 Amendment)

Two new feature flags are added to the `atelier-license` tier map:

| Flag | Tier | Meaning |
|---|---|---|
| `compute` | pro, business, enterprise | Access to the core Compute Worker (ADR-0013): `compute_run`, `compute_status`, `compute_result`, `compute_abort`. All three strategies (grid, random, bayesian). |
| `compute_fabric` | enterprise only | Access to the Compute Fabric (ADR-0026): `compute_job_create`, `compute_parallel_run`, `compute_shard_plan`, `compute_resource_status`, all datasource tools. Requires `compute` implicitly. |

Both flags are added to `VALID_FEATURE_FLAGS` in `atelier_license/verifier.py` and to
the canonical `TIER_FLAGS` dict in `atelier_license/tier_flags.py`.

### 3. Trial Mode (Free Tier / No License)

When no valid license is installed (free tier, evaluation), the gate enters **trial
mode** rather than denying access:

| Parameter | Trial limit |
|---|---|
| Total `compute_run` submissions | 500 (deployment-wide) |
| Strategies allowed | `grid`, `random` (Bayesian requires a license) |
| Max concurrent workers | 1 |
| Expiry | None — the cap is on total iterations, not wall-clock time |
| Watermark | `_license: {_atelier_trial: true, iterations_remaining: N, upgrade: URL}` injected into every `compute_run` response |

Trial state is persisted in:
```
<atelier_home>/global/license/trial_compute.json  (mode 0o600)
```

Schema:
```json
{
  "iterations_used": 0,
  "first_run_at": null,
  "schema_version": 1
}
```

**Tamper protection:** if the state file is world-readable (mode != 0o600),
`iterations_used` is treated as `TRIAL_ITERATION_CAP` immediately. Deleting the file
resets the counter — this is intentional; the file is an advisory limit for honest
operators, not a cryptographic enforcement mechanism. Production enforcement requires
the license JWT.

### 4. License Gate Implementation

The gate lives in `core/compute/atelier_compute/license_gate.py`.
It is called by the Forge MCP server (`operator/forge/forge/mcp_server.py`) at the
`_call_compute_tool()` and `_call_fabric_tool()` entry points.

**Gate invocation rule:**
- `compute_run` — gated.
- `compute_submit` — gated (when ADR-0029 engine tools are fully wired).
- `compute_status`, `compute_result`, `compute_abort`, `compute_gate` — **never gated**. Stranding an in-flight run on a license change would be hostile to operators.
- All Fabric tools (`compute_job_create`, `compute_parallel_run`, etc.) — gated,
  requires `compute_fabric` flag (enterprise tier).

**Access resolution (in order):**

```
atelier-license not installed?
  → trial mode

license.jwt missing?
  → trial mode

license.jwt present but invalid signature / malformed?
  → trial mode  (not denied — invalid token ≠ hostile actor in most cases)

license.jwt expired?
  → check grace period (ADR-0017 Phase III grace machine)
      in-grace  → licensed access + audit warning
      past grace → denied

license.jwt valid, `compute` flag absent?
  → trial mode  (license installed but tier too low)

license.jwt valid, `compute` flag present?
  → licensed access
      `compute_fabric` flag also present? → Fabric also allowed
      otherwise → Fabric denied (ComputeFabricLicenseRequired)
```

**Fail-open on gate errors:** any unexpected exception inside the gate (I/O error,
import failure, unexpected JWT shape) falls through to trial mode and emits a
`compute.license.gate_error` audit event. The gate must never crash the compute
surface.

### 5. Audit Events

Every gate invocation emits a structured event into the forge audit chain:

| Event type | When |
|---|---|
| `compute.license.checked` | Every `compute_run` — includes `allowed`, `mode`, `tier` |
| `compute.license.denied` | Access blocked (cap reached, expired past grace, etc.) |
| `compute.license.fabric_checked` | Every Fabric tool call |
| `compute.license.fabric_denied` | Fabric access blocked |
| `compute.license.gate_error` | Unexpected exception in the gate itself |

No PII enters the audit events. `customer_id` is always fingerprinted (first 12 hex
chars of SHA-256), consistent with the existing `atelier-license` audit contract.

### 6. Trial Onboarding Funnel

```
GitHub README (atelier-compute)
  └── "Trial: 500 iterations free — no signup required"
  └── "Production license: atelier.ai/compute-license"

compute_run response (trial mode)
  └── _license.iterations_remaining: N
  └── _license.upgrade: "https://atelier.ai/compute-license"

compute_run response (cap reached)
  └── isError: true
  └── error: "ComputeLicenseRequired"
  └── message: "Trial limit of 500 iterations reached. Upgrade at ..."

License purchase → JWT emailed to operator
  └── operator installs at <atelier_home>/global/license/license.jwt (mode 0o600)
  └── gate reads it on next compute_run — no restart required
```

---

## Consequences

### Positive

- **Revenue lever without breaking the community**: source is readable, trial is
  genuinely useful (500 iterations covers most evaluation scenarios), production
  deployments require a commercial relationship.
- **No vendor lock-in fear**: BSL with Apache-2.0 Change Date gives operators a
  time-bound guarantee that the code eventually becomes fully open.
- **Consistent with existing infrastructure**: reuses ADR-0017 RS256 JWT + grace
  machine + audit chain — no new cryptographic primitives.
- **Fail-safe**: gate errors fall through to trial mode; operators are never locked
  out by a gate bug.

### Negative / Trade-offs

- **BSL is not OSI-approved**: some enterprise procurement policies prohibit
  non-OSI-approved licenses. Mitigation: the automatic Apache-2.0 conversion at the
  Change Date and the generous Additional Use Grant cover most cases in practice.
- **Bayesian strategy unavailable in trial**: the most powerful optimisation strategy
  is behind the license wall. This is intentional — it is the clearest capability
  differentiation between trial and paid.
- **Trial cap is advisory, not cryptographic**: a determined operator can delete the
  trial state file to reset the counter. Acceptable for the intended audience
  (enterprise operators evaluating in good faith); deliberate abusers who need a
  license for production use will need one regardless of the trial counter.

---

## Must NOT Do (Claude Code load-bearing constraints)

- **Don't gate `compute_status`, `compute_result`, or `compute_abort`** — stranding
  in-flight jobs on a license change is hostile to operators.
- **Don't make the gate fail-closed on exceptions** — gate errors fall through to
  trial mode; the audit event captures the problem.
- **Don't add a `compute_license_off` env-var or bypass flag** — consistent with the
  CLAUDE.md "compliance-off mode" prohibition.
- **Don't add a new tier or move a flag between tiers without an ADR amendment** —
  the tier map is a commercial-policy document, not a code convenience.
- **Don't put `customer_id` raw in any audit event** — always fingerprint via
  `fingerprint_customer_id()`.
- **Don't call `check_compute_access()` from worker-side code** (`worker.py`,
  `client.py`) — the gate lives exclusively in the MCP server entry point.
- **Don't apply the gate to `compute_submit` or `compute_gate`** until ADR-0029
  engine tools are fully wired in `mcp_server.py`.

---

## Implementation Inventory

| File | Change |
|---|---|
| `core/compute/atelier_compute/license_gate.py` | **New** — gate logic, trial state, `ComputeAccessResult` |
| `core/compute/LICENSE` | **New** — BSL-1.1 text |
| `core/license/atelier_license/tier_flags.py` | **Modified** — `compute` added to pro/business/enterprise; `compute_fabric` added to enterprise |
| `core/license/atelier_license/verifier.py` | **Modified** — `compute`, `compute_fabric` added to `VALID_FEATURE_FLAGS` |
| `operator/forge/forge/mcp_server.py` | **Modified** — gate in `_call_compute_tool()` and `_call_fabric_tool()`; lazy import of `license_gate` |
| `core/compute/tests/test_license_gate.py` | **New** — unit tests for gate, trial state, strategy enforcement, tamper detection |

---

## Open Items

- `compute_submit` / `compute_gate` (ADR-0029 engine tools) are not yet wired in
  `mcp_server.py`. When they are, the gate must be applied at those entry points
  as well (ticket to open when ADR-0029 Phase 2 lands).
- A signed trial-JWT flow (email → auto-generated trial JWT, no credit card) is the
  next conversion milestone; requires a signing endpoint in `atelier-cloud`.
- The BSL `LICENSE` file header must be added to every `.py` file in
  `core/compute/` in a follow-up commit (mechanical, low-risk).
