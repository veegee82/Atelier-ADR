# ADR-0024 — Adaptive OS-Turn Model Selection (Layer 29.5 Phase 3)

**Status:** Accepted
**Date:** 2026-05-16
**Companion to:** Layer 29.5 Phase 1 (Helper-Model Cost-Split),
Layer 29.5 Phase 2 (OS-Turn Model Selection — `helper_model_default`
+ env-wide opt-in), Layer 22 (`WorkerEngine`), Layer 29 (Delegation),
ADR-0022 (Engine-agnostic Forge + SkillForge).
**Implements:** Neun sub-phases, jede unabhängig shippable —
29.5.3a Selektor-Modul, 29.5.3b Resolver-Ersatz (Phase 2 raus),
29.5.3c Spawn-Input-Wiring, 29.5.3d Audit-Events,
29.5.3e Retry-on-Thrashing-Backstop, 29.5.3f Prometheus,
29.5.3g Grafana-Panels, 29.5.3h Phase-2-Cleanup
(helper_model_default-Feld + env raus, SITE_OS_TURN entfernen),
29.5.3i ADR-Close + CLAUDE.md.
**Replaces:** Layer 29.5 Phase 2 (`helper_model_default` Persona-Feld
+ `ATELIER_HELPER_MODEL_OS_TURN` env-wide opt-in + `SITE_OS_TURN`
helper_model.py konstante) — strukturell entfernt, kein Dual-Path.

---

## Context

Layer 29.5 Phase 2 hat den OS-Turn-Model-Knopf eingeführt
(`adapter._resolve_os_model(profile)`). Heute (2026-05-16) ist
der Stand:

| Pfad | Ergebnis |
|---|---|
| Persona mit explicit `model:` | das Modell |
| Persona mit `helper_model_default: true` | resolve_helper_model(SITE_OS_TURN) |
| env `ATELIER_HELPER_MODEL_OS_TURN=X` | das Modell für ALLE Personas |
| sonst | None → CLI-Default (Subscription = Opus) |

Akute Lage seit dem Haiku→Sonnet-Switch von heute Vormittag:
`service.env` setzt **global** Sonnet 4.6 für jeden Bridge-Turn.
Das war die Notbremse nach dem Haiku-Autocompact-Thrashing auf
Layer-30 / atelier_data / Layer-24-Tasks. Sonnet 4.6 ist
~70 % billiger als Opus, aber immer noch ~5× teurer als Haiku.

**Das Problem:** ein typischer Bridge-Turn ist klein — eine
Begrüßung, eine Status-Frage, ein `/btw`, ein kurzer
Bestätigungs-Reply. Auch ein Auto-Routing-Schritt oder die
Reformulierung einer Tool-Antwort braucht selten mehr als 50k
Tokens. Diese Tasks **könnten in Haikus 200k-Context-Window
passen** — und wenn doch, wäre Haiku ausreichend. Heute zahlen
sie Sonnet-Preise.

Die offensichtliche Lösung "wieder Haiku als Default" hat genau
das Problem ausgelöst, das wir gerade gefixt haben: bei großen
Tasks (Layer-30-Discussion, atelier_data-Anfragen) reicht Haikus
Context nicht aus → Autocompact-Thrashing → fehlgeschlagener
Turn. Das ist auch keine Operator-Wahl per `/engine`, weil
dieselbe Persona im selben Chat mal kleine, mal große Anfragen
bearbeitet.

Drei Strategien wurden dialektisch durchgespielt:

| Strategie | Mechanik | Pro | Contra |
|---|---|---|---|
| **A — Pre-Spawn-Schätzer** | Vor Spawn die Initial-Bytes (Prompt + System-Prompt + MCP-Schemas + Session-History) summieren. Wenn ≤ Schwellwert → Haiku, sonst → Sonnet. | Schnell, lokal, deterministisch, 1 Spawn pro Turn | Tool-Use-Chain wird ignoriert. Ein kurzer Prompt kann eine lange Coding-Session triggern, deren Tool-Outputs allein Haikus Context sprengen. |
| **B — Retry-on-Thrashing** | Erst Haiku versuchen. Wenn Spawn mit `Autocompact thrashing` / `prompt too long` / `400 context_length_exceeded` fehlschlägt → retry mit Sonnet. | Automatischer Fallback, optimal billig-die-funktioniert | 1. Reply langsam, doppelte Tokens bei den Fehlversuchen. Nicht jeder Crash signalisiert Context-Probleme. |
| **C — Persona-/Site-Floor** | Per-Persona-Tag `os_model_floor: "sonnet"` — verhindert Auto-Downgrade unter ein bestimmtes Niveau. | Klar, vorhersehbar, kein Schätz-Risiko | Statisch, Operator-Aufwand, eine Persona für viele unterschiedlich-große Tasks ist zu grob. |

**Keine der drei alleine reicht.** A unterschätzt Tool-Chain. B
ist langsam beim Erstaufruf. C ist zu grob.

---

## Decision

**Hybrid A + B + C.** Schätzer trifft die Default-Wahl
(billig-wenn-möglich), Retry-Backstop deckt Schätz-Fehler ab,
Persona-Floor pinnt notorisch-große Personas hart.

Das ist eine **erweiterung** von Layer 29.5 Phase 2, kein Ersatz.
Bestehende Pfade (`helper_model_default`, explicit `model:`,
env-wide opt-in) bleiben byte-identisch funktional.

### Vier-Tier Resolution Order

Phase 3 ersetzt Phase 2 **strukturell**, kein Legacy-Pfad. Vier
Tiers in dieser Reihenfolge:

```
_resolve_os_model(profile, *, payload_chars) → str | None

  1. env ATELIER_OS_MODEL_OVERRIDE                   → kill-switch
     (operator-wide pin, gewinnt vor allem inkl. explicit model)

  2. profile.model OR persona.model                  → explicit
     (per-Persona / per-chat-profile Pin)

  3. autoselect_os_model(payload_chars) + persona.os_model_floor
     → adaptive
     (Schätzer + Floor, Default-Pfad wenn AUTOSELECT=on)

  4. None → CLI default                              → subscription
```

Tier 1 ist die **strukturierte Notbremse**, die Phase-2's
versteckten `ATELIER_HELPER_MODEL_OS_TURN`-Pfad ersetzt. Sie
gewinnt vor `model:` Feld bewusst (incident response soll
operator-wide funktionieren ohne jede Persona einzeln zu
patchen).

