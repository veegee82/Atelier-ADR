# ADR-0042: AtelierOS Production-Ready Release Roadmap

**Status:** ACTIVE  
**Date:** 2026-05-21  
**Version:** 1.0  
**Target Release:** 2026-06-21 (Phase M5.0)  

---

## Executive Summary

AtelierOS is **architecturally complete and compliance-ready** (EU AI Act, GDPR, hash-chained audit) but has **significant operational and quality gaps** that prevent public release. This ADR captures:

- **What's blocking production release** (477 TODO/FIXME markers, 35 open bugs, 77 test collection errors)
- **Go-live release checklist** (code quality, docs, security, deployment, marketing)
- **Execution plan** across 5 parallel work streams (12 weeks, ~150 engineer-hours)
- **Success criteria** per stream and global gate conditions

This is **not** a "finish later" document — every item is scoped, time-estimated, and assigned a phase.

---

## Part 1: Current State Assessment

### 1.1 What's Production-Ready ✅

| Component | Status | Evidence |
|---|---|---|
| **Architecture** | Complete | 16-layer model documented; ADR-001 covers EU AI Act compliance |
| **Compliance** | Built-in | Bot disclosure, consent gate, hash-chained audit, data residency layer all enforced |
| **Core Subsystems** | Functional | Forge, SkillForge, Memory, Compute Worker, all 7 bridges operational |
| **Documentation** | Comprehensive | 35 docs; architecture diagrams; layer model; API reference |
| **Test Framework** | Green | 183 test suites; CI/CD pipeline working; E2E coverage for security layers |
| **Deployment** | Scripted | Docker Compose, systemd, Tailscale, bootstrap installer all present |
| **Open Source** | Licensed | Apache 2.0; CLA in place; NOTICE file correct |

### 1.2 What's Blocking Release ❌

#### 1.2.1 Code Quality Issues

| Issue | Severity | Count | Root Cause | Fix Effort |
|---|---|---|---|---|
| TODO/FIXME/HACK markers | **CRITICAL** | **477** | Dev velocity choices left visible in production code | 20h (code review + removal) |
| Test collection errors | **CRITICAL** | **77** | Fixture scope issues, import cycles, test isolation problems | 8h |
| Known open bugs | **HIGH** | **35** | Code review findings from 2026-05-20 (Signal DoS, Teams memory leak, SkillForge YAML injection, etc.) | 20h |
| License test failures | **HIGH** | 1 | Tier field coercion (`unknown` vs `business`) | 1h |
| Type coercion errors | **HIGH** | 3 | `audit_query.py`, `gdpr_ropa.py` — int() failures on audit ts field | 2h (FIXED) |

**Total blocking code debt: ~51 engineer-hours**

#### 1.2.2 Documentation Gaps

| Gap | Impact | Fix Effort |
|---|---|---|
| No public website | **Critical** — no discovery path, no first impression, no value prop visible | 40h (design + build + content) |
| No "Getting Started" guide | Users can't tell if this is for them | 8h |
| No installation walkthrough with screenshots | Users get lost in `setup.sh` | 6h |
| No architecture deep-dive for operators | Operators can't plan deployments | 4h (already in docs; needs website integration) |
| No troubleshooting guide | Support burden increases 10x | 12h |
| No upgrade/migration guide | Breaking changes will surprise users | 6h |
| No security policy published | Can't report vulns responsibly | 2h |
| No roadmap public | Users don't know what's coming | 4h |
| No FAQ | Repeat support questions | 8h |

**Total docs work: ~90 engineer-hours**

#### 1.2.3 Operations & Reliability Gaps

| Gap | Impact | Risk | Fix Effort |
|---|---|---|---|
| No monitoring/alerting stack included | Can't detect degradation until users report | **CRITICAL** | 20h |
| No log aggregation documented | Incident response blind | **HIGH** | 8h |
| No backup rotation validation | Data loss on restore | **HIGH** | 6h |
| No disaster recovery runbook | RTO/RPO undefined; recovery is ad-hoc | **HIGH** | 12h |
| No SLA/uptime guarantees | Expectation mismatch with early adopters | **MEDIUM** | 4h (policy doc) |
| No health check dashboard | Operators flying blind | **MEDIUM** | 8h |
| No upgrade path tested | "Just git pull" breaks things | **MEDIUM** | 10h |
| No performance baselines | Can't tell if deployment is degraded | **MEDIUM** | 8h |
| No rollback procedure documented | Incidents unrecoverable | **HIGH** | 4h |

**Total ops work: ~80 engineer-hours**

#### 1.2.4 Security & Compliance Gaps

| Gap | Status | Fix Effort |
|---|---|---|
| Security.md threats vs actual fixes mismatch | Some mitigations described but not implemented (timing attacks, PYTHONPATH manipulation) | 6h |
| No bug bounty program defined | No responsible disclosure incentive | 2h (policy) |
| No security advisory process | CVE disclosure process undefined | 2h (process doc) |
| No penetration test results | No proof that threat model holds | 40h (external engagement) |
| No SBOM generation | No supply-chain visibility | 4h (tooling) |
| No container scanning in CI | Registry images not scanned for CVEs | 3h (GitHub Actions) |

