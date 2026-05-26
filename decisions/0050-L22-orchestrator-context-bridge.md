# ADR-0050 — Layer 22: Orchestrator Context Bridge

**Status:** Accepted — M1 planned
**Date:** 2026-05-23
**Companion to:** [0002 L22 Engine Migration](0002-phase2-adapter-engine-migration.md),
[0049 L22 Session-Pinned Workers](0049-L22-session-pinned-workers.md),
[0048 L38 A2A](0048-L38-remote-trigger-receiver.md),
[0016 Conversation Recall and User Modeling](0016-conversation-recall-and-user-modeling.md),
[0051 L29 Worker Memory Bridge](0051-L29-worker-memory-bridge.md)

---

## Context

ADR-0049 introduced session-pinned workers for delegation sub-tasks (L29): each
worker subprocess can resume a specific session by ID across turns, accumulating
context without re-injecting domain knowledge into every call.

The **main conversation thread** — the human-facing `ClaudeCodeEngine` session
per Discord/Telegram/WhatsApp chat — was deliberately left out of ADR-0049.
That omission is now a structural gap.

### Phase 26 wipe (commit 08ad5b5, 2026-05-21)

The adapter uses `continue_session=True` (→ `--continue`) to resume the most
recent session in the working directory. A "Phase 26" workaround was added to
intentionally wipe `.session_started` after every Discord/Telegram message.
The reason: `--continue` is keyed only to the working directory, not to a
specific session ID. Autonomous-loop sessions (spawned by orchestrator delegates
in the same working directory) were being picked up by the main thread, causing
the bot to respond to its own internal monologue.

The result: every main-thread turn starts from a blank Claude Code session.
L28 conversation-recall injects a summary, but:

- The summary is a lossy prose compression, not full conversational context.
- Each turn pays the full cold-start cost (tool registration, system-prompt
  injection, CLAUDE.md re-read).
- The session history that Claude Code's own compaction mechanism could maintain
  across turns is never built up.

### Engine memory capability matrix

| Engine | Native persistent memory | Session resume flag | Notes |
|---|---|---|---|
| `ClaudeCodeEngine` | Yes — `~/.claude/projects/<workdir>/` | `--resume <id>` | L28 recall + OS memory also injected automatically |
| `CodexCliEngine` | No | — | Context window only |
| `OpenCodeEngine` | No | — | Provider-agnostic; context window only |
| `GeminiCLI` (planned) | No | — | 1M-token window; no persistent memory |

For non-Claude-Code engines, there is currently no mechanism to transfer
relevant OS-layer memory (L28 recall, per-user model, cowork project context)
into the worker's initial context before spawn. Each worker receives only the
literal instruction string from the caller.

### Gap summary

Two distinct problems share a single root: the OS layer does not persistently
bridge context between turns for either the main thread or non-Claude-Code
workers.

1. **Main thread** — stateless despite `ClaudeCodeEngine` supporting `--resume`.
   The `--continue` mechanism cannot be used safely because it is not session-specific.
2. **Non-Claude-Code workers** — receive no OS-layer memory at spawn time, even
   when L28 has rich per-user context available.

---

## Decision

### §1 — Main-thread session pinning

Replace the `--continue` mechanism for the main conversation thread with
`--resume <specific_session_id>`.

**1.1 Session file**

After each successful main-thread turn, capture the `session_id` from the
`session_started` stream event (`ev.raw["session_id"]`) and write it atomically
to:

```
<workdir>/.main_session.json
```

File mode: **0600**. Contents (JSON, atomic write with `flock`):

```json
{
  "session_id": "<claude-session-id>",
  "chat_key": "<bridge>:<chat>",
  "tenant_id": "<tid>",
  "created_at": "<iso8601>",
  "last_resumed_at": "<iso8601>",
  "resume_count": 0
}
```

`chat_key` and `tenant_id` are stored as guards: if the file is read back for a
different chat key (e.g. after a workdir reuse across tenants), the callsite
MUST treat it as absent and spawn fresh.

