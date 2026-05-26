# ADR-001: EU-DSGVO-konforme Atelier-Architektur mit lokalem OpenCode LLM

**Status:** Proposed  
**Date:** 2026-05-19  
**Deciders:** Datenschutzbeauftragte, Enterprise Architecture, Security  
**Stakeholders:** Legal, Compliance, DevOps, Engineering  

---

## Problem

Atelier OS soll in großen europäischen/deutschen Unternehmen deploybar sein, **ohne dass Quellcode, Prompts oder Verarbeitungsdaten die EU verlassen oder zu US-Providern (Anthropic) fließen**. Die aktuelle Standard-Integration zu Claude API verstößt gegen:

- **DSGVO Kap. 5 (Datenübermittlung):** Standard Contractual Clauses (SCCs) nach Schrems II fragwürdig für USA
- **Nationale Datenschutzgesetze:** DE-DSG, NIS2-Richtlinie (kritische Infrastrukturen)
- **Unternehmensgeheimnis:** Quellcode darf nicht an externe Parteien fließen
- **Supply-Chain-Governance:** Wer haftet, wenn Anthropic gehackt wird?

**Ziel:** Ein **Enterprise-ready Atelier-Setup**, das:
- ✅ OpenCode (oder anderes Open-Source-LLM) lokal lädt
- ✅ Alle Daten on-premises oder EU-hosted verarbeitet
- ✅ Audit-Trails + Compliance-Reporting hat
- ✅ Kein Datenleck zu Anthropic/USA-Servern
- ✅ Praktisch in großen Konzernen deploybar ist

---

## Context

### Aktuelle Situation
- Atelier nutzt Claude API → Daten fließen zu Anthropic (USA)
- OpenCode ist lokal ansprechbar und konfigurierbar
- Atelier hat bereits **starke Governance** (Permissions, Sandbox/Bwrap, Audit-Logs)
- Unternehmen brauchen **Alternative**, die on-prem/EU-safe ist

### Regulatorischer Kontext
| Anforderung | Geltung | Impact |
|---|---|---|
| DSGVO Art. 44-49 | EU/DE | Datenübermittlung nur mit Garantien |
| Schrems II (EuGH C-311/18) | EU/DE | SCCs + zusätzliche Maßnahmen = verpflichtend |
| NIS2-Richtlinie | DE/kritische Infra | Wo liegt der Code? Wer hat Zugriff? |
| DE-DSG § 3 Abs. 11 | Deutschland | Verbot der Übermittlung in Länder ohne Datenschutzniveau |
| Unternehmensgeheimnis-Gesetz | DE | Quellcode = Geschäftsgeheimnis, darf nicht zu Dritten |

---

## Decision

### Core Decisions

**D1: Dual-Mode-Architektur für Atelier**
```
┌─────────────────────────────────────────────────────────┐
│ Atelier Core (unverändert)                              │
│ - Permissions, Sandbox, Audit-Logs                      │
│ - Agent-Orchestrierung                                  │
└──────────┬──────────────────────────────────────────────┘
           │
      ┌────┴─────────────────────────────────────┐
      │ LLM-Abstraktions-Layer (NEU)             │
      └────┬──────────────────────────────────┬──┘
           │                                  │
      ┌────▼──────────┐              ┌────────▼──────────┐
      │ STANDARD MODE │              │ EU-COMPLIANCE MODE│
      ├────────────────┤             ├──────────────────┤
      │ Claude API     │             │ OpenCode (lokal) │
      │ (Anthropic)    │             │ oder Mistral API │
      │ Optional       │             │ (EU-hosted)      │
      └────────────────┘             └──────────────────┘
```

**D2: OpenCode als Default für EU-Enterprises**
- **LLM-Engine:** OpenCode (lokal executable, kein Cloud-Dependency)
- **Fallback:** Mistral API (EU-hosted, SCCs mit französischem Provider)
- **Ablehnung:** Anthropic API in produktiven Workloads (nur Test/Dev)

**D3: Zero-Egress-Architektur**
```
Keine Daten verlassen die Maschine/Netzwerk, außer:
✅ Authorized logging zu Siem (intra-Netzwerk)
✅ Git pushes zu eigenem Gitea/GitLab (intra)
✅ Audit-Reports zu Compliance-System (intra)
❌ Keine Telemetrie/Crash-Reports außen
❌ Keine Model-Training-Daten outside
❌ Keine Prompts/Code nach außen
```

