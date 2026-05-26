# ADR-0039 — Workflow Builder UI

**Status:** Accepted (2026-05-18) — Phases 1-7 implemented
**Plugin:** `core/console/atelier_console/web-next/` (new pages) + `core/console/atelier_console/routes/workflows.py` (new backend)
**Sister-ADRs:** ADR-0001 (AWP), ADR-0006 (DAG walker), ADR-0011 (workflow triggers), ADR-0032 (AWPKG), ADR-0037 (console relaunch)

---

## Context

AtelierOS ships a complete AWP execution stack — `awp_dag_parser.py`,
`awp_validator.py`, `awp_walker.py`, `atelier_workflows/runner.py` — plus the
AWPKG packaging format (ADR-0032) and a partially-wired scheduler daemon
(`atelier-scheduler@<tenant_id>.service`). All of this is accessible only by
editing YAML files on disk or invoking CLI commands. No operator-facing UI
exists for authoring, testing, scheduling, or monitoring AWP workflows.

The operator goal is for end-users to describe workflows in natural language
(text or voice), have Claude Code generate valid AWP YAML, visualise the
resulting DAG, run it live with step-by-step feedback, attach cron schedules,
and package/share the result — all from the console web UI without touching
the command line.

Three concrete gaps motivate this ADR:

1. **No authoring surface.** Creating a workflow requires knowing AWP YAML
   syntax and editing files on disk. The target audience is operators and
   power-users, not developers.

2. **No live feedback.** The walker executes synchronously; there is no UI
   that shows which node is running, what its output was, or where it failed.

3. **Scheduling is not wired.** `atelier-scheduler` is implemented but not
   connected to any UI — operators cannot create, inspect, or disable cron
   triggers without editing service files.

---

## Decision

Add a **Workflow Builder** section to the console web UI. It is a visual
authoring and operations surface built on top of the existing AWP execution
stack — no new runtime is introduced. The UI is the only new layer.

### Route Structure

```
/app/workflows                    Workflow library (list + import)
/app/workflows/:wid               Workflow editor (canvas + chat + inspector)
/app/workflows/:wid/runs          Run history
/app/workflows/:wid/runs/:rid     Single run detail (step-by-step log)
```

### Frontend: Three-Pane Editor

The editor at `/app/workflows/:wid` is a three-column layout:

| Pane | Content |
|---|---|
| Left (320 px) | Guided chat composer — text/voice input, assistant messages, design history. |
| Center (flex) | AWP canvas — React Flow DAG. Nodes typed (agent / delegation_loop / condition / http / forge_tool / delay / trigger). Drag-to-reorder, click-to-select. |
| Right (320 px) | Node inspector — selected node's properties, last input/output, per-step test button. |

Bottom toolbar: **▶ Test · ◼ Stop · 💾 Save · ↓ Export · ↑ Import · ⏱ Schedule**.

---

### Guided Design Assistant

The chat pane is not a one-shot command interface. It is a **persistent,
iterative design session** in which the assistant actively co-designs the
workflow with the operator — asking clarifying questions, surfacing gaps,
proposing next steps, and warning about incomplete configurations. The canvas
updates incrementally after each exchange. The operator never has to know AWP
syntax.

#### Design session phases

The design assistant operates in four sequential phases. The current phase
is shown as a breadcrumb in the chat pane header
(`Discovering → Structuring → Detailing → Ready`). The assistant never
skips forward; it waits for each phase to be sufficiently resolved before
advancing. Back-transitions are allowed at any time via natural language
(see Conversation-driven control below).

| Phase | Assistant behaviour | Exit condition |
|---|---|---|
| **Discovering** | Asks about goal, audience, and trigger. Max 3 questions total. | Goal and trigger type are known. |
| **Structuring** | Proposes a node skeleton on the canvas. Asks one "does this shape look right?" confirmation. | Operator confirms or redirects the skeleton. |
| **Detailing** | Visits each node once — one focused question per node (model, target channel, filter condition, error handling). | Every mandatory node property is filled. |
| **Ready** | Canvas consistent, no open gaps. Assistant announces readiness and offers a dry-run. | Operator runs, saves, or keeps editing freely. |

---

#### Template suggestion

