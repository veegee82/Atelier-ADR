# ADR-0067 — HermesEngine Production Parity: Equal-Standing Local Engine

**Status:** Accepted — 2026-05-29 · M2.1–M2.5 Implemented 2026-05-29
**Date:** 2026-05-29
**Authors:** Claude Code (maintainer session)
**Type:** Architecture — Compliance hardening + UX parity + Observability
**Supersedes:** Nothing (extends [ADR-0066](0066-L22-hermes-engine.md))
**Depends on:** ADR-0066 (M1 done) · L34 · L35 · ADR-0020 L30.1b · ADR-0017 console

---

## Context

ADR-0066 M1 delivered `HermesEngine` as the fourth `WorkerEngine`:

- `hermes_engine.py` — Ollama HTTP streaming engine
- `delegate_hermes` — L29 delegation MCP tool
- `_call_hermes_streaming_via_engine` — adapter direct-dispatch path
- `hermes-worker` persona — `default_engine: "hermes"`

**22/22 E2E checks pass.** Hermes works.

However, a structured production-readiness audit (executed 2026-05-29 against the
live codebase) identified **eight gap categories** between the Hermes OS-turn path
and what Claude Code provides. Until these gaps are closed, Hermes cannot be
treated as equal to Claude Code for production traffic, GDPR/EU AI Act compliance,
or operator visibility.

Additionally, the user experience is asymmetric: Claude Code, Codex, and OpenCode
can all be switched per-chat via `/engine <name>` and via the operator console.
Hermes is only selectable by assigning a persona — it has no first-class UI path.

### Gap inventory (from audit, 2026-05-29)

| # | Gap | Severity | Regulatory impact |
|---|---|---|---|
| G1 | L34 data-classification gate absent at OS-turn | **Critical** | EU AI Act Art. 14, GDPR Art. 32 |
| G2 | L35 network-egress gate absent at OS-turn | **Critical** | EU AI Act Art. 14, GDPR Art. 32 |
| G3 | L30.1b engine-trust gate absent at OS-turn | **Critical** | EU AI Act Art. 14 |
| G4 | No per-turn audit events (`hermes.turn_start/end`) | **High** | GDPR Art. 30 — operator must be able to reconstruct what happened |
| G5 | `/engine hermes` not wired in `engine_switch.py` | **High** | UX parity — Hermes not user-selectable like other engines |
| G6 | No console engine selector for Claude Code ↔ Hermes | **High** | Operator UX — can't set default engine without touching JSON files |
| G7 | No `hermes.*` types in `security_events.py` | **Medium** | Audit integrity — unregistered events bypass voice-audit verify |
| G8 | No Prometheus metrics for Hermes OS-turns | **Medium** | Observability — operator can't alert on Hermes error rate |

### Why G1–G3 are "Critical"

The L34/L35/L30.1b gates are load-bearing GDPR and EU AI Act compliance controls
(CLAUDE.md §Compliance baseline). For Claude Code, they run unconditionally before
`engine.spawn()`. For Hermes, they are **skipped entirely** at the OS-turn level.

Concretely: a tenant with `spec.data_classification.matrix: {SECRET: local}` expects
Hermes (locality=local) to be allowed for SECRET tasks. But the gate also enforces the
*process* — it writes the compliance audit event. Without the gate, there is no record
that the classification check ran. Under GDPR Art. 30, that record is required.

The CLAUDE.md absolute constraint: "Don't weaken audit-chain integrity — no event
skips the hash-chain link."

---

## Decision

Close all eight gaps in four milestones, promoting Hermes to **equal standing** with
Claude Code for OS-turn dispatch.

### Variant analysis

**Variant A — Shared gate helper (chosen)**

Extract the three gate calls (`_check_engine_trust_or_fail`, `_check_compliance_or_fail`,
`_check_egress_or_fail`) into a single `_run_pre_dispatch_gates(engine_id, ...)` helper
that all non-Claude-Code OS-turn paths call. Both `_call_hermes_streaming_via_engine`
and `_call_opencode_streaming_via_engine` call this helper before `engine.spawn()`.

Benefit: single call-site audit, easy to extend for future engines. Shared
code change also closes the same gaps for OpenCode (which currently has identical
compliance gaps — this ADR is the right moment to fix both).

**Variant B — Copy gates into each function**

Duplicate the gate logic into each engine-specific function.

Rejected: violates DRY, creates drift risk, more code to audit.

**Variant C — Integrate gates into WorkerEngine.spawn()**

Move gate checks inside the engine protocol so any `spawn()` call is implicitly gated.