---

## Implementation Strategy

### Phase 1: Architecture Setup (4 Wochen)

#### 1.1 LLM-Engine-Abstraktion in Atelier

**Datei:** `atelier/config/llm-engine.yaml`

```yaml
# Standard Mode (Dev/Testing)
engines:
  claude:
    type: api
    provider: anthropic
    endpoint: https://api.anthropic.com
    enabled: true
    scope: development_only  # Explizit nur Dev

# EU-Compliance Mode (Enterprise)
  opencode:
    type: local
    provider: opencode
    endpoint: http://localhost:8000
    model: opencode-latest
    enabled: true
    scope: production_eu
    config:
      memory: 16GB
      gpu: optional
      max_context: 200k
      
  mistral_eu:
    type: api
    provider: mistral  # Frankreich, EU-safe
    endpoint: https://api.mistral.ai
    enabled: false  # Opt-in
    scope: enterprise_eu_optional
    compliance:
      jurisdiction: EU (France)
      dpa_signed: true
      scc_level: enhanced

# Routing-Policy
routing:
  default: opencode  # Für EU-Deployments
  fallback: mistral_eu
  forbidden_for_sensitive_code: claude  # Explizit blockiert
```

**Implementierung:**
```python
# atelier/llm_engine/factory.py

from abc import ABC, abstractmethod

class LLMEngine(ABC):
    @abstractmethod
    def call(self, prompt: str, context: Dict) -> str:
        pass
    
    @abstractmethod
    def get_compliance_info(self) -> Dict:
        pass

class OpenCodeEngine(LLMEngine):
    def __init__(self, endpoint: str):
        self.endpoint = endpoint  # localhost:8000
        
    def call(self, prompt: str, context: Dict) -> str:
        # POST /v1/completions zu local OpenCode
        response = requests.post(
            f"{self.endpoint}/v1/completions",
            json={"prompt": prompt, **context}
        )
        return response.json()["text"]
    
    def get_compliance_info(self) -> Dict:
        return {
            "jurisdiction": "local",
            "data_egress": "none",
            "gdpr_compliant": True,
            "audit_trail": True
        }

class AnthropicEngine(LLMEngine):
    # Nur für Entwicklung, explizit gekennzeichnet
    pass

# Selektion
engine = select_engine(config.deployment_mode)
# deployment_mode == "eu_production" → OpenCodeEngine
# deployment_mode == "dev" → AnthropicEngine (mit Warnung)
```

#### 1.2 OpenCode lokal deployen

**Docker Setup:**
```dockerfile
# Dockerfile.opencode
FROM nvidia/cuda:12.2-runtime-ubuntu22.04

RUN apt-get update && apt-get install -y python3.11 pip

COPY requirements-opencode.txt /app/
RUN pip install -r /app/requirements-opencode.txt

# OpenCode Model (~7B Parameter, ~15GB)
RUN python3 -m opencode.download --model opencode-7b-instruct

EXPOSE 8000

CMD ["python3", "-m", "opencode.serve", "--port", "8000", "--gpu"]
```

**Docker-Compose (Enterprise Setup):**
```yaml
version: '3.9'

services:
  opencode-llm:
    build:
      context: .
      dockerfile: Dockerfile.opencode
    container_name: atelier-opencode
    ports:
      - "8000:8000"
    volumes:
      - ./models:/models  # Persistent Model Cache
      - ./logs:/var/log/opencode
    environment:
      - GPU_MEMORY=16GB
      - MAX_BATCH_SIZE=4
      - LOG_LEVEL=INFO
    restart: always
    networks:
      - atelier-internal
    
  atelier-core:
    build:
      context: .
      dockerfile: Dockerfile.atelier
    container_name: atelier-engine
    depends_on:
      - opencode-llm
    environment:
      - LLM_ENGINE=opencode
      - LLM_ENDPOINT=http://opencode-llm:8000
      - COMPLIANCE_MODE=eu_strict
    volumes:
      - /var/atelier/audit-logs:/audit
      - /var/atelier/config:/config
    networks:
      - atelier-internal

  audit-logger:
    image: elastic/filebeat:8.x
    container_name: atelier-audit
    volumes:
      - /var/atelier/audit-logs:/logs:ro
      - ./filebeat.yml:/etc/filebeat/filebeat.yml:ro
    networks:
      - atelier-internal

networks:
  atelier-internal:
    driver: bridge
```