Tier 3 ist der **neue Default-Pfad**. Aktiv wenn (a)
`ATELIER_OS_MODEL_AUTOSELECT=on` (Default) UND (b) `payload_chars`
vom Aufrufer übergeben wurde. Bei `AUTOSELECT=off` fällt der
Resolver direkt auf Tier 4 (None → subscription default).

**Edge-Case `estimate_failed`:** wenn der Schätzer
crash't / None zurückgibt (z.B. session_dir-Read-Fehler trotz
try/except), wählt Tier 3 **HIGH** (Sonnet) als safe-default
+ emittet `os_model.selected` mit `reason="estimate_failed"`.
Niemals silently LOW bei Schätz-Failure — wäre ein Stiller
Bug-Maskierer.

### Schätzer-Formel

```python
def estimate_os_turn_chars(
    prompt: str,
    system_prompt: str,
    mcp_config_text: str = "",          # RAM-Pass-Through, kein Disk-Read
    session_dir: Path | None = None,
) -> int:
    chars = (
        len(prompt)
        + len(system_prompt)
        + len(mcp_config_text)
        + _session_history_bytes(session_dir)
        + _INTERNAL_OVERHEAD_CHARS  # 10_000 — claude-internal preamble + tool boilerplate
    )
    return chars


def _session_history_bytes(session_dir: Path | None) -> int:
    """Rekursive Bytes-Summierung über session_dir mit Hard-Cap 5 MB.

    Über 5 MB ist Haiku unabhängig vom genauen Wert raus —
    weiter zählen wäre Verschwendung. Liefert min(actual, 5_242_880).
    Permission-Errors / Missing-Files werden silently 0-gezählt
    (best-effort; Schätzer darf den Spawn nie crashen).
    """
    if session_dir is None or not session_dir.exists():
        return 0
    SESSION_BYTES_CAP = 5 * 1024 * 1024  # 5 MB
    total = 0
    try:
        for path in session_dir.rglob("*"):
            if path.is_file():
                try:
                    total += path.stat().st_size
                except OSError:
                    continue
                if total >= SESSION_BYTES_CAP:
                    return SESSION_BYTES_CAP  # early exit, cap
    except OSError:
        return 0
    return min(total, SESSION_BYTES_CAP)
```

**RAM-Pass-Through (Open Question 2 resolved):** Der MCP-Config-
String wird in `_resolve_spawn_inputs` ohnehin gebaut und in eine
Datei materialisiert. Statt die Datei nach dem Schreiben wieder
zu lesen, reicht der Aufrufer den String direkt an
`estimate_os_turn_chars` durch. **Performanter** (kein extra
Disk-IO pro Bridge-Turn), **robuster** (was geschrieben wird, IST
was geschätzt wird — keine TOCTOU-Lücke, kein Cache-Drift).

**Rekursive session_dir (Open Question 1 resolved):** `rglob('*')`
über alle Files unterhalb des `session_dir`, mit Hard-Cap 5 MB
und Early-Exit. Damit wird sowohl `.claude.json` als auch der
`.claude/`-Subdir (Tool-Use-History) berücksichtigt. Cap-
Begründung: bei 5 MB Session-State ist Haiku raus unabhängig vom
genauen Wert — weiter zählen ist Verschwendung.

Was geht in `system_prompt` ein:
- Base-Prompt (channel-aware "Du bist Claude Code in der …")
- Persona `append_system` (cowork-Resolver-Ergebnis)
- Audience-Block (Layer 12)
- User-Style-Block (Layer 26)
- User-Model-Block (Layer 28.2, `<user_context>` als LETZTER Eintrag)
- Skill-Inject-Block (Layer 7 + 15)
- Observer-Transcript-Block (Layer 16 Phase 2)
- Recall-Block (Layer 28.1)
- SELF-CHECK-Block (Layer 11)

Alle diese Bestandteile werden bereits in `_resolve_spawn_inputs`
gebaut. Der Schätzer wird **nach** dem Build-Step aufgerufen und
misst die Bytes am bereits konstruierten String.

### Threshold-Auswahl

**Default `THRESHOLD_CHARS = 60_000`** (~17k tokens bei 3.5
chars/token konservativ).

Mathe-Begründung:
- Haikus Context-Window: 200k tokens (= ~700k chars bei 3.5 c/t)
- Autocompact-Reserve: 30 % (= 60k tokens müssen für nächste
  Turns + Tool-Outputs frei bleiben)
- Verfügbares Initial-Budget: ~140k tokens = ~490k chars
- Sicherheits-Faktor für unknown Tool-Chain-Expansion: 8×
  (= 60k chars Initial-Bytes erlauben dann bis zu ~480k chars
  Tool-Output expansion, immer noch unter 140k-Budget)

Mit `ATELIER_OS_MODEL_THRESHOLD_CHARS` operator-tunable. Untergrenze
20k chars (sub-20k garantiert in Haiku's Comfort-Zone — Senkung
darunter wäre nur Kosmetik). Obergrenze 200k chars (darüber
ergäbe Haiku keinen Sinn mehr).

### Backstop B — Retry-on-Thrashing

In `_call_claude_streaming_via_engine`:

```python
def _maybe_retry_on_context_overflow(error_text: str, current_model: str) -> str | None:
    """Wenn Fehler kontext-bezogen UND aktuell auf LOW → upgrade auf HIGH.
    Returns the new model id to retry with, or None to give up."""
    if current_model == HIGH:
        return None  # bereits oben, kein weiterer Retry
    if not _is_context_error(error_text):
        return None  # 5xx, network, user-cancel — kein retry
    return HIGH
```

Curated Error-Patterns für `_is_context_error`:
- `"Autocompact is thrashing"`
- `"prompt is too long"`
- `"context_length_exceeded"`
- `"400"` AND `"context"` (HTTP 400 mit context-Begriff)
- `"input length"` AND `"exceeds"` (LLM-API standard phrasing)

Pattern-Liste in `model_selector._CONTEXT_ERROR_PATTERNS` curated;
neue Pattern brauchen E2E-Case mit dem konkreten Error-String.

Maximal **1 Retry** pro Turn (kein Endlos-Loop falls auch Sonnet
crasht — dann ist der Fehler echt und der User soll ihn sehen).

