# ADR-0049 — Layer 22: Session-Pinned Workers

**Status:** Accepted — Implemented (M1–M4 complete 2026-05-23)
**Date:** 2026-05-23
**Companion to:** [0002 L22 Engine Migration](0002-phase2-adapter-engine-migration.md),
[0020 Engine Trust Hardening](0020-engine-trust-hardening.md),
[0048 L38 A2A](0048-L38-remote-trigger-receiver.md)

---

## Context

Every `ClaudeCodeEngine` spawn today is stateless: the subprocess starts with
no conversation history, processes one turn, and exits. The `continue_session`
flag (→ `--continue`) is wired for the *adapter's own* conversation thread, but
delegation spawns in Layer 29 never set it — each orchestrator sub-task begins
from a blank slate.

This creates a structural gap for multi-turn orchestration patterns:

1. **Iterative refinement** — an orchestrator delegates a code-review task to
   a specialist worker across three turns. Without session continuity, the
   worker re-reads every file from scratch each time and cannot build a
   running model of what has already been reviewed.
2. **Long-running compute loops** — L25 compute strategies dispatch multiple
   parameter-sweep iterations. A pinned worker can accumulate intermediate
   results in its context rather than re-establishing domain knowledge every
   call.
3. **A2A peer-chains** (L38) — a remote-triggered worker asked to do a
   two-phase task (explore → patch) today requires the sender to pack full
   context into each envelope. A resident worker session eliminates that
   overhead.

The `claude` CLI exposes two session-continuation flags:

| Flag | Semantics |
|---|---|
| `--continue` | Resume the most-recently-used session for the current working directory |
| `--resume <session-id>` | Resume a specific session by ID |

The existing `continue_session=True` path uses `--continue` and is correct for
the adapter's own main conversation. Delegation spawns need `--resume <id>`
because there may be multiple concurrent workers per chat and "most recent"
is ambiguous.

**What is NOT changing:** The adapter's own main conversation thread (the
human-facing Claude Code session) continues to use `--continue` as today.
This ADR concerns only worker subprocesses spawned via L29 delegation.

---

## Decision

Introduce **Session-Pinned Workers** as an opt-in capability of `ClaudeCodeEngine`
in Layer 22, consumed by the Layer 29 delegation path.

### 1. Scope key

Each pinned worker session is identified by a `(tenant_id, bridge, chat_key,
scope_label)` tuple. `scope_label` is a short alphanumeric string set by the
caller (e.g. the persona name or a delegation purpose tag such as
`"code_review"`, `"compute_sweep_1"`). This allows a single chat to host
multiple concurrent pinned workers with isolated contexts.

### 2. Session file

The session ID received in the `session_started` event is persisted to:

```
<atelier_home>/tenants/<tid>/sessions/<bridge>:<chat>/worker_sessions/<scope_label>.session.json
```

File mode: **0600**. Contents (JSON, one atomic write with `flock`):

```json
{
  "session_id": "<claude-session-id>",
  "scope_label": "<label>",
  "persona": "<persona-name>",
  "created_at": "<iso8601>",
  "last_resumed_at": "<iso8601>",
  "resume_count": 3
}
```

`session_id` is the raw string from `{"type":"system","subtype":"init","session_id":"..."}`.
It is treated as an opaque token and MUST NOT appear in any audit `details` field.

### 3. `ClaudeCodeEngine._build_args` extension

A new optional parameter `resume_session_id: str | None = None` is added.
When set, `--resume <id>` is inserted in the same argv slot as `--continue`:

```
claude --resume <session_id> -p ...
```

`resume_session_id` and `continue_session` are mutually exclusive.
`_build_args` raises `ValueError` if both are truthy (programmer error, not
runtime condition — caught at test time via golden snapshots).

### 4. Delegation callsite (L29)

`a2a_worker.py` (and the main delegation path in `delegate_claude_code`)
accepts a new `pin_session: bool = False` spawn parameter. When `True`:

1. Load `<scope_label>.session.json` if it exists → pass `resume_session_id`
   to engine.
2. If no file exists → spawn fresh (no `--resume`), capture `session_id` from
   the first `session_started` event, write the file atomically.
3. If the spawn fails with a session-not-found error (claude exits non-zero
   with the specific error substring `"session not found"`) → delete the
   stale file, spawn fresh, write new session file. This handles the case
   where the Claude Code backend has expired the session.

`pin_session` is `False` by default — existing delegation call sites are
unaffected.

### 5. Persona-level opt-in

A persona may declare `worker_session_pinned: true` in its YAML. The adapter
reads this flag when constructing the delegation kwargs, setting
`pin_session=True` automatically. Without the flag, pinned sessions require
an explicit caller opt-in.