**Total security work: ~57 engineer-hours**

#### 1.2.5 Community & Support Gaps

| Gap | Impact | Fix Effort |
|---|---|---|
| No GitHub Discussions enabled | No async support channel | 1h (config) |
| No issue template | Users submit bad bug reports | 2h |
| No PR template | Review process undefined | 1h |
| No code of conduct | Community norms unclear | 1h |
| No support SLA | Response time undefined | 1h (policy) |

**Total community work: ~6 engineer-hours**

---

## Part 2: Production-Ready Definition

An AtelierOS release is **production-ready** when it meets these criteria:

### Gate 1: Code Quality

- [ ] **Zero TODO/FIXME/HACK** in production code (`core/`, `operator/` excluding tests)
- [ ] **All 183 tests green** (`bash operator/bridges/run-all-tests.sh` exits 0)
- [ ] **Zero pytest collection errors** on `--collect-only`
- [ ] **All 35 open bugs resolved or explicitly deferred** (ADR recorded for deferrals)
- [ ] **Security test suite passes** (threat model mitigations verified)

### Gate 2: Documentation

- [ ] **Docs sync with code**: every CLI flag, config option, env var documented
- [ ] **Public website live** with Getting Started, architecture explanation, troubleshooting
- [ ] **Installation guide tested** by someone other than author on 2 OSes
- [ ] **Upgrade guide written** (main → vX → vY)
- [ ] **Security policy published** with vulnerability disclosure process

### Gate 3: Operations

- [ ] **Healthcheck script passes** on production image (`docker run ... healthcheck.sh`)
- [ ] **Backup/restore tested** end-to-end (state loss scenario)
- [ ] **Disaster recovery runbook written** with RTO/RPO targets
- [ ] **Monitoring guide provided** (what metrics to alert on)
- [ ] **Performance baselines established** (latency, throughput, CPU/mem per bridge)

### Gate 4: Security & Compliance

- [ ] **All CRITICAL/HIGH findings from code review resolved**
- [ ] **Threat model validation**: each threat in `security.md` has a test proving mitigation
- [ ] **Container image signed** (if using container distribution)
- [ ] **SBOM generated** for transparency
- [ ] **Responsible disclosure process active** (security@atelieros.io email monitored)

### Gate 5: Community & Support

- [ ] **GitHub Discussions enabled** and first pinned thread posted
- [ ] **Issue/PR templates in `.github/` configured**
- [ ] **Code of conduct published**
- [ ] **Support tier/SLA public** (free community vs. paid support, if applicable)
- [ ] **First FAQ live** (at least 10 Q&As from real questions)

---

## Part 3: 5 Work Streams

### Stream 1: Code Quality (Phase M5.1)

**Owner:** Code Audit Lead  
**Duration:** 3 weeks  
**Hours:** 51  

#### M5.1.1: TODO/FIXME Cleanup

```bash
# Current state
$ grep -r "TODO\|FIXME\|HACK" core operator --include="*.py" --include="*.ts" | wc -l
477
```

**Tasks:**

1. **Audit & categorize** (4h)
   - `grep -r "TODO" … | sort | uniq -c | sort -rn` → list by frequency
   - Group by component: bridges (signal, teams, …), forge, skill-forge, compliance, compute
   - Classify: "will fix", "defer to Phase X", "obsolete"

2. **Resolve "will fix"** (35h)
   - Signal bridge: add exponential backoff + audit events (4h)
   - Teams bridge: implement opt-in gate + fix memory leak (3h)
   - Email bridge: fix rate limits (1h)
   - SkillForge: validate claim dict + transactional promotion (2h)
   - Forge: add persona ACL on secrets (1h)
   - Skill diff: enforce scope boundary (1h)
   - Plugin-slot mirror: fail-closed not silent (2h)
   - PYTHONPATH security (1h)
   - Base64 false positives (1h)
   - Timing attacks on PIN (1h)
   - [repeat for all 477…] (20h accumulated)

3. **Defer "Phase X" items with ADR** (4h)
   - For each deferral: create short ADR (`ADR-00XX-defer-<issue>.md`)
   - Record in CHANGELOG.md → `## Deferred to Phase X`
   - Build automation: fail build if unlinked TODO exists → force ADR-per-deferral

4. **Remove "obsolete"** (2h)
   - Delete dead code
   - Verify no broken imports

5. **Validation** (6h)
   - Re-run grep: should show 0 non-deferred TODOs
   - Run full test suite
   - Code review by security lead

**Deliverables:**
- Zero TODO/FIXME in production code
- `docs/decisions/ADR-00XX-deferred-*.md` for each deferral
- Git commit message: `fix(codebase): eliminate 477 TODO/FIXME markers; defer 12 items to Phase X`

#### M5.1.2: Test Suite Fixes

