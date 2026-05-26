# ADR-0016 — Conversation Recall + User Modeling (Layer 28)

**Status:** Proposed
**Date:** 2026-05-15
**Companion to:** Layer 7 (Skill-Inject), Layer 10 (path-gate),
Layer 11 (dialectic — `cli` mode), Layer 23 (STT metadata-only-audit
rule), Layer 24 (PII detector — regex backend),
Layer 26 (user-style learner), ADR-0007 Phase 1 (per-tenant tree),
Compliance baseline (GDPR Art. 5(1)(c), 6, 7, 17, 30, 32; EU AI Act
2026 Art. 14).
**Implements:** Phase 28.1 (Conversation Recall) + Phase 28.2 (User
Modeling) land in the same PR; Phase 28.3 (Provider Plugin Protocol)
and Phase 28.4 (GDPR Forget + Retention Sweep) defer until a second
provider or a regulator request forces them.

---

## Context

AtelierOS today has **no cross-session conversation memory**. Bridge
chats persist in `<tenant_home>/voice/sessions/<chan>:<chat>/` as the
Claude-Code `.claude.json` history (the engine's own per-session
context), but:

- The owner cannot ask "what did we decide three weeks ago about X?"
  because there is no searchable index.
- A persona has no representation of who the owner is — their
  preferences, recurring topics, communication style — beyond what
  Layer 7 skill-inject and Layer 26 auto-learned-style bullets surface
  per turn. Both are signal-driven (outcome-grading), not state-driven
  (cross-session recall of factual context).
- Comparable agent frameworks (Hermes Agent: FTS5 + Honcho user-model;
  Mem0 / Supermemory) ship this as a first-class concept. AtelierOS's
  Layer 26 user-style-learner addresses **how the assistant speaks**;
  it does not address **what the assistant remembers about the user**.

The functional gap is real. The design constraint is that closing it
without violating the compliance baseline is non-trivial: a naive
FTS5 index over every bridge conversation would lay down a
plaintext-PII corpus on disk that the operator never asked for. A
naive user-model distill would write a personality profile of a
human into the audit chain.

This ADR is the design contract for closing the gap while keeping
GDPR Art. 5 (data minimisation), Art. 6/7 (consent), Art. 17 (right
to erasure), Art. 30 (record of processing) and Art. 32 (security of
processing) structurally intact.

---

## Decision

Two complementary mechanisms layered on top of the existing
per-tenant tree, both **opt-in per chat_profile** and both routed
through the unified audit hash chain.

### A) Conversation Recall (Phase 28.1)

A per-tenant FTS5 SQLite index of redacted bridge turn-pairs.

- **Storage:** `<tenant_home>/global/memory/recall.db`, mode `0o600`,
  SQLite WAL + virtual FTS5 table over (`user_text_redacted`,
  `assistant_text_redacted`).
- **Schema (turn table):**

  ```
  CREATE TABLE turns (
      id              INTEGER PRIMARY KEY AUTOINCREMENT,
      ts              REAL NOT NULL,
      channel         TEXT NOT NULL,
      chat_key        TEXT NOT NULL,
      msg_id          TEXT,
      run_id          TEXT,
      persona         TEXT,
      user_chars      INTEGER NOT NULL,    -- pre-redaction length
      asst_chars      INTEGER NOT NULL,
      redacted_classes TEXT NOT NULL       -- comma-list of PII classes hit
  );
  CREATE VIRTUAL TABLE turns_fts USING fts5(
      user_text_redacted, assistant_text_redacted,
      content='turns', content_rowid='id',
      tokenize='unicode61 remove_diacritics 2'
  );
  ```

- **Append-Hook:** in `adapter.process_one`, AFTER the assistant
  reply is produced AND BEFORE the outbox envelope is written, the
  turn-pair is handed to `conversation_recall.index_turn(...)`. Both
  user-msg and assistant-reply are passed through the **text-mode PII
  redactor** (Layer 24's regex backend, adapted for free text instead
  of column samples) before they hit the FTS5 table. The original
  text NEVER lands in the index.
