# ADR-0062 — Console UX Overhaul:
# Bridge Setup Wizard · People Hub · Agent Hub Peer Cards · Workflow Split-View

**Status:** Proposed — 2026-05-28
**Date:** 2026-05-28
**Authors:** Claude Code (maintainer session)
**Type:** Multi-component — frontend UX + backend API additions
**Implements:** UX design concept (Discord voice session, 2026-05-28)
**Depends on:** ADR-0037 (console relaunch, React stack), ADR-0039 (workflow builder),
ADR-0048 (L38 A2A / Agent Hub), L18 (roles), L19 (disclosure), L20 (quota),
L28 (conversation recall), L38 (A2A), ADR-0053 (L39 AtelierFed),
ADR-0054 (L41 social capability grants)
**Scope:** `core/console/atelier_console/web-next/` (frontend) +
`core/console/atelier_console/routes/` (backend REST extensions)

---

## Context

The AtelierOS console (ADR-0037, ADR-0039) has a complete backend and a
technically functional React frontend. Four usability gaps prevent non-developer
operators from self-configuring their instance without CLI access:

| Gap | Current state | Impact |
|---|---|---|
| Bridge installation | Raw JSON textarea; WhatsApp QR present but flow fragmented | Operators cannot connect WhatsApp without file editing |
| User management | Scattered across legacy SPA + new console; no single surface | Assigning roles, quota, consent requires two different UIs |
| Agent Hub | "Inbound Origins" / "Outbound Endpoints" labels — abstract jargon; pairing is multi-step manual | Non-developers cannot pair agents without documentation |
| Workflow editor | Canvas only; no plain-language explanation of what the workflow does; run history is a flat table | Operators cannot inspect or trust workflows they didn't author |

This ADR specifies four concrete UI components to close these gaps. No new
runtime layers are introduced; all changes are frontend pages and thin REST
extensions on existing backend routes.

---

## Component 1 — Bridge Setup Wizard

### Problem

`/app/bridges` currently renders one tab per bridge channel. Each tab contains:
1. A collapsible setup guide (HTML instructions).
2. A raw JSON `<textarea>` for `settings.json`.
3. An enable/disable toggle.

WhatsApp has a QR code display, but it is rendered inline inside the JSON
card with no status feedback, no countdown, and no polling loop. For all other
bridges the only path is editing raw JSON.

Operators who are not developers cannot configure any bridge without reading
documentation and editing JSON by hand.

### Decision

Replace the per-tab card layout with a **3-step guided wizard** per bridge.
The wizard is opened via a "Connect" / "Reconfigure" button on each bridge tile;
the tile grid remains visible underneath as a dashboard.

#### Bridge tile grid (`/app/bridges`)

```
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│ 📱 WhatsApp  │  │ 🤖 Telegram  │  │ 💬 Discord   │
│ ● Verbunden  │  │ ○ Nicht konf.│  │ ● Verbunden  │
│ [Verwalten]  │  │ [Verbinden]  │  │ [Verwalten]  │
└──────────────┘  └──────────────┘  └──────────────┘
```

Status badges: `Verbunden` (emerald) / `Konfiguriert, inaktiv` (amber) /
`Nicht konfiguriert` (slate). A bridge in error state shows `Fehler` (red)
with a detail tooltip.

#### Wizard — Step 1: Authenticate

Bridge-specific credential form. All fields are typed inputs (not a JSON
blob). Field definitions live in a static `BRIDGE_FIELDS` map in the
frontend — no new backend schema required for field rendering.

| Bridge | Fields |
|---|---|
| WhatsApp | _(no credentials — uses QR session)_ → skip to Step 2 immediately |
| Telegram | Bot-Token (password input) + optional Whitelist (comma-separated) |
| Discord | Bot-Token (password input) + Application ID |
| Slack | OAuth Bot Token + Signing Secret (both password inputs) |
| Email | IMAP host/port/user/pass + SMTP host/port/user/pass |
| Signal | Socket path (text input, default `/var/run/signal-cli/socket`) |
| Teams | Microsoft Bot Framework App ID + App Password |

