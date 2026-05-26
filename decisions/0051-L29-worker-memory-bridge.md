# ADR-0051 — Layer 29: Worker Memory Bridge

**Status:** Accepted
**Date:** 2026-05-23
**Companion to:** [0050 L22 Orchestrator Context Bridge](0050-L22-orchestrator-context-bridge.md),
[0049 L22 Session-Pinned Workers](0049-L22-session-pinned-workers.md),
[0016 Conversation Recall and User Modeling](0016-conversation-recall-and-user-modeling.md),
[0002 L22 Engine Migration](0002-phase2-adapter-engine-migration.md)

---

## Context

ADR-0050 introduced two mechanisms: main-thread session pinning (§1) and a basic
memory bridge for non-Claude-Code engines (§2). That ADR was scoped to the
immediate Phase-26 wipe regression.

The broader problem — **workers of all engine types receive no OS-layer memory at
spawn time and contribute nothing back after their turn** — was identified in full
during design review but not yet fully formalised. This ADR captures the complete
four-mechanism architecture for worker engine memory.

### Engine memory capability matrix

| Engine | Native persistent memory | Session resume | Notes |
|---|---|---|---|
| `ClaudeCodeEngine` | Yes — `~/.claude/projects/<workdir>/` + `CLAUDE.md` | `--resume <id>` | Also receives L28 recall + OS memory automatically |
| `CodexCliEngine` | No | — | Context window only |
| `OpenCodeEngine` | No | — | Provider-agnostic; context window only |
| `GeminiCLI` (planned) | No | — | 1M-token window; no persistent memory |

### Structural gap

Workers are currently spawned with a literal instruction string and no OS-layer
context. There are two directions missing:

1. **Export (pre-spawn):** Relevant OS memory is not transferred into the worker's
   initial context — neither as file-based memory for Claude Code workers, nor as
   a structured context block for non-CC workers.

2. **Import (post-turn):** After each worker turn, any new decisions, user
   corrections, or project facts discovered by the worker are lost. The OS-layer
   memory pool does not grow from worker output.

These two gaps compound: each worker starts from a clean slate and leaves no
trace, regardless of how many prior turns have established relevant context.

---

## Decision

### §1 — Native Pass-Through for Claude Code Workers

`ClaudeCodeEngine` workers already run on the same filesystem. When a CC worker
is spawned in the correct working directory, it automatically reads:

- The project's `CLAUDE.md`
- `~/.claude/projects/<workdir>/memory/MEMORY.md` and topic files

**Problem today:** `ClaudeCodeEngine` receives a generic `--project-dir`
argument rather than a scope-specific directory. Autonomous-loop workers and
main-thread workers share the same workdir, so the worker reads the full global
context indiscriminately.

**Fix:** When spawning a CC worker for a delegation task (L29), set
`--project-dir` to the session-scope directory:

```
<atelier_home>/tenants/<tid>/sessions/<bridge>:<chat>/
```

