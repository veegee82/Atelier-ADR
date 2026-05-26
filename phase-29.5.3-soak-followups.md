# Phase 29.5.3 — soak followups (ADR-0024 Adaptive OS Model Selection)

Status snapshot at commit `1f577f1` (2026-05-16, "feat(adr-0024):
Layer 29.5 Phase 3 adaptive OS-turn model selection"). Phase 3 is
**code-complete für 3a-3g und 3i**; **3h ist deferred** für 14-Tage-
Soak. Diese Datei trackt was zwischen Live-Go (heute) und vollem
Phase-3-Default-Mode (in ~2 Wochen) noch passieren muss.

Siehe `docs/decisions/0024-adaptive-os-model-selection.md` für den
vollen Plan, das Cost-Modell und die "What you must NOT do"-Liste.

## Live-Status (2026-05-16, post-restart pid 3291903)

| Komponente | Status |
|---|---|
| `model_selector.py` (Phase 3a) | live, 37 Tests grün |
| `_resolve_os_model` 4-Tier (Phase 3b) | live, 17 Tests grün |
| `_resolve_spawn_inputs` Wiring (Phase 3c) | live |
| Audit-Events `os_model.selected` / `os_model.escalated` (3d) | registered, allow-list aktiv |
| Retry-on-Thrashing-Backstop (Phase 3e) | live, max 1 retry/turn |
| Prometheus `atelier_os_model_*` (Phase 3f) | live, 2 metric families |
| Grafana panels (Phase 3g) | live in `atelier-overview.json` |
| CLAUDE.md Layer 29.5 Phase 3 section (Phase 3i) | live |
| **Safety-Pin** `ATELIER_OS_MODEL_OVERRIDE=claude-sonnet-4-6` | aktiv in `service.env` — alle Bridge-Turns auf Sonnet wie pre-Phase-3 |

## Followup-Trigger gates (für volle Phase-3-Aktivierung)

Alle drei müssen halten, bevor `ATELIER_OS_MODEL_OVERRIDE` aus
`service.env` entfernt wird:

- [ ] Schätzer kalibriert (siehe FU-1 unten) — synthetische
      Bridge-Turn-Tests zeigen realistische `payload_chars`-Werte
      die mit der echten Cache-Creation-Token-Anzahl korrespondieren.
- [ ] Mindestens 7 Tage Sonnet-Pin ohne 400-Errors in der
      Bridge-Adapter-Log (Negativ-Kontrolle: bevor Phase 3 das
      Problem verursacht hat, war die Linie clean).
- [ ] Operator-Decision: Cost-Saving via Autoselect ist die
      Kosten-Latenz-Trade-off wert. Bei "alles weiter auf Sonnet"
      bleibt OVERRIDE permanent gesetzt.

## FU-1 — Schätzer kalibrieren

Beobachtung am 2026-05-16:

- Manueller `claude -p --output-format stream-json --model
  claude-haiku-4-5-20251001 "sag hi"` zeigt `cache_creation_input_tokens:
  146914` im `result`-Event. Bei 3.5 chars/token = ~514k chars
  Initial-Overhead, gegen den der Adapter Phase-3-Schätzer mit
  `_INTERNAL_OVERHEAD_CHARS = 10_000` total daneben liegt.
- Vor Safety-Pin gab es zwei Bridge-Turns mit `engine streaming
  returned error: 400` (14:48 und 17:57) — beide auf `orchestrator`-
  Persona mit großer MCP-Config (forge + skill_forge + delegate_*).
  Hypothese: Schätzer sagt < 60k → Haiku → Initial-Real ~200k →
  400 von der Anthropic-API → Retry-Backstop greift nicht weil der
  Error-Text nur "400" ist (matched keine Context-Overflow-Pattern).

### Vorgehen für die Kalibrierung

1. **Messen statt schätzen.** Adapter um eine optionale `--audit-payload`
   debug-Mode erweitern, der bei jedem Spawn das echte
   `result.usage.cache_creation_input_tokens` aus dem Stream-JSON
   liest und in einem `os_model.spawn_real_usage` (INFO) Audit-Event
   loggt. Damit haben wir nach 100 Turns echte Verteilungsdaten.

2. **Threshold gegen reale Verteilung anpassen.** Wenn 80 % der Turns
   real ≤ 100k tokens sind, dann Schwellwert auf chars-Äquivalent
   setzen (= ~350k chars im Schätzer). Aktuell `60_000` chars =
   ~17k tokens → viel zu niedrig.

3. **`_INTERNAL_OVERHEAD_CHARS` realistischer setzen.** Vermutlich
   ~140_000 chars (≈ 40k tokens Tool-Schemas + Plugin-Boilerplate).
   Konkrete Zahl kommt aus Schritt 1.

4. **Retry-Backstop Pattern erweitern um `"400"` mit `usage`-Hint.**
   Wenn das `result`-Event `is_error=true` UND `api_error_status=400`
   UND `usage.input_tokens` nahe am Model-Max ist, ist es ein
   Context-Overflow auch ohne explizite "Autocompact thrashing"-
   Meldung. Engine `_normalise` in
   `bridges/shared/agents/claude_code.py` müsste dann den `error`-
   Event mit reicherer Diagnostik anreichern.

### Erfolgs-Kriterium

Nach Kalibrierung: synthetische "sag hi" mit dem realen orchestrator-
Persona-Setup liefert über den Adapter eine `os_model.selected`-
Audit-Entry mit `estimate_chars ≈ realer initial-token-Wert × 3.5`
(±20 %). Aktuell wahrscheinlich Faktor 10+ daneben.

## FU-2 — Phase 29.5.3h (Phase-2-Cleanup)

Trigger: FU-1 abgeschlossen UND OVERRIDE entfernt UND 7 Tage clean
ohne Regression.

Was rausgenommen wird (alles dead-code ab heute):

- `service.env`: `ATELIER_HELPER_MODEL_OS_TURN=claude-sonnet-4-6`
  (legacy Phase-2-Knopf, wird vom neuen Code nicht mehr gelesen)
- `operator/bridges/shared/helper_model.py`: `SITE_OS_TURN`
  constant + Entfernung aus `ALL_SITES`
- `operator/bridges/shared/test_helper_model.py` /
  `test_helper_model_sites.py`: alle Cases die `SITE_OS_TURN`
  prüfen
- `operator/cowork/personas/orchestrator-haiku.json`:
  `helper_model_default` ist bereits raus (Commit `1f577f1` /
  Operator-Kopie). Sicherheitscheck dass der Diff stabil ist.
- CLAUDE.md: Phase-2-Section entfernen (Layer 29.5 Phase 3 ist
  dann der einzige OS-Model-Mechanismus).

ADR-0024 Status: `Accepted` → bleibt; Migration-Section um
"Phase 2 vollständig entfernt am <Datum>" ergänzen.

## FU-3 — Drei unverwandte Test-Suite-Fails diagnostizieren

`bash operator/bridges/run-all-tests.sh` meldet `3 / 183
suite(s) failed`. Die drei Suites sind:

- ❓ noch zu identifizieren — der Runner druckt nur die Anzahl,
  nicht die Namen. Eine erste Diagnose zeigte 56 von 183 erwarteten
  Cyan-Headers im stdout — vermutlich Output-Buffering-Artifakt
  beim Runner, der `>/dev/null` Redirection breit anwendet.
- Vorgehen: `run-all-tests.sh` umbauen so dass bei Fail der
  Suite-Name in einer Summary-Liste am Ende erscheint. Heute meldet
  er nur den Counter.
- Wahrscheinlichkeits-Hypothese: pre-existing fragile Tests (z.B.
  live-Engine-Tests die Network brauchen), nicht durch ADR-0024
  ausgelöst — alle Phase-3-Suites sind einzeln grün getestet:
  - `test_model_selector.py` — 37/37
  - `test_adapter_os_model.py` — 17/17
  - `test_helper_model.py` — 17/17
  - `test_helper_model_sites.py` — 13/13
  - `test_audit_metrics.py` — 35/35

### Erfolgs-Kriterium

Runner gibt die Suite-Namen der Fails aus → wir wissen welche drei
es sind → wir entscheiden ob sie real fragil sind oder reparabel.

## FU-4 — Diagnose-Mode für payload_chars-Tracking (optional)

Ein opt-in Debug-Mode der bei jedem OS-Turn `os_model.spawn_real_usage`
emittiert:

```python
# in adapter, after stream completion, when ATELIER_OS_MODEL_DEBUG_PAYLOAD=on
if result_event:
    usage = result_event.get("usage", {})
    _model_selector.emit_real_usage(
        persona=persona,
        channel=channel,
        estimate_chars=payload_chars_recorded,
        real_input_tokens=usage.get("input_tokens", 0),
        real_cache_creation_tokens=usage.get("cache_creation_input_tokens", 0),
        chosen_model=chosen_model,
    )
```

Audit-Event Allow-List: `estimate_chars`, `real_input_tokens`,
`real_cache_creation_tokens`, `chosen_model`. Niemals prompt/system-
prompt content.

Gibt nach 100-200 Turns echte Verteilungsdaten für FU-1.

## Off-Soak-Aufgaben (vor Phase 3 voll-aktiv)

- [ ] FU-1 Schätzer-Kalibrierung — load-bearing für sichere
      Autoselect-Aktivierung
- [ ] FU-3 Test-Runner-Diagnose — sanity-check vor jedem größeren
      Commit-Schub
- [ ] FU-4 Diagnose-Mode (optional) — empfohlen wenn FU-1 ohne
      synthetische Tests schwierig ist

## Soak-Aufgaben (nach Trigger-Gates erfüllt)

- [ ] OVERRIDE aus `service.env` entfernen
- [ ] FU-2 Phase-2-Cleanup (29.5.3h)
- [ ] ADR-0024 mit "Phase 2 vollständig entfernt"-Eintrag
      updaten

## Rollback-Pfad (falls Phase 3 in der Soak Probleme macht)

1. `ATELIER_OS_MODEL_AUTOSELECT=off` in `service.env` setzen
2. Adapter restart
3. → Resolver fällt auf Tier 4 (None → CLI subscription default)
4. ABER: vorher OVERRIDE entfernen, sonst pinnt der weiter
5. Stable-State: Subscription-Default = Opus (kein Sonnet mehr).
   Das ist explizit-default-of-default und sicher.

Ein "Phase 2 zurück" gibt es per Definition nicht — Phase 2 ist
nicht mehr im Code. Rollback ist immer auf Tier 4 (subscription
default). Wenn das nicht akzeptabel ist, OVERRIDE auf den
gewünschten Wert setzen (= Phase-2-Verhalten unter neuem Knopf-
Namen).
