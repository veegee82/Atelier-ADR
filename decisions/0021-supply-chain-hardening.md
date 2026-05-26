# ADR-0021 — Supply-Chain-Härtung (Layer 31)

**Status:** Proposed
**Date:** 2026-05-15
**Companion to:** ADR-0007 Phase 5 (`.atelier-pkg` ed25519
Signatur), ADR-0019 (License-Signing-Pipeline — Vorbild für
Offline-Signing-Trust-Boundary), Layer 10 (path-gate — schützt
SBOM/Lock-Files), Layer 16 (audit-chain als Source of Truth),
Compliance-Baseline (EU AI Act 2026 Art. 15 — Robustness; GDPR
Art. 32 — Security of Processing).
**Implements:** Sechs Sub-Phasen, jede einzeln nutzbar:
31.1 SBOM-Generation, 31.2 Dependency-Hash-Gate, 31.3
CVE-Surveillance-Timer, 31.4 Plugin-Capability-Manifest, 31.5
Sigstore/Rekor-Anchoring (opt-in), 31.6 Audit-Events +
Prometheus-Metriken. Reproducible-Builds bleibt explizit out-of-scope
(Phase 31.7+ wenn Phase-VI/Cloud-Ops das verlangen).

---

## Context

AtelierOS supply-chain-baseline heute:

- **ADR-0007 Phase 5** signiert publizierte `.atelier-pkg`-Bundles
  mit ed25519, verifiziert per öffentlichem Key des Publishers.
- **`requirements.txt`** pinnt Versionen pro Plugin-Venv.
- **bwrap-Sandbox** isoliert Tool-Execution.
- **NOTICE-File** dokumentiert vendored Third-Party-Code (soft).

**Drei strukturelle Lücken bleiben:**

1. **Keine SBOM-Pflicht.** Ein Operator, der ein `.atelier-pkg`
   installiert, hat keine maschinenlesbare Übersicht der
   transitiven Dependencies. EU AI Act Art. 15 fordert
   "appropriate level of robustness"; ohne SBOM kann der Operator
   einer Regulator-Anfrage "welche Open-Source-Komponenten laufen
   in welcher Version" nicht beantworten ohne den Source-Tree zu
   inspizieren.
2. **Kein Hash-Gate beim pip install.** `requirements.txt` pinnt
   die Version aber NICHT die Artefakt-Hashes. Ein PyPI-Mirror,
   der eine kompromittierte `pydantic-2.5.0.tar.gz` ausliefert,
   würde unbemerkt installiert. Mirror-Substitution ist ein
   dokumentiertes Angriffsmuster (PyTorch-Dezember-2022,
   ctx-Mai-2022).
3. **Keine CVE-Überwachung.** Ein Plugin, das `cryptography==42.0.0`
   pinnt und sechs Monate später ein CVE-2026-XXX bekommt, läuft
   auf jedem Operator-Box weiter — bis der Operator manuell
   `pip-audit` ausführt. Daily-Audit-Verify (Roadmap L) deckt
   Audit-Chain-Tampering ab; CVE-Surveillance ist die parallele
   Disziplin für Dependency-Hygiene.

Comparable Frameworks — npm Audit, GitHub Dependabot, OpenSSF
Scorecard, SLSA Level 2/3 — adressieren diese Klassen seit Jahren.
AtelierOS muss aufholen, nicht-blocking, opt-in-friendly für
single-operator-Setups (die heute KEIN package-managment-tooling
zusätzlich installieren wollen).

Microsoft Defense-in-Depth (Mai 2026) listet "Supply-Chain
Compromise" als eigene Risikoklasse für autonome AI-Agenten. Die
Logik überträgt sich 1:1: ein kompromittiertes Forge-Tool aus
einem `.atelier-pkg` ist Code-Execution mit Tenant-Privileg.

---

## Decision

### Honest scope — what this layer is, and what it isn't

