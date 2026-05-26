# ADR-0001 — AWP as Orchestration Layer, Backend-Agnostic Worker Engines

**Status:** Accepted
**Date:** 2026-05-09
**Deciders:** Silvio Jurk (Maintainer)
**Context:** claudeOS v0.10 → multi-backend agent OS

> **Naming note (added Phase 3 of the AtelierOS rebrand):** This ADR
> predates the project rename from `claudeOS` to `AtelierOS`. The body
> preserves the original term for historical accuracy — the decision
> records the moment when the project was still `claudeOS`. Today's
> canonical name is **AtelierOS**. See CLAUDE.md "Project rebrand"
> section for the strangler-fig migration plan.

## Context

claudeOS today is structurally bound to Anthropic Claude Code:

- `bridges/shared/adapter.py::call_claude_streaming` spawns `claude --input-format stream-json --output-format stream-json --verbose` directly.
- The voice-mode TTS pipeline depends on Claude Code's Stop-Hook.
- The path-gate sandbox enforcement depends on Claude Code's PreToolUse-Hook.
- Skills 2.0 are wired through Claude Code's `Skill` tool API.
- Mid-stream `/btw` injection writes to a Claude-Code-specific stdin pipe.

This is fine for the current product, but blocks three concrete classes of future demand:

1. **Enterprise compliance** — large customers want their data to stay on
   their own LLM backend (Azure OpenAI in EU, AWS Bedrock, on-prem Ollama,
   self-hosted vLLM). Today they cannot: claudeOS runs Claude Code or
   nothing.
2. **Multi-vendor failover** — when Anthropic has an incident, the entire
   claudeOS deployment is down. Production users want a fallback chain.
3. **Open-standard positioning** — claudeOS is currently *a tool* on top of
   *one CLI*. The opportunity is to be *a platform* implementing *an open
   spec*, with multiple compliant backends.