**1.2 Spawn-time resume**

At the start of each main-thread turn, read `.main_session.json` if present.
If the `chat_key` matches and the file is valid, pass
`resume_session_id=<id>` to `ClaudeCodeEngine.spawn()` (the `--resume` flag).
Update `last_resumed_at` and `resume_count` atomically after a successful spawn.

**1.3 Phase 26 removal**

Remove the `.session_started` wipe that was introduced as the Phase 26
workaround. The `--resume <specific_id>` flag is precise: it targets the exact
session written by the main thread's last successful turn and cannot accidentally
pick up an autonomous-loop session.

**1.4 Eviction and reset**

`_reset_session_state()` is extended to delete `.main_session.json` alongside
`.session_started` and `.claude.json`. This ensures a clean fresh-start on:

- Explicit user `/reset` / `/clear` / `/new` commands (L8).
- Session corruption detected during `session_started` processing (existing
  error-handling path).
- Session-not-found errors returned by the `claude` subprocess (the engine
  checks for the specific error substring `"session not found"`, deletes the
  file, and respawns fresh — same eviction pattern as ADR-0049 §4).

The L8 audit-first invariant applies: `session.reset` is written to the L16
hash chain BEFORE any file deletion.

**1.5 Audit events**

| Event | Severity | Allowed fields |
|---|---|---|
| `main_session.created` | INFO | `channel`, `chat_key`, `tenant_id` |
| `main_session.resumed` | INFO | `channel`, `chat_key`, `tenant_id`, `resume_count` |
| `main_session.evicted` | WARNING | `channel`, `chat_key`, `tenant_id`, `reason` |

`session_id` MUST NOT appear in any audit field.

---

### §2 — Memory bridge for non-Claude-Code engines

Introduce a `memory_bridge.py` module under `operator/bridges/shared/` that
provides OS-layer memory injection for engine types without native persistent
memory (Codex CLI, OpenCode, Gemini CLI, future engines).

**2.1 Interface**

```python
def build_context_block(
    scope: str,          # tenant:channel:chat_key
    workdir: Path,
    profile: dict,       # per-user profile (L12/L28 fields)
    max_chars: int = 2000,
) -> str:
    ...
```

Returns a UTF-8 string of at most `max_chars` characters, formatted as:

```
<os_context>
...prose summary of relevant memory...
</os_context>
```

Returns an empty string if no relevant memory is found or if the combined
source material yields nothing after compression.

**2.2 Sources**

The block is assembled from:

1. **L28 recall.db** — most recent N turns (FTS5 query, recency-weighted,
   PII-redacted before read).
2. **Per-user model** (`<global>/memory/user_model/<chan>__<chat>.json`) —
   injected verbatim if present and within budget.
3. **OS memory files** (`~/.claude/projects/<path>/memory/MEMORY.md` and
   topic files) — topic files whose filename matches the current task context
   (heuristic: keyword overlap with the instruction string).

**2.3 Haiku-4.5 selector**

Because the raw source material may exceed `max_chars`, `memory_bridge.py`
invokes Haiku-4.5 via `subprocess` (`claude -p --max-turns 1 --no-tools`) to
select and compress the most relevant entries before assembly. The selector
prompt is a self-contained instruction that receives the candidate material and
returns a compressed prose paragraph. If the subprocess fails or times out (10 s
ceiling), `build_context_block` falls back to a simple recency-truncation of the
raw material — no exception is raised to the caller.

**2.4 Injection callsite**

Before spawning a non-Claude-Code worker in the L29 delegation path
(`delegate_codex`, `delegate_opencode`, and the planned `delegate_gemini`),
the adapter calls `memory_bridge.build_context_block()` and prepends the result
to the self-contained delegate prompt. The `<os_context>` block appears before
the `<delegated_skill>` block (if present) and before the
`<a2a_instruction>` framing block (L38). An empty return value is a no-op.

**2.5 Constraints**

- `memory_bridge.py` MUST NOT `import anthropic`. Haiku-4.5 is invoked only
  via `subprocess` using the `claude` CLI.
