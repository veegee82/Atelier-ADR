# concept-os-completion.md — closing the OS gap, messenger-first

## Mental model

AtelierOS is currently an *agent runtime with OS-shaped abstractions*.
Five gaps separate it from a system that legitimately deserves the
"OS" label (see [os-analogy.md](os-analogy.md) for the existing
mapping). This doc is a roadmap to close those gaps **without
breaking the invariant that the messenger is the only UI**.

The five new layers (17–21) all observe the same constraints:

1. **Messenger-first.** Every operation is reachable via slash
   commands inside the existing bridges (WhatsApp / Telegram /
   Discord / Slack / Email). No web UI. No separate desktop app.
   No SSH-only paths. The CLI in `operator/cowork/bin/cowork`
   stays as a debug tool, not a primary interface.
2. **Single owner.** Layer 16's `read_only` role + `observer
   transcript` is the only audience split. No multi-user accounts.
3. **Layered + opt-in.** Each layer is independently disable-able
   via the existing Layer-14 LDD-toggle pattern (`/ldd-set
   layer_17_process_model off`).
4. **Audit-chain integrated.** Every new mutating operation chains
   into the existing SHA-256 audit log via
   `forge.security_events.write_event`.
5. **Path-gate respected.** No new direct-write surfaces. All state
   mutations route through MCP servers or signed side-channels.
6. **Backwards-compatible.** Existing chats keep working without
   touching any new command. Layers 17–21 are net additions.

```
Existing AtelierOS (Layers 1–16):
  bridges → adapter → claude subprocess → MCP tools → audit chain

New OS-completion layers:
  + Layer 17: process model — /ps, /kill, signals, process tree
  + Layer 18: inter-session pipes — composing agents
  + Layer 19: service manager — dependency-graph init, journals
  + Layer 20: context memory manager — token budgets, paging
  + Layer 21: federation — multi-host coordination (optional)