Rejected: The gates need adapter-level context (channel, chat_key, profile, persona)
that the WorkerEngine Protocol intentionally does not carry (engines are infrastructure,
not policy). Mixing policy into the engine breaks the layer separation.

---

## Milestones

### M2.1 — Compliance gates at OS-turn (G1–G3) · **Load-bearing**

**Files:**
- `operator/bridges/shared/adapter.py`

**Change:** Add `_run_pre_dispatch_gates(engine_id, channel, chat_key, profile)` helper
that calls — in order — the same three gates that the Claude path already calls:

```python
def _run_pre_dispatch_gates(
    engine_id: str,
    channel: str,
    chat_key: str,
    profile: dict | None,
) -> str | None:
    """Run L30.1b / L34 / L35 pre-dispatch gates for non-ClaudeCode engines.

    Returns an error string if any gate denies the dispatch; None to allow.
    Mirrors what _call_claude_streaming_via_engine does at lines 2810-2848.
    Emits audit events on deny; never on allow (gate semantics: silence = pass).
    """
    # L30.1b — engine-trust gate
    trust_result = _check_engine_trust_or_fail(engine_id, channel, chat_key, profile)
    if trust_result is not None:
        return trust_result

    # L34 — data-classification + flow-guard gate
    compliance_result = _check_compliance_or_fail(engine_id, channel, chat_key, profile)
    if compliance_result is not None:
        return compliance_result

    # L35 — network-egress gate
    # Note: HermesEngine connects only to localhost:11434 (loopback). The L35
    # gate's loopback-unconditional-permit rule means this gate is a near-no-op
    # for Hermes in normal operation. It still runs to produce the compliance
    # audit record (GDPR Art. 30 process-record requirement).
    egress_result = _check_egress_or_fail(engine_id, channel, chat_key, profile)
    if egress_result is not None:
        return egress_result

    return None
```

Called in both `_call_hermes_streaming_via_engine` and
`_call_opencode_streaming_via_engine` immediately after argument validation,
before `engine.spawn()`.

**Must NOT do (M2.1):**
- Fail-open if a gate function raises — treat exceptions as deny (same semantics as
  the Claude Code path, which also fails-closed on gate errors).
- Skip the L35 gate for Hermes because "it's loopback." The compliance record is the
  requirement, not the network decision.
- Add a "hermes_exempt" flag to bypass gates — no bypass mode, full stop.

---

### M2.2 — Per-turn audit events (G4, G7) · **High**

**Files:**
- `operator/forge/forge/security_events.py`
- `operator/bridges/shared/adapter.py`

**New event types in `EVENT_SEVERITY`:**

| Event | Severity | Trigger |
|---|---|---|
| `hermes.turn_start` | INFO | HermesEngine spawn begins |
| `hermes.turn_end` | INFO | HermesEngine turn completed successfully |
| `hermes.turn_error` | WARNING | HermesEngine turn ended with error |
| `hermes.ollama_unavailable` | WARNING | Ollama unreachable at spawn time |
| `hermes.stream_timeout` | WARNING | Idle watchdog fired |

Audit fields (allow-list — NEVER prompt/output text):
`engine_id`, `model`, `persona`, `channel`, `chat_key`, `duration_ms`,
`token_count`, `error_class` (for error events), `base_url_hash` (16-hex prefix of base URL).

Emitted from `_call_hermes_streaming_via_engine` wrapping the event loop:

```python
_audit_event("hermes.turn_start",
    channel=channel, chat_key=chat_key,
    details={"engine_id": "hermes", "model": engine.model,
             "persona": profile.get("name", "")})
# ... event loop ...
if error_text:
    _audit_event("hermes.turn_error", ...)
else:
    _audit_event("hermes.turn_end", ..., details={"duration_ms": ..., "token_count": ...})
```

OpenCode gets the same treatment (same five event types, `opencode.*` prefix).

