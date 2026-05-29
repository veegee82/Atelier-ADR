# ADR-0066 — Layer 22 Extension: HermesEngine — Locally-Hosted Open-Source Agent via Ollama

**Status:** Accepted — 2026-05-29 · M1 Implemented & tested 2026-05-29
**Date:** 2026-05-29
**Authors:** Claude Code (maintainer session)
**Type:** Architecture — WorkerEngine extension + local inference integration
**Extends:** [ADR-0007 (Multi-tenant)](0007-multi-tenant-axis.md), Layer 22 (WorkerEngine Protocol)
**Depends on:** L22 (WorkerEngine), L29 (Delegation), L29.1 (Hardening), L29.2 (Confinement), L34 (Data Classification), L35 (Egress Lockdown)
**Scope:** `operator/bridges/shared/agents/hermes_engine.py` · `operator/bridges/shared/delegate_hermes.py` · `operator/cowork/personas/hermes-worker.json`

---

## Context

AtelierOS currently ships three `WorkerEngine` implementations (L22):

| Engine | Binary | Cloud dependency | Locality |
|---|---|---|---|
| `ClaudeCodeEngine` | `claude` | Anthropic API | Remote (EU cloud w/ zone) |
| `CodexCliEngine` | `codex` | OpenAI API | Remote |
| `OpenCodeEngine` | `opencode` | Configurable provider | Depends on model |

All three can be routed through EU-zone endpoints, but all require an external
API call for inference. There is no fully-local, zero-egress engine that can
serve as an air-gapped or cost-zero worker for appropriate task classes.

**NousResearch Hermes** (Hermes-2, Hermes-3) is a family of open-source LLMs
fine-tuned for strong tool/function-calling and agentic behavior. Key properties:

- Licensed Apache-2.0 / Llama-Community — compatible with AtelierOS's Apache-2.0
  licensing baseline (CLAUDE.md §Licensing).
- Available as quantized GGUF models via **Ollama** (which most operators already
  run for local inference). Sizes: 7 B through 70 B, selectable per task.
- ChatML prompt format with native `<tool_call>` / `<tool_response>` tokens;
  Hermes-3 adds structured JSON-schema output enforcement.
- Zero network egress when running via local Ollama: `localhost:11434`. Satisfies
  L35 `default_action: deny` with no `allowed_hosts` entry needed for inference.
- Data-classification matrix maps to `locality: local` and `network_egress: none`
  — the highest trust tier in L34. Enables `CONFIDENTIAL` task classes that
  `ClaudeCodeEngine` cannot serve under strict EU_PRODUCTION preset.

**Gap being filled:** there is no engine that an operator can use for
CONFIDENTIAL / SECRET classification tasks where data must not leave the host.
Hermes-via-Ollama fills this gap.

### Design goals

1. Minimal footprint — `HermesEngine` wraps Ollama's existing HTTP streaming API
   (`/api/chat`); no new binary dependency beyond `ollama` (which is already a
   recommended local-compute tool in the deploy guide).
2. Conform to the `WorkerEngine` Protocol exactly — `spawn()` → `Iterator[StreamEvent]`,
   `cancel()`, capability dict.
3. Plug into L29 Delegation as a first-class target (`delegate_hermes` MCP tool).
4. Support Hermes native tool-calling (ChatML `<tool_call>` tokens) mapped to
   AtelierOS forge/skill-forge MCP tool format in M2.
5. Never `import anthropic` — enforcement parity with all other L22/L29/L34/L35
   modules (CI AST lint).

---

## Decision

Introduce `HermesEngine` as the fourth `WorkerEngine` implementation, staged
across three milestones.

### Variant analysis

Three integration shapes were considered:

**Variant A — Thin HTTP wrapper (chosen for M1)**
`HermesEngine.spawn()` sends a streaming POST to `http://localhost:11434/api/chat`
and maps Ollama's NDJSON chunks to `StreamEvent`. No subprocess; uses Python
`httpx` (already a dependency). Capabilities: `stream_json=True`, `mcp=False`,
`mid_stream_inject=False`, `hooks=False`, `skills_tool=False`.

**Variant B — `ollama run` CLI subprocess**
Spawns `ollama run <model> "<prompt>"` as a subprocess, parses stdout. Simpler
but loses streaming granularity, cannot pass system prompts cleanly, and
adds a shell-injection surface. Rejected.

**Variant C — Hermes + MCP bridge (target for M3)**
`HermesEngine` translates Hermes native `<tool_call>` tokens into forge/skill-forge
MCP calls, returning `<tool_response>` tokens. Gives Hermes access to the same
tool surface as `ClaudeCodeEngine`. Requires an MCP-over-stdio bridge process.
Deferred to M3.

### Milestone plan

#### M1 — Thin streaming engine (HTTP, no tool-calling) ✓ DONE

**New file:** `operator/bridges/shared/agents/hermes_engine.py`