Inline validation on blur: token format check (regex per bridge type).
"Weiter" button disabled until validation passes.

#### Wizard — Step 2: Connect

Bridge-specific connection step.

**WhatsApp — Live QR flow:**

```
┌──────────────────────────────────────┐
│  📱 WhatsApp verbinden               │
│                                      │
│      ┌────────────────────┐          │
│      │                    │          │
│      │   [QR-CODE SVG]    │  256×256 │
│      │                    │          │
│      └────────────────────┘          │
│                                      │
│  ⏳ Warte auf Scan…  ●●●             │  ← pulse animation
│  Läuft ab in: 00:38                  │  ← countdown (60 s)
│                                      │
│  1. Öffne WhatsApp auf deinem Handy  │
│  2. Tippe auf ⋮ → Verknüpfte Geräte  │
│  3. Scanne diesen Code               │
└──────────────────────────────────────┘
```

- Frontend polls `GET /v1/console/bridges/whatsapp/qr-status` every 3 s.
- Response: `{ status: "waiting" | "scanned" | "linked" | "expired", qr_svg?: string, ttl_s: number }`
- On `scanned`: replace QR with spinner + "Wird bestätigt…"
- On `linked`: auto-advance to Step 3, confetti micro-animation.
- On `expired`: show "Code abgelaufen" + "Neuen Code laden"-Button (re-polls endpoint).
- QR SVG is generated server-side by the existing `whatsapp_qr_bridge.py`;
  the new endpoint is a thin wrapper.

**Token-based bridges (Telegram, Discord, etc.):**

Step 2 calls `POST /v1/console/bridges/{channel}/test-connection` with the
credentials from Step 1. Response: `{ ok: bool, detail: string }`.

- `ok: true` → green check, "Verbindung erfolgreich", auto-advance after 1.5 s.
- `ok: false` → red banner with `detail` message, "Zurück"-button to fix credentials.

#### Wizard — Step 3: Test & Activate

- Sends a test message through the bridge to the operator's own UID.
- "Testnachricht gesendet. Hast du sie erhalten?" — two buttons: ✓ Ja / ✗ Nein.
- On confirm: `PATCH /v1/console/bridges/{channel}/settings` sets `enabled: true`,
  wizard closes, tile updates to `● Verbunden`.
- On deny: shows troubleshooting tip specific to the bridge type.
- "Überspringen" link for bridges where a test message is not possible (Signal).

#### Advanced config

An "Erweiterte Konfiguration" accordion at the bottom of Step 1 exposes the
raw JSON textarea for power users. Changes there override the typed fields.
The accordion is collapsed by default and must remain accessible — it is the
only path for settings not covered by the field map (e.g. custom rate limits).

#### New backend endpoints

| Method | Route | Purpose |
|---|---|---|
| GET | `/v1/console/bridges/whatsapp/qr-status` | Live QR SVG + status + TTL |
| POST | `/v1/console/bridges/{channel}/test-connection` | Credential validation |
| POST | `/v1/console/bridges/{channel}/test-message` | Send test message to self |

All three are thin wrappers over existing bridge daemon state. They emit no
new audit events (bridge state changes already emit `bridge.settings_changed`).

#### New frontend files

```
web-next/src/pages/bridges.tsx           (replace existing — wizard logic)
web-next/src/components/bridge/
  BridgeTileGrid.tsx
  BridgeSetupWizard.tsx
  WhatsAppQrStep.tsx
  TokenConnectStep.tsx
  TestActivateStep.tsx
```

---

## Component 2 — People Hub

### Problem

User/member management is split:
- **Legacy SPA** (`/members`, `/members/{chat_key}`): role assignment, quota
  display, consent status. Functional but not linked from the new console.
- **New console** (`/app/space`): partially implemented, scope unclear.

No single surface shows the operator "who has access, with what role, what quota
consumed, and what consent status" across all bridge channels at once.

### Decision

Add a new route `/app/people` to the new console. The legacy `/members` pages
remain as-is and are not deleted — the new page is additive.

#### Layout: List + Side-Panel