**Must NOT do (M2.2):**
- Put prompt text, output text, or model names in audit `details`.
- Emit `base_url` directly — hash-prefix only (16 hex chars of SHA-256).
- Register `hermes.*` events without adding them to `voice-audit verify`'s known-event
  list (otherwise verify exits non-zero on old chains that don't have them).

---

### M2.3 — `/engine hermes` in-chat switcher (G5) · **High**

**Files:**
- `operator/bridges/shared/engine_switch.py`

**Changes:**

```python
# engine_switch.py, line ~114
VALID_ENGINES: tuple[str, ...] = ("claude_code", "codex_cli", "opencode", "hermes")

# ENGINE_ALIASES, add:
"hermes":       "hermes",
"hermes-fast":  "hermes",   # maps alias → canonical id; model is separate
"local":        "hermes",   # convenience alias: "use local engine" = hermes
```

**Behaviour spec:**
- `/engine hermes` → switches the current chat's `default_engine` to `"hermes"`.
- `/engine hermes hermes-fast` → switches engine AND sets `model=hermes-fast`
  (stored in session profile, not the persona file).
- `/engine claude` (existing) → reverts to `claude_code`.
- `/engine` (no arg) → shows current engine + available choices including Hermes.
- Capability-degradation warning injected into the reply when switching to Hermes:
  "Note: /btw, forge tools, and skill injection are not available with the Hermes
  engine. Use `/engine claude` to re-enable them."

**Audit:** `/engine` switches emit `bridge.engine_switched` (existing event type).

**Must NOT do (M2.3):**
- Allow `/engine hermes` when `allowed_engines` policy excludes `hermes`.
- Accept arbitrary model tags from the user without validating against
  `HERMES_MODEL_ALIASES` or a length/charset check (injection surface).
- Make `/engine local` an alias without documenting that `local` = `hermes` in
  the help text.

---

### M2.4 — Console engine selector (G6) · **High**

**Scope:** The operator console (`core/console/`) Settings page gains a
"Default Engine" selector for each tenant, with live preview of consequences.

**UI design:**

```
Engine
┌────────────────────────────────────────────────────────┐
│  ● Claude Code (default)   — Full feature set           │
│    /btw · skills · forge · hooks · session pinning      │
│                                                         │
│  ○ Hermes (fully local)    — Zero egress                │
│    Model: [hermes-balanced ▼]                           │
│    No /btw · No forge MCP · No skills injection         │
│    ✓ CONFIDENTIAL-capable · ✓ No cloud API key needed   │
└────────────────────────────────────────────────────────┘
  [ Save ]
```

The selector writes to `tenant.atelier.yaml::spec.default_engine` (new field,
resolves via `current_tenant().spec.default_engine` in the adapter's
`call_claude_streaming` dispatch).

**Resolution order (spec):**

```
1. per-chat profile.default_engine  (chat_profiles.*.persona / /engine command)
2. tenant spec.default_engine       (console setting, new in M2.4)
3. ClaudeCodeEngine                 (hardcoded fallback)
```

Hermes model at tenant level: `spec.hermes_model` (optional, falls back to
`ATELIER_HERMES_MODEL` env var → `"nous-hermes-2"` built-in default).

**Console REST endpoints (new):**

```
GET  /v1/console/settings/engine          → {default_engine, hermes_model, ...}
PUT  /v1/console/settings/engine          → body: {default_engine, hermes_model}
GET  /v1/console/settings/engine/health   → {ollama_reachable, model_count, ...}
```

**Console React component:** `EngineSelector` in `core/console/web/src/components/`.
Hard-coded option set (no free-text engine field on UI — operator uses YAML for custom
engines). Reads engine health from the health endpoint before rendering.

**Must NOT do (M2.4):**
- Allow the console to set `default_engine` to a value not in `VALID_ENGINES`.
- Expose the raw Ollama base URL in console responses (hash-prefix only for logs).
- Auto-switch all existing chats when the tenant default changes — only new chats
  use the new default; active chats keep their profile.
- Gate the console setting behind the enterprise license — engine selection is
  Apache-core.

---

### M2.5 — Prometheus metrics (G8) · **Medium**

**Files:**
- `core/gateway/atelier_gateway/audit_metrics.py`

**New counter families:**

```python
# hermes_turns_total{outcome="success|error|timeout", model="...", persona="..."}
HERMES_TURNS        = Counter("atelier_bridge_hermes_turns_total", ...)
# hermes_turn_duration_seconds{outcome="success|error"} histogram
HERMES_DURATION     = Histogram("atelier_bridge_hermes_turn_duration_seconds", ...)
# hermes_ollama_unavailable_total — when spawn fails due to Ollama down
HERMES_UNAVAILABLE  = Counter("atelier_bridge_hermes_ollama_unavailable_total", ...)
```

Labels: `outcome` (success/error/timeout), `model` (resolved model name),
`persona` (profile name). **Never** channel or chat_key in labels (PII risk).

Metric emission happens in `_call_hermes_streaming_via_engine`, parallel to the
audit event emission from M2.2. The `audit_metrics.py` module uses lazy-loaded
Prometheus client; missing dependency → metrics silently disabled (best-effort,
same as other metric sites).

OpenCode gets the same treatment (`atelier_bridge_opencode_turns_total` etc.).

**Must NOT do (M2.5):**
- Add chat_key, channel, or user identifiers as Prometheus labels.
- Make metrics emission blocking (it must be fire-and-forget).
- Require prometheus_client to be installed for the adapter to boot.

---

## Consequences

### Positive

- **Regulatory:** Hermes OS-turns now produce the same compliance audit record as
  Claude Code turns. GDPR Art. 30 satisfied for Hermes-routed chats.
- **UX parity:** `/engine hermes` and the console selector make Hermes a
  first-class choice alongside Claude Code. Operators no longer need to edit JSON
  persona files to enable local inference.
- **Operator visibility:** `hermes.*` audit events + Prometheus metrics give
  oncall the same signal quality for Hermes as for other engines.
- **Security:** L34/L35 gates close a real bypass — a sufficiently motivated
  attacker who could control `profile.default_engine` could currently route
  traffic around the data-classification gate. M2.1 closes this.

### Negative / Risks

- **M2.1 latency:** Three gate calls add ~1 ms per Hermes turn (gate functions
  are in-process; no subprocess). Acceptable.
- **M2.4 complexity:** Console engine selector is a non-trivial React component
  with health-check integration. Risk of desync if Ollama goes up/down while
  the console is open. Mitigated by health endpoint and "save with warning" UX.
- **M2.3 `/engine local` alias:** The convenience alias `local → hermes` is
  intuitive but could conflict with future local-first engines. The alias table
  is operator-visible (not end-user-configurable); risk is low.
- **OpenCode parity:** M2.1 and M2.2 also fix OpenCode's identical compliance gaps.
  This is correct behaviour but widens the scope of the change. Scoped under this ADR
  to avoid a separate ADR for what is effectively the same fix.

---

## Must NOT do (cross-cutting)

- Never set `default_engine: "hermes"` as the global system default — Claude Code
  remains the default. Hermes is opt-in (per-chat, per-tenant, or per-persona).
- Never make gate failure for `hermes.ollama_unavailable` CRITICAL — Ollama is
  optional; adapter must boot without it (ADR-0066 invariant preserved).
- Never put prompt text, output text, model-response content, or full URLs in any
  audit `details` field added in M2.2.
- Never let M2.4 console save propagate silently if Ollama is unreachable — show a
  warning: "Hermes/Ollama is not reachable at {url}. Engine saved but turns will
  fail until Ollama is running."
- Never allow M2.3 `/engine` command to bypass the engine-policy gate that
  `call_claude_streaming` runs at line ~3721.
- Never merge M2.x milestones without the corresponding E2E test update to
  `test_hermes_e2e_full.py` (per-subtask E2E is mandatory for security-touching
  changes — CLAUDE.md §Working method).

---

## Milestone status

| Milestone | Status | Notes |
|---|---|---|
| ADR-0066 M1 — engine + delegation + dispatch | ✅ Done 2026-05-29 | 22/22 E2E checks |
| M2.1 — compliance gates | ✅ Done 2026-05-29 | `_run_pre_dispatch_gates()` + `agents/trust/hermes.yaml` + hermes in `data_classification.py` |
| M2.2 — per-turn audit events | ✅ Done 2026-05-29 | 10 new event types; hermes.turn_start/end/error + opencode parity |
| M2.3 — `/engine hermes` switcher | ✅ Done 2026-05-29 | ENGINE_ALIASES + VALID_ENGINES + supported_aliases() |
| M2.4 — console engine selector | ✅ Done 2026-05-29 | `routes/engine.py` — GET/PUT /settings/engine + health endpoint |
| M2.5 — Prometheus metrics | ✅ Done 2026-05-29 | `engine_metrics.py` — lazy prometheus_client, record_hermes/opencode_turn() |

---

## References

- ADR-0066: `decisions/0066-L22-hermes-engine.md`
- ADR-0020 L30.1b engine-trust: `AtelierOS/docs/claude-ref/layer-engines.md`
- ADR-0042 L34 data-classification: `AtelierOS/docs/claude-ref/layer-34-data-classification.md`
- ADR-0043 L35 egress lockdown: `AtelierOS/docs/claude-ref/layer-35-egress-lockdown.md`
- ADR-0017 console: `decisions/0017-enterprise-control-plane.md`
- Audit: `AtelierOS/operator/bridges/shared/adapter.py` lines 2810–2848 (Claude gates)
- Audit: `AtelierOS/operator/bridges/shared/engine_switch.py` (VALID_ENGINES)
