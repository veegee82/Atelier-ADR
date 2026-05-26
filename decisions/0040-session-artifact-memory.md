# ADR-0040 — Session Artifact Memory (Layer 33)

**Status:** Accepted (2026-05-19)
**Touches:** `operator/forge/forge/artifacts.py` (NEW), `operator/forge/forge/mcp_server.py` (EXTEND), `operator/voice/hooks/path_gate.py` (EXTEND), `operator/bridges/shared/adapter.py` (EXTEND), `operator/bridges/shared/conversation_recall.py` (EXTEND)
**Sister-ref:** `docs/claude-ref/layer-33-artifacts.md`

---

## Context

Sessions in AtelierOS generate **non-text artifacts** — PDFs, charts,
CSVs, generated images, exported HTML — that the model and user want
to reference in **later tasks of the same session** without re-pasting
the content or paying token costs for unconditional context injection.

Current state (before this ADR):

- **Layer 6 (Forge)** stores generated *tools* per scope, not arbitrary
  files.
- **Layer 7 (SkillForge)** stores generated *skills* (markdown prompt
  fragments).
- **Layer 28 (Recall)** indexes PII-redacted *conversation turns*, not
  binary artifacts.
- **No convention** exists for "files the model emitted that should
  survive across turns but die with the session reset."

Operators today work around this by either re-uploading every turn
(token cost) or by hand-pasting paths (brittle). Neither scales.

## Non-goals

- **Auto-inject** of artifact contents into the prompt. Token cost is
  prohibitive and most artifacts are not relevant to the next turn.
- **Cross-tenant** sharing. Artifacts are tenant-scoped per ADR-0007.
- **Vector embeddings.** FTS5 + LLM-generated description is sufficient
  for realistic session sizes (≤ 80 artifacts). Migration path exists.

---

## Decision

Introduce **Layer 33 — Session Artifact Memory**, a thin file-system
layer with six MCP tools and one auto-register hook.

### Storage layout

```
<atelier_home>/tenants/<tid>/
├── sessions/<bridge>:<chat>/artifacts/        # session-scope
│   ├── .manifest.jsonl                        # append-only source of truth
│   ├── ab/ab3f5c2e_q3_budget.pdf              # sha256-prefix sharded
│   └── ...
└── global/artifacts/                          # project-scope (pinned)
    └── ...
```

The session-scope tree shares its parent with all other session-bound
state (forge, skill-forge, voice). Lifecycle is bound to Layer 8
(session reset). Pinned artifacts move to the global scope and survive
session reset.

### Manifest entry (one line per artifact)

```json
{
  "ts": 1747688400.123,
  "name": "q3_budget_v2.pdf",
  "sha256": "ab3f5c2e9...",
  "size": 184320,
  "mime": "application/pdf",
  "path_rel": "ab/ab3f5c2e_q3_budget_v2.pdf",
  "by_tool": "forge.gen_pdf",
  "run_id": "r_482...",
  "description": "Q3 budget proposal, 14 pages, German, financial tables.",
  "tags": ["budget", "internal"],
  "pinned": false
}
```

### MCP tools (registered on the existing Forge MCP server)

| Tool | Returns | Notes |
|---|---|---|
| `artifact_list(after_ts?, mime?, limit=20)` | metadata array | no content |
| `artifact_search(query)` | FTS5 hits from `recall.db` | reuses Layer 28 index |
| `artifact_get(name, max_bytes=65536)` | text content OR `{too_large, hint}` | binary returned as base64 |
| `artifact_extract(name, range)` | page / line range | PDFs via `pdftotext`, images via metadata-only |
| `artifact_register(path, description?, tags?)` | manifest entry | manual fallback for tools that don't auto-register |
| `artifact_pin(name)` | new path | promotes to global scope |

### Auto-register

A PostToolUse hook fires after `Write` / `Edit` / `NotebookEdit` and
checks two predicates:

1. Output path is inside `<session>/artifacts/`, OR
2. Output MIME ∈ {`application/pdf`, `image/*`, `text/csv`,
   `application/vnd.openxmlformats-*`, `image/svg+xml`, `text/html`}.

If either matches, an entry is appended to `.manifest.jsonl`. A
Haiku-4.5 helper (Layer 29.5) generates a one-sentence description
in the artifact's natural language (auto-detect) and stores it
alongside. The description is also indexed into `recall.db` with
class `artifact_summary` so that `recall_search` and `artifact_search`
share a single FTS5 index.