```
Personen (12)                    [Filter ▾]  [+ Einladen]
──────────────────────────────────────────────────────────
🟢  silvio.jurk     Admin    Discord    ████░  42/100
⚪  max.mustermann  Member   WhatsApp   ██░░░  18/100
⚪  lisa.test       Observer Telegram   ░░░░░   0/100
```

Columns: status-dot, display-name/uid, role badge, primary bridge channel,
quota bar (messages used / limit). Clicking a row opens a side-panel without
navigating away from the list.

Status dot: green = active today, blue = online now (L28 recall last-seen),
grey = inactive >7 days.

#### Per-person side-panel

```
┌─────────────────────────────────┐
│ max.mustermann                  │
│ WhatsApp · discord:#general     │
├─────────────────────────────────┤
│ Rolle:  [Member              ▾] │  ← L18 role dropdown
│                                 │
│ Nachrichten-Quota               │
│ ████░░░░░░  18 / 100 heute      │  ← L20 quota bar
│ [Limit anpassen]                │
│                                 │
│ Zustimmung (GDPR)               │
│ ● Aktiv  Läuft ab 2026-06-28    │  ← L16 consent TTL
│ [Verlängern] [Widerrufen]       │
│                                 │
│ Offenlegung (EU AI Act)         │
│ ✓ Gesendet 2026-05-10           │  ← L19 disclosure status
│                                 │
│ Letzte Aktivität: vor 2 Stunden │
│ Gesamt: 234 Nachrichten         │
└─────────────────────────────────┘
```

Role change: `PATCH /v1/console/members/{chat_key}/{uid}/role` — existing
endpoint. Confirmation dialog for demotions (member → observer removes
write access).

Quota adjust: inline number input → `PATCH /v1/console/members/{chat_key}/{uid}/quota`.
Validation: admin cannot set quota above their own tier ceiling.

Consent: buttons call existing `POST /v1/console/consent/...` routes.

#### Invite flow

"+ Einladen"-Button → modal:

```
Über welchen Kanal einladen?
[WhatsApp] [Telegram] [Discord] [E-Mail] [Link kopieren]
```

Generates a one-time invite URL with a short TTL (24 h). When the invited
user opens it, they are routed through the L19 disclosure flow for that bridge.
Backend: `POST /v1/console/invites` (new endpoint, stores token in session
store with TTL; no new persistent table needed).

#### Bulk actions

Checkbox column in the list → action bar appears:

```
3 ausgewählt  [Rolle setzen ▾]  [Quota setzen]  [Zustimmung verlängern]
```

Calls the same per-user endpoints sequentially. Progress shown inline per row.

#### New frontend files

```
web-next/src/pages/people.tsx
web-next/src/components/people/
  PeopleList.tsx
  PersonSidePanel.tsx
  RoleSelect.tsx
  QuotaBar.tsx
  ConsentWidget.tsx
  InviteModal.tsx
```

Sidebar nav entry: "Personen" under the "Channels" group, between Bridges
and Voice. Icon: `Users` (Lucide).

#### New backend endpoint

| Method | Route | Purpose |
|---|---|---|
| POST | `/v1/console/invites` | Create time-limited invite token |
| GET | `/v1/console/invites/{token}` | Redeem invite (redirects to disclosure) |

---

## Component 3 — Agent Hub Peer Cards

### Problem

`/app/agent-hub` presents two tabs: "Inbound Origins" and "Outbound Endpoints".
The terms are correct (L38 vocabulary) but unfamiliar to non-developers.
The pairing flow requires: generate JSON files, copy them out-of-band, place
them on the peer, restart — no UI-driven invite path exists.

### Decision

Redesign `/app/agent-hub` around the mental model of "peers" rather than
directional connection types. The underlying L38 data structures (origin /
endpoint files) are unchanged; only the presentation changes.

#### Main layout

```
Verbundene Agenten (3)                [+ Agent verbinden]
──────────────────────────────────────────────────────────
[Alle] [Kann senden] [Kann empfangen] [Deaktiviert]
```

Peers are displayed as cards in a 3-column grid (responsive: 1 on mobile,
2 on tablet):