- The `max_chars` cap is enforced after subprocess compression: if the returned
  block exceeds the cap, it is truncated at the last sentence boundary before
  `max_chars`. Hard truncation at `max_chars` is the final fallback.
- No memory contents are injected into the system prompt automatically — the
  block is part of the per-turn user-turn message only.
- `build_context_block` is always called asynchronously (background thread or
  async task) before spawn; it MUST NOT block the main event loop for more than
  the 10 s subprocess ceiling.

---

## Consequences

### Positive

- The main conversation thread accumulates context across turns within a single
  Claude Code session, reducing cold-start cost and improving response coherence.
- Removing the Phase 26 workaround eliminates a fragile state-wipe that would
  break whenever autonomous-loop and main-thread working directories happened to
  differ.
- Non-Claude-Code workers gain a best-effort memory bridge without requiring
  protocol changes to their spawn interface.
- Session eviction is deterministic: a single `_reset_session_state()` call
  cleans all session state consistently.

### Risks and mitigations

| Risk | Mitigation |
|---|---|
| Main-thread `--resume` picks up a stale/expired session | Eviction-and-respawn on `"session not found"` (§1.4); `main_session.evicted` audit event |
| `.main_session.json` read for wrong chat key after workdir reuse | `chat_key` guard in file; mismatch treated as absent |
| Haiku selector subprocess adds latency before every non-CC worker spawn | 10 s ceiling + fallback to recency-truncation; called asynchronously |
| Haiku selector output exceeds `max_chars` | Post-subprocess cap enforced before injection |
| Memory bridge injects stale user model from a prior session | Per-user model is timestamped; `build_context_block` skips files older than `CLAUDEOS_SESSION_TTL_DAYS` |
| Concurrent writes to `.main_session.json` from parallel turns | `flock` + atomic rename pattern (same as ADR-0049 session files) |
| `session_id` leaks into logs via error messages | Log sanitiser covers the regex `[a-f0-9-]{32,}`; `session_id` explicitly excluded from all audit fields |

---

## Must NOT do

- Don't put `session_id` in any audit `details` field — it is an opaque token
  that enables partial session reconstruction.
- Don't share `.main_session.json` across chat keys or tenants; the `chat_key`
  and `tenant_id` guards in the file are structural, not advisory.
- Don't use `--continue` for the main conversation thread after this ADR is
  implemented — `--resume <specific_id>` is the only admissible resume mechanism
  for the main thread.
- Don't call `import anthropic` from `memory_bridge.py` — Haiku-4.5 is invoked
  via subprocess only (same constraint as all `shared/` modules).
- Don't auto-inject memory contents into the system prompt — injection is
  per-turn, user-message only.
- Don't skip the `max_chars` guard — auto-injection without a size cap would
  silently consume the model's context budget on high-recall chats.
- Don't apply the memory bridge to `ClaudeCodeEngine` workers — they already
  have native persistent memory and L28 recall injection via the OS memory path.
- Don't retry the Haiku selector subprocess on failure — fall back to
  recency-truncation immediately (a retry loop would double the latency risk).

---

## Milestones

| Milestone | Scope | Status |
|---|---|---|
| **M1** | `.main_session.json` write/read/evict in `adapter.py`; Phase 26 `.session_started` wipe removal; `_reset_session_state()` cleanup integration; `session_id` capture in `ClaudeCodeEngine` streaming loop; audit events `main_session.*` | Planned |
| **M2** | `memory_bridge.py` module: `build_context_block()`, Haiku-4.5 selector subprocess, fallback truncation, `<os_context>` injection in `delegate_codex` / `delegate_opencode` L29 callsites | Planned |
| **M3** | E2E integration test: two Discord turns, verify second turn passes `--resume <id>` to `ClaudeCodeEngine`, verify `session_id` in `.main_session.json` matches the `session_started` event of the first turn, verify `resume_count=1` and `main_session.resumed` in audit chain | Planned |