---

### Phase 2: Compliance-Barrieren (4 Wochen)

#### 2.1 Datenfluss-Kontrolle

**Datei:** `atelier/security/data_flow_guard.py`

```python
import logging
from enum import Enum

class DataClassification(Enum):
    PUBLIC = "public"
    INTERNAL = "internal"
    CONFIDENTIAL = "confidential"
    SECRET = "secret"

class DataFlowGuard:
    """
    Verhindert, dass sensitive Daten zu unautorisierten Systemen fließen.
    """
    
    ALLOWED_ENDPOINTS = {
        "opencode": "http://localhost:8000",
        "mistral_eu": "https://api.mistral.ai",  # Only if explicitly enabled
    }
    
    FORBIDDEN_ENDPOINTS = {
        "anthropic": "https://api.anthropic.com",
        "openai": "https://api.openai.com",
    }
    
    def __init__(self, config: Dict, audit_log: AuditLog):
        self.config = config
        self.audit_log = audit_log
        
    def validate_endpoint(self, endpoint: str, data_class: DataClassification) -> bool:
        """
        Prüft, ob Daten an einen Endpoint gehen dürfen.
        Throws exception bei Verstoß.
        """
        
        # Sensitive Daten nur zu lokalen Endpoints
        if data_class in [DataClassification.CONFIDENTIAL, DataClassification.SECRET]:
            if endpoint not in ["http://localhost:8000"]:
                self.audit_log.log_violation(
                    event="DATA_EGRESS_ATTEMPT",
                    endpoint=endpoint,
                    classification=data_class,
                    action="BLOCKED"
                )
                raise SecurityException(
                    f"Sensitive data cannot be sent to external endpoint {endpoint}"
                )
        
        # Whitelist-Check
        if endpoint in self.FORBIDDEN_ENDPOINTS:
            self.audit_log.log_violation(
                event="FORBIDDEN_ENDPOINT",
                endpoint=endpoint,
                action="BLOCKED"
            )
            raise SecurityException(
                f"Endpoint {endpoint} is explicitly forbidden in EU-Compliance mode"
            )
        
        self.audit_log.log(
            event="DATA_FLOW_APPROVED",
            endpoint=endpoint,
            classification=data_class
        )
        return True

# Usage in Agent
class AttelierAgent:
    def __init__(self, llm_engine, data_guard: DataFlowGuard):
        self.engine = llm_engine
        self.guard = data_guard
    
    def process(self, prompt: str, data_class: DataClassification):
        # Vor jedem API-Call: Guard prüfen
        self.guard.validate_endpoint(
            self.engine.endpoint,
            data_class
        )
        return self.engine.call(prompt)
```

#### 2.2 Audit & Compliance Logging

**Datei:** `atelier/audit/compliance_logger.py`

```python
from datetime import datetime
import json

class ComplianceLogger:
    """
    Vollständiger Audit-Trail für DSGVO-Compliance.
    Kann vom Datenschutzbeauftragten eingesehen werden.
    """
    
    def __init__(self, log_dir: str):
        self.log_dir = log_dir
        self.current_session_id = str(uuid.uuid4())
        
    def log_agent_execution(self, agent_id: str, prompt: str, data_class: str, 
                            llm_engine: str, result: str, duration_ms: int):
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "session_id": self.current_session_id,
            "event_type": "AGENT_EXECUTION",
            "agent_id": agent_id,
            "llm_engine": llm_engine,
            "data_classification": data_class,
            "prompt_hash": hashlib.sha256(prompt.encode()).hexdigest(),
            "prompt_length": len(prompt),
            "result_length": len(result),
            "duration_ms": duration_ms,
            "compliance_check": {
                "dsgvo": True,
                "data_egress_check": True,
                "audit_trail": True
            }
        }
        self._write_log(entry)
        
    def log_data_access(self, user_id: str, resource: str, purpose: str, 
                        access_type: str):
        """Logging für DSGVO Art. 32 (Datensicherheit)"""
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": "DATA_ACCESS",
            "user_id": user_id,
            "resource": resource,
            "purpose": purpose,
            "access_type": access_type  # read, write, delete
        }
        self._write_log(entry)
        
    def log_violation(self, event_type: str, details: Dict):
        """Kritische Sicherheitsverletzungen"""
        entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "event_type": "SECURITY_VIOLATION",
            "violation_type": event_type,
            "details": details,
            "severity": "CRITICAL"
        }
        self._write_log(entry)
        self._alert_security_team(entry)
        
    def _write_log(self, entry: Dict):
        """Atomic write zu Audit-Log (unveränderlich)"""
        log_file = f"{self.log_dir}/audit-{self.current_session_id}.jsonl"
        with open(log_file, "a") as f:
            f.write(json.dumps(entry) + "\n")
            
    def _alert_security_team(self, entry: Dict):
        # Benachrichtigung an SIEM oder Slack-Channel
        pass
        
    def generate_compliance_report(self, start_date: str, end_date: str) -> Dict:
        """Bericht für Datenschutzbeauftragten"""
        return {
            "period": f"{start_date} to {end_date}",
            "total_executions": self._count_executions(),
            "data_egress_attempts": self._count_blocked_egress(),
            "llm_engines_used": self._get_used_engines(),
            "compliance_violations": self._get_violations(),
            "attestation": "All data processed locally or within EU-approved endpoints"
        }
```