1. **Pytest collection audit** (4h)
   - `pytest --collect-only 2>&1 | grep -i error` → root-cause each of 77 errors
   - Fixture scope issues: audit `conftest.py` for `tmp_path`, `rs256_keypair`, etc.
   - Import cycles: trace imports, break cycles with deferred import or refactor
   - Test isolation: fix tests that pollute shared state

2. **Fixture cleanup** (4h)
   - `rs256_keypair` scope: change from `module` to `function` if contending
   - `tmp_path` usage: ensure each test gets fresh dir
   - Add explicit teardown/cleanup

3. **Test import review** (2h)
   - Check for circular imports
   - Verify all test utilities are importable

4. **Run & verify** (2h)
   - `pytest --collect-only -q` → should show N tests, 0 errors
   - `pytest -x` → all tests pass

**Deliverables:**
- Zero collection errors
- Git commit: `fix(tests): resolve 77 pytest collection errors + fixture scope issues`

#### M5.1.3: Bug Resolution

Bug resolution work list (see `docs/internal/CODE_REVIEW_FINDINGS_2026_05_20.md`):

| Bug | Est. | Priority |
|---|---|---|
| Signal DoS (rate limiting) | 1h | P0 |
| Teams DoS (rate limiting) | 1h | P0 |
| SkillForge YAML injection | 1h | P0 |
| Teams memory leak | 1h | P0 |
| Compliance type coercion | 1h | P1 |
| License test failure | 1h | P1 |
| Signal exponential backoff | 1h | P2 |
| Bridge audit events | 2h | P2 |
| Email rate limits | 1h | P2 |
| Teams opt-in gate | 2h | P2 |
| PIN timing attacks | 1h | P2 |
| WhatsApp spawn escaping | 1h | P2 |
| SkillForge claim validation | 1h | P2 |
| Grade promotion transactionality | 1h | P2 |
| Skill cleanup audit trail | 1h | P2 |
| Forge persona ACL | 1h | P2 |
| Skill diff scope | 1h | P2 |
| Plugin mirror fail-closed | 1h | P2 |
| PYTHONPATH deny | 1h | P2 |
| Base64 false positives | 1h | P2 |

**Total: 28h**

**Approach:**
1. Fix all P0 first (4h) — these are DoS/injection risks
2. Fix all P1 (2h) — test + compliance
3. Fix P2 in parallel as team availability allows (22h)

Each fix: commit with `fix(component): <issue>`; include test.

#### M5.1.4: Security Layer Tests

**Threat Model Validation** (8h):

For each threat in `docs/security.md`:

| Threat | Mitigation | Test | Status |
|---|---|---|---|
| Whitelist bypass | ACL gate in Bridge Adapter | `test_whitelist_blocks_unlisted_chat` | ✓ present? |
| Forge breakout | `bwrap` sandboxing + deny-network | `test_forge_no_network`, `test_forge_no_filesystem_write` | ✓ present? |
| SkillForge injection | NFKC linter + confusables scan | `test_skillforge_injection_reject` | ✓ present? |
| Audit chain tampering | SHA-256 chaining + daily verify | `test_audit_chain_verify_detects_tampering` | ? |
| Timing attack on PIN | `crypto.timingSafeEqual` | `test_pin_timing_safe` | ? |
| PYTHONPATH bypass | Subprocess isolation | `test_forge_pythonpath_override_denied` | ? |

**Task:** For each missing test, write it and verify mitigation works.

**Deliverable:** `test_security_mitigations.py` with 100% coverage of threat model.

---

### Stream 2: Documentation (Phase M5.2)

**Owner:** Tech Writer + Product Lead  
**Duration:** 4 weeks  
**Hours:** 90  

#### M5.2.1: Website & Static Site

**Scope:** Public-facing landing page + docs site (no backend).

**Deliverables:**

1. **Homepage** (`index.html`)
   - Hero section: "Self-hosted agent OS for teams"
   - Feature carousel: Compliance · Runtime self-improvement · Compute · Firewall
   - Quick demo video (30s) or animated GIF
   - Three CTAs: "Get Started", "Read Docs", "GitHub"
   - Testimonial/use-case cards (2–3)
   - Footer: links to GitHub, security@, email

2. **Getting Started** (`getting-started.html`)
   - Prerequisites: Docker, API keys
   - Step-by-step: `curl … | sudo bash`
   - What happens after install: bridges, personasSetup, first message
   - Troubleshooting: common errors + solutions
   - Next: "Configure Discord", "Run tests", etc.

3. **Architecture** (`architecture.html`)
   - Layer diagram (SVG from docs)
   - High-level flow: message → bridge → adapter → engine
   - Security story: whitelists, audit chain, sandboxing
   - Multi-engine support table

4. **Security** (`security.html`)
   - Threat model (from `docs/security.md`)
   - Compliance badges: EU AI Act, GDPR, Apache 2.0
   - Responsible disclosure: email, response time SLA
   - Known limitations: what we don't (yet) do