```

---

## Layer 17 — process model

> **Status: MVP slice landed.** `bridges/shared/process_table.py`
> implements the registry protocol (register / update / deregister /
> list / get / cleanup_terminated / format_ps_table). 17 E2E cases
> cover the full lifecycle including concurrent register from 4
> threads (lock validated). Adapter integration (calling `register`
> on subprocess spawn, `update` on tool transitions, `deregister` on
> exit), the `/ps` and `/kill` slash commands, and the signals
> system (PLAN / SUMMARIZE / CONTEXT_DROP / QUIET) are the
> follow-up slices and intentionally not yet wired — they would
> conflict with currently-uncommitted Layer-16 work in `adapter.py`
> and `daemons/`. See `test_process_table.py` for the contract
> guaranteed by the MVP.

### Goal

Make session lifecycle **visible** and **controllable** from the
messenger. Today the per-chat-lock + in-flight tracker already form a
scheduler; this layer surfaces it as first-class processes.

### Slash-command surface

| Command | Effect |
|---|---|
| `/ps` | Table: session_id, chat, persona, status, tokens, runtime, parent |
| `/ps -t` | Tree view (parent/child relationships) |
| `/ps -a` | Include zombies and terminated sessions |
| `/top` | Auto-refreshing single message (10s edit cadence where the bridge supports message edits; one-shot fallback otherwise) |
| `/kill <session_id>` | SIGTERM equivalent (graceful) |
| `/kill -9 <session_id>` | SIGKILL equivalent (force) |
| `/nice <session_id> <±N>` | Adjust priority (-20…+19, lower = higher prio) |
| `/sig <session_id> <name>` | Send custom signal: `PLAN`, `SUMMARIZE`, `CONTEXT_DROP`, `QUIET`, `RESUME` |

Custom signals are the load-bearing innovation — they let the
operator nudge a running session without canceling it. `PLAN`
switches the persona to plan-mode for the next turn; `SUMMARIZE`
asks for a mid-task summary; `CONTEXT_DROP` triggers paging
(see Layer 20); `QUIET` mutes outbound until `RESUME`.

### Data model

`<atelier_home>/run/sessions.jsonl` — one line per active session,
atomically appended on state changes:

```json
{
  "session_id": "s_abc12",
  "chat_key": "discord:1501540900529246251",
  "persona": "coder",
  "status": "running",
  "started_at": "2026-05-08T12:34:56Z",
  "tokens_total": 87340,
  "tokens_in_window": 12404,
  "parent_session": null,
  "nice": 0,
  "in_flight_tool": "Read",
  "last_activity": "2026-05-08T12:35:12Z"
}
```

Slash-commands read this file (mtime-cache); signals write side-channel
envelopes (`{_signal: "PLAN", target: "s_abc12"}`) routed by the
adapter to the matching subprocess via existing `_running_stdins`
infrastructure.

### Implementation hooks

- Builds on existing per-chat lock + `_in_flight` + side-channel
  envelope routing (`/btw`, `/cancel`)
- New module `bridges/shared/process_table.py` — registry write/read
- Slash-commands in `shared/js/in_chat_commands.js`
- Signals dispatched via `_signal_handler(session_id, name)` in adapter
- Audit events: `process.started`, `process.signaled`,
  `process.killed`, `process.exited`

### Per-subtask E2E

`shared/test_process_model.py` — spawn 3 fake sessions, verify `/ps`
table content, send signals, verify `/kill -9` SIGTERMs the right
PID, child sessions get reaped on parent kill.

### Effort estimate

Medium (~3 weeks). Highest hebel-per-investment of all five layers.

---

## Layer 18 — inter-session pipes

> **Status: Phase 1 + Phase 2 landed.**
> Phase 1: `bridges/shared/pipe_registry.py` implements all three
> pipe modes (named FIFO / anonymous / broadcast) plus subscribe /
> unsubscribe / list / remove. 17 E2E cases.
> Phase 2: `core/pipe/` MCP server exposes the registry
> as 9 MCP tools (`pipe_create`, `pipe_write`, `pipe_read`,
> `pipe_subscribe`, `pipe_unsubscribe`, `pipe_list`, `pipe_remove`,
> `pipe_get_meta`, `pipe_queue_depth`) over JSON-RPC 2.0 stdio.
> 10 E2E cases via real subprocess + real stdin/stdout pipe.
> Personas can opt into the MCP server via their JSON config (same
> pattern as forge / skill-forge).
> Phase 3 landed: adapter side-channel routing for pipe-readers
> and the `/pipe-*` slash commands — both touch in-progress M-files
> in adapter.py and in_chat_commands.js.

### Goal

Sessions can **compose**. Output of `research` becomes input of
`coder`. Unix-philosophy for agents.

### Slash-command surface

| Command | Effect |
|---|---|
| `/pipe-create <name>` | Create a named pipe |
| `/pipe-list` | List pipes (name, owner, write-count, read-count) |
| `/pipe-rm <name>` | Remove pipe |
| `/pipe-write <name>` | Write current session's last reply to pipe |
| `/pipe-read <name>` | Inject pipe content as next-turn message |
| `/pipe-stream <name>` | Subscribe; new writes appear as auto-injected messages |
| `/run-with <persona1>:"prompt1" \| <persona2>` | One-shot composite |

### Three pipe modes

| Mode | Lifetime | Storage |
|---|---|---|
| Anonymous | One-shot (`/run-with`) | In-memory, dropped after consumer reads |
| Named FIFO | Until `/pipe-rm` | `<scope>/pipes/<name>.jsonl` |
| Broadcast | Until last subscriber unsubscribes | Same as named FIFO + `subscribers.json` |

### MCP server

`core/pipe/` — new optional plugin:

- `mcp__atelier_pipe__pipe_write(name, payload)`
- `mcp__atelier_pipe__pipe_read(name)`
- `mcp__atelier_pipe__pipe_subscribe(name)`
- `mcp__atelier_pipe__pipe_list()`

Personas opt in via `pipes_enabled: true` in their JSON. The cowork
resolver auto-injects the MCP server config + tools, mirroring the
forge / skill-forge pattern.

### Worked example

```
User in #research-chat:
  /persona research
  find me the top 3 papers on memory paging in agents
Bot:
  [research output with citations]
User:
  /pipe-write inbox-to-coder
Bot:
  📤 written to pipe inbox-to-coder

User in #coder-chat:
  /pipe-read inbox-to-coder
  → implement an LRU paging strategy based on these