**Layer 31 ist Drift-Detection + Regulator-Paper-Trail.**
Es ist NICHT *active defense* gegen die häufigste real
beobachtete Klasse von Supply-Chain-Compromise:
**legitimate-author-account-compromise** (ctx, event-stream,
ua-parser-js, PyTorch torchtriton 2022). Bei dieser Klasse passt
der `pip install`-Hash, weil der bösartige Upload DER legitime
Upload ist. CVE-Detection schlägt Tage bis Wochen verspätet zu
(PyTorch: 5 Tage). In diesem Fenster läuft Layer 31 ehrlich
gesagt blind.

Was diese Klasse von Angriff strukturell verhindern würde:
- Reproducible Builds (Phase 31.7+, eigener ADR)
- Content-Trust-mandatory (Sigstore non-opt-in via Phase 31.5
  + Phase 31.7 zusammen)
- Sandboxed Dependency Execution (eBPF Egress-Verification, eBPF
  syscall-trace, Phase 31.8+)

Bis diese drei Phasen landen, leistet Layer 31:
- **Drift-Detection:** wenn Plugin X gestern `pydantic 2.5.3`
  installiert hatte und heute `2.5.4`, sieht der Operator den
  Hash-Wechsel im Audit + den Capability-Drift wenn Imports neu
  sind.
- **CVE-Forensik:** wenn ein PyPI-Package nachträglich als
  kompromittiert deklariert wird, hat der Operator den
  paper-trail zur Schadens-Eingrenzung.
- **Regulator-Defensibility:** EU CRA 2027, EU AI Act 2026
  Art. 15, GDPR Art. 32 — alle drei verlangen SBOM + 
  Vulnerability-Management-Pflichten. Layer 31 erfüllt sie.

Diese Framing-Ehrlichkeit ist load-bearing: jeder, der Layer 31
aktiviert, soll wissen, was er nicht bekommt. False sense of
security ist schlimmer als kein security claim.

### Mechanism overview

Sechs additive Schichten, alle **opt-in pro Operator** über neue
Felder in `tenant.atelier.yaml::spec.supply_chain` UND ein
operator-globales `<atelier_home>/global/supply_chain.yaml` für
Cross-Tenant-Defaults.

### A) SBOM-Generation @ build (Phase 31.1)

`atelier_gateway.packaging.PackageBuilder` erweitert um
SBOM-Emission:

- Format: **CycloneDX 1.5** JSON (de-facto-Standard, Apache-2.0
  Tooling, EU/CRA-konform).
- Inhalt:
  - Pinned Python-Deps (Name, Version, sha256-Hash der `.whl` oder
    `.tar.gz`).
  - Pinned npm-Deps (Name, Version, integrity-Hash aus
    `package-lock.json`).
  - Vendored Third-Party-Code aus NOTICE (Name, License, Version
    soweit bekannt).
  - Plugin-Source-Tree-Hash (sha256 des sortierten `path||NUL||
    content`-Stream, gleicher Algorithmus wie Phase-5
    `payload_sha256`).
- Datei: `sbom.cdx.json` im Bundle-Root, neben
  `manifest.atelier.yaml`.
- Wird mit-signiert: `payload_sha256` umfasst sbom.cdx.json
  automatisch (zero Code-Change am Signing-Path).

`package verify` erweitert um:

- `--require-sbom` (Default: true ab Phase 31.1.2): refuses
  Bundles ohne SBOM.
- Optional `--check-cves <feed>`: ruft `pip-audit
  --requirement <derived-from-sbom>` auf den extrahierten Deps,
  reportet CRITICAL/HIGH-Findings als Warning (kein
  install-blocking — Operator entscheidet).

Audit-Event `supply_chain.sbom_verified` (INFO) bei jedem
erfolgreichen Verify; `supply_chain.sbom_missing` (WARNING) bei
Bundles ohne SBOM, falls `--require-sbom` False (Operator hat
explizit opt-out).

### B) Dependency-Hash-Gate (Phase 31.2)

`bootstrap.sh` jedes Plugins wird modifiziert:

```bash
# Vor: pip install -r requirements.txt
# Nach:
pip install --require-hashes -r requirements.txt
```

`requirements.txt`-Files bekommen `--hash=sha256:…` pro Entry
via `pip-compile --generate-hashes` (pip-tools). Beispiel:

```
fastapi==0.136.0 \
    --hash=sha256:abcd... \
    --hash=sha256:ef01...   # multiple hashes for wheel + sdist
pydantic==2.5.3 \
    --hash=sha256:1234...
```

Mismatch → pip exits non-zero → bootstrap fail-closed. Kein
"warn and continue", kein Auto-Refresh der Hashes (das wäre
silent supply-chain-substitution).

**Operator-Workflow für Dep-Update:**

1. `pip-compile --upgrade --generate-hashes
   core/<name>/requirements.in` (oder `operator/<name>/requirements.in`
   je nach Plugin-Tree, operator-side, on a build box).
2. Resultierendes `requirements.txt` checken-in.
3. Audit-Event `supply_chain.dep_hashes_updated` (INFO) bei
   nächstem `bridge.sh up` (mit count + sha256 des neuen
   requirements-Files — nie Inhalt, nie Dep-Namen-Liste).

**Frozen-baseline mode** (operator-opt-in via
`<atelier_home>/global/supply_chain.yaml::supply_chain_mode: frozen`):
ein gesetzter `frozen`-Modus blockiert `bootstrap.sh` von ALLEN
Hash-Refreshes. Der Operator zwingt sich, jedes Update als
explizite Aktion durchzuziehen (`pip-compile --upgrade` MUSS
manuell laufen), und das resultierende `requirements.txt`-Update
emittiert ein zusätzliches Audit-Event
`supply_chain.frozen_baseline_breach_attempted` wenn ein
unautorisierter Hash-Drift erkannt wird. Sinnvoll für
high-assurance-Tenants (regulierte Branchen, air-gap), die jede
Dep-Änderung als bewusste Entscheidung dokumentieren wollen.
Default ist `unmanaged` (existing-behavior-Kompatibilität).

### C) CVE-Surveillance-Timer (Phase 31.3)

Zwei systemd-User-Timer parallel zum existierenden
`atelier-audit-verify.timer` (Layer 16 Roadmap L):

- `atelier-supply-chain-weekly.timer`
  (`OnCalendar=Mon *-*-* 05:00:00`, `Persistent=true`) — der
  **Default-Pfad**. Aggregiert MEDIUM/HIGH/CRITICAL findings als
  Wochen-Digest, ein Bridge-Notification pro Plugin pro
  Severity-Tier. Reduziert Alert-Fatigue auf ein
  Operator-Augen-Pensum.
- `atelier-supply-chain-critical.timer`
  (`OnCalendar=*-*-* 05:00:00`, `Persistent=true`) — der
  **Notfall-Pfad**. Triggert NUR bei neu auftauchenden CRITICAL-
  Findings (Diff gegen letzten Run). Bestehende CRITICAL-Findings
  werden NICHT täglich re-alarmiert — sonst pendelt der Operator
  in derselben Alert-Fatigue.

Begründung: `pip-audit` und `npm audit` haben dokumentiert hohe
False-Positive-Raten und MEDIUM-Findings, die seit Jahren
unverändert sind. Daily-Notifications für unveränderte Befunde
trainieren den Operator, sie zu ignorieren — exakt der
Failure-Mode, vor dem Operations-Praxis seit Jahrzehnten warnt.
Die Wochen-Digest-Cadence ist bewusst träger; CRITICAL-Diff ist
das einzige Live-Signal.

Der Timer triggert
`operator/voice/scripts/supply_chain_verify.py`:

1. Iteriert über alle `core/*/` und `operator/*/`.
2. Pro Plugin: ruft `pip-audit --requirement
   <tree>/<name>/requirements.txt --format json` (Python-Side)
   und `npm audit --json` (Node-Side, falls `package.json`
   vorhanden).
3. Aggregiert Findings nach Severity (CRITICAL/HIGH/MEDIUM/LOW).
4. Bei CRITICAL ODER HIGH:
   - Audit-Event `supply_chain.cve_detected` (WARNING) pro
     Plugin pro Severity-Tier (mit `package`, `version`, `cve_id`,
     `severity`, `fix_available`, `plugin_name` — **niemals**
     Vulnerability-Body, nie Exploit-Detail).
   - Bridge-Notification via `relay.json` (mirror der
     Layer-16-Roadmap-L `--notify-bridge`-Mechanik):
     `🚨 Supply-chain alert: <plugin> hat <N> CRITICAL/<M> HIGH
     CVE(s). Run "atelier-supply-chain-verify show" für Details.`