### Persona-Floor (Pfeiler C)

Neues optionales Feld in Persona-JSON:

```json
{
  "name": "forge",
  "os_model_floor": "sonnet"
}
```

Werte:
- `"haiku"` (oder unset) — default, kein Floor
- `"sonnet"` — Auto-Downgrade auf Haiku verboten, mindestens Sonnet
- `"opus"` — Auto-Downgrade verboten, mindestens Opus

Resolver-Logik:
```python
floor = persona.get("os_model_floor")  # haiku | sonnet | opus
chosen = autoselect_os_model(payload_chars)
if _model_rank(chosen) < _model_rank(floor):
    chosen = floor_model(floor)
```

**Floor-Entscheidung (Open Question 3 resolved):** Nur `forge`
bekommt `os_model_floor: "sonnet"`. Alle anderen Bundle-Personas
bleiben unset → reines Auto-Select.

| Persona | os_model_floor | Begründung |
|---|---|---|
| `forge` | `"sonnet"` | Forge-Operationen lesen policy.json + tool-manifest-Listings; Tool-Creation und Policy-Edits sind safety-kritisch — Sonnet's bessere Reasoning ist hier den geringen Aufpreis wert. Auto-Select würde forge zwar meistens auf Sonnet schicken (große Initial-Bytes), aber der Floor ist die strukturelle Garantie ohne Schätz-Vertrauen. |
| `orchestrator-haiku` | unset | Existierendes explizites `model: "claude-sonnet-4-6"` (Phase-2-Notbremse vom 2026-05-16) gewinnt via Tier 2 weiterhin. Phase 3 entfernt nur `helper_model_default: true` aus dieser Persona (29.5.3h). |
| `coder` / `research` / `assistant` / `orchestrator` / `inbox` / `browser` / `homeassistant` | unset | Auto-Select arbeitet — Initial-Bytes sind moderat (≤ 60k chars), passen typisch in Haikus Comfort-Zone. |

---

### Audit-Chain — zwei neue Event-Types (metadata-only)

Registriert in `forge/security_events.py::EVENT_SEVERITY`:

| Event | Severity | Per-Event Allow-List (metadata) |
|---|---|---|
| `os_model.selected` | INFO | `persona`, `channel`, `estimate_chars`, `chosen` (`haiku`/`sonnet`/`opus`/`other`), `reason` (`explicit`/`floor`/`autoselect_low`/`autoselect_high`/`helper_default`/`env_pin`) |
| `os_model.escalated` | WARNING | `persona`, `channel`, `from`, `to`, `reason` (`autocompact-thrash`/`context-overflow`/`http-400`) |

`_FORBIDDEN_FIELDS` (global): `prompt`, `prompt_text`, `system_prompt`,
`system_prompt_text`, `body`, `payload`, `final_text`. Per-Event-
Allow-List + Forbidden-Set werden im Emitter strict erzwungen
(raise `OsModelAuditFieldNotAllowed` bei smuggled fields).

Mirror der Layer-23 / Layer-25 / Layer-28 / Layer-29-Regel: der
Audit-Chain dokumentiert **dass** ein Modell gewählt wurde,
**nicht** den Prompt-Inhalt. `estimate_chars` (Zahl) ist genug
für Operator-Forensics; ein 200-Char-Prompt-Snippet würde
DSGVO-Grenzwertig.

### Prometheus (ADR-0007 Phase 6 Integration)

Zwei neue Metric-Familien in `atelier_gateway/audit_metrics.py`:

| Metrik | Type | Label allow-list |
|---|---|---|
| `atelier_os_model_selected_total` | counter | `model` ∈ `{haiku, sonnet, opus, other}`, `reason` ∈ `{explicit, floor, autoselect_low, autoselect_high, helper_default, env_pin}` |
| `atelier_os_model_escalated_total` | counter | `from`, `to` ∈ `{haiku, sonnet, opus}`, `reason` ∈ `{autocompact-thrash, context-overflow, http-400}` |

`other` ist das Catch-All für unbekannte Modelle (z.B. eine
zukünftige Sonnet-Variante, die im allow-list noch nicht
gemappt ist). Cardinality bleibt strukturell bounded.

### Grafana-Panels

`docs/observability/grafana/atelier-overview.json` bekommt zwei
neue Panels:

| Panel | Query (PromQL) | Sinn |
|---|---|---|
| **OS Model Selection (1h)** | `sum by (model) (rate(atelier_os_model_selected_total[1h]))` als Stacked Area | Operator sieht Mix Haiku/Sonnet/Opus über Zeit |
| **OS Model Escalations / 5min** | `sum by (reason) (rate(atelier_os_model_escalated_total[5m]))` | Alarm wenn anhaltend > 1/min → Schwellwert zu hoch, Threshold senken |

Beide Panels haben den existierenden `tenant`-Variable-Filter
für per-Tenant-View (mehrmandantenfähig wie alle anderen
Phase-6 Panels).

### Operator-Knöpfe

| Env-Var | Default | Effekt |
|---|---|---|
| `ATELIER_OS_MODEL_OVERRIDE` | _(unset)_ | **Kill-Switch / Notbremse.** Wenn gesetzt, gewinnt dieses Modell vor allem (Tier 1) — auch vor `model:`-Feld. Ersetzt strukturell die Phase-2-`ATELIER_HELPER_MODEL_OS_TURN`-Notbremse. Werte = `claude-sonnet-4-6` / `claude-haiku-4-5-20251001` / etc. Leer = unset = kein Override. |
| `ATELIER_OS_MODEL_AUTOSELECT` | `on` | Master-Knopf für Tier 3 (Adaptive). `off` → Resolver fällt auf Tier 4 (subscription default) zurück. Notfall-Knopf falls Autoselect Probleme macht. |
| `ATELIER_OS_MODEL_LOW` | `claude-haiku-4-5-20251001` | Low-Tier-Modell |
| `ATELIER_OS_MODEL_HIGH` | `claude-sonnet-4-6` | High-Tier-Modell |
| `ATELIER_OS_MODEL_THRESHOLD_CHARS` | `60000` | Schaltschwelle. Clamped [20_000, 200_000]. |
| `ATELIER_OS_MODEL_RETRY_ON_THRASH` | `on` | Backstop B aktiv |

Per-Persona-JSON-Feld:
- `os_model_floor`: `"haiku"` (unset) / `"sonnet"` / `"opus"`