```
┌─────────────────────────┐
│ 🤖  Agent B (Laptop)    │
│ ● Online · vor 4 Min.   │
│                         │
│ Darf: Senden, Beobachten│
│ Letzte Task: 14:32       │
│                         │
│ [Bearbeiten]  [Log ▾]   │
└─────────────────────────┘
```

"Log ▾" expands an inline collapsible feed of the last 10 A2A audit events
for this peer (from the existing `GET /v1/console/remote-trigger/log` endpoint,
filtered by `origin_id` or `endpoint_id`).

#### Permission matrix (in Bearbeiten-Modal)

```
Berechtigung     Aktiv
─────────────────────
Aufgaben senden   ☑
Ergebnisse lesen  ☑
Admin-Zugriff     ☐
```

Saves to the origin/endpoint JSON file via `PATCH /v1/console/agent-hub/peers/{id}`.

#### "+ Agent verbinden" — 3-way invite modal

```
Wie möchtest du verbinden?

┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ 📤 Einladen      │  │ 📥 Code eingeben │  │ ⚙ Manuell        │
│ Link / QR für    │  │ Code vom anderen  │  │ Dateien + Restart │
│ den anderen Peer │  │ Peer eingeben     │  │ (Fortgeschritte) │
└──────────────────┘  └──────────────────┘  └──────────────────┘
```

**"Einladen" flow:**
1. Frontend calls `POST /v1/console/agent-hub/invites` → returns `invite_code`
   (a short base58 string, TTL 24 h, stored in session store).
2. UI shows: the code as text + a QR code (rendered client-side via `qrcode` npm package).
3. Peer operator enters the code in their own console → "Code eingeben" flow.
4. On redeem: `POST /v1/console/agent-hub/invites/{code}/accept` → server
   generates matched origin + endpoint files on both sides via a signed
   exchange (HMAC key generated server-side, written to both instances via
   the existing `atelier-a2a pair` logic wrapped in a REST call).

**"Code eingeben" flow:** single input + "Verbinden"-button → calls accept endpoint.

**"Manuell" flow:** exposes the existing raw JSON import form (current UI).

#### New backend endpoints

| Method | Route | Purpose |
|---|---|---|
| POST | `/v1/console/agent-hub/invites` | Generate invite code + QR payload |
| POST | `/v1/console/agent-hub/invites/{code}/accept` | Redeem invite, write origin + endpoint files |
| PATCH | `/v1/console/agent-hub/peers/{id}` | Update permissions on origin/endpoint JSON |
| GET | `/v1/console/agent-hub/peers` | List all peers (merged origin + endpoint view) |

#### New frontend files

```
web-next/src/pages/agent-hub.tsx          (replace existing)
web-next/src/components/agent-hub/
  PeerCard.tsx
  PeerEventLog.tsx
  PermissionMatrix.tsx
  ConnectModal.tsx
  InviteCodeDisplay.tsx
```

---

## Component 4 — Workflow UX Overhaul

### Problem

ADR-0039 Phases 1–7 delivered a working workflow editor with a node canvas,
a guided chat composer, and a run history table. Three concrete usability
problems remain:

1. **The canvas is opaque.** A user looking at a workflow they did not author
   cannot tell what it does without reading YAML. There is no plain-language
   explanation.

2. **Node creation is friction-heavy.** Adding a node requires right-clicking
   the canvas, picking from a flat type list, and then filling in a modal form.
   There are no templates.

3. **Run history is a flat table.** It shows status per run but not *where*
   time was spent per node, making performance problems invisible.

### Decision

Three targeted additions to the existing workflow editor. The three-pane layout
(ADR-0039) is preserved; only the centre pane (canvas) and the runs page
are extended.

#### 4a — Explanation Panel (Split-View)

The canvas pane gains a resizable right drawer (default 380 px, collapsible)
called "Was es tut".

