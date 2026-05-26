# ADR-0005 — AWP as Standards Only, Engines Own Execution

**Status:** Accepted
**Date:** 2026-05-10
**Supersedes:** ADR-0003 (Phase-3 AWP-runtime dispatch — rolled back)
**Companion to:** ADR-0001 (AWP adoption), ADR-0002 (Engine migration), ADR-0004 (Compliance zones — survives)

## Context

ADR-0003 wired AWP's Python runtime (`awp.AWPAgent.run`) into the bridge
adapter as an opt-in dispatch path for complex tasks. ADR-0001's
"hexagonal" promise was that AWP would orchestrate, engines would
execute. In practice — verified by reading
`reference/python/src/awp/runtime/llm.py` — AWP's runtime owns its own
provider-agnostic HTTP client (OpenAI / Ollama / OpenRouter / Groq /
Anthropic via proxy / vLLM / LiteLLM). When AWP-workers fire an LLM
call inside `AWPAgent.run`, those calls flow over AWP's HTTP path, NOT
through the AtelierOS engine layer. Phase 4 attempted to close this gap
by injecting a `worker_engine_factory` into AWP's state — but factory
adoption is a per-worker code change in the AWP repo, not something
AtelierOS can guarantee.

The maintainer's call (2026-05-10): the goal of AtelierOS' AWP
integration is **standards adoption for organisational peripherals**
— let large companies wire their tooling against AWP's declarative
schemas. Runtime execution belongs to the LLM-CLI engines (Claude
Code / Codex CLI / Gemini CLI / future Ollama / vLLM), full stop.

## Decision

**AWP is consumed by AtelierOS strictly as a protocol / declarative
standard layer, never as a runtime.** Concretely:

* AWP's YAML schemas (workflow, agent identity, capabilities, state,
  orchestration, output contract R17, 5D budget shape) are the
  declarative interface AtelierOS exposes for organisational
  peripherals.
* AWP's validation rules (R17, output contract, persona schema,
  reserved-key conventions) are honoured by AtelierOS-side validators.
* AWP's `AWPAgent.run` is **not** called from any AtelierOS code path.
  No `awp_runtime.dispatch()` wrapper. No adapter dispatch hook.
* All execution flows through `engine_registry.get_engine(engine_id)`
  → `WorkerEngine.spawn(...)`. The engine catalog (Claude Code / Codex
  CLI / future Gemini / Ollama) is the *only* execution path in
  AtelierOS.

## What survives, what's deleted

### Deleted on 2026-05-10

| Artefact | Reason |
|---|---|
| `bridges/shared/awp_runtime.py` | Wrapped `AWPAgent.run` — runtime path |
| `bridges/shared/test_awp_integration.py` | Tests the deleted runtime wrapper |
| `bridges/shared/test_adapter_awp_dispatch.py` | Tests the deleted dispatch helper |
| `adapter._try_awp_dispatch` (function) | Adapter-side dispatch into AWP runtime |
| `adapter` call site of `_try_awp_dispatch` | Replaced by direct `call_claude_streaming` |
| `run-all-tests.sh` lines for the two deleted suites | No suite, no entry |

### Survives

| Artefact | Why |
|---|---|
| `bridges/shared/engine_registry.py` | Engine catalog — pure standards, no AWP runtime |
| `bridges/shared/engine_policy.py` | Declarative policy schema — no AWP runtime |
| `bridges/shared/compliance_zone_classifier.py` | Heuristic classifier — no AWP runtime |
| `bridges/shared/task_complexity.py` | Heuristic classifier — no AWP runtime |
| `adapter._resolve_engine_via_policy` | Standards-side engine picker; ready for Phase-6 wiring into `call_claude_streaming` |
| `bridges/shared/test_engine_*` (3 suites) | Test the surviving standards modules |
| `docs/decisions/0004-phase5-compliance-zones.md` | Compliance-zone schema is a declarative standard |
| AWP-PR #6 in `veegee82/agent-workflow-protocol` | Spec + helper module live in the AWP repo, not in AtelierOS — useful for any future AWP consumer |
| `awp` Python package as an *importable peer* | Operators may still call AWP standalone (CLI, scripts); AtelierOS just doesn't call into it |

## Consequences

### Positive

* The architecture invariant from ADR-0001 ("Body / Brain / Engines")
  becomes structurally enforceable: there is no code path in AtelierOS
  that bypasses the engine layer. Audit-event ↔ engine_id ↔ awp_task_id
  integrity is automatic — `awp_task_id` events simply don't exist
  anymore in AtelierOS' chain.
* Organisational adoption story sharpens: "AtelierOS speaks AWP as a
  schema, not as a runtime — your tooling integrates against the same
  YAML spec the rest of the AWP ecosystem uses."
* No optional-dep contract to maintain — AtelierOS does not import
  `awp.*` anywhere. Bridges keep working without the `awp-agents`
  package installed at all.
* The hexagonal rejection of the per-call HTTP shortcut keeps Tool-Use,
  MCP, hooks, sandbox and audit symmetric across every execution.

### Negative

* Phase 3c, Phase 4 (AtelierOS-side wiring) and Phase 5 (adapter wiring
  of policy resolver) are the work-in-vain of 2026-05-10. The standards
  modules (engine_registry, engine_policy, compliance_zone_classifier,
  task_complexity) survive — about half the volume.
* AWP's manager-intelligence, delegation-loop, 5D-budget-enforcer,
  2-tier-validation are not consumed runtime-wise. To get those
  features, AtelierOS will need to re-implement them on top of the
  engine layer (Phase 6), or operators run AWP standalone (the
  one-time-CLI path documented in `docs/integration-paths.md`).
* The maintainer holds dual citizenship: AWP-side PRs (like
  `feat/atelier-worker-engine-factory`) remain useful for *other* AWP
  consumers but are not load-bearing for AtelierOS.

## Phase 6 (next session, optional)

AtelierOS-side DAG-Walker that reads AWP-style `workflow.awp.yaml` and
walks each node through `engine_registry.get_engine(node.engine_id) →
engine.spawn(...)`. This honours AWP's *declarative* contract while
keeping AWP's runtime out of the loop. Hybrid execution_kind
(`http` direct vs `engine` spawn) is the natural extension when AWP
declarative DAGs become the load-bearing input shape.

## Standards integration story (the marketing line)

> AtelierOS adopts the **Agent Workflow Protocol** as its declarative
> integration standard for orchestration. Organisational peripherals
> wire against AWP-spec schemas (`workflow.awp.yaml`, agent identity,
> capability declarations, state model, output contract R17, 5D
> budget). Execution is the responsibility of the engine layer
> (Claude Code / Codex CLI / Gemini CLI / Ollama / vLLM via the
> `WorkerEngine` protocol). AtelierOS never bypasses the engine layer
> for an LLM call — that integrity rule is structural, not policy.