**Phase-2-Knöpfe (entfernt in 29.5.3h):**
- ~`ATELIER_HELPER_MODEL_OS_TURN`~ — wird in 29.5.3h aus `service.env` und aus `helper_model.py::SITE_OS_TURN` entfernt
- ~`persona.helper_model_default: true`~ — wird in 29.5.3h aus `orchestrator-haiku.json` entfernt; Persona behält explizites `model: "claude-sonnet-4-6"`

### Cost-Modell (Schätzung, Soak-Phase verifiziert)

Annahme: 80 % der Bridge-Turns sind unter 60k chars Initial-Payload
(Routing, kurze Replies, /btw, Status-Fragen, Begrüßungen, Tool-
Output-Reformulierung). 20 % sind darüber (Code-Reviews mit großem
Skill-Block, Forge-Tool-Listings, Layer-X-Diskussionen mit großer
Recall-History).

Heutige Kosten (alles Sonnet via service.env):
- 80 % × Sonnet + 20 % × Sonnet = 100 % Sonnet-Preise

Nach Phase 3 (Autoselect):
- 80 % × Haiku + 20 % × Sonnet
- Haiku ~ Sonnet × 0.2 (Input) / × 0.25 (Output)
- Gesamtkosten: 0.8 × 0.22 + 0.2 × 1.0 ≈ 0.376
- **~62 % Kostenreduktion** gegenüber Phase-2-Stand
- ~88 % Kostenreduktion gegenüber Phase-0-Stand (alles Opus)

Diese Zahlen brauchen 14-Tage-Soak zur Verifikation. Bei
Operator-Beschwerde "fühlt sich dumm an" → `ATELIER_OS_MODEL_AUTOSELECT=off`
als Notbremse, fällt auf Phase-2-Verhalten zurück.

---

## Implementation Plan — neun sub-phases

| Phase | Was | LOC ≈ | Test-Cases | Abhängigkeiten |
|---|---|---|---|---|
| **29.5.3a** | `model_selector.py` — pure Logic | 180 | 22 | – |
| **29.5.3b** | `_resolve_os_model` **ersetzen** (4-Tier mit OVERRIDE, Phase-2-Pfade raus) | 70 | 10 (in `test_adapter_os_model.py`) | 29.5.3a |
| **29.5.3c** | `_resolve_spawn_inputs` ruft Schätzer (RAM-Pass-Through) | 30 | 4 (in `test_adapter_*.py`) | 29.5.3b |
| **29.5.3d** | Audit-Events | 100 | 8 | 29.5.3c |
| **29.5.3e** | Retry-Backstop | 80 | 5 | 29.5.3d |
| **29.5.3f** | Prometheus | 60 | 6 (in `test_audit_metrics.py`) | 29.5.3d |
| **29.5.3g** | Grafana-Panels | 50 (JSON) | Parity-Check via `test_observability_dashboards.py` | 29.5.3f |
| **29.5.3h** | **Phase-2-Cleanup** — `helper_model_default` aus `orchestrator-haiku.json`, `ATELIER_HELPER_MODEL_OS_TURN` aus `service.env`, `SITE_OS_TURN` aus `helper_model.py::ALL_SITES`, Phase-2-Tests entfernen | 100 (löschend) | 4 (Regression: alte Knöpfe sind kein no-op-Pfad mehr) | 29.5.3a–g grün **+ 7 Tage soak** |
| **29.5.3i** | ADR auf `Accepted`, CLAUDE.md L29.5 Phase 3 ergänzt, Phase-2-Section entfernt | 250 (Markdown) | – | 29.5.3h grün |

**Per-Phase-E2E-Rule:** Jede sub-phase ist ein eigener Commit
mit eigener grüner Test-Suite. Mirror der Layer-29 / Layer-30
Discipline. Keine sub-phase wird gemerged, bevor die vorherige
soak-stabil ist.

### Phase 29.5.3a — `model_selector.py` (pure Logic)

Neues Modul `operator/bridges/shared/model_selector.py`:

```python
from __future__ import annotations
import json
import os
from pathlib import Path
from typing import Final

DEFAULT_LOW: Final[str] = "claude-haiku-4-5-20251001"
DEFAULT_HIGH: Final[str] = "claude-sonnet-4-6"
DEFAULT_THRESHOLD_CHARS: Final[int] = 60_000
_MIN_THRESHOLD: Final[int] = 20_000
_MAX_THRESHOLD: Final[int] = 200_000
_INTERNAL_OVERHEAD_CHARS: Final[int] = 10_000

# Curated set — Patterns die "claude hat zu wenig Context"
# signalisieren. Erweiterung nur mit E2E-Case + concreteem
# Error-Sample.
_CONTEXT_ERROR_PATTERNS: Final[tuple[str, ...]] = (
    "Autocompact is thrashing",
    "prompt is too long",
    "context_length_exceeded",
    "input length",        # Anthropic API standard phrase part
)

_MODEL_RANK = {
    "claude-haiku-4-5-20251001": 1,
    "claude-haiku-4-5":          1,
    "claude-sonnet-4-6":         2,
    "claude-opus-4-7":           3,
}
_FLOOR_TO_MODEL = {
    "haiku":  DEFAULT_LOW,
    "sonnet": DEFAULT_HIGH,
    "opus":   "claude-opus-4-7",
}

def estimate_os_turn_chars(
    prompt: str,
    system_prompt: str,
    mcp_config_text: str = "",
    session_dir: Path | None = None,
) -> int: ...

def _session_history_bytes(session_dir: Path | None) -> int: ...

def autoselect_os_model(
    payload_chars: int,
    *,
    threshold: int | None = None,
    low: str | None = None,
    high: str | None = None,
) -> str: ...

def autoselect_enabled() -> bool: ...

def threshold_chars() -> int: ...
def low_model() -> str: ...
def high_model() -> str: ...
def retry_on_thrash_enabled() -> bool: ...

def is_context_error(error_text: str) -> bool: ...
def apply_floor(chosen: str, floor: str | None) -> str: ...
```

**Cost-Contract:** Modul MUSS NICHT `import anthropic`. Reine
stdlib + os.environ + pathlib. CI-Lint (AST-Walk) in
`test_model_selector.py::NoSdkImportContractTests` blockt
forbidden imports — selbe pattern wie `dialectic.py`,
`helper_model.py`, `user_model.py`.