```
┌────────────────────────┬─────────────────────────┐
│   CANVAS               │  WAS ES TUT             │
│                        │                          │
│  [Trigger]             │  📋 Zusammenfassung      │
│      ↓                 │  Wenn eine neue Nachricht│
│  [Agent: Analyse]      │  eintrifft, analysiert   │
│      ↓                 │  "Analyse" den Inhalt    │
│  [Bedingung]           │  und leitet ihn weiter.  │
│   ↙       ↘           │                          │
│ [Ja]     [Nein]        │  🔵 Ausgewählt:          │
│                        │  Bedingung               │
│                        │  Prüft: score > 0.7      │
│                        │  Ja  → Senden            │
│                        │  Nein → Verwerfen        │
└────────────────────────┴─────────────────────────┘
```

**Summary generation:** When a workflow is saved or loaded, the frontend
calls `POST /v1/console/workflows/{wid}/explain`. The backend spawns a
single Haiku-4.5 call (`claude -p --max-turns 1 --no-tools`) with the
AWP YAML and a prompt: "Describe this workflow in plain German in 3
sentences maximum. Then list each node as one line: NodeName — what it does."

The response is cached in the workflow metadata (`meta.explanation`,
`meta.explanation_ts`). The explanation is regenerated on each save.
It is never stored in the audit chain. Node-level explanations are
extracted from the structured Haiku response by splitting on node names.

Clicking a node on the canvas highlights its explanation entry in the panel.
The panel scrolls to the selected node's line.

**Drawer toggle:** a `PanelRight` icon button at the top-right of the canvas
shows/hides the drawer. State persists in `localStorage`.

#### 4b — Node Quick-Add Palette

Clicking the "+" FAB in the canvas (bottom-right, already present) opens an
inline floating palette instead of a modal:

```
  📨 Nachricht empfangen
  🤖 Agent ausführen
  🔀 Bedingung prüfen
  📤 Antworten / Senden
  ⏱ Warten
  🔗 Webhook aufrufen
  ─────────────────────
  📁 Vorlagen…
```

Each entry maps to a pre-filled AWP node stub. Clicking inserts the node at
the canvas centre. The node is immediately selected; its properties panel
opens in the left chat-composer pane so the user fills in details inline.

"Vorlagen…" opens the template gallery (§4c).

#### 4c — Template Gallery

New tab at `/app/workflows` alongside "Meine Workflows": **"Vorlagen"**.

Each template is a static AWPKG file shipped with the console bundle under
`web-next/src/assets/workflow-templates/`. Initial set (5 templates):

| Template | Description |
|---|---|
| `daily-briefing.awpkg` | Tägliche Zusammenfassung um 08:00 per WhatsApp |
| `alert-routing.awpkg` | Eingehende Alerts → Analyse → an zuständigen Agenten weiterleiten |
| `content-approval.awpkg` | Entwurf → Review-Agent → Freigabe oder Ablehnung |
| `qa-pipeline.awpkg` | Frage stellen → Retrieve-Agent → Antwort generieren → Senden |
| `webhook-to-message.awpkg` | Eingehender HTTP-Webhook → Nachricht an Channel |

Template card layout:

```
┌─────────────────────────────────────────────┐
│  [mini canvas preview — static SVG]         │
├─────────────────────────────────────────────┤
│  📋 Tägliche Zusammenfassung                │
│  Sendet jeden Morgen um 08:00 eine          │
│  Zusammenfassung per WhatsApp.              │
│                          [Verwenden →]      │
└─────────────────────────────────────────────┘
```

"Verwenden" calls `POST /v1/console/workflows` with the AWPKG content →
opens the new workflow in the editor pre-filled. No template is modified;
each use creates a fresh copy.

Mini canvas previews are pre-rendered static SVGs shipped in the bundle —
no live rendering.

#### 4d — Run Timeline View

`/app/workflows/:wid/runs/:rid` currently shows a flat event log table.
Replace with a Gantt-style timeline:

```
Node               0 s   2 s   4 s   6 s   8 s
─────────────────────────────────────────────────
Trigger            ████
Agent: Analyse           ████████████
Bedingung                            ████
Senden                                    ████████
```

Bar colours match node state: emerald (ok), red (failed), amber (running),
slate (waiting/skipped). Clicking a bar opens a side drawer with:
- Input payload (collapsed, "Eingabe anzeigen" accordion)
- Output payload (collapsed)
- Duration in ms
- Error message if failed