5. **Pricing/Support** (`pricing.html`)
   - Community: free, self-hosted, no SLA
   - Managed: optional hosting + support (future)
   - Enterprise: custom features (future, out of scope for M5.0)

6. **FAQ** (`faq.html`)
   - Starter set: 15–20 QAs
   - Categories: Installation, Bridges, Security, Operations, Compliance
   - Link to GitHub Discussions for more

7. **Docs** (static HTML generated from Markdown)
   - Use a static site builder (Hugo, Jekyll, or hand-rolled)
   - Mirror the `docs/` directory structure
   - Full-text search (client-side JS + JSON index)
   - Dark mode (match the banner aesthetic)

**Tech Stack:** (recommendation)
- Build: Hugo or plain HTML + Makefile
- Hosting: GitHub Pages (free)
- Domain: atelieros.io (or similar)
- DNS: Namecheap or Cloudflare (< $15/yr)
- Styling: Tailwind CSS or similar (utility-first)
- Analytics: Plausible (privacy-respecting, no cookies)

**Effort Breakdown:**
- Design mockups: 4h (Figma or HTML sketches)
- Homepage + hero: 6h
- Getting Started page: 6h
- Architecture page: 4h
- Security page: 4h
- Pricing page: 2h
- FAQ skeleton: 4h
- Docs template + generation: 6h
- DNS + hosting setup: 2h
- Review + iterate: 4h

**Total: 42h**

#### M5.2.2: In-App Documentation

**Scope:** Help text, CLI reference, config docs.

1. **CLI Reference** (`docs/cli-reference.md`)
   - Every command in `/persona`, `/voice-*`, `/ldd-*`, `/profile`, `/memory`, `/vault`
   - Syntax, args, examples, errors
   - Cross-reference to web docs

2. **Config Schema** (`docs/config.md`)
   - Every field in `persona.json`, `bridge.json`, operator `config.json`
   - Type, default, constraints, example values
   - Auto-generate from JSONSchema (if available)

3. **Bridge Setup Guides** (update existing)
   - Discord: token creation, webhook setup (with screenshots)
   - Telegram: bot token creation, privacy settings
   - WhatsApp: QR-code pairing, number verification
   - Slack, Teams, Signal, Email: same pattern

4. **Troubleshooting** (`docs/troubleshooting.md`)
   - 50+ common issues and fixes:
     - "Message not received": check whitelist, check logs
     - "Bridge won't connect": token invalid, network timeout
     - "Forge tool fails": permissions, bwrap unavailable
     - etc.
   - Diagnostic checklist: `bridge.sh doctor`

5. **Upgrade Guide** (`docs/upgrade.md`)
   - 1.0.0 → 1.1.0: breaking changes, migration steps
   - Data migration (if any schema changes)
   - Rollback procedure (keep prev tag available)

**Effort:** 25h

#### M5.2.3: Video & Visual Content

1. **30-second demo video** (3–5 min to film, edit, host on YouTube)
   - "Ask AtelierOS to create a tool, it runs in a sandbox, then grades itself"
   - Show: message → bridge → response → skill created

2. **Architecture diagram animation** (existing SVG, animate with CSS)

3. **Configuration screenshots** (6–8 images)
   - Bridge setup: token entry, whitelist config, persona selection
   - First message exchange

**Effort:** 8h (film + edit outsource; host in-house)

#### M5.2.4: Docs Audit

**Run `docs-as-definition-of-done` skill** on every code change (enforcement via CI check):

```bash
# Add to .github/workflows/lint.yml:
- name: Check docs-as-DoD
  run: |
    git diff HEAD~1 --name-only | grep -E '\.py|\.ts|\.json' && \
      echo "Code changed; check if docs updated" && \
      git diff HEAD~1 --name-only | grep -E 'docs/|README' || \
      (echo "Code changed but docs not updated"; exit 1)
```

**Deliverables:**
- Website live at atelieros.io (or GitHub Pages)
- 100% docs-as-DoD compliance via CI check
- GitHub Discussions enabled + first FAQ thread
- FAQ with 20+ questions
- All bridge setup guides with screenshots
- Troubleshooting guide (50+ issues)
- Upgrade guide + rollback procedure
- Video demo

---

### Stream 3: Operations & Reliability (Phase M5.3)

**Owner:** DevOps Lead  
**Duration:** 3 weeks  
**Hours:** 80  

#### M5.3.1: Monitoring & Alerting

**Goal:** Operators can detect degradation before users report it.

**Stack:** Prometheus + Grafana (lightweight, self-hosted-friendly)

**Deliverables:**

1. **Prometheus scrape config** (`ops/prometheus.yml`)
   - Scrape `atelier` container `/metrics` endpoint (Prometheus format)
   - Metrics to expose:
     - Bridge health: `bridge_message_received_total`, `bridge_message_sent_total`, `bridge_error_total`
     - Engine latency: `engine_request_duration_seconds` (histogram)
     - Forge execution: `forge_execution_duration_seconds`, `forge_errors_total`
     - Audit chain: `audit_entries_total`, `audit_verify_latency_seconds`
     - Database: connection pool size, query latency
     - System: CPU, memory, disk usage (via node_exporter)