### 6. Lifecycle (L8 integration)

`session_reset.py` is extended to prune the `worker_sessions/` directory as
part of the session-reset sweep. The audit-first invariant applies:

1. `session.reset` event written to the L16 hash chain (as today).
2. Worker session files purged (`worker_sessions/*.session.json`).
3. Worker session workspace dirs pruned (existing L8 logic).

Order of operations within step 2: individual `worker_session.purged` events
are emitted per file before each deletion (best-effort; a write failure MUST
NOT block the rmtree).

Daily TTL sweep (`atelier-session-timeout.timer`) also prunes
`worker_sessions/` for sessions older than `CLAUDEOS_SESSION_TTL_DAYS`
(default: 7 days), using `last_resumed_at` as the age reference.

### 7. Audit events

Two new events are added to the L16 hash chain:

| Event | Severity | Allowed fields |
|---|---|---|
| `worker_session.created` | INFO | `scope_label`, `persona`, `channel`, `chat_key`, `tenant_id` |
| `worker_session.resumed` | INFO | `scope_label`, `persona`, `channel`, `chat_key`, `tenant_id`, `resume_count` |
| `worker_session.stale_evicted` | WARNING | `scope_label`, `persona`, `channel`, `chat_key`, `tenant_id` |
| `worker_session.purged` | INFO | `scope_label`, `channel`, `chat_key`, `tenant_id` |

`session_id` MUST NOT appear in any audit field. `resume_count` is an integer
and carries no PII.

### 8. Engine-capability gate

`ClaudeCodeEngine.capabilities` gains a boolean key `session_pinning: True`.
Codex CLI and OpenCode engines expose `session_pinning: False`. Any caller
requesting `pin_session=True` on a non-supporting engine raises `CapabilityError`
at spawn time (not silently ignored).

### 9. Context growth + compaction

Claude Code compacts its internal context automatically when the session
approaches the model's context limit. This ADR inherits that behaviour without
any additional machinery. The operator may force a session reset (which purges
the worker session file) to start fresh if compaction is undesirable for a
specific use-case.

---

## Consequences

### Positive

- Orchestrator sub-tasks gain cumulative context at zero protocol overhead —
  no need to re-inject domain knowledge or prior-turn summaries into each
  delegation envelope.
- L38 A2A senders can drive multi-phase workflows without re-packing full
  context into every `TaskEnvelope`.
- L25 compute workers can maintain a running parameter-sweep model across
  iterations.
- Lifecycle integration is clean: L8 reset already owns the session directory;
  the new `worker_sessions/` subdirectory slots into the existing sweep.

### Risks and mitigations

| Risk | Mitigation |
|---|---|
| Stale session after claude backend expires the ID | Automatic eviction-and-respawn on `"session not found"` error (§4, step 3) |
| Context accumulation leaks data across unrelated tasks | `scope_label` isolates sessions; `/reset` purges all; TTL sweep prunes stale files |
| session_id in logs enables partial context reconstruction | File mode 0600; `session_id` explicitly excluded from all audit fields; log sanitiser covers the regex `[a-f0-9]{32,}` |
| Concurrent writes to the session file | `flock` + atomic rename pattern (same as other per-chat JSON files) |
| Pin used across tenant boundary | Session file path includes `tenant_id`; resolver enforces per-tenant isolation |

---

## Must NOT do

- Don't set `session_id` in any audit `details` field (opaque token, not PII,
  but still allows partial session reconstruction).
- Don't share a pinned session across chat keys or tenants.
- Don't make `pin_session=True` the default for any delegation path without
  an explicit persona-level opt-in.
- Don't allow the stale-eviction path to retry more than once (double-eviction
  loop indicates a deeper engine problem — surface as `CapabilityError`).
- Don't add `session_pinning` capability to Codex CLI or OpenCode engines
  without a separate ADR verifying their `--resume` semantics match Claude Code's.
- Don't prune `worker_sessions/` from the daily TTL sweep before the L8
  `session.reset` audit event has been written for any in-progress reset.
- Don't add an env-flag to bypass the `CapabilityError` on unsupported engines.

---

## Milestones

| Milestone | Scope | Status |
|---|---|---|
| **M1** | `_build_args` extension (`resume_session_id`), golden snapshot tests, `CapabilityError` for non-supporting engines | Done |
| **M2** | Session file write/read/evict in `a2a_worker.py` + main delegation path; audit events; L8 `session_reset.py` integration | Done |
| **M3** | Persona YAML `worker_session_pinned` flag; adapter reads flag and sets `pin_session`; TTL sweep integration | Done |
| **M4** | E2E integration test: two-turn orchestrator delegation with session pin, verify `resume_count=2` in session file and `worker_session.resumed` in audit chain | Done |