Before spawn, write a task-scoped `CLAUDE.md` and populate a
`.claude/memory/` sub-directory with the memory files that Haiku-4.5 selected
as relevant (the same selector used by §2's `build_context_block()`). The CC
worker reads these files via its native mechanism; they are deleted on session
cleanup (L8 reset audit-first invariant applies).

### §2 — Memory Bridge (reference to ADR-0050 §2)

The `memory_bridge.py` module (`operator/bridges/shared/`) is specified in
ADR-0050 §2. This ADR extends the scope of that module to include both the
**export** and the **import** paths described below.

**Before spawn (export) — ADR-0050 §2 covers this:**

- Haiku-4.5 selector (`claude -p --max-turns 1 --no-tools`) reads the OS memory
  pool and returns the most relevant entries.
- For **Claude Code** workers: writes selected memory to the session-scope
  `.claude/memory/` directory (§1 above).
- For **all other engines**: compresses selected entries into an `<os_context>`
  prose block injected at the start of the self-contained delegate prompt.
- Fallback on selector timeout (10 s ceiling): recency-truncation of raw
  material; no exception raised to caller.

**After each worker turn (import/harvest) — NEW in this ADR:**

See §4 (Post-Turn Memory Harvest).

### §3 — Structured Context Seeding Format

The current self-contained delegate prompt passes OS context as unstructured
freetext. Replace this with a canonical XML-scoped block:

```
<atelier_context>
  <user_profile>…</user_profile>
  <project_facts>…</project_facts>
  <session_learnings>…</session_learnings>
</atelier_context>
```

**Sources per sub-block:**

| Block | Source |
|---|---|
| `<user_profile>` | L28 per-user model (`<global>/memory/user_model/<chan>__<chat>.json`) |
| `<project_facts>` | Project-scope OS memory files (topic files with keyword overlap to the task) |
| `<session_learnings>` | Session-scope memory entries + last N recall.db turns |

This block format applies identically to all engine types. For Claude Code
workers it is injected into the task-scoped `CLAUDE.md` written by §1. For all
others it is prepended to the delegate prompt before the `<delegated_skill>`
block and before the `<a2a_instruction>` framing block (L38 ordering).

`memory_bridge.build_context_block()` (ADR-0050 §2.1) is extended to produce
this structured format rather than an unstructured prose paragraph.

**Size budget per block:**

| Block | Default cap |
|---|---|
| `<user_profile>` | 600 chars |
| `<project_facts>` | 800 chars |
| `<session_learnings>` | 600 chars |
| Total `<atelier_context>` | 2 000 chars (configurable via `ATELIER_CONTEXT_MAX_CHARS`) |

Haiku-4.5 compression applies per block; each block falls back to
recency-truncation independently.

### §4 — Post-Turn Memory Harvest

After each worker turn completes, a **harvest step** extracts memory-worthy
facts from the worker output and writes them back into the OS memory pool.

**4.1 Trigger**

The harvest runs as a PostToolUse hook after every `delegate_*` tool call that
produces a non-empty `final_text`. It MUST NOT block the main-thread response;
it runs in a detached background task with a 30 s wall-clock ceiling.

**4.2 Haiku-4.5 extractor**

```
claude -p --max-turns 1 --no-tools
```

Receives a self-contained prompt with:
- The original task instruction (truncated to 2 KB).
- The worker's `final_text` output (truncated to 4 KB).
- A structured extraction schema asking for: new decisions, project facts,
  user corrections, codebase learnings — each as a short labelled entry.

Returns a JSON array of `{ "scope": "session|project|user", "type": "feedback|project|user", "body": "…" }`.

If the subprocess fails or returns no extractable entries, the harvest is silently
skipped. No exception is propagated.

**4.3 Write path**

Extracted entries are written as OS memory files into the appropriate scope:

| Scope | Target directory |
|---|---|
| `session` | `<session_dir>/.claude/memory/` (cleaned on L8 reset) |
| `project` | `<atelier_home>/tenants/<tid>/global/memory/worker_harvest/` |
| `user` | `~/.claude/projects/<path>/memory/` (same pool as OS-layer memory) |

Each entry is a self-contained markdown file using the standard memory
frontmatter schema (name, description, metadata.type).

**4.4 Audit event**

| Event | Severity | Allowed fields |
|---|---|---|
| `memory.worker_harvest` | INFO | `channel`, `chat_key`, `tenant_id`, `scope`, `entry_count`, `engine_id` |

Entry content MUST NOT appear in any audit field.

**4.5 Integration with L33 Artifact Memory**

If the worker's output references a file path that was registered in the L33
artifact store during the same turn, the harvest step adds a cross-reference
from the extracted memory entry to the artifact ID. This allows future context
seeding to pull the artifact description via `artifact_search` rather than
re-reading the file.

---

## Interface summary

```python
# operator/bridges/shared/memory_bridge.py

def build_context_block(
    scope: str,               # "tenant:channel:chat_key"
    workdir: Path,
    profile: dict,
    engine_type: str,         # "claude_code" | "codex" | "opencode" | ...
    instruction_hint: str = "",  # used for keyword-based topic-file selection
    max_chars: int = 2000,
) -> str:
    """Return <atelier_context>...</atelier_context> or empty string."""
    ...

def harvest_worker_output(
    scope: str,
    instruction: str,
    output: str,
    engine_id: str,
    chat_key: str,
    tenant_id: str,
) -> None:
    """Fire-and-forget: extract memory entries from worker output."""
    ...
```

`memory_bridge.py` MUST NOT `import anthropic`. Both functions invoke
Haiku-4.5 only via `subprocess`.

---

## Consequences

### Positive

- CC workers gain scoped, task-relevant memory via the native file-based
  mechanism — no protocol change to the CC engine.
- Non-CC workers receive structured, engine-agnostic context in a canonical
  format that LLMs parse reliably.
- The OS memory pool grows over time from worker output: learnings, corrections,
  and project facts accumulate across turns and sessions.
- Structured `<atelier_context>` blocks are easier to audit for PII leaks than
  unstructured freetext injection.

### Risks and mitigations

| Risk | Mitigation |
|---|---|
| Harvest writes stale or hallucinated entries | Haiku prompt is instruction-grounded and schema-constrained; entries are never auto-promoted to user-scope without operator confirmation |
| PII in worker output flows into memory files | Harvest prompt includes explicit PII-redaction instruction; `memory_bridge.py` post-processes entries through the same NFKC + redaction pipeline as L28 recall |
| Harvest background task outlives session reset | L8 reset sets a cancellation event that harvest tasks check at write time; a task that arrives after the cancellation event is silently discarded |
| `<atelier_context>` block exceeds model context budget on high-recall chats | Per-block caps + total `max_chars` guard enforced after Haiku compression |
| CC worker reads wrong project CLAUDE.md if session dir is reused | Session-scope `.claude/memory/` is keyed to `chat_key`; mismatch → fresh directory |
| Harvest subprocess adds latency visible to the user | Fire-and-forget detached task; harvest never blocks the main-thread reply |

---

## Must NOT do

- Don't call `import anthropic` from `memory_bridge.py` — subprocess only.
- Don't apply `build_context_block()` to `ClaudeCodeEngine` main-thread turns
  — the native OS memory path already handles this; double-injection wastes
  context budget.
- Don't write harvest entries to the L16 audit chain with entry content — only
  `entry_count` and `scope` are allowed in audit fields.
- Don't auto-promote worker-harvested entries from `project` to `user` scope
  — promotion requires operator/user confirmation (same rule as SkillForge L7).
- Don't skip the per-block PII-redaction pass before writing harvest entries.
- Don't share session-scope `.claude/memory/` directories across chat keys.
- Don't allow harvest to write into the L33 artifact store directly — only the
  cross-reference link from a memory entry to an existing artifact ID is
  permitted.
- Don't retry Haiku selector or harvest subprocesses on failure — fall back
  silently; a retry loop doubles the latency risk.
- Don't put instruction text or worker output in any audit `details` field.

---

## Milestones

| Milestone | Scope | Status |
|---|---|---|
| **M1** | `build_context_block()` extended to structured `<atelier_context>` format (§3); CC worker session-scope `--project-dir` + scoped `.claude/memory/` write (§1) | Done |
| **M2** | `harvest_worker_output()` implemented; PostToolUse hook wired for `delegate_*` tools; session-scope write path + audit event | Done |
| **M3** | Project-scope and user-scope harvest write paths; L33 artifact cross-reference; `ATELIER_CONTEXT_MAX_CHARS` env var; E2E test: two delegation turns verify memory round-trip | Done |