- **Recall surface:** a Python function
  `conversation_recall.recall(query, *, channel=None, chat_key=None, since=None, until=None, limit=20)`
  returning a list of redacted turn-pair excerpts with metadata. The
  function is consumable from three call sites:
  1. **MCP tool** (`mcp__memory__recall`) exposed by a new
     bundled-but-opt-in MCP server `atelier-memory`. Wired via the
     cowork resolver's `_inject_memory_capability(merged, persona)`
     analogous to `_inject_forge_capability`. Persona-ACL field:
     `memory_recall_enabled: true|false` (default false; the
     `assistant` / `research` / `coder` bundle personas flip to
     true). Same per-persona-injection pattern as forge / skill-forge.
  2. **Slash-command** `/recall <query>` (owner + admin via Layer-18
     roles), reachable through the existing
     `in_chat_commands.js` dispatcher. Returns the top-N hits as a
     terse chat reply with `[ts, persona, snippet]` lines.
  3. **Adapter-inject** when a chat_profile flag
     (`recall_inject_on_question: true`, default false) is set —
     under that flag, every user-msg that ends with `?` triggers a
     5-result recall and prepends a `<recall>...</recall>` block to
     the prompt. This is the **lazy-recall** mode for chats where
     the owner wants the LLM to spontaneously remember.
- **Audit events** (registered in `EVENT_SEVERITY`, metadata only):
  - `memory.turn_indexed` (INFO) — channel, chat_key, msg_id,
    user_chars, asst_chars, redacted_classes_count. NEVER any text.
  - `memory.recall_query` (INFO) — channel, chat_key, query_chars,
    result_count, caller_persona, since, until.
  - `memory.indexing_failed` (WARNING) — reason
    (`db-locked` / `redact-failed` / `disk-full`), msg_id,
    error (200-char cap).