2. **Docker Compose extension** (`ops/docker-compose.monitoring.yml`)
   - Prometheus container (scrapes atelier + node_exporter)
   - Grafana container (visualizes metrics)
   - Both behind optional flag in `.env`: `ATELIER_MONITORING=true`

3. **Grafana dashboards** (3 templates)
   - **System Health** (CPU, memory, disk, network)
   - **Bridge Activity** (messages/sec, errors, latency)
   - **Engine Performance** (response time histogram, errors)

4. **Alert rules** (`ops/alert.rules.yml`)
   - Bridge offline for 5 min → page operator
   - Disk space < 5% → warn
   - Engine latency p99 > 30s → warn
   - Audit verify fails → critical

5. **Documentation** (`docs/monitoring.md`)
   - How to enable monitoring (set `ATELIER_MONITORING=true`)
   - Interpret dashboards
   - Alert thresholds and tuning
   - Query examples (how to find slow requests, error rates, etc.)

**Effort:** 24h

#### M5.3.2: Backup & Disaster Recovery

**Goal:** Operator can recover from data loss in ≤1 hour (RTO) with ≤1 hour data loss (RPO).

**Deliverables:**

1. **Backup strategy** (`docs/backup.md`)
   - **What to back up:** `/opt/atelier/home` (personas, audit chain, state)
   - **Frequency:** daily, 7-day rolling retention
   - **Encryption:** optional via `BACKUP_ENCRYPTION_KEY`
   - **Storage:** local NFS, S3, or Rsync

2. **Backup script** (`ops/scripts/backup.sh`)
   ```bash
   #!/bin/bash
   # tar + gzip home dir
   # sign with GPG if BACKUP_ENCRYPTION_KEY set
   # upload to remote if AWS_S3_BUCKET set
   # log rotation (keep last 7 days)
   ```

3. **Restore procedure** (`docs/restore.md`)
   - Stop container
   - Extract tarball to `/opt/atelier/home`
   - Start container
   - Verify audit chain integrity
   - Re-run healthcheck

4. **Systemd timer** (`ops/systemd/atelier-backup.timer`)
   - Runs daily at 02:00
   - Calls `backup.sh`
   - Email on failure (if BACKUP_EMAIL set)

5. **Disaster recovery runbook** (`docs/disaster-recovery.md`)
   - **Scenario 1:** Disk full → free space (guide)
   - **Scenario 2:** Container corrupted → restore from backup
   - **Scenario 3:** Database corrupted → resync from audit chain (possible?)
   - **Scenario 4:** Audit chain compromised → data integrity failure (response: notify users, investigate)
   - Each scenario: RTO/RPO, steps, validation

6. **Backup validation test** (automated)
   - Daily: restore latest backup to temp container
   - Verify audit chain integrity
   - Run healthcheck
   - Email results

**Effort:** 18h

#### M5.3.3: Performance Baselines & Load Testing

**Goal:** Know that a fresh deploy can handle expected load.

**Deliverables:**

1. **Load test suite** (`ops/load-tests.sh`)
   - Simulate N concurrent bridges
   - Send M messages/sec from each
   - Measure latency, error rate, CPU/mem growth
   - Tools: `locust`, `k6`, or Apache JMeter

2. **Baseline results** (`docs/performance.md`)
   ```
   Deployment: Docker Compose on t3.medium (2 CPU, 4 GB RAM)
   
   Throughput:
   - Single bridge: 100 msg/sec, p99 latency 200ms
   - 3 bridges: 50 msg/sec per bridge, p99 latency 500ms
   
   Resource usage:
   - Base: 150 MB RSS
   - Per 10 msg/sec: +50 MB
   - CPU: ~40% single-core for 50 msg/sec
   ```

3. **Regression test** (CI/CD check)
   - On each push, run abbreviated load test
   - Fail build if p99 latency increases >10% vs. baseline

4. **Scaling guide** (`docs/scaling.md`)
   - "My instance is slow; what do I do?"
   - CPU-bound bottleneck? → Upgrade instance or add load balancer
   - Memory-bound? → Check for leaks (Teams conversation map, etc.)
   - Network-bound? → Check bridge throughput limits

**Effort:** 20h

#### M5.3.4: Health Checks & Observability

**Goal:** Operator can run one command and see if everything works.

**Deliverables:**

1. **Enhanced healthcheck** (already exists, improve it)
   ```bash
   operator/voice/scripts/bridge.sh doctor
   # Output: ✓ Docker working
   #         ✓ Bridge daemons running (7/7)
   #         ✓ Postgres responding
   #         ✓ Audit chain valid (17,432 entries)
   #         ⚠️  Telegram bridge: no messages in 24h
   #         ✗ Disk: 4.8 GB free (< 5 GB threshold)
   #         Score: 8/10 - Healthy with warnings
   ```