**E2E-Surface** (`test_model_selector.py`, ~20 Cases):

`EstimateTests` (5):
- estimate sums prompt + system + mcp_config + overhead
- session_dir None → 0 added
- session_dir missing file → 0 added
- session_dir present → exact byte count added
- session_dir unreadable (permission) → 0 added + no raise

`AutoselectTests` (4):
- payload < threshold → LOW
- payload == threshold → LOW (edge)
- payload > threshold → HIGH
- threshold/low/high args override defaults

`EnvTests` (5):
- `ATELIER_OS_MODEL_AUTOSELECT=on` → True
- `ATELIER_OS_MODEL_AUTOSELECT=off` → False
- `ATELIER_OS_MODEL_THRESHOLD_CHARS=10000` → clamped to 20_000
- `ATELIER_OS_MODEL_THRESHOLD_CHARS=500000` → clamped to 200_000
- `ATELIER_OS_MODEL_THRESHOLD_CHARS=invalid` → falls back to default

`ContextErrorTests` (3):
- known pattern matches case-insensitive
- unknown pattern returns False
- empty / None / non-string input returns False

`FloorTests` (3):
- floor=`"sonnet"`, chosen=`"haiku"` → upgrades to sonnet
- floor=`"haiku"`, chosen=`"sonnet"` → keeps sonnet
- floor=`None` / `""` → returns chosen as-is

`NoSdkImportContractTests` (1):
- AST walk: kein `anthropic` / `openai` / `google.*` import

### Phase 29.5.3b — `_resolve_os_model` ersetzen (4-Tier, Phase 2 raus)

Adapter-`_resolve_os_model` wird **strukturell ersetzt**, kein
Dual-Path. Phase-2-Pfade (helper_model_default + env-wide
opt-in) sind weg.

```python
def _resolve_os_model(
    profile: dict | None,
    *,
    payload_chars: int,  # required — kein None-Fallback mehr
) -> str | None:
    """Layer 29.5 Phase 3 — 4-Tier adaptive OS model selection.

    Resolution order (top wins):
      1. ATELIER_OS_MODEL_OVERRIDE env       → kill-switch
      2. profile.model OR persona.model      → explicit pin
      3. autoselect + payload_chars + floor  → adaptive
      4. None → CLI default                  → subscription
    """
    # Tier 1 — operator-wide kill-switch
    override = _model_selector.os_model_override()
    if override:
        return override

    # Tier 2 — explicit per-persona / per-chat-profile pin
    profile = profile or {}
    explicit = profile.get("model")
    if isinstance(explicit, str) and explicit.strip():
        return explicit.strip()

    # Tier 3 — adaptive autoselect
    if _model_selector.autoselect_enabled():
        try:
            chosen = _model_selector.autoselect_os_model(payload_chars)
            floor = profile.get("os_model_floor")
            chosen = _model_selector.apply_floor(chosen, floor)
            return chosen
        except _model_selector.EstimateError:
            # Edge-case Q7: estimate failure → safe-default HIGH
            return _model_selector.high_model()  # Sonnet

    # Tier 4 — fallthrough to CLI subscription default
    return None
```

**E2E-Surface** (`test_adapter_os_model.py` — alte 19 Phase-2-
Cases werden ersetzt durch ~10 Phase-3-Cases):

- Tier 1: `ATELIER_OS_MODEL_OVERRIDE=opus-4-7` → opus-4-7,
  auch wenn `profile.model=haiku` (Override gewinnt vor explicit)
- Tier 2: explicit `model:` ohne Override → das Modell
- Tier 3 a: payload unter threshold → LOW
- Tier 3 b: payload über threshold → HIGH
- Tier 3 c: payload + floor=sonnet + chosen=haiku → sonnet
- Tier 3 d: `AUTOSELECT=off` → fällt auf Tier 4 (None)
- Tier 3 e: estimate-Error → HIGH (safe-default)
- Tier 4: nichts gesetzt + `AUTOSELECT=off` → None
- profile=None → kein crash
- forge persona round-trip: explicit floor=sonnet + payload-low → Sonnet

### Phase 29.5.3c — `_resolve_spawn_inputs` ruft den Schätzer

`_resolve_spawn_inputs` baut bereits `prompt` und `system_prompt`.
Was noch fehlt: `mcp_config_text` (wird in `_build_args` materialisiert)
und `session_dir`.

Lösung: Schätzer-Aufruf **nach** `_resolve_spawn_inputs`, **vor**
`_build_args`. Das `_resolve_spawn_inputs`-Output bekommt ein
neues Feld `"payload_chars"`:

```python
# in _resolve_spawn_inputs, am Ende:
mcp_config_text = ""
if mcp_config_path:
    try:
        mcp_config_text = Path(mcp_config_path).read_text(encoding="utf-8")
    except OSError:
        mcp_config_text = ""
payload_chars = _model_selector.estimate_os_turn_chars(
    prompt=prompt,
    system_prompt=system_prompt_full,
    mcp_config_text=mcp_config_text,
    session_dir=session_dir,
)
return {
    ...
    "payload_chars": payload_chars,
    "model": _resolve_os_model(profile, payload_chars=payload_chars),
}
```

**E2E-Surface** (im bestehenden `test_adapter_os_model.py`, +4 Cases):
- end-to-end: kleine inbox-msg → `model` ist LOW
- end-to-end: riesige skill-inject → `model` ist HIGH
- end-to-end: MCP-config-text wird mitgezählt
- end-to-end: session_dir mit großer .claude.json → zählt mit

### Phase 29.5.3d — Audit-Events

`security_events.py::EVENT_SEVERITY` erweitern:
```python
"os_model.selected":   "INFO",
"os_model.escalated":  "WARNING",
```

Emitter in `model_selector.py` (NICHT in adapter — keep selector
self-contained):

```python
_ALLOWED_FIELDS_SELECTED: Final[frozenset[str]] = frozenset({
    "persona", "channel", "estimate_chars", "chosen", "reason",
})
_ALLOWED_FIELDS_ESCALATED: Final[frozenset[str]] = frozenset({
    "persona", "channel", "from", "to", "reason",
})
_FORBIDDEN_FIELDS: Final[frozenset[str]] = frozenset({
    "prompt", "prompt_text", "system_prompt", "system_prompt_text",
    "body", "payload", "final_text",
})

def emit_selected(...) -> None: ...
def emit_escalated(...) -> None: ...
```