- **Cost contract:** the module MUST NOT `import anthropic` (or any
  LLM SDK). Indexing is pure SQLite + regex; recall is pure SQLite
  FTS5 query. The dialectic / distill pipeline that DOES want LLM
  reasoning lives in Phase 28.2 and goes through `dialectic.py`'s
  `cli` mode (operator's Max-Abo).

### B) User Modeling (Phase 28.2)

A per-(tenant, chat_key) structured representation of the owner,
distilled periodically from the recall index.

- **Storage:**
  `<tenant_home>/global/memory/user_model/<safe_channel>__<safe_chat>.json`,
  mode `0o600`. Schema-versioned (`apiVersion: atelier/v1`,
  `kind: UserModel`).
- **Schema (curated, Pydantic-like — no free-form keys):**

  ```jsonc
  {
    "apiVersion": "atelier/v1",
    "kind":       "UserModel",
    "metadata": {
      "channel":      "discord",
      "chat_key":     "<safe>",
      "created_at":   1778204770.0,
      "updated_at":   1778205000.0,
      "distill_count": 7
    },
    "spec": {
      "communication_style": "concise, asks for trade-offs explicitly",
      "preferences":         ["German chat", "voice-note replies", "..."],
      "recurring_topics":    ["AtelierOS layer design", "compliance audit", "..."],
      "goals":               ["..."],
      "patterns":            ["..."],
      "do_not_assume":       ["..."]
    }
  }
  ```

  Each `spec` field is `list[str]` (or scalar for `communication_style`)
  with hard caps: max 10 entries per field, max 200 chars per entry.
  The distiller is instructed to write **observable patterns**, not
  inferred psychology — "asks for trade-offs" is fine, "is detail-
  oriented" is not.
- **Distill pipeline:** triggered by
  - `cron`: a periodic adapter-side timer (default every 50
    successful turns per chat, configurable via
    `chat_profile.user_model_distill_every_n_turns`), OR
  - explicit `/memory distill` slash-command (owner).

  The distiller calls `dialectic.cli_judge(...)`-style subprocess:
  `claude -p --max-turns 1 --no-tools` with a fixed system prompt
  that takes the previous `UserModel.spec` + the redacted last-30
  turn-pairs from `conversation_recall` as input and returns the
  refreshed `spec` as JSON. Cost: one Max-Abo subprocess call per
  distill (operator-subscription-native; no Anthropic SDK).
- **Two-layer context injection:** `skill_inject.collect_active_skills`
  is extended to also fold the active `UserModel.spec` (rendered as
  a `<user_context>...</user_context>` markdown block) into the
  per-turn append. The order is:

  ```
  <skill-inject block from Layer 7>
  <user_style bullets from Layer 26>
  <user_context from Layer 28>     ← new
  ```

  User context lands LAST so it is the most-recent instruction for
  the LLM (mirroring the `summarize.py::SELF_CHECK_BLOCK` ordering
  rule).
- **Opt-in default-off:** `chat_profile.user_model_enabled: true`
  is required. Without it, the distill pipeline never fires, the
  injection block stays empty, and no `memory.user_model_*` audit
  event is emitted. GDPR Art. 6 — explicit opt-in for
  personality-shaped processing.
- **Audit events:**
  - `memory.user_model_distilled` (INFO) — channel, chat_key, fields
    that changed (names only, NEVER values), distill_count,
    judge_wall_clock_s.
  - `memory.user_model_distill_failed` (WARNING) — reason
    (`judge-unparseable` / `judge-timeout` / `recall-empty` /
    `schema-violation`), error (200-char cap).
  - `memory.user_model_forgotten` (INFO) — emitted on `/forget user`
    or on operator-side delete.
- **The distilled value is operator-only on disk; the audit chain
  records WHEN distill happened and WHICH fields changed, never the
  values.** Mirror of L23 voice-transcribe-metadata-only rule.

### C) Path-gate extension

`<atelier_home>/**/memory/**` joins the existing `forge`, `skill-forge`,
`audit.jsonl`, `policy.json`, `data_policy.{yaml,yml,json}`, `compute`,
`admin` protected-path set. The LLM may NOT direct-write to the
recall DB, the user-model JSON, or any future memory artefact via
Write / Edit / Bash / sed. All writes go through the
`conversation_recall.index_turn(...)` / `user_model.save(...)` Python
API, which is reached only from adapter code or operator CLI. The MCP
recall tool is read-only; there is no MCP write surface in Phase 28.1.

### D) GDPR Right to Erasure (deferred to Phase 28.4, designed here)

The architecture pre-anticipates Art. 17 even though the slash-command
ships later:

- `conversation_recall.forget(filter)` deletes matching rows from the
  `turns` table and rebuilds the FTS5 index over the remaining rows.
  The deletion is **opaque to the hash chain** — the audit event
  `memory.forgotten` records that a forget happened, the pattern, and
  the row count, but **not** the deleted text. The chain stays intact
  through the deletion (DSGVO requires deleting the data, not the
  audit record of the deletion).
- `user_model.forget(channel, chat_key)` deletes the JSON file +
  emits `memory.user_model_forgotten`.
- Daily retention sweep (deferred): per-tenant
  `memory.retention_days` (default 30 in Phase 28.4) triggers a
  systemd-timer that purges turns older than the TTL. User-scope
  opt-out via `retention_days: 0` (unlimited).

---

## What this layer deliberately does NOT do

- **No vector embedding.** FTS5 lexical + redacted text is the
  Phase 28.1 contract. Vector embeddings would either require an
  external embedding service (Anthropic API — violates the cost
  contract) or a local model (~500 MB dependency — violates the
  zero-dependency invariant for the bridge process). A future
  `MemoryProvider` plugin can layer vector search on top; the
  bundled default stays lexical.
- **No external memory backend.** Honcho-style external providers
  are out-of-scope for the bundled implementation. Phase 28.3 lands
  the `MemoryProvider` Protocol and an example external-provider
  stub when an operator actually requests one.
- **No cross-chat recall by default.** A recall query is scoped to
  the current `(channel, chat_key)` unless the caller explicitly
  passes `chat_key=None` AND the persona-ACL grants
  `memory_recall_cross_chat: true` (default false; ONLY the
  `assistant` bundle persona could plausibly need this and it does
  not have it set today). Cross-tenant recall is structurally
  impossible because the DB lives in the tenant tree.
- **No raw text in any audit event.** Mirror of L23 / L24 / L25 /
  L26 — metadata only. The redacted-text-in-FTS5-on-disk is the
  ONLY persistence of conversation content this layer introduces
  beyond what the engine already keeps in `.claude.json`.
- **No automatic distill without opt-in.** A user who never set
  `user_model_enabled: true` sees zero user-model-related code paths
  fire, zero audit events, zero JSON files on disk.
- **No "memory inheritance" across tenants.** A migrated owner-uid
  in tenant ACME does NOT carry recall history into tenant GLOBEX.
  The tenant tree is the recall scope, structurally.

---

## What you, as Claude Code, must NOT do (Layer 28)

- **Don't index raw conversation text.** Every text that lands in
  the FTS5 table goes through the text-mode PII redactor first. The
  regression gate (`test_conversation_recall.py::test_redaction_runs_before_index`)
  inserts a known-PII string and asserts the indexed row contains
  the redaction tokens, not the raw value. DSGVO Art. 5(1)(c)
  baseline.
- **Don't fold the user-model JSON values into the audit chain.**
  `memory.user_model_distilled` carries the list of changed field
  **names** plus the `distill_count`, NEVER the actual `spec` fields.
  Mirror of the L23 metadata-only rule.
- **Don't allow `recall.recall(...)` to cross tenants.** The DB
  path is tenant-resolved before the SQLite open call; there is no
  multi-tenant aggregation method. A future "operator-side
  cross-tenant analytics" feature belongs in a separate ADR with its
  own auth gate, not as an extension here.
- **Don't make the recall query bypass the persona-ACL.** The MCP
  tool `mcp__memory__recall` is only injected into personas with
  `memory_recall_enabled: true`. Adding a back-door function call
  (e.g. via the adapter shortcut path) would defeat the per-persona
  visibility gate. The persona-ACL is the structural defence against
  a `forge` persona reading the owner's recall history mid-tool-run.
- **Don't make the distiller call `anthropic` SDK directly.** The
  CI lint (`test_user_model.py::case_no_anthropic_sdk_import`) walks
  `user_model.py` and `conversation_recall.py` AST and fails on
  forbidden imports. The cost contract is operator-subscription-
  native via `claude -p` subprocess.
- **Don't drop the opt-in gate for user-modeling.** GDPR Art. 6
  requires an explicit lawful basis for processing personality data
  about a natural person. `chat_profile.user_model_enabled: true`
  IS that basis (operator opts in for their own chat). Auto-enabling
  it across all chats would convert the layer from "opt-in
  feature" to "ambient surveillance".
- **Don't widen the persona-ACL list silently.** The bundle defaults
  (`assistant`/`research`/`coder` → true; everything else → false)
  are the contract. New personas land with `memory_recall_enabled:
  false` unless an explicit decision says otherwise.
- **Don't store the SQLite DB outside the tenant tree.** A "shared
  recall index" path (e.g. `<atelier_home>/global/memory/recall.db`
  at the root) breaks per-tenant isolation: tenants would see each
  other's redacted turn metadata via cardinality side-channels.
