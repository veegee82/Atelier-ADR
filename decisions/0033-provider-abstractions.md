# ADR-0033 — Provider Abstractions: Notification, Recall, Summary, Router

**Status:** Proposed  
**Date:** 2026-05-17  
**Author:** Maintainer  
**Depends on:** ADR-0030 (plugin system), ADR-0016 (L28 recall/user-model),
ADR-0007 (multi-tenant), ADR-0011 (workflow plugins)  
**Extends:** ADR-0030 §"Extension points" table — adds four new `plugin_type` values  
**Governs:** `NotificationBackend`, `RecallBackend`, `SummaryProvider`,
`RouterBackend` protocols and their default implementations  

---

## Context

### The gap ADR-0030 left

ADR-0030 introduced six layer-scoped extension points
(`compute_engine`, `worker_engine`, `bridge_channel`, `stt_provider`,
`data_connector`, `audit_backend`) and a unified `AtelierPlugin` lifecycle
contract. Four subsystems in the existing codebase still bypass this pattern
and call their implementation directly with no pluggable indirection:

| Subsystem | Current coupling | Missing protocol |
|---|---|---|
| Notification dispatch | Hardcoded in each bridge daemon (`discord.js`, `telegram.js`, …) — alerts go to the same channel that issued the request | `NotificationBackend` |
| Conversation recall / memory storage | SQLite FTS5 at a fixed path in `recall.py`; `user_model.py` writes JSON next to it | `RecallBackend` |
| TTS summarization | `summarize.py` calls `claude -p` subprocess; model and endpoint hardcoded | `SummaryProvider` |
| Persona routing | `router.py` tries four backends in a fixed chain (fake → heuristic → OpenAI embeddings → Anthropic SDK → CLI); chain is not externally swappable | `RouterBackend` |

### Why this matters for operators

A firm that runs AtelierOS in a regulated environment needs to:

- Route alerts to PagerDuty / OpsGenie / Jira Webhooks instead of a chat channel.
- Store conversation history in Postgres (audit-required), Pinecone (semantic search),
  or Chroma (on-prem), rather than SQLite files.
- Run summarization on an on-premises model (EU AI Act Art. 14 zone gate) instead of
  calling Anthropic's API.
- Inject custom routing logic (department-specific persona keywords, LDAP-group-based
  persona selection) without patching `router.py`.

None of these is possible today without forking core code. ADR-0030 defines the
general lifecycle contract; this ADR defines the four missing protocols and how
existing code becomes their default implementations.

---

## Decision

Add **four new plugin types** to ADR-0030's extension point table:

| `plugin_type` | Layer | Protocol | Default implementation |
|---|---|---|---|
| `notification_backend` | L3+ | `NotificationBackend` | `LogNotificationBackend` (stdout + audit chain) |
| `recall_backend` | L28 | `RecallBackend` | `SqliteRecallBackend` (current `recall.py` behavior) |
| `summary_provider` | L11 | `SummaryProvider` | `ClaudeCliSummaryProvider` (current `summarize.py` behavior) |
| `router_backend` | L5 | `RouterBackend` | `ChainRouterBackend` (current `router.py` chain) |

Each protocol lives in `atelier_plugins/providers/<type>.py`. Each protocol
is `@runtime_checkable` and follows the structural subtyping convention already
used by `AtelierPlugin`. Default implementations are the authoritative wrappers
of current behavior — they are **not** copies; they delegate to the existing
modules.

`PluginContext` gains four optional handles (`notification_registry`,
`recall_registry`, `summary_registry`, `router_registry`) so a plugin's
`on_load()` can self-register the same way `compute_engine` plugins do.

---

## Protocols

### `NotificationBackend`

```python
@runtime_checkable
class NotificationBackend(Protocol):
    """Deliver an event notification to an external system (ADR-0033).

    Called by the bridge adapter and daemons when they need to emit an
    alert that is NOT part of the normal reply flow — e.g. a rate-limit
    warning, a health-check failure, or a per-subtask completion signal.

    Implementations MUST be non-blocking (fire-and-forget or async-queued).
    MUST NOT block the calling thread for more than 100 ms.
    MUST NOT put message content or PII in the payload — metadata only
    (event name, severity, tenant_id, timestamp).
    """

    def notify(
        self,
        event: str,          # dot-separated slug: "adapter.rate_limited"
        payload: dict,       # metadata only — no PII, no message content
        *,
        tenant_id: str = "_default",
        severity: str = "info",   # "info" | "warn" | "error" | "critical"
    ) -> None: ...
```

