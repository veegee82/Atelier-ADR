# ADR-0004 — Phase 5: Compliance Zones + Engine Policy

**Status:** Skeleton implemented; adapter wiring + outage fallback pending
**Date:** 2026-05-10
**Builds on:** ADR-0001 (AWP Adoption), ADR-0002 (Engine Migration), ADR-0003 (Phase 3 AWP Dispatch)

## Goal

Add **declarative engine routing with compliance zones** for enterprise
deployments. An operator drops a single `engine_policy.json` per workspace
and gets:

  * a default engine + ordered fallback chain
  * compliance zones (`personal_data`, `code_only`, `external_facing`,
    custom) with engine allowlists per zone
  * a per-task zone classifier (PII regex + persona-hints)
  * audit-chain events tagged with both `engine_id` and `compliance_zone`
    so a regulator can answer "which LLM saw what data" with one query

This is the **enterprise-adoption threshold** referenced in
`docs/for-organizations.md` § 7.4 — the moment a regulated customer can
sign without negotiating around the data path.

## Architecture

```
   incoming task ──► classify_zone(task, persona)
                          │
                          ▼
                    zone = "personal_data"
                          │
                          ▼
   engine_policy.allow_engines_for(zone) ──► ["azure_openai_eu", "vllm_eu_west"]
                          │
                          ▼
   for engine_id in allowed:
       engine = engine_registry.get_engine(engine_id)
       if engine and engine.healthy():
           audit("engine_id_selected", engine_id=..., zone=...)
           return engine
   audit("engine_policy_no_engine", zone=..., tried=[...])
   return None  # caller falls through to single-turn default
```

Three new modules + tests:

| Module | Responsibility |
|---|---|
| `bridges/shared/engine_policy.py` | Load `engine_policy.json`, validate schema, expose `allow_engines_for(zone)` and `default_chain()` |
| `bridges/shared/compliance_zone_classifier.py` | Heuristic classifier — PII regex (e-mail, phone, IBAN, credit-card, AHV/SSN/national-id), persona hints (`inbox` → `personal_data`), explicit user marker `[zone:foo]` |
| Adapter integration | `_resolve_engine_via_policy(prompt, profile)` returns engine_id; replaces direct `engine_registry.resolve_engine_id` for AWP-dispatched flows |

## Schema — `engine_policy.json`

```jsonc
{
  "default_engine": "claude_code",
  "fallback_chain": ["claude_code", "vllm_eu_west", "ollama_local"],
  "compliance_zones": {
    "personal_data": {
      "allow_engines": ["azure_openai_eu", "vllm_eu_west"],
      "deny_engines": ["claude_code"],
      "audit_severity": "WARNING"
    },
    "code_only": {
      "allow_engines": ["claude_code", "codex_cli"]
    },
    "external_facing": {
      "allow_engines": ["claude_code"],
      "deny_engines": ["vllm_eu_west"]
    }
  },
  "task_zone_classifier": "regex_pii"
}
```

Zone-resolution order: `deny` beats `allow` (a deny-listed engine never
fires even if it's also on the allow-list). Empty `allow_engines` means
"any engine that isn't explicitly denied" — useful for catch-all
zones.

## Sub-phases

K_MAX = 5 per sub-phase. Each ships with its own per-subtask E2E.

### **5a — `engine_policy.py` schema + loader + validator**

Pure-Python:

* `EnginePolicy.from_file(path)` — JSON load + schema validation
* `EnginePolicy.from_dict(data)` — for tests
* `default_chain() -> list[str]` — engine_ids in fallback order
* `allow_engines_for(zone) -> list[str]` — applies allow ∩ ¬deny
* `validate_against_registry(known_ids) -> list[str]` — returns
  warnings (typo'd engine_id in policy)

E2E: 12 cases — happy load, missing file, malformed JSON, unknown
zone, empty allow + non-empty deny, deny-beats-allow, validation
warnings.

### **5b — `compliance_zone_classifier.py`**

Pure-Python heuristic:

* PII regex bank (e-mail, +49/+1 phone, IBAN, 16-digit credit-card,
  SSN/AHV/national-id patterns by country)
* Persona-hint table (`inbox` → `personal_data`, `coder` →
  `code_only`)
* Explicit user marker recognition (`[zone:foo]` prefix)
* Returns `(zone, signals)` for audit attribution

E2E: 18 cases — PII variants per country, persona hints, marker
override, edge cases (PII in code comments, ambiguous prompts).

### **5c — Adapter wiring**

In `_try_awp_dispatch` (or before): call
`_resolve_engine_via_policy(prompt, profile)` if `engine_policy.json`
exists in `<atelier_home>/global/`. Falls through to
`engine_registry.resolve_engine_id` when no policy, no zone match,
or no engine in the allowed list is healthy.

Audit:

* `engine.policy_resolved` (INFO) — engine_id picked, zone, allowed list
* `engine.policy_no_engine` (WARNING) — zone matched, no allowed engine
  available; fell back to default
* `engine.zone_classified` (INFO) — zone + signals used for the pick

### **5d — Audit-tag wiring**

Every audit event involving an LLM call carries `engine_id` AND
`compliance_zone` fields. Backwards-compatible: existing events keep
their fields, new fields are optional (default empty string).

### **5e — Outage-fallback E2E**

Per-subtask E2E that:
1. registers a stub-engine that always raises
2. exercises the fallback chain
3. asserts the second engine in `default_chain` got called
4. asserts the audit event `engine.fallback_used` was emitted with
   the failed engine's id

## What ships in this PR (skeleton)

- `bridges/shared/engine_policy.py` — schema + loader + validator
- `bridges/shared/compliance_zone_classifier.py` — heuristic classifier
- `bridges/shared/test_engine_policy.py` — 12 cases
- `bridges/shared/test_compliance_zone_classifier.py` — 18 cases
- `run-all-tests.sh` — both wired in
- `docs/decisions/0004-phase5-compliance-zones.md` — this ADR

**Not in this PR (deliberately):**

- Adapter wiring (`_resolve_engine_via_policy` integration)
- Outage-fallback execution path
- AWP-side worker integration that propagates `compliance_zone` into
  AWP-dispatched workers (depends on AWP PR #6 being merged)
- Concrete engine adapters for `azure_openai_eu`, `vllm_eu_west`,
  `ollama_local` (Phase-6 scope — just `claude_code` + `codex_cli`
  in the registry today)

The skeleton is honest: importable, testable, plug-and-play for the
adapter wiring in the next session.

## How this connects to LDD

Phase 5 is the *outer-loop* step that closes the
"engine-must-honour-policy" loss term. The integrity rule from
ADR-0003 is reformulated:

> Every audit event with `engine_id` set has an enclosing `awp_task_id`
> *AND* a `compliance_zone` field that matches one of the engine's
> allowed zones in the active policy.

If the rule is violated, the policy was bypassed — that's a critical
audit event, not a minor drift.