- **Don't widen the audit-event allow-list to free-form details.**
  Each new memory event has a curated detail-key set (mirroring
  L25 `_ALLOWED_FIELDS` pattern). Extra keys raise.
- **Don't lower the text-redaction confidence threshold.** The
  text-mode redactor uses the same regexes as Layer 24's value-regex
  backend, with a 1-hit threshold (every match becomes a redaction).
  Tuning it down to "redact only on 3+ hits" would let an isolated
  e-mail address slip through.
- **Don't add a "verbose log" mode that prints recall query results
  to stderr.** The recall surface is one-way (query → result to the
  caller). Logging results would defeat the per-persona-ACL because
  the operator's log file would carry text the LLM was structurally
  prevented from seeing.

---

## Implementation phases

| # | Scope | Files | Approx LOC |
|---|---|---|---|
| 28.1 | Conversation Recall storage + indexing + recall + audit + persona-ACL | `bridges/shared/conversation_recall.py`, `test_conversation_recall.py`, `forge/security_events.py` (+3 events), `hooks/path_gate.py` (memory tree) | ~600 + ~300 test |
| 28.2 | User Modeling storage + distiller + adapter-inject extension + audit | `bridges/shared/user_model.py`, `test_user_model.py`, `forge/security_events.py` (+3 events), `bridges/shared/skill_inject.py` (extension hook) | ~500 + ~250 test |
| 28.3 | MemoryProvider Protocol + plugin slot + tenant.atelier.yaml schema | `bridges/shared/memory_provider.py`, `forge/atelier_gateway/tenant_config.py` (schema) | deferred |
| 28.4 | `/forget` + retention sweep + systemd timer | `bridges/shared/memory_forget.py`, `voice/scripts/memory_retention_sweep.py`, systemd unit | deferred |

Adapter integration (the `process_one` append-hook in 28.1, the
inject extension in 28.2) lands inside the same PR as 28.1/28.2 — it's
small enough that a separate phase isn't warranted.

The MCP server `atelier-memory` lands in 28.1 because the recall
surface is useless without a call site; it's a thin JSON-RPC wrapper
around `conversation_recall.recall()`.

---

## References

- Layer 23 (STT) — metadata-only audit precedent (`voice.transcribed`
  carries provider/duration/char-count, never `result.text`)
- Layer 24 (PII detector) — regex backend reused for text mode
- Layer 26 (user-style learner) — closed-loop bullet pipeline; same
  cost-contract pattern (no Anthropic SDK; dialectic CLI subprocess)
- Layer 11 (dialectic) — `cli` mode = `claude -p --max-turns 1
  --no-tools`; the distiller follows the same shape
- Hermes Agent docs — FTS5 cross-session recall + Honcho user-model
  (external comparison, ADR-0016 designs an AtelierOS-shaped analogue
  with compliance gates the Hermes design does not have)