Timeline data comes from the existing run event stream
(`GET /v1/console/workflows/{wid}/runs/{rid}/events`). The frontend
computes relative start/end times from `started_at` / `completed_at`
fields already present on each event.

#### New backend endpoints

| Method | Route | Purpose |
|---|---|---|
| POST | `/v1/console/workflows/{wid}/explain` | Generate + cache Haiku explanation |
| GET | `/v1/console/workflows/{wid}/explain` | Return cached explanation |

The explain endpoint must NOT emit the AWP YAML or the Haiku output to the
audit chain. Only `workflow.explain_generated` (count + duration_ms) is audited.

#### Modified frontend files

```
web-next/src/pages/workflows.tsx           (add template tab)
web-next/src/pages/workflow-editor.tsx     (add explanation drawer)
web-next/src/pages/workflow-run-detail.tsx (replace table with timeline)
web-next/src/components/workflow/
  ExplanationDrawer.tsx                    (new)
  NodeQuickAddPalette.tsx                  (new)
  TemplateGallery.tsx                      (new)
  TemplateCard.tsx                         (new)
  RunTimeline.tsx                          (new)
web-next/src/assets/workflow-templates/    (new directory, 5 × .awpkg)
```

---

## Cross-cutting design rules

These apply to all four components and must not be violated in implementation:

| Rule | Rationale |
|---|---|
| **Progressive disclosure** | Form-first, raw JSON as opt-in accordion. Never the inverse. |
| **No JSON by default** | Operators who are not developers must be able to complete every wizard step without knowing JSON syntax. |
| **Every action has a feedback state** | Loading spinner → success toast or error banner. No silent failures. |
| **Responsive layout** | Side-panels, not modals, for detail views — the list stays visible on desktop. On mobile, side-panels become bottom sheets. |
| **Status clarity** | Every resource tile shows: configured? active? error? at a glance — no drill-down needed for status. |
| **No audit leakage** | Explanation text, QR payloads, invite tokens, and member notes must never appear in `audit.jsonl` detail fields. |
| **L18 enforcement in UI** | Role dropdowns must respect the L18 kerze rule: a user cannot promote themselves or demote a role higher than their own. |
| **L19/L20 immutability** | Consent and quota widgets call existing REST endpoints — they do not write directly to JSON files. |

---

## Milestones

All four components are independent and can be developed in parallel.
Suggested order by impact/effort ratio:

| Milestone | Component | Deliverable |
|---|---|---|
| **M1** | Bridge Wizard | Tile grid + wizard shell (Steps 1–3), WhatsApp live QR poll |
| **M2** | People Hub | List + side-panel + role/quota/consent widgets |
| **M3** | Agent Hub Peer Cards | Card grid + invite modal (Einladen + Code-eingeben paths) |
| **M4** | Workflow Explanation | Haiku explain endpoint + drawer, cached per workflow |
| **M5** | Workflow Templates | Gallery tab + 5 bundled AWPKG files |
| **M6** | Run Timeline | Gantt-style run detail view |
| **M7** | Node Quick-Add Palette | FAB palette replacing the modal flow |

M1 and M2 unblock the most operator-facing self-service. M3 unblocks the
A2A demo path. M4–M7 complete the workflow polish.

---

## Must NOT do

- Do not delete the raw JSON textarea — keep it as the advanced accordion.
- Do not bypass L18 role enforcement in the People Hub role dropdown.
- Do not put AWP YAML, explanation text, invite token payloads, or QR content
  in audit chain `details` fields.
- Do not introduce a new persistence layer — all state goes through existing
  REST routes or the session store.
- Do not remove the legacy `/members` pages until People Hub has reached M2
  and has been validated in production for ≥14 days.
- Do not run the Haiku explain call synchronously in a frontend request — it
  must be async (fire-and-forget on save, result polled via GET).
- Do not ship the invite-code endpoints without HMAC signing (replay protection
  is structural, not optional).
- Do not widen the `/v1/console/agent-hub/invites` TTL beyond 24 h without
  an ADR update.