The maintainer also owns and maintains [AWP (Agent Workflow Protocol)](https://github.com/veegee82/agent-workflow-protocol),
a separate open-source project with a published 1.0 spec, conformance
suite, and PyPI package `awp-agents`. AWP already documents an
integration pattern for ClaudeClaw / openclaw under
`docs/openclaw_integration.md` — the same pattern applies here.

## Decision

claudeOS adopts **AWP as its orchestration layer**, with LLM backends
plugged in through a stable `WorkerEngine` protocol.

The architecture is intentionally **lopsided** (mirroring AWP's own
integration philosophy):

| Layer | Owner | Responsibility |
|---|---|---|
| **Body** | claudeOS | Bridges (Discord/WA/Telegram/Slack/Email), Voice/TTS, Personas, Sandbox (bwrap), Audit-Hash-Chain, Layer 1–21 (Roles/Quota/Consent/Disclosure/Proposal), MCP-Server (forge/skill-forge), Path-Gate |
| **Brain** | AWP (`awp-agents`, optional dep) | DAG-Engine + Delegation-Loop, 5D-Budget, 2-Tier-Validation, Manager-Intelligence, A3-Runtime-Tool-Creation, Worker-Routing |
| **Engines** | pluggable | Claude Code, Codex CLI, Gemini CLI, Anthropic SDK, Azure OpenAI Assistant, Bedrock, Vertex, Ollama, vLLM |

The architectural pattern is **Hexagonal Architecture** (Ports and
Adapters, Cockburn 2005): AWP defines the **Port** (`WorkerEngine`
Protocol), each backend is an **Adapter**.

## WorkerEngine Protocol (normative)

```python
class WorkerEngine(Protocol):
    name: str                       # canonical identifier ("claude_code", "codex_cli", ...)
    capabilities: dict[str, Any]    # capability flags, see below

    async def spawn(
        self,
        prompt: str,
        system: str | None = None,
        tools: list[ToolSpec] | None = None,
        mcp_config: dict[str, Any] | None = None,
        persona: str | None = None,
        working_dir: Path | None = None,
        add_dirs: list[Path] | None = None,
    ) -> AsyncIterator[StreamEvent]: ...

    async def inject(self, stream_id: str, text: str) -> bool: ...
    async def cancel(self, stream_id: str) -> None: ...
```

Capability flags (each engine declares what it supports):

| Flag | Semantics |
|---|---|
| `mid_stream_inject` | Engine supports `inject()` while a turn is streaming (Claude Code: yes, Codex: no) |
| `hooks` | Engine supports filesystem-based pre/post hooks (Claude Code: yes, others: no — emulate via MCP) |
| `skills_tool` | Engine has a first-class `Skill` tool (Claude Code: yes; others: emulate via system-prompt append) |
| `mcp` | Engine supports MCP servers natively (Claude Code: yes, Codex: yes, Gemini: limited) |
| `stream_json` | Engine emits structured JSONL events on stdout (Claude Code: yes, Codex: yes via `exec --json`) |
| `permission_modes` | Engine has multi-level permission modes (Claude Code: 4 modes; Codex: sandbox modes) |
| `add_system_prompt` | Engine supports `--append-system-prompt` or equivalent (Claude Code: yes, Codex: via prompt) |

Adapter logic gates features on capabilities — a missing capability is
**never a crash, always a degraded mode**.

## Phasing

| Phase | Scope | E2E gate |
|---|---|---|
| **0** | This ADR | doc review |
| **1** | `WorkerEngine` protocol + `ClaudeCodeEngine` (extract) + `CodexCliEngine` (new) + per-subtask E2E running real `claude` AND real `codex exec` | E2E green for both backends |
| **2** | `adapter.py` consumes engines through the protocol; existing behaviour preserved on Claude Code path | full `run-all-tests.sh` green |
| **3** | AWP integration: `awp-agents` as optional dep, `complex-task` classifier dispatches to AWP DAG/Delegation-Loop, AWP workers reach claudeOS Forge / SkillForge / Personas / MCP / Sandbox / Audit through existing surfaces | AWP conformance-suite green for claudeOS-backed runs |
| **4** | Multi-engine per worker: AWP YAML declares `engine: codex_cli` or `engine: claude_code` per worker; persona has `default_engine` field | E2E with at least 2 backends in one workflow |
| **5** | Enterprise compliance: workspace `engine_policy.json` with default + fallback chain + compliance zones (EU/US/self-hosted); audit chain tags `engine_id` + `compliance_zone` per event | E2E exercising fallback + zone routing |

Hard rule: **every phase ships with at least one per-subtask E2E that
exercises the new behaviour against real subprocesses**. Phase 1 in
particular runs both `claude` and `codex` as real binaries (no mocks
beyond what is needed to keep the test deterministic).

## Non-Goals

- **Replacing AWP with claudeOS-internal orchestration.** AWP is a
  separate, published spec with its own conformance suite. claudeOS
  consumes AWP, never re-implements it.
- **Re-writing claudeOS' security envelope.** Layers 1–21 stay
  unchanged; AWP workflows run *inside* the existing sandbox / audit
  chain / role / quota system, not around it.
- **Forcing AWP on every turn.** Simple single-worker tasks bypass AWP
  entirely — the cost contract is "AWP eskaliert nur bei
  classification == complex". Default ~30 % of bridge turns reach AWP;
  ~70 % stay on the legacy direct-spawn path.

## Risks

| Risk | Mitigation |
|---|---|
| AWP spec drifts | Pin `awp-agents>=1.0,<2.0`; run AWP conformance-suite in CI |
| AWP-Manager LLM calls increase per-turn cost | Eskalation only on `classification == complex`; default classifier is local heuristic |
| Engine adapters drift from upstream CLI changes | Per-engine smoke-test in CI; adapters expose `version` from CLI |
| Path-Gate / Hooks unavailable on Non-Claude engines | Implement Path-Gate-equivalent as MCP-Tool-Wrapper (Phase 1 inkludiert); Voice-TTS via stream-output-parser |
| Maintenance burden of 4+ engines | Don't ship a backend until there's a real E2E that runs against it; community can contribute the rest |

## Consequences

**Positive**

- claudeOS becomes vendor-agnostic at the orchestration layer.
- Enterprise customers can plug in their own LLM backend without
  forking claudeOS.
- AWP gets a flagship reference implementation; AWP's network effect
  benefits both projects.
- Marketing narrative shifts from "tool on top of Claude Code" to
  "open-spec agent OS".

**Negative**

- Refactor cost (Phases 1–2 are ~5 weeks of focused work).
- Capability-gating spreads conditional logic through the adapter
  (mitigated by central `WorkerEngine.capabilities` table).
- Test matrix grows: E2E now runs N×M (N engines × M test scenarios).
  Mitigated by per-engine smoke + a single shared scenario suite.

## How this connects to LDD

Each phase is one LDD outer-loop step (`∂L/∂method` — the *method*
being the orchestration architecture). Each phase's E2E is the inner-loop
loss signal. Phase 1 specifically converges when:

```
loss(phase1) = max(
    1 - claude_code_engine_e2e_pass_rate,
    1 - codex_cli_engine_e2e_pass_rate
)
```

reaches 0 (both engines green on the same E2E suite).

## References

- AWP Spec: <https://github.com/veegee82/agent-workflow-protocol>
- AWP `docs/openclaw_integration.md` — the integration pattern this ADR adapts
- `docs/architecture.md` (claudeOS) — current Body-only architecture
- `docs/layer-model.md` (claudeOS) — Layer 1–21 reference
- `CLAUDE.md` (claudeOS) — full per-layer documentation
- Cockburn, A. (2005). *Hexagonal Architecture*.