Before the first Discovering question the assistant checks whether the
opening description matches a known pattern. If a template confidence
score ≥ 0.7 is reached (heuristic keyword match, no API call), the
assistant offers scaffolding instead of starting from scratch:

```
Operator:  "Fetch RSS feeds every morning and post a digest to Slack."
Assistant: "That sounds like the 'daily digest' pattern I know well.
            I have a template with 4 steps:
            TRIGGER > fetch_feed > summarise > send_message.
            Should I load it as a starting point? You can still
            change anything."
```

If the operator accepts, the canvas is pre-populated and the session
enters the Detailing phase directly, skipping Structuring entirely.

Built-in templates shipped with the console bundle:

| Template key | Skeleton |
|---|---|
| `daily-digest` | cron-trigger → fetch → summarise → deliver |
| `inbox-monitor` | email-trigger → filter → classify → route |
| `data-pipeline` | http_fetch → transform → store → notify |
| `approval-loop` | delegation_loop → condition → accept / reject branch |

---

#### Session lifecycle

When the operator creates a new workflow (and no template is offered or
accepted), the assistant sends the first message rather than waiting:

```
Assistant: "What should this workflow do? Describe the goal in one or
            two sentences — I will ask follow-up questions to fill in
            the details."

Operator:  "Monitor my inbox and send a daily summary to Discord."
```

The assistant then asks at most one focused follow-up question per turn,
waits for the answer, updates the canvas, and continues. A full exchange
for a fresh workflow typically looks like this:

```
Assistant: "Good. A few quick questions to shape the design:"
           "How often should it run -- daily, hourly, or on demand?"

Operator:  "Every morning at 7."

Assistant: "Should it only check unread emails, or all emails from the
            last 24 hours?"

Operator:  "Only unread."

Assistant: "Should the Discord message be a bullet list or a paragraph?"

Operator:  "Bullet list."

Assistant: "Perfect -- creating a first draft now."
           [canvas: TRIGGER > fetch_inbox > filter_unread > summarise > send_discord]
           "4 steps created. Want me to add a condition that skips Discord
            if there are no unread emails?"

Operator:  "Yes."

Assistant: "Done. Added a condition node after filter_unread."
           [canvas updates: condition node inserted]
```

#### Proactive gap-detection

After every canvas mutation the assistant inspects the workflow for common
gaps and surfaces at most one warning per turn:

- No trigger defined -- "This workflow has no trigger. Should it run on a
  schedule or be started manually?"
- A node has no depends_on and is not a trigger -- "Step X is disconnected.
  Which step should feed into it?"
- No output delivery step -- "The workflow produces output but does not send
  it anywhere. Where should the result go?"
- Budget likely exceeded -- "8 LLM steps may exceed the default token budget.
  Want me to raise it?"

Gap warnings appear as assistant messages in the chat pane and highlight the
affected node in amber on the canvas.

#### Iterative refinement

Once a first draft exists the operator continues the session to refine it:

```
Operator:  "Make the summarise step use Haiku instead of Sonnet."
Assistant: "Changed. Haiku is faster for list-style summaries."

Operator:  "What if the inbox fetch fails?"
Assistant: "Currently the workflow stops on error. I can add a retry
            (up to 3x, 30 s apart) or a fallback Discord message."

Operator:  "Retry; if it still fails, send a short error message."
Assistant: "Done. Added retry config and an error branch."
```

#### Checkpoint summaries

At the end of the Structuring phase and again at the transition into Ready,
the assistant emits a compact summary card in the chat pane:

```
Assistant: "Here is what we have so far:
            • Goal: morning Slack digest of top HN stories
            • Trigger: cron, daily 07:00 Europe/Berlin
            • Steps: fetch_hn → filter_top10 → summarise (Haiku) →
                     send_slack
            • Condition: skip Slack if fewer than 3 stories pass the filter
            Does this look right before I fill in the details?"
```