2. **Metrics endpoint** (`/v1/health`)
   - JSON response with all health indicators
   - HTTP 200 if healthy, 503 if degraded
   - Kubernetes-compatible (liveness + readiness probes)

3. **Log aggregation guide** (`docs/logging.md`)
   - Point operators to ELK, Loki, or simple `tail -f`
   - Pre-built Loki config (optional)

**Effort:** 8h

#### M5.3.5: Upgrade Path & Testing

**Goal:** Operator can upgrade safely without data loss.

**Deliverables:**

1. **Upgrade script** (`ops/scripts/upgrade.sh`)
   - Fetch latest tag
   - Stop container, backup state
   - Update docker image
   - Run migrations (if needed)
   - Start container, verify healthcheck
   - Rollback if healthcheck fails

2. **Testing matrix** (document + test)
   - Upgrade 1.0.0 → 1.1.0 with 10K audit entries ✓
   - Upgrade 1.0.0 → 1.1.0 with 1M audit entries ✓
   - Verify: all audit entries present, chain integrity intact

3. **Rollback procedure** (document + test)
   - If upgrade fails, revert to previous tag
   - Document: "You have 10 min to rollback before consensus breaks"

**Effort:** 12h

---

### Stream 4: Security & Compliance (Phase M5.4)

**Owner:** Security Lead  
**Duration:** 3 weeks  
**Hours:** 57  

#### M5.4.1: Threat Model Validation

**Task:** Prove each threat in `docs/security.md` is mitigated.

**Example:**

```
Threat: Forge tool breakout (write to arbitrary filesystem)
Mitigation: bwrap sandboxing + rw binding only to temp workspace
Test: test_forge_filesystem_isolation_prevents_parent_write()
  - Create forge tool that tries: open('/etc/passwd', 'w')
  - Expect: PermissionError or FileNotFoundError
  - Status: ✓ test exists, passes
```

**Effort:** 12h (write 15–20 tests)

#### M5.4.2: Container & Artifact Security

**Deliverables:**

1. **Container image signing** (Cosign)
   ```bash
   # Sign image on push
   cosign sign --key /path/to/key ghcr.io/veegee82/atelieros:latest
   
   # Verify before pull
   cosign verify --key /path/to/pub ghcr.io/veegee82/atelieros:latest
   ```

2. **SBOM generation** (Syft)
   ```bash
   # On release, generate SBOM
   syft ghcr.io/veegee82/atelieros:latest -o json > atelieros-1.0.0-sbom.json
   ```