---

### Phase 3: Governance & Dokumentation (2 Wochen)

#### 3.1 Konfiguration für Unternehmen

**Datei:** `enterprise/atelier-compliance-config.yaml`

```yaml
# Enterprise Deployment Configuration
# Für Datenschutzbeauftragte und IT-Security

deployment:
  mode: EU_PRODUCTION
  jurisdiction: Germany  # oder EU
  
compliance:
  dsgvo:
    enabled: true
    article_32: "Alle Daten on-premises mit Encryption at rest"
    article_44_49: "Keine Datenübermittlung außerhalb der EU"
    audit_trail: "Vollständiges Audit-Logging, unveränderlich"
    
  nis2:
    applicable: true
    critical_infrastructure: false
    managed_security_provider: internal
    
  legal:
    data_processor_agreement: "./documents/dpa-signed.pdf"
    scc_level: "not_applicable (local processing)"
    
llm_engine:
  default: opencode
  allowed_engines:
    - opencode  # Lokal, kein Datenleck
  forbidden_engines:
    - anthropic  # USA, zu riskant
    - openai     # USA, zu riskant
    
data_handling:
  classification_levels:
    public: 
      allowed_endpoints: [opencode, mistral_eu]
    internal:
      allowed_endpoints: [opencode]
    confidential:
      allowed_endpoints: [opencode]  # Nur lokal
    secret:
      allowed_endpoints: [opencode]  # Nur lokal
      
  encryption:
    at_rest: "AES-256"
    in_transit: "TLS 1.3"
    key_management: "Local key store, no cloud HSM"
    
audit:
  log_retention: "7 years"
  log_location: "/var/atelier/audit-logs"
  log_immutability: true
  siem_integration: "splunk-internal"
  
approval_chain:
  data_protection_officer: true
  legal_review: true
  security_review: true
  ciso_sign_off: true
```

#### 3.2 Checkliste für Datenschutzbeauftragten

**Datei:** `docs/DSB-CHECKLIST.md`

```markdown
# Checkliste für Datenschutzbeauftragte

## ✅ Technische Kontrollen

- [ ] OpenCode LLM läuft lokal auf isoliertem Netzwerk
- [ ] Keine Datenlecks zu Anthropic/OpenAI/externen APIs
- [ ] Firewall-Regel: Port 8000 (OpenCode) nur intern, keine externen Callouts
- [ ] Encryption at rest (AES-256) für alle Logs und Modelle
- [ ] TLS 1.3 für alle Netzwerkverbindungen
- [ ] Audit-Logs sind unveränderlich (write-once, append-only)
- [ ] Regelmäßige Backups der Audit-Logs (mindestens täglich)
- [ ] Keine Telemetrie/Crash-Reports zu Anthropic

## ✅ Governance-Kontrollen

- [ ] Data Processing Agreement (DPA) mit Atelier-Team unterzeichnet
- [ ] Purpose Limitation: Atelier darf nur für Softwareentwicklung genutzt werden
- [ ] Retention Policy: Code-Inputs werden nach 30 Tagen gelöscht (konfigurierbar)
- [ ] Access Control: Nur autorisierte Entwickler haben Zugriff
- [ ] Data Subject Rights: Prozess für Datenzugriff/Löschung definiert
- [ ] Breach Notification Plan: Wer wird informiert, wenn Audit-Log gehackt?

## ✅ Compliance-Prüfungen

- [ ] Penetration Test für OpenCode-Setup durchgeführt
- [ ] DSGVO-Folgenabschätzung (DPIA) abgeschlossen
- [ ] Schrems II: Keine USA-Übermittlung, lokale Verarbeitung
- [ ] NIS2: Falls kritische Infrastruktur, zusätzliche Sicherheitsmaßnahmen

## ✅ Dokumentation

- [ ] Security Policy für Atelier geschrieben
- [ ] Incident Response Plan für Datenlecks
- [ ] Audit-Log Review Schedule: Wöchentlich mindestens
- [ ] Regelmäßige Berichterstellung an Geschäftsleitung

## 📋 Regelmäßige Überprüfung

- Monatlich: Audit-Logs auf Anomalien prüfen
- Quartal: Compliance-Bericht generieren
- Jährlich: Penetration Test + Compliance Audit
```