The operator can answer "yes", correct a bullet inline ("change Haiku to
Sonnet"), or say "redo the trigger" to back-transition. The summary card
is rendered as a visually distinct quoted block in the chat log, not as a
plain assistant message. The card is also stored in `<wid>.chat.jsonl`
so it is visible when reopening a session.

---

#### Conversation-driven control

Beyond the guided flow the operator can use natural phrases at any point:

| Phrase pattern | Effect |
|---|---|
| "undo / revert / go back" | Restores the last canvas state. The reverted exchange stays visible in the chat log but is struck through. |
| "start over / reset the workflow" | Clears the canvas, restarts at Discovering. Conversation history is retained as a collapsible "Previous session" block. |
| "explain [node / step / concept]" | Assistant explains in plain language. No canvas change. |
| "what if [variant]" | Assistant describes the canvas variant without applying it. Operator says "do that" to apply. |
| "show me the YAML" | Current AWP YAML rendered as a code block in chat. Read-only; no canvas change. |
| "I'm done / looks good / ship it" | Readiness check runs, any remaining gaps listed, then save + optional schedule offered. |

Voice users can speak any of these phrases. The STT pipeline normalises
common spoken variants ("go back one step", "undo that last thing").

---

#### Session persistence

The design conversation is stored alongside the workflow YAML in
<wid>.chat.jsonl (role / content pairs). Reopening the workflow restores
the full conversation so the operator can continue where they left off.
A short summary of the last session is shown at the top of the chat pane
on re-open.

#### Voice mode

When the operator uses the microphone the assistant responses are also
spoken via TTS (reusing the voice bridge TTS path). The canvas updates
visually in parallel, making the design loop fully hands-free.

#### Assistant system prompt contract

The WorkerEngine subprocess for the design assistant receives:

1. The current AWP YAML (as context).
2. The full conversation history (compressed if too long).
3. The current design phase (`discovering | structuring | detailing | ready`).
4. The template match result (key + confidence) if one was offered.
5. A system prompt that instructs the engine to:
   - Ask at most one question per turn.
   - Operate within the current phase; do not jump ahead.
   - Emit a `YAML_UPDATE:` sentinel followed by the complete new AWP YAML
     when the canvas should change, nothing otherwise.
   - Emit a `PHASE_UPDATE: <new_phase>` sentinel (on its own line) when
     the phase should advance or back-transition.
   - Never output partial or invalid YAML.
   - Warn about workflow gaps once per gap, not repeatedly.
   - Emit checkpoint summary cards using the `SUMMARY_CARD:` sentinel
     (JSON payload: `{goal, trigger, steps[], conditions[]}`) at the
     two mandatory checkpoints.

The backend processes each sentinel block independently before displaying
the chat text:

- `YAML_UPDATE:` — stripped, validated with `awp_dag_parser.parse_dag_dict`,
  persisted on success, rejected silently on validation failure (the chat
  message is still shown without the block).
- `PHASE_UPDATE:` — stripped, phase state updated in `<wid>.meta.json`,
  breadcrumb in the chat pane header refreshed.
- `SUMMARY_CARD:` — stripped, rendered as a distinct card component in the
  chat pane, stored in `<wid>.chat.jsonl` with `role: summary_card`.

---

### Chat-Driven Ad-hoc Commands

In addition to the guided design flow the operator can issue direct
one-shot commands at any point:

```
Operator: "Rename fetch_inbox to email_fetcher."
Operator: "Schedule this every Monday at 08:00 Europe/Berlin."
Operator: "Export as AWPKG."
Operator: "Run a dry-run and show me what would happen."
```

The assistant handles these immediately without asking follow-up questions.

---

### Test Mode

Clicking Test calls POST /v1/console/workflows/:wid/runs with
dry_run=false (or dry_run=true for schema-only validation). The backend
streams WorkflowRunEvent SSE:

```jsonc
{"type": "node_started",   "node_id": "fetch_news", "ts": 1716040800}
{"type": "node_completed", "node_id": "fetch_news", "tokens": 4200, "elapsed_s": 8.1}
{"type": "node_failed",    "node_id": "summarize",  "error": "..."}
{"type": "run_completed",  "ok": false, "budget": {...}}
```

The canvas reflects each event: nodes turn yellow (running), green (OK),
red (failed). A five-dimension budget bar (tokens, time, loops, workers,
tool_calls) updates in real time.

---

### Scheduling UI

A modal lets the operator set a cron trigger without knowing cron syntax.
Friendly presets (hourly, daily at HH:MM, weekdays, custom) with a live
preview of the next three runs. Saving calls PUT /:wid/schedule, which
writes the triggers.schedule block into the YAML and registers the job with
atelier-scheduler. Overrun policy (skip / queue / parallel) is selectable.

The assistant can also set the schedule via chat: "Schedule this every
Monday at 8 am" opens the modal pre-filled for confirmation.

---

### Export / Import

- Export .awp.yaml -- raw YAML for version control and sharing.
- Export .awpkg -- full package via core/awpkg/builder.py.
- Import -- drag-and-drop or paste; backend validates before persisting.

---

### Run History

/app/workflows/:wid/runs lists all runs with status, trigger source,
start time, duration, and token usage. Each row links to a step-by-step
log stored in <tenant>/workflows/<wid>/runs/<rid>.jsonl.

---

### Backend Routes (new, under /v1/console/workflows/)

| Method | Path | Description |
|---|---|---|
| GET | / | List workflows |
| POST | / | Create workflow |
| GET | /:wid | Get workflow |
| PATCH | /:wid | Update title/description |
| DELETE | /:wid | Delete workflow + runs |
| GET | /:wid/yaml | Download AWP YAML |
| PUT | /:wid/yaml | Replace AWP YAML (validated) |
| POST | /:wid/runs | Start a run (dry_run flag) |
| GET | /:wid/runs | List runs |
| GET | /:wid/runs/:rid | Run detail + event log |
| DELETE | /:wid/runs/:rid | Delete run record |
| GET | /:wid/schedule | Get cron schedule |
| PUT | /:wid/schedule | Set / update schedule |
| DELETE | /:wid/schedule | Remove schedule |
| POST | /:wid/chat | WebSocket -- guided design assistant |

---

### On-disk Storage

```
<atelier_home>/tenants/<tid>/workflows/
  <wid>.awp.yaml
  <wid>.meta.json
  <wid>.chat.jsonl        design session conversation history
  <wid>/runs/
    <rid>.jsonl
    <rid>.meta.json
```

---

### Audit Events

| Event type | Trigger |
|---|---|
| workflow.created | POST / |
| workflow.updated | PUT /:wid/yaml |
| workflow.deleted | DELETE /:wid |
| workflow.run_started | POST /:wid/runs |
| workflow.run_completed | run terminal state |
| workflow.scheduled | PUT /:wid/schedule |
| workflow.unscheduled | DELETE /:wid/schedule |

---

## Phase Plan

| Phase | Scope | Prerequisite |
|---|---|---|
| 1 | Library page + read-only canvas | None |
| 2 | Guided design assistant (chat > YAML > canvas, gap detection) | Phase 1 |
| 3 | Test mode with live SSE visualisation | Phase 2 |
| 4 | Schedule builder UI + atelier-scheduler wiring | Phase 3 |
| 5 | Export / Import (YAML + AWPKG) | Phase 1 |
| 6 | Run history + step-by-step logs | Phase 3 |
| 7 | Voice mode (TTS responses in design session) | Phase 2 |

Phases 1-3 are the minimum viable surface. Phases 4-7 extend the
operations experience.

---

## Consequences

**Positive:**
- AWP workflows become a first-class, operator-accessible feature.
- The guided assistant lowers the barrier to entry: operators describe
  intent; the assistant handles syntax, validation, and structure.
- Natural-language and voice authoring make complex multi-agent pipelines
  accessible to non-developers.
- Scheduling, history, and export/import complete the programs-in-an-OS
  mental model.

**Negative / Risks:**
- React Flow adds ~250 kB to the bundle; the workflows route must be
  code-split to avoid bloating other pages.
- The guided assistant adds one WorkerEngine round-trip per design turn;
  perceived latency must be masked with optimistic canvas updates and
  typing indicators.
- atelier-scheduler wiring (Phase 4) requires operator-level permissions
  at install time.

**Must NOT do:**
- Do not allow the worker engine to write workflow files directly --
  only through the validated PUT /:wid/yaml endpoint.
- Do not skip AWP validation on import.
- Do not persist run event logs in the main audit chain.
- Do not auto-start atelier-scheduler from bridge or adapter code.
- Do not surface raw YAML editing as the primary authoring mode.
- Do not ask more than one follow-up question per assistant turn --
  phone-first UX.
- Do not persist the design conversation in the main audit chain --
  it goes into <wid>.chat.jsonl only.