```python
# WorkerEngine implementation — wraps Ollama /api/chat streaming endpoint.
# MUST NOT import anthropic (CI AST lint).

class HermesEngine:
    name = "hermes"
    capabilities = {
        "stream_json": True,
        "mcp": False,
        "mid_stream_inject": False,
        "hooks": False,
        "skills_tool": False,
        "permission_modes": ["read_only"],
    }

    def __init__(self, model: str = "nous-hermes-2", base_url: str = "http://localhost:11434"):
        self.model = model
        self.base_url = base_url
        self._cancel_event = threading.Event()

    def spawn(self, prompt, *, system=None, model=None, ...) -> Iterator[StreamEvent]:
        # POST /api/chat with stream=True
        # yield StreamEvent(type="text_delta", text=chunk["message"]["content"])
        # yield StreamEvent(type="turn_completed")
        ...

    def cancel(self):
        self._cancel_event.set()
```

**Supported Hermes models (default aliases):**

| Alias | Ollama model tag | Use-case |
|---|---|---|
| `hermes-fast` | `nous-hermes-2:7b-mistral-q4_K_M` | Quick summaries, routing |
| `hermes-balanced` | `nous-hermes-2:13b-q4_K_M` | General delegation |
| `hermes-capable` | `nous-hermes-3:8b-llama3.1-q5_K_M` | Tool-calling M2 |
| `hermes-large` | `hermes-3:70b-llama3.1-q4_K_M` | Complex reasoning |

Operator overrides via `ATELIER_HERMES_MODEL` env var or per-persona `engine_config.model`.

**M1 delivery notes (2026-05-29):**

- `hermes_engine.py` uses Python stdlib `urllib.request` (not `httpx`) — no new dependency introduced.
- Test model default in `test_hermes_engine.py`: `qwen3:1.7b` (any Ollama-compatible model works; not restricted to NousResearch family).
- 21/21 tests green: 12 protocol-contract unit tests + 7 live tests against local Ollama `qwen3:1.7b`.
- `delegation.py` → `AVAILABLE_ENGINES` includes `"hermes"`; `_default_engine_factory` routes to `HermesEngine()`.
- `mcp_server.py` → `delegate_hermes` tool registered; `_TOOL_NAME_TO_ENGINE` mapping added.
- `self_test.py` → `_check_hermes_ollama()` wired in `run_self_test()` (WARNING severity, never blocks boot).
- `resolver.py` → `_DELEGATE_BRIEF` and `_inject_delegate_capability()` updated to mention four engines.
- `hermes-worker.json` persona created; `routing_anchors` set for L5 auto-router.

**L29 delegation tool:**

```python
# delegate_hermes MCP tool — same contract as delegate_claude_code / delegate_codex.
# Parameters: prompt, model (optional), budget_s (10..600, default 60), working_dir.
# Returns: {ok, engine, final_text, duration_ms, usage, model, error}
```

**Persona stub:**

```json
// operator/cowork/personas/hermes-worker.json
{
  "name": "hermes-worker",
  "engine": "hermes",
  "engine_config": { "model": "hermes-balanced" },
  "ldd_preset": "off",
  "delegate_enabled": true,
  "memory_recall_enabled": false,
  "skill_forge_enabled": false,
  "description": "Fully-local Hermes worker — zero egress, CONFIDENTIAL-capable."
}
```

**L34 data-classification matrix entry:**

```yaml
# tenant.atelier.yaml::spec.data_classification.engine_compliance
hermes:
  locality: local
  network_egress: none
  # Hermes via Ollama localhost — qualifies for CONFIDENTIAL tasks.
  max_classification: CONFIDENTIAL
```

**L35 egress note:** `HermesEngine` connects to `localhost:11434` only. This is
loopback — not subject to L35 `allowed_hosts` (loopback is unconditionally
permitted by the egress gate). No `allowed_hosts` entry needed.

**Audit events (L16 hash chain):**

| Event | Fields |
|---|---|
| `hermes.engine_spawned` | `engine_id`, `model`, `persona`, `channel`, `chat_key`, `duration_ms`, `token_count` |
| `hermes.engine_cancelled` | same, plus `reason` |
| `hermes.ollama_unavailable` | `engine_id`, `base_url_hash` (never full URL), `error_class` |

`base_url_hash`: SHA-256 prefix (16 hex chars) of the base URL — never full
URL or path in audit details.

**Boot self-test addition:**

```python
# self_test.py: WARNING-level check — not CRITICAL (Ollama is optional).
# probe: GET http://localhost:11434/api/tags → rc==200 within 2 s.
# Reports "hermes_ollama_reachable: false" as WARNING, never blocks startup.
```

---

#### M2 — Native tool-calling (ChatML `<tool_call>` tokens)

Hermes-3 emits structured tool-calls in ChatML format:

```
<tool_call>{"name": "forge_exec", "arguments": {"tool_name": "code.csv_diff", ...}}</tool_call>
```