---

### Phase 4: Testing & Validation (2 Wochen)

#### 4.1 Compliance Test Suite

**Datei:** `tests/test_eu_compliance.py`

```python
import pytest
from atelier.security.data_flow_guard import DataFlowGuard, DataClassification

class TestEUCompliance:
    """
    Compliance-Tests, die sicherstellen, dass Atelier die 
    Anforderungen einhält.
    """
    
    @pytest.fixture
    def guard(self):
        return DataFlowGuard({}, audit_log=MockAuditLog())
    
    def test_no_data_egress_to_anthropic(self, guard):
        """Anthropic darf NIEMALS erreicht werden"""
        with pytest.raises(SecurityException):
            guard.validate_endpoint(
                "https://api.anthropic.com",
                DataClassification.SECRET
            )
    
    def test_sensitive_data_only_local(self, guard):
        """Confidential data darf nur lokal verarbeitet werden"""
        # ✅ Lokal OK
        assert guard.validate_endpoint(
            "http://localhost:8000",
            DataClassification.CONFIDENTIAL
        )
        
        # ❌ Extern nicht OK (auch mit Mistral)
        with pytest.raises(SecurityException):
            guard.validate_endpoint(
                "https://api.mistral.ai",
                DataClassification.CONFIDENTIAL
            )
    
    def test_audit_trail_complete(self):
        """Jeden Agent-Call müssen wir tracken"""
        agent = AttelierAgent(mock_engine, guard=mock_guard)
        agent.process("test prompt", DataClassification.INTERNAL)
        
        audit_log = read_audit_log()
        assert len(audit_log) > 0
        assert audit_log[-1]["event_type"] == "AGENT_EXECUTION"
        assert audit_log[-1]["llm_engine"] == "opencode"
    
    def test_encryption_at_rest(self):
        """Alle Logs müssen verschlüsselt sein"""
        # Prüfe, dass Audit-Logs auf der Festplatte encrypted sind
        log_file = "/var/atelier/audit-logs/audit-*.jsonl"
        for log in glob.glob(log_file):
            # File ist mit AES-256 verschlüsselt
            assert is_encrypted(log, algorithm="AES-256")
    
    def test_no_external_calls_in_compliance_mode(self):
        """Im EU-Compliance-Mode: Null externe Calls außer Whitelist"""
        with network_monitor() as monitor:
            agent = AttelierAgent(opencode_engine)
            agent.process("write a function", DataClassification.INTERNAL)
        
        external_calls = monitor.get_external_calls()
        assert len(external_calls) == 0, f"Unexpected external calls: {external_calls}"
    
    def test_dsgvo_right_to_deletion(self):
        """Datenschutzrecht: Löschung muss funktionieren"""
        user_id = "user123"
        
        # Agent lädt Code von user123
        agent.process_for_user(user_id, "code snippet")
        
        # Datenschutzanfrage: Alles von user123 löschen
        agent.delete_user_data(user_id)
        
        # Prüfe: Alle Spuren sind weg
        logs = read_audit_logs_for_user(user_id)
        assert len(logs) == 0, "User data not properly deleted"
```

#### 4.2 Penetration Test Checkliste