### Lifecycle

- **/new /clear /reset** (Layer 8) → `artifact.session_purged` audit
  event **before** rmtree (audit-first rule). Pre-warn list is sent
  to the bridge if unpinned artifacts exist; user must `/reset ack`
  to override.
- **Daily TTL sweep** (`atelier-session-timeout.timer`, 03:30 UTC)
  → artifacts inherit the session's 7-day TTL.
- **Pin** → file is moved to `<global>/artifacts/`, manifest entry
  copied with `pinned: true`, audit event `artifact.pinned` emitted.

### Audit events (in the unified hash chain)

| Event | Severity | Details policy |
|---|---|---|
| `artifact.registered` | INFO | name, sha256, size, mime, by_tool — never path content or description text |
| `artifact.read` | INFO | name, requested_range |
| `artifact.pinned` | INFO | name, sha256 |
| `artifact.purged` | INFO | name, sha256 |
| `artifact.session_purged` | CRITICAL | count, session_key — never artifact names |

---

## Compliance posture (load-bearing)

- **GDPR Art. 17 (Erasure):** `/forget` (Layer 28) extends to purge
  matching artifact entries + on-disk files in the active scope.
- **GDPR Art. 32 (Integrity):** Manifest writes go through `fcntl.flock`
  + in-process lock (same pattern as `audit.jsonl`). Direct LLM writes
  to `<session>/artifacts/` are denied by Layer 10 path-gate; the only
  legitimate write path is the MCP server.
- **EU AI Act Art. 14 (Transparency):** Every register / read / pin /
  purge is one audit event. `voice-audit verify` covers the chain.
- **Description PII redaction:** Haiku-generated descriptions go
  through the same `pii_redact` pipeline that Layer 28 uses before
  being written to manifest or indexed into `recall.db`.

---

## Token-cost characterisation

- **One system-prompt hint per turn:** ~30 input tokens, fixed.
- **`artifact_list` default response:** ~200 output tokens for 10
  artifacts (name, mime, size, ts, 1-sentence description).
- **`artifact_search` response:** ~150 output tokens for 3 hits + FTS5
  snippets.
- **`artifact_get` cap:** 65 536 bytes / ~16 000 tokens at most;
  larger artifacts force the caller to `artifact_extract` with an
  explicit range.

Net effect: the model can locate any artifact in <500 tokens and
fetch only the bytes it actually needs.

---

## Alternatives considered

| Alt | Reject reason |
|---|---|
| **FS-convention only (no manifest)** | No semantic search; user pays the cost of remembering paths. |
| **Per-session SQLite + FTS5** | Doubles Layer 28's machinery for no gain at realistic session sizes. Migration path preserved (see *Migration to dedicated DB* below). |
| **Extend `recall.db` table** | Conflates binary-artifact metadata with PII-redacted text turns; Lifecycle of Layer 28 is owner-of-session-state, artifacts have a different reset semantics. |
| **SkillForge-style 4-scope promotion** | Overkill for the dominant use-case ("I want yesterday's PDF"). Pin to global is the only promotion users actually need. |
| **External object store (S3/MinIO)** | Breaks on-prem / EU data-residency posture (ADR-0007 zone gate). Future backend option, not v1. |

---

## Migration to dedicated DB

If a deployment exceeds ~500 active artifacts per session (telemetry
threshold), a config flag in `<atelier_home>/global/artifacts.config.json`
flips the storage backend from `"jsonl"` to `"sqlite"`. The
manifest-JSONL schema is forward-compatible with the SQLite-table shape
(`artifacts(ts, name, sha256, size, mime, path_rel, by_tool, run_id,
description, tags, pinned)`); the migration helper ingests the JSONL
and writes the DB without touching the on-disk files.

---

## Must-NOT do (locked-in constraints)

- Don't auto-inject artifact contents into the system prompt.
- Don't write the description text or path content into any audit-event
  details field.
- Don't allow direct LLM writes to `<session>/artifacts/` — only via
  the MCP server.
- Don't promote pinned artifacts beyond `global/artifacts/` (no
  cross-tenant, no cross-user).
- Don't share the `recall.db` artifact-summary class outside the
  active tenant.
- Don't run the Haiku description on artifacts marked `tags:
  ["no-summary"]` (operator escape-hatch).

See `docs/claude-ref/layer-33-artifacts.md` for the full reference.
