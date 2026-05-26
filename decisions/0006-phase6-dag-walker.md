# ADR-0006 — Phase 6: Standards-Only DAG Walker

**Status:** Accepted
**Date:** 2026-05-10
**Builds on:** ADR-0005 (AWP standards-only)

## Context

ADR-0005 established the standards-only line: AWP is consumed as a
declarative protocol, never as runtime. Phase 6 closes the loop by
adding the **engine-driven DAG executor** that reads AWP-style YAML
schemas and walks each node through `engine_registry.get_engine()` →
`engine.spawn(...)`. No code path imports `awp.runtime.*`.

## Decision

Three new modules in `bridges/shared/`:

| Module | Role |
|---|---|
| `awp_dag_parser.py` | Schema validator for AWP-style `workflow.awp.yaml` (or dict). Returns immutable `DAG` / `DAGNode` / `DAGBudget` dataclasses. Detects cycles at parse time. Topological-sort helper preserves source order on ties. |
| `awp_validator.py` | R17 output-contract check (`result[worker_id].confidence ∈ [0..1]`), 5D `BudgetTracker` with `charge()` / `validate_budget()`. Pure-Python re-implementation of AWP's spec rules — no AWP-runtime call. |
| `awp_walker.py` | Engine-driven executor. Reads a parsed `DAG`, walks nodes in topological order, calls `engine_factory(node.engine_id)` per node, validates output via R17, accumulates state + budget. Hybrid: nodes with `execution_kind: "http"` use an injected `http_caller` instead of an engine. Aborts walk on failure / R17 violation / budget breach. |

## Per-node execution contract

```python
# Engine path (default)
engine = engine_factory(node.engine_id)   # WorkerEngine instance
result = engine.spawn(prompt=substituted_prompt)   # returns R17 dict

# HTTP path (hybrid escape hatch from ADR-0005)
result = http_caller(prompt=..., model=node.model)
```

The walker never imports an HTTP client itself — the caller injects
`http_caller` if any node uses `execution_kind: "http"`. This keeps
the walker's dependency surface to standard library only (plus the
optional `pyyaml` for YAML files).

## State model

The walker maintains a flat dict that grows as nodes complete:

```
state = {
  ...initial state from DAG...
  "<node_a.output_key>": {a: {confidence, ...}},  # R17 result
  "<node_b.output_key>": {b: {confidence, ...}},
  "meta": {
    "workflow_id": "...",
    "dag_started":   <unix_ts>,
    "dag_completed": <unix_ts>,
    "<node_id>":  {elapsed_s, engine_id, execution_kind, ...},
    ...
  },
  "worker_engine_factory": <callable>,   # ADR-0005 carry-along
}
```

Prompt templates support shallow `{state.foo}` substitution — full
Jinja2 etc. is opt-in via wrapping the walker, intentionally kept
out of core to limit dependency surface.

## Standards-integrity rule (load-bearing)

The test suite **lints** all three modules for:

```
"from awp.runtime"   "import awp.runtime"
"from awp "          "import awp\n"
"awp.AWPAgent"
```

Any of these strings landing in `awp_dag_parser.py`,
`awp_validator.py` or `awp_walker.py` is a CI-fail. This is the
structural guarantee that ADR-0005's separation holds — not a
convention but a check.

## What ships in this PR

* `bridges/shared/awp_dag_parser.py` — schema parser + topo-sort
* `bridges/shared/awp_validator.py` — R17 + budget
* `bridges/shared/awp_walker.py` — engine-driven executor
* `bridges/shared/test_awp_walker.py` — 60+ assertions covering
  parser, validator, walker, R17 violations, budget breach,
  HTTP-hybrid path, audit emission, **standards integrity**
* `run-all-tests.sh` — Phase-6 suite wired in
* `docs/decisions/0006-phase6-dag-walker.md` — this ADR

## What's deliberately NOT in this PR

* **Adapter wiring**. `_resolve_engine_via_policy` exists in
  `adapter.py` but the walker is not yet hooked into `process_one`.
  Operators who want to test the walker do so via direct call from
  a forged tool / user script. Adapter hot-path wiring is Phase 6.5
  and needs its own soak.
* **YAML-file discovery convention**. Where workflows live (per-chat
  override, persona-pinned, repo-wide) is operator decision; the
  walker just parses what it's given.
* **Manager-driven dynamic DAG generation** (option B from the
  pre-Phase-6 discussion). The walker today consumes a *given* DAG
  — generating one from a free-form prompt via an LLM call is a
  Phase 6.5 add-on (one engine spawn, one DAG returned, walker
  resumes its standard path).

## Phase 6.5 roadmap (after soak)

1. Adapter hook: when `chat_profile.workflow_path` is set OR a
   `[workflow:<id>]` marker is in the prompt, parse + walk that
   DAG instead of `call_claude_streaming`.
2. Hybrid execution: per-node `execution_kind` heuristic
   classifier (already in `task_complexity.py` — extend to
   classify per-node, not whole-prompt).
3. Manager-driven DAG generation: one engine spawn produces a
   YAML-shaped DAG from the user's prompt; walker takes over.
4. Dynamic Engine-Outage-Fallback inside walker: when
   `engine_factory(node.engine_id)` returns None, walk the
   policy's allowed engines for the node's compliance zone.
   (Reuses Phase-5 `_resolve_engine_via_policy` per node.)

## Coverage / evidence

- 60+ test assertions across parser, validator, walker
- standards-integrity linter test catches accidental
  `awp.runtime` imports
- HTTP-hybrid path tested with injected fake `http_caller`
- Audit-event emission verified for `walker.node_complete`,
  `walker.node_failed`, `walker.r17_violation`,
  `walker.budget_exceeded`