**Default:** `LogNotificationBackend` — writes to `logging.getLogger("atelier.notify")`
and emits a `notification.sent` audit event. Used when no plugin is registered.

---

### `RecallBackend`

```python
@runtime_checkable
class RecallBackend(Protocol):
    """Store and retrieve conversation turns for recall and user modeling (ADR-0033).

    Text passed to index_turn() MUST already be PII-redacted by the caller
    (same contract as current recall.py).  The backend receives clean text.
    Implementations MUST NOT re-introduce PII into any stored representation.

    search() returns a list of turn dicts; shape:
        {"channel": str, "chat_key": str, "text": str, "ts": float}
    forget() returns the number of rows deleted (for audit purposes).
    """

    def index_turn(
        self,
        channel: str,
        chat_key: str,
        text: str,           # PII-redacted by caller
        *,
        tenant_id: str = "_default",
    ) -> None: ...

    def search(
        self,
        query: str,
        *,
        channel: str = "",
        limit: int = 10,
        tenant_id: str = "_default",
    ) -> list[dict]: ...

    def forget(
        self,
        channel: str,
        chat_key: str,
        *,
        tenant_id: str = "_default",
    ) -> int: ...
```

**Default:** `SqliteRecallBackend` — thin adapter over the current `recall.py`
module (`index_turn`, `search`, `forget` map 1-to-1).

---

### `SummaryProvider`

```python
@runtime_checkable
class SummaryProvider(Protocol):
    """Summarize a long assistant reply into a TTS-friendly spoken form (ADR-0033).

    The returned string MUST satisfy the faithfulness contract:
    - No new facts invented beyond what text contains.
    - All main points present.
    - No Markdown, code tokens, or shell paths in the output.

    Implementations MUST NOT raise — return a truncated fallback instead
    (same contract as the current naive-summary fallback in summarize.py).
    MUST NOT import the anthropic SDK directly.
    """

    def summarize(
        self,
        text: str,
        *,
        lang: str = "de",
        max_chars: int = 400,
        tenant_id: str = "_default",
    ) -> str: ...
```

**Default:** `ClaudeCliSummaryProvider` — thin adapter over the current
`summarize.py` module invoked via subprocess (`claude -p`), with the same
naive-truncation fallback when the CLI is unavailable.

---

### `RouterBackend`

```python
@runtime_checkable
class RouterBackend(Protocol):
    """Select a persona for an incoming user message (ADR-0033).

    Returns a dict on confident match:
        {"persona": "<name>", "confidence": <0.0-1.0>, "why": "<reason>"}
    Returns None on no-match or any error — caller falls back to its default
    persona.  Implementations MUST NOT raise.
    """

    def route(
        self,
        text: str,
        personas: list[dict],   # list of persona dicts with "name", "description",
                                 # optional "routing_anchors", "routing_exclude"
        *,
        min_confidence: float = 0.5,
        tenant_id: str = "_default",
    ) -> dict | None: ...
```

**Default:** `ChainRouterBackend` — thin adapter over the current `router.py`
module (fake → heuristic → embeddings → Anthropic SDK → CLI fallback chain).

---

## Provider registries

Each provider type has a lightweight per-process registry in its module:

```python
# providers/summary_provider.py (representative)

class SummaryProviderRegistry:
    """Holds the active SummaryProvider for the process.

    Only one provider is active at a time. The last call to set_active()
    wins. Thread-safe.
    """
    def set_active(self, provider: SummaryProvider) -> None: ...
    def get_active(self) -> SummaryProvider: ...  # returns default if none set

_registry: SummaryProviderRegistry = SummaryProviderRegistry()

def get_active() -> SummaryProvider:
    return _registry.get_active()

def set_active(provider: SummaryProvider) -> None:
    _registry.set_active(provider)
```

The `PluginContext.summary_registry` handle points at this module-level
`_registry` instance, so a plugin's `on_load()` can call
`ctx.summary_registry.set_active(self)` — exactly the same pattern as
`ctx.compute_registry.register(self)` in ADR-0030.

---

## Integration path

### Existing code: no breaking change in Phase 1

Phase 1 (this ADR) introduces the protocols and default implementations.
Existing callers (`summarize.py`, `router.py`, bridge daemons, `recall.py`)
are **not modified** yet. The default implementations delegate to them
unchanged. This means operators can install a custom plugin and it takes
effect; operators who do nothing see identical behavior.

### Phase 2 (follow-on): thin adapter wiring

A follow-on commit updates each existing caller to go through the registry:

| File | Change |
|---|---|
| `summarize.py` | Replace direct `claude -p` call with `summary_provider.get_active().summarize(...)` |
| `router.py` | Replace `route()` body with `router_backend.get_active().route(...)` |
| Bridge daemons | Replace hardcoded `emitToChat()` notification with `notification_backend.get_active().notify(...)` |
| `recall.py` | Expose `index_turn`, `search`, `forget` as `recall_backend.get_active().*` |

Phase 2 is a pure refactor (no behavior change for default configurations)
and is covered by the existing test suites (`test_summarize.py`,
`test_router.py`, bridge boot tests).

---

## `PluginContext` extension

```python
@dataclass
class PluginContext:
    # ... existing fields unchanged ...
    compute_registry:       Any | None = None
    engine_factory:         Any | None = None
    channel_registry:       Any | None = None
    # — ADR-0033 additions —
    notification_registry:  Any | None = None  # providers.notification_backend._registry
    recall_registry:        Any | None = None  # providers.recall_backend._registry
    summary_registry:       Any | None = None  # providers.summary_provider._registry
    router_registry:        Any | None = None  # providers.router_backend._registry
    extra: dict = field(default_factory=dict)
```

Existing plugins that do not use these handles are unaffected (fields default
to `None`; `on_load()` is free to ignore them).

---

## Tenant configuration

```yaml
# tenant.atelier.yaml — example with custom summary provider

spec:
  plugins:
    installed:
      - id: "com.acme.summary-llama"
        class_path: "acme_atelier_plugins.llama_summary:LlamaSummaryProvider"
        config:
          endpoint: "https://llama.internal/v1/chat"
          model: "llama-3-70b-instruct"
          # API key read from vault; never in config block
```

The loader wires `ctx.summary_registry` when it discovers a
`summary_provider` plugin, so the plugin's `on_load()` can call
`ctx.summary_registry.set_active(self)`.

---

## Must NOT

- **Must NOT** put message content, transcript text, or PII in a notification
  `payload` — metadata only (event name, severity, tenant_id, timestamp,
  error class name without stack trace).
- **Must NOT** let a `RecallBackend` implementation store raw text before
  it has been PII-redacted by the caller — the redaction contract is on the
  call site, not the backend.
- **Must NOT** let a `SummaryProvider` implementation import the `anthropic`
  SDK directly — use subprocess (`claude -p`) or an OpenAI-compat endpoint.
- **Must NOT** let a `RouterBackend` raise — return `None` on any error so
  the caller's fallback persona fires.
- **Must NOT** cache `get_active()` results across calls — the registry may
  be updated by a hot-reload (tenant plugin swap at runtime).
- **Must NOT** add a "trusted provider" bypass that skips the `AtelierPlugin`
  lifecycle (`on_load` / `on_unload` / `health_check` must always run).
- **Must NOT** auto-install a provider from bridge or adapter boot code —
  providers are installed via the plugin loader only.
- **Must NOT** expose `set_active()` as an MCP tool or bridge command — it is
  an operator-level operation gated behind `on_load()`.

---

## Implementation checklist

- [x] `atelier_plugins/providers/__init__.py`
- [x] `atelier_plugins/providers/notification_backend.py`
- [x] `atelier_plugins/providers/recall_backend.py`
- [x] `atelier_plugins/providers/summary_provider.py`
- [x] `atelier_plugins/providers/router_backend.py`
- [x] `atelier_plugins/protocol.py` — add 4 protocols + 4 `PluginContext` fields + `KNOWN_PLUGIN_TYPES`
- [x] `templates/notification_backend_plugin.py`
- [x] `templates/recall_backend_plugin.py`
- [x] `templates/summary_provider_plugin.py`
- [x] `templates/router_backend_plugin.py`
- [x] `tests/test_plugin_system.py` — extend with new-type coverage (55 tests, all green)
- [x] Phase 2 wiring: `adapter.py` — router, recall, summary, notifications wired through provider registries

---

## Open questions

1. **Multi-provider fan-out for `NotificationBackend`:** should the registry
   support multiple active backends (fan-out to PagerDuty AND Slack), or is
   one active provider sufficient? Current decision: single-active; fan-out
   can be implemented inside a custom plugin.

2. **`RecallBackend` migrations:** if an operator switches from SQLite to
   Postgres, existing recall data is not migrated automatically. Accept as
   limitation for v1; document migration path.

3. **`RouterBackend` confidence calibration:** custom backends may use
   different confidence scales. Accept `None` return as "no match" regardless
   of internal confidence; `min_confidence` is advisory.