Bot:
  [reads research output as initial-message context, then implements]
```

### Audit events

`pipe.created`, `pipe.write`, `pipe.read`, `pipe.subscribed`,
`pipe.removed`. Each chains into the unified audit log.

### Implementation hooks

- New file-based protocol mirrors inbox/outbox pattern
- Adapter watches `<scope>/pipes/*.jsonl` mtime; new content for
  subscribed sessions triggers an `_pipe_msg` side-channel envelope
- Path-gate hook covers `<scope>/pipes/**` writes — only the MCP
  server may write

### Per-subtask E2E

`shared/test_pipes.py` — fake research session writes, fake coder
session reads, verify content arrives as next-turn input. Stream
test: third session subscribes, gets all writes within 100ms.

### Effort estimate

High (~5 weeks). Depends on robust Layer-17 process model first.

---

## Layer 19 — service manager

> **Status: Phase 1 + Phase 2 landed.**
> Phase 1: `core/init/init.py` implements ServiceDef /
> ServiceState, `discover_services`, `topological_order`, and the
> Supervisor with real-subprocess spawn, three backoff modes,
> max_restarts cap, hot_reload signal delivery, reverse-topo
> `shutdown_all`, and per-service journal capture. 15 E2E cases
> including real-subprocess restart loops with an injectable clock
> for deterministic backoff testing.
> Phase 2: `*.service.yaml` manifests written for the seven existing
> services (forge-mcp, skill-forge-mcp, voice-adapter, and four
> bridge daemons). Smoke-tested via `discover_services` +
> `topological_order` — dep graph valid, startup order is
> forge-mcp → skill-forge-mcp → voice-adapter → daemons in parallel.
> Phase 3 landed: bridge.sh migration to call into init.py
> (touches the bridge.sh M-file).

### Goal

Replace `bridge.sh up/down` with proper init: dependency graph,
restart policies, journal-style logs. Controllable via messenger.

### Slash-command surface

| Command | Effect |
|---|---|
| `/svc list` | All services + status (running / stopped / failed / restarting) |
| `/svc start <name>` | Start service |
| `/svc stop <name>` | Stop service |
| `/svc restart <name>` | Restart |
| `/svc enable <name>` | Auto-start on boot |
| `/svc disable <name>` | Don't auto-start |
| `/svc deps <name>` | Dependency tree (ASCII art for chat) |
| `/svc journal <name> [--tail N]` | Last N log lines |
| `/svc journal -f <name>` | Follow journal (auto-edit message) |
| `/svc reload <name>` | Config reload (sends SIGHUP if hot_reload set) |

### Service definition

`*.service.yaml` files inside each plugin:

```yaml
name: forge
type: oneshot           # or "daemon"
exec_start: python3 -m forge.mcp_server
restart: on-failure     # never / always / on-failure
restart_sec: 5
backoff: exponential    # exponential / linear / none
max_restarts: 5
requires:
  - audit
  - workspace
wants:
  - skill-forge
journal: "<atelier_home>/run/log/forge.log"
hot_reload: SIGHUP
```

### Implementation hooks

- New plugin `core/init/`:
  - `init.py` — reads `*.service.yaml`, builds dep graph, topological
    sort, supervises children
  - `journal.py` — structured log capture with rotation
  - `mcp_server.py` — `svc_start`, `svc_stop`, `svc_status`,
    `svc_journal_tail`
- Slash-commands talk to init via Unix socket at
  `<atelier_home>/run/init.sock`
- `bridge.sh` stays as compatibility shim, calls into init under
  the hood — operators don't need to change muscle memory

### Audit events

`svc.started`, `svc.stopped`, `svc.crashed`, `svc.restart_attempted`,
`svc.config_reloaded`.

### Migration path

1. Add `*.service.yaml` to each existing plugin (forge, skill-forge,
   bridges)
2. Init plugin learns to read them
3. `bridge.sh up` switches from `parallel ssh-style` to
   `init.py start --all`
4. Slash-commands land
5. Old `bridge.sh` direct-calls deprecated, old direct-systemctl
   patterns documented as "legacy escape hatch"

### Per-subtask E2E

`shared/test_init.py` — yaml fixture defines 3 services with
A→B→C dependency, verify topo-sort startup order, simulate B crash,
verify restart, verify cascade-stop on A failure.

### Effort estimate

Medium (~4 weeks). Foundation for everything else, so worth doing
early after Layer 17.

---

## Layer 20 — context memory manager

> **Status: Phase 1 + Phase 2 landed.**
> Phase 1: `bridges/shared/context_budget.py` implements the
> bookkeeping API with the ok-warn-oom action ladder and three
> policies (evict / compress / reject). 18 E2E cases.
> Phase 2: `bridges/shared/context_cold_storage.py` adds the
> cold-storage tier with a pluggable EmbeddingProvider Protocol —
> the MVP ships a deterministic HashEmbeddingProvider that lets the
> machinery run end-to-end without an API call; production swaps in
> OpenAI's `text-embedding-3-small` or a local sentence-transformer
> without code changes. 20 E2E cases including 4-thread concurrent
> page_out, cosine-similarity ranking, cross-provider safety, and
> identical-text retrieval at similarity ≈ 1.0.
> Phase 3 landed: adapter pre-flight gate integration (calling
> `check_budget` before subprocess spawn, paging out evicted turns
> to cold storage automatically). Touches adapter.py M-file.

### Goal

Treat the LLM context window as **managed resource** with quotas,
eviction policies, and observability. Today's harness compresses
silently when context bloats; this layer makes that explicit and
operator-controllable.

### Slash-command surface

| Command | Effect |
|---|---|
| `/budget show` | Current usage per session, quotas, OOM policy |
| `/budget set <session\|chat> <tokens>` | Set quota |
| `/budget global <tokens>` | System-wide limit |
| `/working-set <session>` | Show what's active vs paged out |
| `/page-out <session> [--older-than Nh]` | Force compaction |
| `/page-in <session> <vector_id>` | Restore paged content |
| `/oom-policy <evict\|compress\|reject>` | Global strategy |
| `/oom-policy --persona <name> <strategy>` | Per-persona override |

### Mechanisms

| Component | What it does |
|---|---|
| Token counter | Wraps `claude` subprocess invocation, tallies per-session usage |
| Pre-flight gate | Before subprocess spawn: check budget, choose action |
| Working-set tracker | Recent N turns + active skills + active todos = "hot" |
| Cold storage | Vector store (text-embedding-3-small) for paged-out turns |
| Eviction | LRU on inactive turns when working set exceeds budget |

### OOM policies

| Policy | Behavior |
|---|---|
| `evict` | Drop oldest turns silently (fastest, lossy) |
| `compress` | Summarize old turns into one compact block (default) |
| `reject` | Refuse new turn until operator intervenes |

### Audit events

`context.budget_exceeded`, `context.compressed`, `context.evicted`,
`context.paged_in`, `context.oom_rejected`.

### UI in chat

```
User: /budget show
Bot:
  session s_abc12 (#research):     43k / 100k tokens (43%) ✓
  session s_def34 (#coder):        87k / 100k tokens (87%) ⚠️
  global:                          238k / 500k tokens (48%)
  
  next OOM action: compress
  per-persona overrides:
    - research: evict (older than 12h)
```

### Implementation hooks

- New module `bridges/shared/context_budget.py`
- Token counter via `tiktoken` library (matches Anthropic's tokenizer
  closely enough for budgeting)
- Cold storage via embeddings — uses existing `OPENAI_API_KEY` if set,
  falls back to local sentence-transformers
- Pre-flight gate is the only mutating point; runs before the
  per-chat lock so a budget-rejected message never starts the queue

### Per-subtask E2E

`shared/test_context_budget.py` — seed a session with 80% budget,
verify `/budget show` accuracy, fake a turn that pushes to 110%,
verify chosen OOM policy fires, verify audit event lands.

### Effort estimate

High (~6 weeks). Cross-cutting; needs careful UX design for OOM
behaviors. Genuinely valuable because context is the only scarce
resource an agent has.

---

## Layer 21 — federation (optional)

### Goal

Multiple AtelierOS instances coordinate. Skills, tools, and audit
shared across hosts. Turns single-machine AtelierOS into distributed
agent runtime.

### Slash-command surface

| Command | Effect |
|---|---|
| `/host list` | Known peers + status (online / offline / degraded) |
| `/host add <name> <url>` | Add peer (interactive PIN auth via /auth-up) |
| `/host remove <name>` | Remove peer |
| `/host status <name>` | Health, latency, audit-chain head, last-sync |
| `/mount <peer>:<scope>` | Mount foreign workspace read-only |
| `/umount <local-name>` | Unmount |
| `/sync-skills <peer>` | Pull skill registry; linter re-runs locally |
| `/audit-cross-verify <peer>` | Verify peer's chain head matches our cross-signed entry |

### Wire protocol

- mTLS over WireGuard tunnel (or SSH ProxyCommand for
  zero-additional-infra setups)
- JSON-RPC for control plane, NDJSON streaming for log/journal sync
- Shared CA cert distributed during initial pairing
- Per-host client cert pinned in `<atelier_home>/global/auth/peers/<name>/cert.pem`

### Federated audit

Each host has its own chain. Cross-host events (mount, sync, pipe-
across-hosts) **double-chain** — written into both hosts' chains,
each entry's payload includes the other host's current head hash.
Daily reconciliation job verifies cross-signatures match.

### Mount semantics

- Foreign skills/tools appear in `mounted/<peer>/` virtual scope
- Read-only by default
- Local linter re-checks every mounted skill (defense in depth — a
  compromised peer can't push a poisoned skill into your chain)
- Operator can promote a mounted skill to local user-scope via
  `/skills promote-from-mount <peer>:<name>`

### Operator-elevation surface

PIN-elevation (Layer 16) gates `/host add`, `/mount`, and
`/sync-skills` — adding a peer is a high-consequence operation.

### Implementation hooks

- New plugin `plugins/atelier-federate/`
- Each host's MCP servers expose `peer_*` tools to authenticated peers
- Adapter learns to recognize "foreign session_id" envelopes and
  route them via the federation tunnel
- New slash-commands

### Audit events

`peer.added`, `peer.removed`, `peer.contacted`, `peer.sync_started`,
`peer.sync_completed`, `peer.audit_cross_verified`,
`peer.audit_mismatch`, `mount.created`, `mount.removed`.

### Per-subtask E2E

`shared/test_federation.py` — spawn two AtelierOS instances on
loopback (different `CLAUDEOS_HOME`), pair them via PIN, mount peer's
skill scope, verify a skill from peer A is invokable on host B,
inject a chain-mismatch and verify cross-verify catches it.

### Effort estimate

Very high (~8 weeks). Only worth doing if multi-host demand is
real. For most users, the four prior layers are the entire OS-
completion they need.

---

## Phase 3 — bridge integration (landed)

After the M-files for Layer-16-Phase-2 (observer transcript) committed
(`db2ff9f`), the runway was clear for Phase 3 — bringing all four new
layers into the live bridge. What landed:

**Adapter-side hooks in `call_claude_streaming`:**
- `process_table.register_session(session_id, chat_key, persona)` right
  after `_register_subproc` / `_register_stdin`
- `process_table.update_session(session_id, in_flight_tool=...)` on
  every tool_use event
- `process_table.deregister_session(session_id, exit_reason=...,
  keep=True)` in the `finally` block (records stay for `/ps -a`
  forensics; periodic `cleanup_terminated` sweep handles TTL)
- All hooks are best-effort (`try/except` swallow + log) so a registry
  hiccup never breaks the streaming loop. `_process_table is None`
  fallback for environments without the module.

**Slash-command surface in `in_chat_commands.js`:**
- `/ps` and `/ps -a` → list active / all-including-terminated sessions
- `/pipe <list|create|write|read|rm|meta>` → all pipe lifecycle
- `/svc <list|deps>` → discover services from `*.service.yaml` and
  show dep graph
- `/budget <show|policy>` → context-budget bookkeeping

All four route through the unified `phase3_cli.py` wrapper (mirroring
the existing `schedule_cli` / `profile_cli` / `vault_cli` pattern).
Output wrapped in fenced code blocks so chat clients render the
fixed-width tables aligned.

**Test coverage added in Phase 3:**
- `js/test_phase3_dispatch.js` — 31 cases against the real CLI
  subprocess (no mocks): /ps empty/all, /pipe full lifecycle, /svc
  list discovers all 7 manifests + deps for voice-adapter, /budget
  show empty + default sub
- Existing 13 test suites all stay green (97 module cases + 5
  adapter tests + JS tests + MCP server test)

**What stays Phase-4 (deferred):**
- `/kill <session_id>` and `/sig <session> <SIGNAL>` — need
  daemon-side signal-routing for PLAN/SUMMARIZE/CONTEXT_DROP/QUIET
- `/svc start/stop/restart/journal` — needs init.py running as a
  supervisor under bridge.sh, not just CLI manifest read
- Layer-20 adapter pre-flight budget gate that auto-evicts/compresses
  before each turn — currently the budget is updated at tool boundaries
  but isn't acting as a gate yet
- Bridge.sh full migration to call into init.py for service supervision

The Phase-3 work that landed gives the user *visibility* into all
four new subsystems from the messenger; Phase-4 (when it lands) adds
*active control*.

## Phasing

A phased rollout with hard gates:

| Phase | Scope | Duration | Gate to next phase |
|---|---|---|---|
| **1** | Layer 17 (process model + signals) | 3 weeks | `/ps` and `/kill` work in production for ≥1 week, no regressions in existing turn flow |
| **2** | Layer 19 (service manager) | 4 weeks | All existing services migrated to `*.service.yaml`, `bridge.sh` becomes compatibility shim |
| **3** | Layer 18 (inter-session pipes) | 5 weeks | Anonymous pipes work end-to-end, named pipes pass audit-chain integrity |
| **4** | Layer 20 (context budget) | 6 weeks | OOM policies fire correctly, eviction is reversible (page-in works) |
| **5** | Layer 21 (federation) | 8 weeks | OPTIONAL — only if multi-host need is established |

Phases 1–4 = ~18 weeks for a single committed engineer, with
realistic LDD discipline (per-subtask E2E rule, docs-as-DoD,
loss-curve tracking).

## What we are NOT building

To keep scope honest:

- **Multi-user accounts.** Layer 16's `read_only` is the right
  audience split. Real user-accounts contradict the single-owner
  design intent.
- **Eigene Hardware-Abstraktion.** Linux/macOS already provide it;
  re-abstracting would add complexity without value.
- **Web UI.** Messenger-first is the load-bearing constraint. A web
  UI splits the user's attention surface and breaks the audit-chain
  uniformity (every interaction goes through one bridge today).
- **Custom kernel.** The LLM stays the kernel. We build userspace
  around it.

## What this concept does to AtelierOS's identity

Currently AtelierOS is described as "agent runtime with OS-shaped
abstractions." Completing layers 17–20 closes most of the gap:

| OS concept | Today | After Layers 17–20 |
|---|---|---|
| Processes | hidden lock | visible `/ps`, `/kill`, signals |
| Scheduling | per-chat sequential | priority-aware (`/nice`) |
| IPC | bridges only | inter-session pipes |
| Memory mgmt | implicit harness compression | explicit budget + eviction |
| Init system | `bridge.sh` | dependency-graph supervisor |
| Filesystem | scope ladder | scope ladder (already OS-shaped) |
| Drivers | bridges (already) | bridges (already) |
| Shell | slash commands (already) | slash commands + pipes |
| Syscalls | MCP tools (already) | MCP tools (already) |
| Audit | hash chain (already) | hash chain (already) |
| Kernel | LLM (not owned) | LLM (still not owned) |

After this work, calling AtelierOS "an OS for agents" becomes
**accurate, not aspirational**. The remaining gap (kernel ownership)
is structural — and arguably the right architecture, because the LLM
*should* be a black-box dependency, not a forked subsystem.

## Cross-references

- [os-analogy.md](os-analogy.md) — current OS mapping (pre-completion)
- [layer-model.md](layer-model.md) — existing 16 layers
- [security.md](security.md) — six surfaces, audit chain (load-bearing
  for every new layer)
- [forge.md](forge.md) / [skills.md](skills.md) — generation
  primitives the new layers build on