```markdown
# Security Penetration Test für EU-Compliant Atelier

## Network Segmentation
- [ ] OpenCode läuft in separatem Container/VM
- [ ] Keine Outbound-Connections zu Anthropic/OpenAI möglich (Firewall)
- [ ] Atelier-Core kann OpenCode nur via localhost:8000 erreichen
- [ ] Alle externe APIs sind whitelisted und getestet

## Data Isolation
- [ ] Prompts werden nicht in externen Systemen gecacht
- [ ] Model-Weights sind lokal, nicht sync'd zu Cloud
- [ ] Keine GPU-Memory-Leaks von Prompts
- [ ] Temp-Files werden nach Ausführung sicher gelöscht

## Compliance Violations
- [ ] Versuche, Anthropic API zu erreichen → BLOCKED
- [ ] Versuche, Debug-Logs außen zu senden → BLOCKED
- [ ] Versuche, Audit-Logs zu verändern → IMMUTABLE, logged

## Social Engineering
- [ ] Env-Var ANTHROPIC_API_KEY ist nicht gesetzt → Fallback zu OpenCode
- [ ] Konfiguration kann nicht zur Laufzeit zu Anthropic umgestellt werden
- [ ] Dokumentation warnt vor unsicherer Verwendung
```

---

## Consequences (Auswirkungen)

### ✅ Positive Consequences

1. **Volle DSGVO-Compliance:**
   - Keine Datenflüsse in die USA
   - Art. 44-49 erfüllt durch lokale Verarbeitung
   - Schrems II nicht mehr relevant

2. **Enterprise-Ready:**
   - Große Konzerne können Atelier produktiv nutzen
   - Datenschutzbeauftragter kann zustimmen
   - Legal/Compliance gibt grünes Licht

3. **Geistiges Eigentum geschützt:**
   - Quellcode verlässt nie das Unternehmen
   - Keine Angst vor Spionage

4. **Governance integriert:**
   - Audit-Logs für 7 Jahre
   - Jede Aktion ist nachvollziehbar
   - Compliance-Berichte automatisch

### ⚠️ Trade-offs

1. **Performance:**
   - OpenCode 7B ist kleiner als Claude 3 Opus
   - Lokale Verarbeitung ist langsamer (keine GPU-Cluster)
   - **Mitigation:** GPU-Support + Quantisierung für schneller Inference

2. **Kapazität:**
   - OpenCode kennt nur Was bis Feb 2025 (Knowledge-Cutoff)
   - Claude hat Zugriff auf aktuelle APIs/Libraries
   - **Mitigation:** Fine-tuning mit unternehmenseigenen Daten

3. **Operational Overhead:**
   - Model muss selbst gehostet werden
   - GPU-Ressourcen notwendig
   - **Mitigation:** Docker-Setup ist automated, GPU optional

4. **Feature Completeness:**
   - Einige Claude-Features (z.B. Vision, Artifact Rendering) nicht sofort verfügbar
   - **Mitigation:** Phase 2 Implementation mit OpenCode-Erweiterungen

---

## Implementation Timeline

```
Week 1-4:   LLM Engine Abstraction + OpenCode Local Setup
Week 5-8:   Data Flow Guard + Compliance Logging
Week 9-10:  Enterprise Configuration + DSB Checkliste
Week 11-12: Testing + Penetration Tests + Validation

Total: 3 Monate bis Production Ready
```

---

## Validation Criteria

Ein Deployment ist **EU-Compliance-ready**, wenn:

- ✅ **Zero Egress Test:** Kein Byte verlässt das Netzwerk außer authorisierten Logs
- ✅ **Audit Trail Immutability:** 7-Jahres-Logs sind unveränderlich
- ✅ **Compliance Report:** Automatischer monatlicher Report für DSB
- ✅ **Penetration Test:** Keine Datenlecks oder Sicherheitslücken
- ✅ **Legal Sign-off:** Datenschutzbeauftragter und Legal haben zustimmend unterschrieben
- ✅ **Production Deployment:** 3 Monate stabil laufen ohne Vorfälle

---

## References

- DSGVO Art. 44-49 (Internationale Datenübermittlung)
- EuGH C-311/18 (Schrems II, SCCs Invalidity)
- NIS2-Richtlinie (2022/2555/EU)
- Deutsches Datenschutzgesetz (DSG)
- NIST Cybersecurity Framework 2.0
- CIS Controls v8

---

**Approved by:** [To be signed]  
**Review date:** 2026-08-19 (3 Monate nach Decision)