Strict-Boundary in `_validate_details` — raise
`OsModelAuditFieldNotAllowed` bei smuggled fields.

**E2E-Surface** (`test_model_selector.py`, +8 Cases):
- emit_selected happy path, alle felder durch
- emit_selected mit forbidden `prompt` → raises
- emit_selected mit unbekanntem feld → raises
- emit_escalated happy path
- emit_escalated mit forbidden body → raises
- chain-integrity: 10 events round-trip via `verify_chain`
- reason ist auf curated set beschränkt → unknown reason raises
- chosen ist auf curated set beschränkt → unknown model fällt auf `"other"`

### Phase 29.5.3e — Retry-on-Thrashing-Backstop

In `_call_claude_streaming_via_engine`:

```python
def _call_claude_streaming_via_engine(...):
    chosen_model = kwargs.get("model")  # vom Phase-3c Output
    try:
        result = _spawn_and_collect(chosen_model, ...)
    except (EngineSpawnFailed, EngineStreamingError) as e:
        if not _model_selector.retry_on_thrash_enabled():
            raise
        retry_model = _model_selector.escalate_for_error(
            str(e), current=chosen_model,
        )
        if retry_model is None:
            raise
        _model_selector.emit_escalated(
            persona=persona, channel=channel,
            from_=chosen_model, to=retry_model,
            reason=_classify_error(str(e)),
        )
        result = _spawn_and_collect(retry_model, ...)  # 1× retry max
    return result
```

`escalate_for_error` ist die single-source-of-truth: prüft
`is_context_error` + Rank-Check (nur upgrade, nie downgrade) +
gibt None bei No-go.

**E2E-Surface** (`test_adapter_engine_path.py` erweitern, +5 Cases):
- fake-claude crasht mit "Autocompact thrashing" → retry mit
  HIGH succeeds → final_text ist die HIGH-Response
- fake-claude crasht mit 500 (network) → keine Retry, raises
- fake-claude crasht ATELIER_OS_MODEL_RETRY_ON_THRASH=off → kein Retry
- fake-claude HIGH crasht auch mit "Autocompact thrashing" → kein
  zweiter Retry (max 1 pro Turn), raises
- Audit `os_model.escalated` lands in chain mit reason

### Phase 29.5.3f — Prometheus

`atelier_gateway/audit_metrics.py`:
- Aggregator-Funktionen für die zwei neuen Events
- Label-Allow-List erweitern (`model`, `reason`, `from`, `to`)
- TTL-Cache-Verhalten bleibt unverändert

**E2E-Surface** (`test_audit_metrics.py`, +6 Cases):
- selected→counter increments per (model, reason)
- escalated→counter increments per (from, to, reason)
- unknown model → label="other"
- unknown reason → metric NICHT emitted (statt other — würde
  Label-Cardinality verwässern)
- cache invalidation funktioniert
- per-tenant isolation

### Phase 29.5.3g — Grafana-Panels

Zwei neue Panel-Definitions in
`docs/observability/grafana/atelier-overview.json`:
- "OS Model Selection (1h)" — Stacked Area Time Series
- "OS Model Escalations / 5min" — Stat-Panel mit Threshold-Alarm

`test_observability_dashboards.py` fängt Drift automatisch (jeder
referenzierte Metric-Name muss in `audit_metrics`-Output existieren).

### Phase 29.5.3h — ADR-0024 close + CLAUDE.md-Section Layer 29.5 Phase 3

- ADR-0024 status `Proposed` → `Accepted`
- CLAUDE.md-Section "Layer 29.5 Phase 3 — Adaptive OS-Model Selection"
  analog Phase 2, mit:
  - 5-Tier Resolution-Tabelle
  - Schätzer-Formel + Beispiel
  - Threshold-Mathe (60k chars Default)
  - Audit-Events + Allow-List
  - Prometheus-Labels
  - Operator-Knöpfe
  - "Was du, als Claude Code, NICHT machen darfst" — siehe unten

---

## Was du, als Claude Code, NICHT machen darfst (Layer 29.5 Phase 3)

- **Don't put the prompt or system_prompt body into any
  `os_model.*` audit-event detail field.** Per-Event-Allow-List
  + `_FORBIDDEN_FIELDS` in `model_selector.py::_validate_details`
  sind die structural defence. Regression gate:
  `test_emit_rejects_prompt_field`.
- **Don't auto-retry on non-context errors (5xx, network,
  user-cancel).** `is_context_error` matched nur curated Patterns.
  Erweiterung der Pattern-Liste braucht echten Error-String und
  einen E2E-Case mit dem konkreten Sample. Endless-Loop-Risiko.
- **Don't retry more than once per turn.** Max-1-Retry ist der
  load-bearing Stop. Wenn auch HIGH crasht, ist der Fehler echt
  und der User soll ihn sehen — silently doubling tokens wäre
  schlimmer als ein klarer Fehler.
- **Don't widen the model whitelist beyond Haiku/Sonnet/Opus
  without an ADR amendment.** Neue Modelle (z.B. eine zukünftige
  Sonnet-Variante) brauchen einen Rank-Eintrag, einen Floor-
  Mapping-Eintrag, ein Label-Allow-List-Update UND einen
  Migrationspfad für bestehende Personas. Silent-widening
  fragmentiert die Cost-Modell-Annahmen.
- **Don't lower `_MIN_THRESHOLD` below 20_000 chars.** Sub-20k
  ist garantiert in Haikus Comfort-Zone (5k Tokens initial-
  budget lassen 195k tokens für Tool-Chain). Untergrenze ist die
  structurelle "selektor kann nichts kaputtmachen"-Garantie.
- **Don't change `_resolve_os_model`'s call signature without
  preserving payload_chars=None semantics.** Phase-2-Aufrufer
  müssen weiterlaufen ohne Code-Änderung. Eine `*`-Trennung +
  Default-Wert = vertraglich abgesicherter Back-Compat-Pfad.
- **Don't `import anthropic` in `model_selector.py`.** Cost-
  Contract mirror der Layer-11 / 29.5-Phase-1 / Layer-28 Regel.
  AST-Lint enforced.