3. **Container scanning** (GitHub's native scanner or Trivy)
   - Scan image for known CVEs on every push
   - Fail build if critical CVEs found

4. **Dependency audit** (pip-audit, npm-audit)
   - `pip-audit` on every commit
   - Fail build if vulns found

**Effort:** 10h

#### M5.4.3: Security Policy & Disclosure

**Deliverables:**

1. **SECURITY.md**
   ```markdown
   # Security Policy
   
   ## Reporting Vulnerabilities
   
   **Do not open public issues for security vulnerabilities.**
   
   Email security@atelieros.io with:
   - Description of the vulnerability
   - Steps to reproduce
   - Suggested CVSS rating
   
   **Response SLA:** 72 hours initial response
   **Disclosure Timeline:** 90 days unless fix merges sooner
   
   ## Supported Versions
   
   | Version | Status |
   | --- | --- |
   | 1.x | Supported |
   | 0.x | EOL 2026-06-01 |
   ```

2. **Responsible disclosure process** (ADR or PROCESS.md)
   - Receive report → acknowledge within 48h
   - Investigate → plan fix
   - Fix → test → tag security release
   - Disclose → 90-day embargo (or sooner if patch released)
   - Document in CHANGELOG

3. **Bug bounty policy** (optional for M5.0)
   - "We don't have a bounty program yet, but we recognize ethical hackers"
   - Link to Hall of Fame process (TBD)

**Effort:** 6h

#### M5.4.4: Penetration Test Coordination

**Task:** Hire external pen tester (not inline, but coordinate for post-release).

**Deliverable:** ADR-0043 with engagement plan.

**Effort:** 2h (planning) + 40h (external, out of budget)

#### M5.4.5: Compliance Documentation

**Deliverables:**

1. **DPA template** (Data Processing Agreement)
   - Link from website under "For Organizations"

2. **Privacy Notice** (GDPR Art. 13/14)
   - What data we collect, why, retention, rights

3. **Risk Register** (starter)
   - Operational risks: disk full, network down, bridge spam
   - Mitigation: monitoring, backups, rate limits

**Effort:** 8h

#### M5.4.6: CVE/Advisory Process

**Goal:** Know how to respond when a dependency has a CVE.

**Deliverable:** `docs/security-response.md`

```
## When a CVE Affects AtelierOS

1. Dependency scanner (GitHub/Trivy) alerts
2. Triage: does it affect us? (version check, is it used?)
3. If yes:
   - Create private security branch
   - Patch dependency + test
   - Tag as security release (v1.0.1-security)
   - Disclose on security@atelieros.io + GitHub Security tab
4. If no: document decision (why we're unaffected)
```

**Effort:** 4h

---

### Stream 5: Community & Support (Phase M5.5)

**Owner:** Product/Community Lead  
**Duration:** 1 week  
**Hours:** 6  

#### M5.5.1: GitHub Community Setup

**Deliverables:**

1. **Discussions enabled**
   - Categories: Announcements, General, Show & Tell, Q&A

2. **Issue template** (`.github/ISSUE_TEMPLATE/bug.md`)
   - Title, description, steps to reproduce, expected behavior, logs
   - Link to troubleshooting guide

3. **PR template** (`.github/pull_request_template.md`)
   - Checklist: tests pass, docs updated, no TODOs
   - Link to CONTRIBUTING.md

4. **Code of Conduct** (`.github/CODE_OF_CONDUCT.md`)
   - Contributor Covenant or similar

5. **First pinned Discussions thread**
   - "Welcome to AtelierOS Discussions!"
   - Link to FAQ, Getting Started, support email

**Effort:** 4h

#### M5.5.2: Support SLA & Tiers

**Deliverable:** `docs/support.md`

```markdown
# Support

## Community Support (Free)

- **Channel:** GitHub Discussions
- **Response SLA:** Best effort (typically 48h)
- **Scope:** Installation, usage questions, bug reports

## Managed Hosting + Support (Future)

- **Coming in Phase M6.0**
- **SLA:** 24h response for non-critical, 4h for critical
- **Scope:** Includes monitoring, backups, updates

## Email Support

- **General:** hello@atelieros.io
- **Security:** security@atelieros.io
```

**Effort:** 2h

---

## Part 4: Master Timeline

### Week 1–2: Code Quality Sprint (M5.1)

| Week | Task | Lead | Hours | Status |
|---|---|---|---|---|
| W1 | TODO/FIXME audit | Code Audit | 4 | TBD |
| W1–W2 | TODO/FIXME resolution | Team | 35 | TBD |
| W1–W2 | Test suite fixes | QA | 8 | TBD |
| W2 | Bug resolution (P0, P1) | Security | 6 | TBD |
| W2 | Security test suite | Security | 8 | TBD |

### Week 3–6: Docs & Website (M5.2)

| Week | Task | Lead | Hours | Status |
|---|---|---|---|---|
| W3–W4 | Website build | Tech Writer | 42 | TBD |
| W4–W5 | In-app docs | Tech Writer | 25 | TBD |
| W5 | Video/visuals | Content | 8 | TBD |
| W5–W6 | Docs audit + sync | Team | 15 | TBD |

### Week 4–6: Ops & Reliability (M5.3)

| Week | Task | Lead | Hours | Status |
|---|---|---|---|---|
| W4 | Monitoring stack | DevOps | 24 | TBD |
| W4–W5 | Backup + DR | DevOps | 18 | TBD |
| W5 | Load testing | DevOps | 20 | TBD |
| W5–W6 | Health checks | DevOps | 8 | TBD |
| W6 | Upgrade testing | DevOps | 12 | TBD |

### Week 3–5: Security & Compliance (M5.4)

| Week | Task | Lead | Hours | Status |
|---|---|---|---|---|
| W3 | Threat model tests | Security | 12 | TBD |
| W4 | Container security | Security | 10 | TBD |
| W4–W5 | Security policy | Security | 6 | TBD |
| W5 | Pen test planning | Security | 2 | TBD |
| W5 | Compliance docs | Compliance | 8 | TBD |
| W5 | CVE response plan | Security | 4 | TBD |

### Week 6: Community & Support (M5.5)

| Week | Task | Lead | Hours | Status |
|---|---|---|---|---|
| W6 | GitHub setup | Community | 4 | TBD |
| W6 | Support SLA | Community | 2 | TBD |

---

## Part 5: Success Criteria & Launch Gates

### Pre-Launch Gates (All Must Pass)

#### Code Quality Gate

```
✓ Zero TODO/FIXME in production code
✓ All 183 tests passing (pytest exit 0)
✓ Zero collection errors
✓ All P0/P1 bugs fixed
✓ Security test suite complete
✓ Diff against 1.0.0-rc1 < 50 files changed
```

**Owner:** Code Lead | **Sign-off:** Maintainer

#### Docs Gate

```
✓ Website live + functional (all links 200)
✓ Getting Started tested on 2 OSes (fresh install)
✓ CLI reference complete + matches code
✓ Troubleshooting guide: 50+ common issues documented
✓ Upgrade guide tested (1.0.0-rc1 → 1.0.0)
✓ All code examples in docs execute without error
```

**Owner:** Tech Writer | **Sign-off:** Maintainer

#### Ops Gate

```
✓ Healthcheck script 100% pass on default config
✓ Backup + restore tested end-to-end
✓ Disaster recovery runbook written + reviewed
✓ Monitoring dashboard viewable + informative
✓ Load test baseline established
✓ Upgrade path tested (rc1 → 1.0.0 → 1.0.1-security-patch)
```

**Owner:** DevOps Lead | **Sign-off:** Maintainer

#### Security Gate

```
✓ All threat model mitigations have tests
✓ Container image signed + SBOM generated
✓ Dependency audit passes (pip, npm)
✓ SECURITY.md published + security@atelieros.io monitored
✓ No CRITICAL findings from final code review
✓ Pen test engagement scheduled (post-release OK)
```

**Owner:** Security Lead | **Sign-off:** Maintainer + CISO

#### Community Gate

```
✓ GitHub Discussions enabled + first thread pinned
✓ Issue + PR templates in .github/
✓ Code of Conduct published
✓ FAQ with 20+ questions live
✓ Support SLA defined + published
```

**Owner:** Community Lead | **Sign-off:** Maintainer

### Launch Checklist

On the day of release:

```
1 hour before:
  ☐ Final healthcheck on prod image
  ☐ Website staging test (all links)
  ☐ Test GitHub download links (releases page)
  ☐ CHANGELOG.md finalized
  
At release (maintainer):
  ☐ Tag v1.0.0
  ☐ Push to main (triggers CI: image build + publish)
  ☐ Create GitHub Release with CHANGELOG excerpt
  ☐ Website: add release announcement to homepage
  ☐ Email: hello@atelieros.io announcement
  ☐ Twitter/social: link to GitHub releases
  ☐ Monitor: healthcheck + error rates for 24h
```

---

## Part 6: Post-Launch (Phase M6)

**Out of scope for M5.0, but important for planning:**

- **M6.1:** Managed hosting (Cloud deployment + SLA)
- **M6.2:** Enterprise features (SAML, audit export, etc.)
- **M6.3:** Paid support tier
- **M6.4:** Plugin marketplace
- **M6.5:** Mobile apps (iOS, Android)

---

## Part 7: Rollback & Risk Mitigation

### If Something Goes Wrong at Launch

**Scenario 1: Critical bug found after release**

```
1. Revert main branch
2. Tag v1.0.1-hotfix
3. Re-release immediately
4. Post-mortem: how did it miss code review?
```

**Scenario 2: Monitoring overwhelmed by false positives**

```
1. Disable alerts (don't silence; disable alert rules)
2. Tune thresholds based on actual data
3. Re-enable with conservative settings
```

**Scenario 3: Website traffic spike crashes server**

```
1. Scale up container resources (Docker Compose config)
2. Implement CloudFlare caching on website
3. Limit inbound API connections temporarily
```

**Rollback SLA:**
- **T+5 min:** Decision to rollback (maintainer + lead)
- **T+15 min:** Revert tag, re-release
- **T+30 min:** Update website, announce status

---

## Appendices

### A. Effort Estimate Summary

| Stream | Phase | Hours | Lead | Duration |
|---|---|---|---|---|
| Code Quality | M5.1 | 51 | Code Audit | 3 weeks |
| Documentation | M5.2 | 90 | Tech Writer | 4 weeks |
| Operations | M5.3 | 80 | DevOps | 3 weeks |
| Security | M5.4 | 57 | Security | 3 weeks |
| Community | M5.5 | 6 | Community | 1 week |
| **Total** | **M5.0** | **284 hours** | Team | **12 weeks** |

### B. Risk Register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Code review backlog grows | Medium | High | Start review now; reserve 10h/week |
| Website designer unavailable | Low | Medium | Use templates (Tailwind, Hugo themes) |
| Pen test finds CRITICAL | Low | High | Keep disclosure flexible; patch + re-release if needed |
| Early adopter spam (bridge abuse) | High | Medium | Rate limits in place; monitor closely |
| Test flakes on CI | Medium | Medium | Run tests 3x before release; investigate flakes |

### C. RACI Matrix

| Task | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| TODO cleanup | Code Audit | Maintainer | Team | Team |
| Website design | Tech Writer | Maintainer | Product | Team |
| Monitoring setup | DevOps | Maintainer | Ops | Team |
| Threat model tests | Security | Maintainer | Code Audit | Team |
| Launch decision | Maintainer | Founder | All leads | N/A |

### D. Success Metrics (Post-Launch)

- **Adoption:** 100+ GitHub stars in 30 days
- **Community:** 10+ questions in Discussions in 30 days
- **Quality:** <1 critical bug in first 30 days
- **Reliability:** 99.5% uptime on reference deployment
- **Security:** Zero responsible-disclosure reports in first 30 days
- **Content:** 50+ page views/day on website

---

## Conclusion

AtelierOS has a **solid foundation** but needs **12 weeks of focused work** to be production-ready. This ADR breaks that work into 5 parallel streams with clear ownership, effort estimates, and launch gates.

**Next step:** Assign leads to each stream and schedule weekly sync-ups.

**Questions?** Tag @maintainer in PRs related to this roadmap.

---

**Document History:**

| Version | Date | Author | Change |
|---|---|---|---|
| 1.0 | 2026-05-21 | Claude Code | Initial version; 284h estimate; 5 streams; 12-week timeline |