5. Bei MEDIUM/LOW: nur Log nach `~/.config/claude-voice/voice.log`,
   kein Audit, keine Notification (sonst Alert-Fatigue).
6. Exit-Code: 0 wenn no-CRITICAL-no-HIGH, 1 sonst (timer-readable).

`bridge.sh up` aktiviert den Timer; `bridge.sh down` deaktiviert
ihn.

### D) Plugin-Capability-Manifest (Phase 31.4)

Jedes Plugin deklariert in `plugin.atelier.yaml`:

```yaml
apiVersion: atelier/v1
kind: PluginCapabilities
metadata:
  plugin_name: atelier-gateway
  version: 0.1.0
spec:
  declared_imports:
    python:
      - fastapi
      - pydantic
      - httpx
      - uvicorn
      - pyyaml
      - PyJWT
      - cryptography
    npm: []                    # plugins ohne Node-Side
  declared_network:
    egress:
      - "api.openai.com"       # für STT-Layer 23
      - "api.anthropic.com"    # für dialectic CLI
    ingress:
      - "0.0.0.0:8000"         # für Gateway HTTP
    notes: "ed25519-Verifikation rein lokal, Sigstore opt-in."
  declared_filesystem:
    writes:
      - "<atelier_home>/tenants/*/global/auth/**"
      - "<atelier_home>/tenants/*/global/gateway/**"
      - "<atelier_home>/tenants/*/global/scim/**"
    reads:
      - "<atelier_home>/tenants/*/global/tenant.atelier.yaml"
```

Bei Plugin-Install (über `.atelier-pkg`) ODER bei `bridge.sh up`:

1. **AST-Walk** über alle `*.py` des Plugins (im Plugin-Dir, NICHT
   im `.venv/`).
2. Sammelt jeden `import X` und `from X import Y` Top-Level —
   Lokal-Imports innerhalb von Funktionen werden ignoriert (zu
   teuer).
3. Vergleicht gegen `declared_imports.python`. Divergenz →
   `supply_chain.capability_drift` (WARNING) mit `plugin_name`,
   `undeclared_imports: [pkg1, pkg2]` (max 20 Einträge,
   alphabetisch sortiert), `unused_declared: [pkg3]`.
4. Operator-Confirm-Prompt erscheint NICHT — der Audit-Event ist
   das Operator-Signal. Aktion bleibt operator-discretion (Plugin
   ist installiert, läuft aber mit dokumentierter Drift).

**AST-Walk fängt Ehrliche, keine Adversaries.** Ein bösartiges
Plugin verwendet `importlib.import_module(__import__('base64').
b64decode('cGFuZGFz').decode())` und der AST-Walk sieht den
literal string nicht. Pattern wie `__import__('os')` mit
dynamischem Argument, `eval('import sys')`, oder
`exec(compile(open('/tmp/x').read(), 'x', 'exec'))` umgehen die
Detection trivial. Layer 31.4 detektiert *honest drift*
(Entwickler vergisst, ein neu hinzugefügtes `httpx` ins Manifest
einzutragen), nicht *malicious obfuscation*. Letzteres erfordert
Sandboxed Dependency Execution (Phase 31.8+).

**Network und Filesystem sind soft.** AST kann sie nicht
verifizieren ohne Runtime-Trace. Phase 31.4 schreibt sie nur
deklarativ in den Manifest; Phase 31.4b (separater ADR-Amendment)
könnte später eBPF-basierte Runtime-Verification anlegen — heute
zu komplex und nicht unbedingt nötig.

### E) Sigstore/Rekor-Anchoring (opt-in, Phase 31.5)

Erweiterung der Phase-5-Signing-Pipeline:

`package build` neuer Flag `--publish-rekor`:

- Veröffentlicht zusätzlich zur ed25519-Signatur einen
  Rekor-Eintrag (Sigstore Public Transparency Log,
  https://rekor.sigstore.dev/).
- Generiert `<bundle>.atelier-pkg.rekor.json` mit Inclusion-Proof.

`package verify` neuer Flag `--require-rekor`:

- Default: false (air-gapped Operator bleibt funktional).
- True: verifiziert Inclusion-Proof gegen Rekor-Log; Mismatch →
  `supply_chain.signature_chain_break` (WARNING), Verify schlägt
  fehl.

Sigstore-Anchoring ist **default-on-recommended** pro Operator-Policy
ab Phase 31.5.2 — der Default flippt von "opt-in" auf
"opt-out-required":

```yaml
# <atelier_home>/global/supply_chain.yaml
require_rekor_inclusion: true    # default ab Phase 31.5.2
                                 # explizit auf false setzen für Air-Gap
```

Operator kann pro-Tenant überschreiben:

```yaml
# tenant.atelier.yaml
spec:
  supply_chain:
    require_rekor_inclusion: false   # nur für air-gap-tenants nötig
```

Begründung der Default-Inversion: opt-in-Mechanismen werden
strukturell selten aktiviert. AtelierOS hat heute ~30 opt-in-Layer
und kein Operator schaltet alle ein. Die Praxis-Surface von
"fast niemand nutzt Sigstore" zu "alle außer Air-Gap nutzen es"
ist nur durch Default-Flipping erreichbar. Air-Gap-Operatoren
müssen bewusst opt-out wählen — das ist die richtige Hürde, weil
sie wissen, dass sie air-gap betreiben.

### F) Audit-Events + Prometheus-Metriken (Phase 31.6)

Audit-Events (alle Tier-1, metadaten-only) registriert in
`forge/security_events.py::EVENT_SEVERITY`:

| Event | Severity | Details (allow-listed) |
|---|---|---|
| `supply_chain.sbom_verified` | INFO | `bundle_sha256`, `dep_count`, `cdx_version` |
| `supply_chain.sbom_missing` | WARNING | `bundle_sha256` |
| `supply_chain.dep_hashes_updated` | INFO | `plugin_name`, `requirements_sha256`, `dep_count` |
| `supply_chain.dep_hash_mismatch` | WARNING | `plugin_name`, `package_name` (no version, no actual hash, no expected hash) |
| `supply_chain.frozen_baseline_breach_attempted` | WARNING | `plugin_name`, `requirements_sha256_old`, `requirements_sha256_new` |
| `supply_chain.cve_detected` | WARNING | `plugin_name`, `package_name`, `package_version`, `cve_id`, `severity` (CRITICAL/HIGH only), `fix_available`, `cadence` (`weekly` / `critical-diff`) |
| `supply_chain.capability_drift` | WARNING | `plugin_name`, `undeclared_imports` (max 20), `unused_declared` (max 20) |
| `supply_chain.signature_rekor_verified` | INFO | `bundle_sha256`, `rekor_log_index` |
| `supply_chain.signature_chain_break` | WARNING | `bundle_sha256`, `reason` (`rekor-not-found` / `inclusion-proof-invalid` / `pubkey-mismatch`) |

Per-Event-Allow-list. Globaler `_FORBIDDEN_FIELDS`-Set:
`exploit_text`, `vuln_body`, `dep_full_list`, `signature_bytes`,
`private_key`, `requirements_full_text`. Mirror der
L23/L24/L25/L28/L29/L30-Regel.

Prometheus-Metriken (ADR-0007 Phase 6 Surface), aggregiert in
`atelier_gateway/audit_metrics.py`:

| Metric | Labels | Source |
|---|---|---|
| `atelier_supply_chain_sbom_verified_total` | — | `sbom_verified` |
| `atelier_supply_chain_sbom_missing_total` | — | `sbom_missing` |
| `atelier_supply_chain_dep_hash_mismatch_total` | `plugin_name` | `dep_hash_mismatch` |
| `atelier_supply_chain_cve_detected_total` | `plugin_name`, `severity` | `cve_detected` (CRITICAL/HIGH only) |
| `atelier_supply_chain_capability_drift_total` | `plugin_name` | `capability_drift` |
| `atelier_supply_chain_signature_chain_break_total` | `reason` | `signature_chain_break` |

Label-Allow-Listen: `plugin_name` ∈ kuratiertem Set (entspricht
den Plugin-Verzeichnissen unter `core/` und `operator/`); `severity` ∈
`CRITICAL` / `HIGH` (LOW/MEDIUM erscheinen nicht in Metrics);
`reason` ∈ kuratierter Reason-Allow-List pro Event-Type.

### Path-Gate-Erweiterung (Layer 10)

`operator/voice/hooks/path_gate.py` schützt zusätzlich:

- `<atelier_home>/global/supply_chain.yaml` (Operator-Policy)
- `<atelier_home>/global/supply_chain/` (CVE-Cache, Drift-Records)
- `core/*/requirements.txt` (alle Core-Plugin-Requirements-Files)
- `operator/*/requirements.txt` (alle Operator-Plugin-Requirements-Files)
- `core/*/package-lock.json` (alle Core-Plugin-Node-Lock-Files)
- `operator/*/package-lock.json` (alle Operator-Plugin-Node-Lock-Files)
- `sbom.cdx.json` (in Bundle-Trees)

Hint-Strings für fail-closed Bash-Detection: `"requirements.txt"`,
`"package-lock.json"`, `"sbom.cdx.json"`,
`"supply_chain.yaml"`. Der LLM kann via Forge-Tool keine
Dependency-Hashes umschreiben; Operator-Workflow läuft via
`pip-compile` außerhalb der LLM-Kontroll-Schicht.

---

## What this ADR deliberately does NOT do

- **Kein Auto-Update bei CVE-Detection.** Mirror der existierenden
  `auto-update`-Regel (siehe CLAUDE.md): "the unattended hook never
  runs a package manager." CVE-Findings sind Operator-Signal, nicht
  System-Aktion. Der Operator entscheidet, ob ein Patch eingespielt
  wird.
- **Kein erzwungenes Sigstore.** Air-gapped Deployments
  (Government, Defense, On-Prem-Bank) müssen funktionsfähig bleiben.
  ed25519-Pinning ist die Mindestanforderung; Sigstore ist die
  öffentlich-verifizierbare Erweiterung für Operator, die das
  wollen.
- **Kein Reproducible-Builds-Anspruch.** Aktuelle pip + npm
  produzieren keine bit-identischen Artefakte über
  Build-Maschinen-Grenzen. Ein Reproducible-Builds-Pfad würde
  PyOxidizer / Nix / Bazel als Build-Substrat verlangen — zu teuer
  für ROI heute. Phase 31.7+ wenn Phase-VI/Cloud-Ops das verlangen.
- **Kein automatischer Dep-Hash-Refresh.** Operator-side via
  `pip-compile --upgrade` only. Auto-Refresh wäre silent
  supply-chain-substitution: ein kompromittierter PyPI-Mirror
  liefert eine neue Hash, die Datei aktualisiert sich, und niemand
  bemerkt es.
- **Keine Runtime-Network-Egress-Verification.** Capability-Manifest
  declared `egress`-Endpunkte — die Verifizierung passiert NICHT
  per eBPF / netfilter. Phase 31.4b könnte das nachholen; heute
  bleibt es deklarativ + Operator-Audit. Die bwrap-Sandbox hat
  bereits `network: deny` als Default; das ist die strukturelle
  Defense, nicht das Manifest.
- **Keine Vulnerability-Body-Persistierung.** Audit-Chain trägt
  nur `cve_id` + `severity`. Wer Details will, schaut die CVE-ID
  in NVD nach. Body-Storage wäre PII-frei (CVEs sind public), aber
  würde die Chain unnötig aufblasen.
- **Keine Cross-Plugin-Dep-Deduplication.** Jedes Plugin hat sein
  eigenes Venv (Phase-Separation, ADR-0007 Phase 4). Doppelte
  Installation einer Dep in zwei Plugins ist akzeptierter Cost.
- **Kein Build-Reproducibility-Audit.** Wenn ein Operator
  Plugin X auf Box A baut und auf Box B installiert, wird
  NICHT verifiziert, dass beide Bauten identisch wären. Out of
  scope.

---

## Test-Surface

Per-Subtask-E2E (load-bearing):

| File | Cases | Coverage |
|---|---|---|
| `core/gateway/tests/test_packaging_sbom.py` | ~12 | SBOM-Schema-Validation gegen CycloneDX 1.5 JSON-Schema, Round-trip build → verify mit SBOM, `--require-sbom` schlägt fehl ohne SBOM, `--check-cves` reportet aber blockt nicht, `payload_sha256` umfasst SBOM, Audit-Allow-List für `sbom_verified`/`sbom_missing` |
| `operator/voice/scripts/test_supply_chain_verify.py` | ~10 | `pip-audit`-Stub-Integration, npm-audit-Stub, Severity-Aggregation, CRITICAL/HIGH triggert Notification, MEDIUM/LOW silent, Exit-Code-Contract, Audit-Event-Allow-List, fail-open auf pip-audit-missing |
| `core/gateway/tests/test_capability_manifest.py` | ~14 | Schema-Validation, AST-Walk gegen Plugin-Source, declared-vs-actual-Diff, Drift-Event-Generation, Allow-List für `capability_drift`, Top-Level-vs-In-Function-Imports korrekt unterscheidet, max-20-Cap |
| `operator/bridges/shared/test_path_gate_supply_chain.py` | ~8 | path-gate denies Write/Edit/Bash-redirect/sed-i auf supply_chain.yaml, requirements.txt, sbom.cdx.json, fail-closed Bash mit Hint, Read bleibt erlaubt, Path-Traversal-Resistenz |
| `core/gateway/tests/test_sigstore_rekor.py` | ~8 (opt-in: `ATELIER_SIGSTORE_LIVE=1`) | Build mit `--publish-rekor` produziert valides `.rekor.json`, verify mit `--require-rekor` erfolgreich, tampered bundle wird rejected, Audit-Event `signature_rekor_verified`/`signature_chain_break` |

Wired into `operator/bridges/run-all-tests.sh` als fünf neue
Suites (Layer 31 a–e). Sigstore-Live-Test ist `ATELIER_SIGSTORE_LIVE`
gegated wie der existierende OpenCode-Live-Test.

---

## What you, as Claude Code, must NOT do (ADR-0021)

- **Don't put dependency lists, vulnerability bodies, exploit
  text, signature bytes, or private-key material into any
  audit-event detail field.** Per-Event-`_AUDIT_ALLOWED_FIELDS`
  + globaler `_FORBIDDEN_FIELDS`-Set in `audit.py` enforce es am
  Boundary. Mirror der L23/L24/L25/L28/L29/L30-Regel.
  Regression-Gate pro Modul: ein Test, der jeden `_emit_*`-Pfad
  gegen einen Spy auf `forbidden field`-Detection fährt.
- **Don't auto-update dependency hashes from any code path.** Der
  Hash-Gate ist die strukturelle Defense gegen
  PyPI-Mirror-Substitution; ein "convenience-mode auto-refresh"
  hebelt es exakt für die Klasse von Angriffen aus, gegen die
  es schützt. Hash-Update ist ausschließlich operator-side via
  `pip-compile --upgrade`.
- **Don't fail-closed on `pip-audit`-missing.** CVE-Surveillance
  ist additiv; ein Box ohne `pip-audit` darf weiterhin Bridge
  starten. Audit-Event `supply_chain.cve_check_skipped` (WARNING)
  ist das Operator-Signal, dass der Surveillance-Path nicht
  funktional ist — nicht das Bridge-Down-Signal.
- **Don't gate Bridge-Boot on `supply_chain.cve_detected`.** CVE
  ist Operator-Signal, nicht Bridge-Action. Auto-Stop bei CVE
  würde einen Operator mit "die Bridge geht jetzt nicht mehr an"
  konfrontieren — schlechter Trade als
  "Bridge läuft, Operator sieht Notification, entscheidet".
- **Don't auto-publish to Rekor when building.**
  `--publish-rekor` ist explizit, kein Default. Operator entscheidet
  pro Build, ob die Signatur öffentlich nachvollziehbar wird.
  Privacy + air-gap-Setups dürfen nicht silent ihre Pakete
  publizieren.
- **Don't widen the AST-walk to include in-function imports.**
  Cost wäre prohibitiv (jeder Funktions-Body wird gescannt), und
  In-Function-Imports sind ein legitimes Pattern für
  optional-dep-handling. Top-Level-only ist die richtige
  Granularität.
- **Don't drop the `<atelier_home>/global/supply_chain.yaml`-
  protection in path-gate.** LLM überschreibt die
  Operator-Policy → CVE-Surveillance kann silent disabled werden.
  Genau die Klasse, gegen die path-gate strukturell schützt.
- **Don't conflate `npm audit` with `pip-audit` Severity-Skala.**
  Beide haben CRITICAL/HIGH/MEDIUM/LOW, aber die Schwellwerte
  unterscheiden sich. Mapping bleibt 1:1 namentlich; eine
  Cross-Tool-Normalisierung ist out of scope.
- **Don't gate the SBOM-Generation behind the License-Plugin.**
  SBOM ist regulatorische Mindestanforderung (EU CRA ab 2027,
  EU AI Act Art. 15); jeder Operator muss sie ohne Lizenz
  produzieren können. Premium-Variante (z.B. SBOM-Diff-Reports
  über Versionen, Vendor-Risk-Scoring) wäre Phase-V-Material.
- **Don't add a network-side enforcement to declared `egress`
  endpoints in this ADR.** Phase 31.4 ist deklarativ; runtime
  enforcement (eBPF / netfilter) braucht eigenen ADR mit eigenem
  Test-Setup. Heute ist der Audit-Event das Signal, nicht der
  Block.
- **Don't ship `pip-audit` / `npm audit` / `pip-compile` / `cyclonedx`
  als Hard-Dependency der Bridge oder des Gateways.** Die Tools
  sind operator-installed; Bridge-Boot fail-closed wenn sie
  fehlen wäre Regression. Audit-Event "tooling missing" + skip ist
  der richtige Default.

---

## Future work (not in this ADR)

- **Phase 31.7 — Reproducible Builds**: PyOxidizer / Nix /
  Bazel-Substrat für bit-identische Plugin-Bauten. Voraussetzung
  für SLSA Level 3+. Eigener ADR wenn Cloud-Ops (ADR-0017 Phase VI)
  das verlangen.
- **Phase 31.8 — Runtime Egress-Verification**: eBPF-basierte
  Echtzeit-Verifikation des declared `egress`-Set. Eigener ADR
  mit Linux-Kernel-Version-Anker und Falcoplugin-Vergleich.
- **Phase 31.9 — Vendor-Risk-Scoring**: aggregiertes
  Trust-Rating pro Dependency-Tree (OpenSSF Scorecard
  Integration), Dashboard-Panel, premium-gated.
- **Phase 31.10 — SBOM-Diff über Plugin-Versionen**: zeigt was
  sich zwischen `v0.12.0` und `v0.13.0` an Deps geändert hat,
  inkl. CVE-Delta. Premium-gated.

---

## References

- ADR-0007 Phase 5 — `.atelier-pkg`-Format (Substrat dieser
  Erweiterung)
- ADR-0017 Phase III — License-Gate (Pattern für opt-in
  Plugin-Mount)
- ADR-0019 — License-Signing-Pipeline (Vorbild für Offline-
  Signing-Trust-Boundary, Sigstore-Decision-Logik)
- Layer 10 — path-gate (erweitert für supply_chain-Tree)
- Layer 16 Roadmap L — daily-audit-verify-Timer (Vorbild für
  daily-supply-chain-verify-Timer, Bridge-Notification-Pattern)
- Layer 23 / 24 / 25 / 28 / 29 / 30 — Metadaten-only-Audit-
  Präzedenz
- Microsoft Defense-in-Depth for Autonomous AI Agents
  (microsoft.com/security blog, May 2026) — externer Anker für
  "Supply-Chain Compromise" als eigene Risikoklasse
- EU AI Act 2026 Art. 15 (Accuracy/Robustness) +
  EU Cyber Resilience Act 2027 (SBOM-Pflicht für digital
  products) — Compliance-Baseline-Anker
- OpenSSF Scorecard, SLSA Level 2/3 — externe Best-Practice-
  Anker für Future-Phasen