- **Don't fold the escalation retry into the dispatcher of a
  delegate worker.** Worker-Engines haben ihre eigene
  Model-Auswahl (Layer 22 + Layer 29). Der OS-Turn-Retry ist
  strikt OS-Turn-only.
- **Don't make `os_model_floor` per-chat-profile-overridable.**
  Floor ist eine Persona-Eigenschaft. Per-Chat-Override würde
  bedeuten, dass ein Owner via `chat_profile.os_model_floor=haiku`
  die Persona's strukturelle Pin-Entscheidung silently
  unterläuft. Wenn das mal nötig ist, neue ADR mit Audit-Event
  `os_model.floor_overridden` für Forensics.
- **Don't make `ATELIER_OS_MODEL_AUTOSELECT=off` the default
  for any new install.** Default-on ist die published baseline.
  Operator-side off ist die Notbremse, nicht der Default-Modus.
- **Don't emit `os_model.selected` for Worker-Engine-spawns
  (delegate_*).** Das Event ist OS-Turn-spezifisch. Worker
  bekommen ihre Model-Auswahl von der Delegate-MCP-Schicht
  durchgereicht; ein eigenes Event-Type-Set würde die Audit-
  Chain saturate für eine Information, die in
  `delegate.invoked` schon implicit steht (via Model-Parameter).
- **Don't merge phases out of order.** 29.5.3a ist
  load-bearing prerequisite für b/c/d/e/f/g. Jede sub-phase ist
  ein eigener Commit mit eigener grüner Test-Suite. Mirror der
  Layer-29 / Layer-30 Discipline.

---

## Out of Scope (für Phase 3, geplant als ADR-0025 / 0026)

- **ADR-0025 — Audit-Chain Trim mit cryptographic Anchoring**
  (Phase 4, operator-bestätigt 2026-05-16). Generischer
  Trim-Mechanismus für alle high-frequency events (`os_model.*`,
  `bridge.btw_inject`, etc.) mit Hash-Chain-überlebender
  Anchor-Event-Lösung. Sobald `voice-audit verify` durch das
  steigende Volumen merklich langsamer wird (Schwellwert: > 5 s
  auf einem typischen Operator-Setup) startet die ADR. Bis
  dahin: volle Chain.
- **1M-Context-Variante** (Sonnet 4.6 mit `anthropic-beta:
  context-1m-2025-XX-XX` Header) als dritte Tier. 1M kostet 2×
  pro Token, lohnt nur für die wirklich-großen Tasks. Wenn
  Phase-3-Soak zeigt, dass auch Sonnet 200k regelmäßig trash't,
  dann eigene ADR für 1M-Variante.