`HermesEngine.spawn()` in M2 intercepts these tokens, dispatches the matching
forge/skill-forge MCP tool via the existing MCP client, injects the result as
`<tool_response>` tokens, and continues streaming. This gives `mcp: True` in
capabilities.

The tool registry passed to Hermes is a restricted subset of the operator's
forge namespace: only tools whose `meta.hermes_allowed: true` policy flag is
set. Default: all tools denied. Operator opt-in per tool.

Capability delta from M1:

```python
capabilities = {
    ...
    "mcp": True,          # M2
    "tool_call_format": "hermes_chatml",  # M2 — for L29 capability gating
}
```

---

#### M3 — Full MCP bridge + skills injection

- `HermesEngine` spawns a lightweight MCP-over-stdio bridge process
  (`hermes_mcp_bridge.py`) that proxies the full forge/skill-forge surface.
- Active session skills (auto-injected at L7) are serialised into the Hermes
  system prompt as a `<delegated_skill>` block (same format as L30 for other
  engines).
- `mid_stream_inject` remains `False` (Ollama HTTP API has no mid-stream
  side-channel); the L13 `/btw` feature is a no-op for `hermes-worker` persona.
- Capability delta: `skills_tool: True` (via prompt injection, not native tool).

---

## Consequences

### Positive

- **GDPR Art. 5 / EU AI Act Art. 14:** Inference stays on the operator's host.
  No personal data sent to any external API for tasks routed to `hermes-worker`.
- **L34 CONFIDENTIAL class unlocked** for delegation tasks (the only engine
  that qualifies without a compliance-zone exception).
- **Cost:** zero inference cost per token once Ollama is installed.
- **Offline operation:** `HermesEngine` works without internet connectivity —
  useful for air-gapped deployments.
- **Apache-2.0 model license:** no legal ambiguity for commercial deployments;
  aligns with AtelierOS's own license (CLAUDE.md §Licensing).
- **Fallback for rate-limits:** when Anthropic/OpenAI rate-limits hit, tasks can
  be rerouted to `hermes-worker` automatically via the L5 auto-router
  (future: `ROUTER_FALLBACK_ENGINE=hermes`).

### Negative / Risks

- **No `mid_stream_inject`:** L13 `/btw` mid-stream injection is not supported.
  Operators must document this limitation for `hermes-worker` persona.
- **Quality ceiling:** Hermes models, even the 70 B variant, are below
  Claude Opus in reasoning depth. Not a drop-in replacement for
  `ClaudeCodeEngine` on complex coding tasks.
- **Ollama prerequisite:** operator must install and run Ollama separately.
  Boot self-test emits WARNING (not CRITICAL) if Ollama is unreachable —
  adapter starts normally, `hermes-worker` tasks fail gracefully with
  `hermes.ollama_unavailable` audit event.
- **GPU memory:** 70 B Q4 requires ~40 GB VRAM or CPU-offload. Operators
  without capable hardware must use smaller aliases.
- **M2 tool-call surface:** the per-tool `hermes_allowed` opt-in gate adds
  operator configuration burden. Default-deny is non-negotiable (L10 parity).
- **CI AST lint:** `hermes_engine.py` and all L38/L29/L34 parity — must not
  `import anthropic`. Covered by existing lint rule.

---

## Must NOT do

- Bypass L10 path-gate for Hermes-spawned tool calls.
- Route CONFIDENTIAL tasks to Hermes without verifying `locality: local` in
  the data-classification matrix (engine_compliance check is mandatory).
- Pass `base_url` as a full URL into any audit `details` field (hash-prefix only).
- Add `hermes_allowed: true` to any tool as a global default — opt-in per tool,
  operator decision.
- Make `hermes.ollama_unavailable` a CRITICAL self-test failure — Ollama is
  optional; adapter must boot without it.
- Call `cancel()` by deleting the httpx client object — set `_cancel_event` and
  let the stream reader exit cleanly.
- Use the `ollama run` CLI subprocess path (Variant B) — no shell-injection
  surface for model names or prompts.
- Hard-code `localhost` in `HermesEngine.__init__` — `base_url` must be
  configurable for docker-networking scenarios (Ollama in sidecar container).

---

## References

- NousResearch Hermes-3: https://huggingface.co/NousResearch/Hermes-3-Llama-3.1-8B
- Ollama API docs: https://github.com/ollama/ollama/blob/main/docs/api.md
- AtelierOS Layer 22 (WorkerEngine): `docs/claude-ref/layer-engines.md`
- AtelierOS Layer 29 (Delegation): `docs/claude-ref/layer-engines.md §Delegation`
- AtelierOS Layer 34 (Data Classification): `docs/claude-ref/layer-34-data-classification.md`
- AtelierOS Layer 35 (Egress Lockdown): `docs/claude-ref/layer-35-egress-lockdown.md`
- Apache-2.0 license compatibility: `docs/claude-ref/licensing.md`