- **Pre-Spawn-LLM-Judge** ("Frage Haiku ob die Task hart ist und
  dann entscheide"). Aktuell kein Bedarf — der Byte-Schätzer ist
  empirisch gut genug, und ein LLM-Judge addiert 3-5 s Latenz
  pro OS-Turn.
- **Streaming-aware mid-turn Model-Switch.** Falls Tool-Output
  during stream zu groß wird, könnte man theoretisch
  "mittendrin" auf Sonnet wechseln. Aber: claude-CLI lässt das
  nicht zu (Modell ist per spawn). Erfordert eine architekto-
  nische Änderung am `claude -p`-Layer, die wir nicht treiben.
- **Per-Tenant Cost-Budgets** (Layer 20 Quota → Layer 29.5.3
  Integration). Wenn ein Tenant ein Sonnet-Budget hat, könnte
  Phase 3 das berücksichtigen. Heute nicht implementiert; gehört
  in eine eigene ADR (Layer 20 Phase 4).

---

## Migration und Rollout

### Vor Phase-3-Merge
- Heutiger Stand (2026-05-16): `service.env` carries
  `ATELIER_HELPER_MODEL_OS_TURN=claude-sonnet-4-6`. Phase-2-Pfad
  greift, alle Bridge-Turns auf Sonnet 4.6.

### Phase-3a-bis-3h Soak
- Jede sub-phase: lokal in `outputs/`-Vorschau, Operator
  reviewed, kopiert manuell ins live-Tree (path-gate blockt
  direkten Edit auf persona-JSONs).
- Tests grün → commit → 24h soak in der Operator-Umgebung →
  nächste sub-phase.

### Nach Phase-3h-Merge
- `ATELIER_HELPER_MODEL_OS_TURN` bleibt in service.env als
  legacy-Knopf (Phase-2-Pfad). Solange `ATELIER_OS_MODEL_AUTOSELECT=on`,
  überstimmt Phase 3 ihn aber für Personas ohne explicit `model:`.
- Operator kann jederzeit `AUTOSELECT=off` setzen und auf
  Phase-2-Verhalten zurückfallen.

### Rollback (falls Phase 3 in der Soak-Phase Probleme zeigt)
- `ATELIER_OS_MODEL_AUTOSELECT=off` in service.env
- Adapter restart (Hot-Reload sieht die Variable beim nächsten
  Spawn)
- Phase-2-Pfad greift wieder. Keine Datei-Rollbacks nötig.

### Hard-Cut (Phase-3 als Default für alle Operatoren)
- Frühestens 14 Tage nach 29.5.3h-Merge.
- Trigger: `os_model.escalated`-Rate < 0.1 % aller Turns, keine
  Operator-Beschwerden, audit-chain-integrität durchgehend grün.
- Action: `ATELIER_OS_MODEL_AUTOSELECT` als Default `"on"` in
  die `bootstrap.sh`-Persistenz-Liste aufnehmen.

---

## Cost-Vergleich (Erwartet)

Annahme-Verteilung (basierend auf typischen Bridge-Logs):

| Kategorie | Anteil | Initial-Bytes typisch | Modell unter Phase 3 |
|---|---|---|---|
| Begrüßung, Status, /btw | 25 % | ≤ 5k chars | Haiku |
| Auto-Routing, Tool-Reply, Bestätigung | 25 % | 5-15k chars | Haiku |
| Mittlere Coder/Forge-Anfrage | 30 % | 15-50k chars | Haiku |
| Große Code-Reviews, Forge-Tool-Listings | 15 % | 50-80k chars | Haiku (knapp) → Retry-B greift bei ~3 % |
| Layer-X-Diskussionen mit großem Recall/Skill-Block | 5 % | > 80k chars | Sonnet |

Modell-Mix nach Phase 3: ~95 % Haiku, ~5 % Sonnet, ~0 % Opus.

| Stand | Effektive Kosten pro 1000 Turns |
|---|---|
| Phase 0 (Opus everywhere) | 100 (baseline) |
| Phase 2 (Sonnet everywhere, heute) | ~33 |
| Phase 3 (Adaptive) | ~10 |

→ **~70 % Kostenreduktion gegenüber Phase 2, ~90 % gegenüber
Phase 0.** Diese Zahlen brauchen Soak-Verifikation.

---

## Beziehung zu anderen Layern

| Layer | Beziehung |
|---|---|
| L11 dialectic | Pattern für Cost-Aware-Entscheidung (heat-score → mode); Phase 3 mirror'd das auf model-selection |
| L22 WorkerEngine | Worker-Engines haben EIGENE Model-Auswahl. Phase 3 ist strikt OS-Turn-only. |
| L23 voice-transcribe | Metadata-only-audit precedent |
| L25 compute worker | Operator-OS-Turn-Kosten sind separater Knopf als per-tenant compute-budgets |
| L28 user-model | Liefert `<user_context>`-Block; trägt zu `system_prompt`-Size bei → Phase-3-Schätzer berücksichtigt es automatisch |
| L29 delegation | OS-Turn-Modell ist orthogonal zu Worker-Modell. Phase 3 ändert OS, nicht Worker. |
| L29.5 Phase 1 | Helper-Subprozesse (voice-summary, dialectic-judge etc.) — Phase 3 berührt sie nicht, sie haben eigenes SITE_*-Schema. |
| L29.5 Phase 2 | Phase 3 ist erweiterung; Phase-2-Pfade bleiben byte-identisch unter `AUTOSELECT=off`. |
| L30 forge-skillforge | Hinzugefügte MCP-Schemas in `system_prompt` → größerer Initial-Bytes → Phase 3 wählt eher Sonnet → bezahlte Optionalität |

---

## Resolved Decisions (Operator-Antworten 2026-05-16)

Alle ursprünglichen Open Questions sind beantwortet. Status
kann nach 29.5.3i-Merge direkt auf `Accepted` gehen.

1. **Q: `session_dir`-Bytes-Schätzung — wie tief lesen?**
   **A: Rekursiv** über `rglob('*')` mit Hard-Cap 5 MB und
   Early-Exit. Über 5 MB ist Haiku unabhängig vom Wert raus;
   weiter zählen ist Verschwendung. Implementiert in
   `_session_history_bytes` (Decision-Section oben).

2. **Q: `mcp_config_text`-Lesen — Datei vs RAM?**
   **A: RAM-Pass-Through** — der ohnehin gebaute String wird
   direkt durchgereicht, keine Disk-Re-Lesung. Performanter UND
   robuster (kein TOCTOU, kein Cache-Drift, was geschrieben
   wird IST was geschätzt wird).

3. **Q: Persona-Floor — welche Bundle-Personas?**
   **A: Nur `forge` → `sonnet`**. Alle anderen Bundle-Personas
   bleiben unset → reines Auto-Select. Forge ist safety-kritisch
   (Tool-Creation + Policy-Edits), Sonnet's Reasoning ist hier
   den geringen Aufpreis wert. Floor-Tabelle in Decision-Section
   oben.

4. **Q: Audit-Event-Volumen — trim-Mechanismus?**
   **A: Eigene ADR-0025 in Phase 4**, NICHT in Phase 3 bundled.
   Audit-Trim ist ein generischer Mechanismus mit cryptographic
   anchoring (Hash-Chain muss überleben), gehört strukturell
   sauber designed — touch nicht nur `os_model.*` sondern alle
   high-frequency events. Phase 3 läuft bis dahin mit voller
   Chain (3 MB JSONL/Monat bei 1000 Turns/Tag ist `voice-audit
   verify` ohne Probleme).

5. **Q: `reason="helper_default"` / `"env_pin"` für Phase-2-
   Fall-Through?**
   **A: Kein Legacy-Code.** Phase 2 wird in 29.5.3h vollständig
   entfernt — `helper_model_default`-Feld raus aus
   `orchestrator-haiku.json`, `ATELIER_HELPER_MODEL_OS_TURN`
   raus aus `service.env`, `SITE_OS_TURN` raus aus
   `helper_model.py::ALL_SITES`. Damit existieren die zwei
   `reason`-Werte gar nicht mehr im Code. Stattdessen vier
   curated Werte: `override` / `explicit` / `autoselect_low` /
   `autoselect_high` / `floor` / `estimate_failed`.

**Zusatz-Entscheidung (Q6, operator-genehmigt 2026-05-16):**
**`ATELIER_OS_MODEL_OVERRIDE`** als Tier-1-Kill-Switch ist
load-bearing. Ersetzt strukturell die heutige
`ATELIER_HELPER_MODEL_OS_TURN`-Notbremse als sauberer Knopf
(`OVERRIDE` ist semantisch klarer als `HELPER_MODEL_OS_TURN`).
Gewinnt vor `model:`-Feld bewusst — Incident-Response ohne
Persona-Touch.

**Zusatz-Entscheidung (Q7, intern entschieden):**
**Edge-Case `estimate_failed` → HIGH** (nicht LOW). Wenn
Schätzer crash't / None zurückgibt, fällt Tier 3 auf Sonnet
+ Audit-Event mit `reason="estimate_failed"`. Niemals LOW bei
Schätz-Failure — wäre ein stiller Bug-Maskierer.

---

## References

- Layer 29.5 Phase 2 — `_resolve_os_model`, `helper_model.py`,
  `test_adapter_os_model.py` (19 Cases)
- ADR-0022 — Pfeiler-Struktur (Skill-Block / MCP / Audit)
  als Stilvorbild für mehrteilige Layer-Erweiterungen
- ADR-0007 Phase 6 — Audit-Chain als Projektionsquelle für
  Prometheus
- ADR-0013 — Compute-Worker als Vorbild für "claude -p ist
  kein anthropic-SDK-Aufruf" cost-contract
- `operator/bridges/shared/adapter.py:1244` — heutige
  `_resolve_os_model`-Implementierung
- `operator/bridges/shared/helper_model.py:76` —
  `resolve_helper_model` als Vorbild für Resolver-Pattern
- `operator/bridges/shared/dialectic.py` — heat-score-Pattern
  für quantitative Cost-Aware-Entscheidungen
