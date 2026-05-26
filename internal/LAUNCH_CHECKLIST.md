# AtelierOS v1.0.0 Launch Checklist

**Release Target:** 2026-06-21  
**Status:** NOT READY (Planning → Execution → Testing → Launch)  
**Last Updated:** 2026-05-21  

---

## Overview

This is the **master checklist** for v1.0.0 release. Five work streams run in parallel (Weeks 1–6), then all gates must pass before launch (Week 6–7).

**How to use this:**
1. Assign one person per stream (see RACI below)
2. Each week, update status: ` ` (not started) → `⏳` (in progress) → `✓` (complete) → `✗` (blocked)
3. Move blocked items to "Blockers" section and flag in standup
4. Sign off on each gate once all items in the section are `✓`

---

## Stream 1: Code Quality (Lead: _______)

**Duration:** Weeks 1–3  
**Hours:** 51  
**Status:** [ ] NOT STARTED · [⏳] IN PROGRESS · [ ] COMPLETE  

### 1.1: TODO/FIXME Cleanup

- [ ] **Audit & categorize TODOs** (4h)
  - [ ] Run grep for all TODO/FIXME/HACK across core + operator
  - [ ] Group by component (bridges, forge, skill-forge, compliance, compute)
  - [ ] Classify: "will fix", "defer", "obsolete"
  - [ ] CSV output: `TODOS_audit_20260521.csv`

- [ ] **Resolve "will fix" items** (35h)
  - [ ] Signal bridge exponential backoff (4h)
  - [ ] Teams memory leak + opt-in gate (3h)
  - [ ] Email bridge rate limits (1h)
  - [ ] SkillForge YAML injection fix (1h)
  - [ ] SkillForge claim validation (1h)
  - [ ] Forge persona ACL on secrets (1h)
  - [ ] Skill diff scope boundary (1h)
  - [ ] Plugin-slot mirror fail-closed (1h)
  - [ ] PYTHONPATH security (1h)
  - [ ] Base64 false positives (1h)
  - [ ] PIN timing attacks (1h)
  - [ ] WhatsApp spawn escaping (1h)
  - [ ] Grade promotion transactionality (1h)
  - [ ] Skill cleanup audit trail (1h)
  - [ ] (+ additional issues per code review) (15h)

- [ ] **Defer "Phase X" items with ADR** (4h)
  - [ ] For each deferral, create ADR-00XX-defer-<issue>.md
  - [ ] Record in CHANGELOG.md under "Deferred"
  - [ ] Ensure deferred items have TODO comment linking to ADR

- [ ] **Remove "obsolete" code** (2h)
  - [ ] Delete unused code
  - [ ] Verify no import errors
  - [ ] Tests still pass

- [ ] **Validation** (6h)
  - [ ] `grep -r "TODO\|FIXME\|HACK" core operator | grep -v "Deferred" | wc -l` → **0**
  - [ ] Full test suite passes
  - [ ] Code review by security lead

**Sign-off:** [ ] Code Audit Lead

### 1.2: Test Suite Fixes

- [ ] **Pytest collection audit** (4h)
  - [ ] `pytest --collect-only 2>&1 > pytest_errors.log`
  - [ ] Root-cause each of 77 errors
  - [ ] Create fix PRs per issue category

- [ ] **Fixture cleanup** (4h)
  - [ ] Audit `conftest.py` scopes
  - [ ] Fix `tmp_path` contention
  - [ ] Fix `rs256_keypair` scope issues
  - [ ] Add explicit teardown where needed

- [ ] **Import cleanup** (2h)
  - [ ] Detect and fix circular imports
  - [ ] Verify all test utilities importable

- [ ] **Final validation** (2h)
  - [ ] `pytest --collect-only -q` → 0 errors
  - [ ] `pytest -x` → all pass

**Sign-off:** [ ] QA Lead

### 1.3: Bug Resolution (from CODE_REVIEW_FINDINGS_2026_05_20.md)

**P0 (Critical)** — Start immediately:

- [ ] Signal bridge DoS (rate limiting) — 1h
- [ ] Teams bridge DoS (rate limiting) — 1h
- [ ] SkillForge YAML injection — 1h
- [ ] Teams memory leak (LRU eviction) — 1h

**P1 (High)** — Next:

- [ ] Compliance type coercion (audit_query, gdpr_ropa) — 1h
- [ ] License test failure (tier coercion) — 1h

**P2 (Medium)** — Parallel track:

- [ ] Signal exponential backoff — 1h
- [ ] Bridge audit events — 2h
- [ ] Email rate limits — 1h
- [ ] Teams opt-in gate — 2h
- [ ] PIN timing attacks — 1h
- [ ] WhatsApp spawn escaping — 1h
- [ ] (+ 8 more from code review) — 8h

**Tracking:**
- [ ] All P0 fixed
- [ ] All P1 fixed
- [ ] All P2 fixed or deferred with ADR

**Sign-off:** [ ] Security Lead

### 1.4: Security Test Suite

- [ ] **Threat model validation** (8h)
  - [ ] Whitelist bypass test: `test_whitelist_blocks_unlisted_chat` — [ ]
  - [ ] Forge breakout test: `test_forge_no_filesystem_write` — [ ]
  - [ ] SkillForge injection test: `test_skillforge_injection_reject` — [ ]
  - [ ] Audit chain tampering test: `test_audit_chain_verify_detects_tampering` — [ ]
  - [ ] PIN timing safe test: `test_pin_timing_safe` — [ ]
  - [ ] PYTHONPATH isolation test: `test_forge_pythonpath_override_denied` — [ ]

- [ ] **Create `test_security_mitigations.py`** with 100% threat coverage

**Sign-off:** [ ] Security Lead

---

## Stream 2: Documentation (Lead: _______)

**Duration:** Weeks 3–6  
**Hours:** 90  
**Status:** [ ] NOT STARTED · [⏳] IN PROGRESS · [ ] COMPLETE  

### 2.1: Website & Static Site (42h)

- [ ] **Design & setup** (6h)
  - [ ] Choose static site builder (Hugo, Jekyll, or HTML + CSS)
  - [ ] Setup domain (atelieros.io or GH Pages)
  - [ ] Setup hosting (GitHub Pages, Netlify, or custom)

- [ ] **Homepage** (6h)
  - [ ] Hero section (value prop)
  - [ ] Feature carousel (4 main features)
  - [ ] Demo video or GIF (30s)
  - [ ] 3 CTAs: Get Started, Docs, GitHub
  - [ ] Testimonial/use-case cards
  - [ ] Footer

- [ ] **Getting Started** (6h)
  - [ ] Prerequisites
  - [ ] Step-by-step: `curl … | sudo bash`
  - [ ] What happens next
  - [ ] First message walkthrough
  - [ ] Common errors + fixes

- [ ] **Architecture** (4h)
  - [ ] Layer diagram (SVG from docs/)
  - [ ] Message flow explanation
  - [ ] Multi-engine support table

- [ ] **Security** (4h)
  - [ ] Threat model summary
  - [ ] Compliance badges (EU AI Act, GDPR)
  - [ ] Responsible disclosure info
  - [ ] Known limitations

- [ ] **Pricing/Support** (2h)
  - [ ] Community tier (free, self-hosted)
  - [ ] Managed tier (future)
  - [ ] Enterprise (future)

- [ ] **FAQ** (4h)
  - [ ] 15–20 common questions
  - [ ] Categories: Install, Bridges, Security, Ops, Compliance
  - [ ] Link to GitHub Discussions

- [ ] **Docs template + generation** (6h)
  - [ ] Mirror docs/ directory as HTML
  - [ ] Full-text search (client-side)
  - [ ] Dark mode toggle
  - [ ] Navigation sidebar

- [ ] **DNS + hosting setup** (2h)
  - [ ] Domain registered
  - [ ] DNS configured
  - [ ] Hosting ready
  - [ ] SSL cert (auto-renewal)

- [ ] **Review + iterate** (4h)
  - [ ] Internal review (team)
  - [ ] Fix issues
  - [ ] Final QA

**Sign-off:** [ ] Tech Writer

### 2.2: In-App Documentation (25h)

- [ ] **CLI Reference** (4h)
  - [ ] Every `/persona` command
  - [ ] Every `/voice-*` command
  - [ ] Every `/ldd-*` command
  - [ ] Every `/profile`, `/memory`, `/vault` command
  - [ ] Syntax, args, examples, errors
  - [ ] Cross-refs to web docs

- [ ] **Config Schema** (4h)
  - [ ] persona.json fields
  - [ ] bridge.json fields
  - [ ] operator config.json fields
  - [ ] Auto-generate from JSONSchema if available

- [ ] **Bridge Setup Guides** (8h)
  - [ ] Discord (token, webhook, whitelist setup)
  - [ ] Telegram (bot token, privacy)
  - [ ] WhatsApp (QR-code pairing, verification)
  - [ ] Slack, Teams, Signal, Email (same pattern)
  - [ ] All with screenshots

- [ ] **Troubleshooting** (6h)
  - [ ] 50+ common issues
  - [ ] Root-cause explanations
  - [ ] Diagnostic checklist
  - [ ] Escalation path (GitHub Issues)

- [ ] **Upgrade Guide** (3h)
  - [ ] vX.Y.Z → v(X+1).Y.Z migration
  - [ ] Breaking changes
  - [ ] Data migration (if needed)
  - [ ] Rollback procedure

**Sign-off:** [ ] Tech Writer

### 2.3: Video & Visual Content (8h)

- [ ] **30-second demo video** (4h)
  - [ ] Film & edit
  - [ ] Upload to YouTube
  - [ ] Embed on homepage

- [ ] **Architecture diagram animation** (2h)
  - [ ] Animate SVG with CSS

- [ ] **Configuration screenshots** (2h)
  - [ ] Bridge setup (token, whitelist, persona)
  - [ ] First message exchange
  - [ ] 6–8 images total

**Sign-off:** [ ] Content Lead

### 2.4: Docs Audit & Sync (15h)

- [ ] **Add CI check: docs-as-DoD**
  - [ ] Fail build if code changed but docs not updated
  - [ ] Add to `.github/workflows/lint.yml`

- [ ] **Run docs-as-DoD audit**
  - [ ] Review every code change → docs sync
  - [ ] Update ADRs as needed

- [ ] **GitHub Discussions setup** (2h)
  - [ ] Enable Discussions
  - [ ] Create categories (Announcements, General, Q&A, Showcase)
  - [ ] Pin welcome thread + FAQ

**Sign-off:** [ ] Tech Writer

---

## Stream 3: Operations & Reliability (Lead: _______)

**Duration:** Weeks 4–6  
**Hours:** 80  
**Status:** [ ] NOT STARTED · [⏳] IN PROGRESS · [ ] COMPLETE  

### 3.1: Monitoring & Alerting (24h)

- [ ] **Prometheus metrics** (6h)
  - [ ] Add `/metrics` endpoint to atelier
  - [ ] Expose: bridge health, engine latency, forge exec, audit chain, system metrics

- [ ] **Docker Compose extension** (4h)
  - [ ] Add Prometheus + Grafana containers
  - [ ] Tie to ATELIER_MONITORING env var

- [ ] **Grafana dashboards** (8h)
  - [ ] System Health dashboard
  - [ ] Bridge Activity dashboard
  - [ ] Engine Performance dashboard

- [ ] **Alert rules** (4h)
  - [ ] Bridge offline > 5 min → page
  - [ ] Disk space < 5% → warn
  - [ ] Engine latency p99 > 30s → warn
  - [ ] Audit verify fails → critical

- [ ] **Documentation** (2h)
  - [ ] How to enable monitoring
  - [ ] Dashboard interpretation guide
  - [ ] Alert tuning guide

**Sign-off:** [ ] DevOps Lead

### 3.2: Backup & Disaster Recovery (18h)

- [ ] **Backup strategy doc** (2h)
  - [ ] What to back up (home dir, audit chain)
  - [ ] Frequency (daily, 7-day rolling)
  - [ ] Encryption (optional GPG)
  - [ ] Storage (NFS, S3, Rsync)

- [ ] **Backup script** (4h)
  - [ ] tar + gzip
  - [ ] GPG signing (if enabled)
  - [ ] S3 upload (if enabled)
  - [ ] Rotation (keep 7 days)
  - [ ] Error handling + logging

- [ ] **Restore procedure** (4h)
  - [ ] Stop container
  - [ ] Extract tarball
  - [ ] Verify audit chain
  - [ ] Start container
  - [ ] Run healthcheck

- [ ] **Systemd timer** (2h)
  - [ ] `atelier-backup.timer` (daily 02:00)
  - [ ] `atelier-backup.service`
  - [ ] Email on failure

- [ ] **Disaster recovery runbook** (4h)
  - [ ] Disk full scenario
  - [ ] Container corrupted scenario
  - [ ] Database corrupted scenario
  - [ ] Audit chain compromised scenario
  - [ ] Each: RTO, RPO, steps, validation

**Sign-off:** [ ] DevOps Lead

### 3.3: Performance Baselines & Load Testing (20h)

- [ ] **Load test suite** (8h)
  - [ ] Simulate N concurrent bridges
  - [ ] Send M messages/sec
  - [ ] Measure latency, errors, CPU, memory

- [ ] **Baseline results** (4h)
  - [ ] Document: throughput (msg/sec), latency (p99), resource usage
  - [ ] Publish in `docs/performance.md`

- [ ] **Regression test in CI** (4h)
  - [ ] Abbreviated load test on every push
  - [ ] Fail if p99 latency increases > 10%

- [ ] **Scaling guide** (4h)
  - [ ] CPU-bound: upgrade instance or load balancer
  - [ ] Memory-bound: check for leaks
  - [ ] Network-bound: check bridge limits

**Sign-off:** [ ] DevOps Lead

### 3.4: Health Checks & Observability (8h)

- [ ] **Enhanced healthcheck** (3h)
  - [ ] Improve `bridge.sh doctor` output
  - [ ] Show bridge status (7/7 running)
  - [ ] Show audit chain status
  - [ ] Show disk/memory/CPU
  - [ ] Overall score

- [ ] **Metrics endpoint** (3h)
  - [ ] `GET /v1/health` → JSON
  - [ ] HTTP 200 if healthy, 503 if degraded
  - [ ] Kubernetes-compatible (liveness + readiness)

- [ ] **Log aggregation guide** (2h)
  - [ ] Point to ELK, Loki, or `tail -f`
  - [ ] Pre-built Loki config (optional)

**Sign-off:** [ ] DevOps Lead

### 3.5: Upgrade Path & Testing (12h)

- [ ] **Upgrade script** (4h)
  - [ ] Fetch latest tag
  - [ ] Backup state
  - [ ] Stop → update → start
  - [ ] Verify healthcheck
  - [ ] Rollback if needed

- [ ] **Testing matrix** (6h)
  - [ ] Test upgrade: 1.0.0 → 1.1.0 with 10K audit entries
  - [ ] Test upgrade: 1.0.0 → 1.1.0 with 1M audit entries
  - [ ] Verify: all audit entries present, chain integrity intact

- [ ] **Rollback procedure** (2h)
  - [ ] Document rollback steps
  - [ ] Test rollback end-to-end

**Sign-off:** [ ] DevOps Lead

---

## Stream 4: Security & Compliance (Lead: _______)

**Duration:** Weeks 3–5  
**Hours:** 57  
**Status:** [ ] NOT STARTED · [⏳] IN PROGRESS · [ ] COMPLETE  

### 4.1: Threat Model Validation (12h)

- [ ] **Whitelist bypass test** (1.5h)
  - [ ] `test_whitelist_blocks_unlisted_chat()` ✓

- [ ] **Forge breakout test** (1.5h)
  - [ ] `test_forge_filesystem_isolation_prevents_parent_write()` ✓
  - [ ] `test_forge_no_network_access()` ✓

- [ ] **SkillForge injection test** (1.5h)
  - [ ] `test_skillforge_injection_reject()` ✓
  - [ ] `test_skillforge_yaml_injection_reject()` ✓

- [ ] **Audit chain integrity test** (1.5h)
  - [ ] `test_audit_chain_verify_detects_tampering()` ✓
  - [ ] `test_audit_hash_chain_continuity()` ✓

- [ ] **PIN timing attack test** (1.5h)
  - [ ] `test_pin_timing_safe()` ✓

- [ ] **PYTHONPATH isolation test** (1.5h)
  - [ ] `test_forge_pythonpath_override_denied()` ✓

- [ ] **Additional security tests** (3h)
  - [ ] Path-gate enforcement tests
  - [ ] Persona ACL tests
  - [ ] Data flow guard tests
  - [ ] Egress lockdown tests

**Sign-off:** [ ] Security Lead

### 4.2: Container & Artifact Security (10h)

- [ ] **Container image signing** (3h)
  - [ ] Setup Cosign key pair
  - [ ] Sign image on release
  - [ ] Document verification process

- [ ] **SBOM generation** (2h)
  - [ ] Setup Syft
  - [ ] Generate SBOM on release
  - [ ] Publish alongside release

- [ ] **Container scanning in CI** (3h)
  - [ ] Enable GitHub container scanning
  - [ ] Fail build if CRITICAL vulns found
  - [ ] Document CVE response

- [ ] **Dependency audit** (2h)
  - [ ] `pip-audit` on every commit
  - [ ] `npm-audit` (if Node code present)
  - [ ] Fail build on vulns

**Sign-off:** [ ] Security Lead

### 4.3: Security Policy & Disclosure (6h)

- [ ] **SECURITY.md** (2h)
  - [ ] Vulnerability reporting process
  - [ ] Response SLA (72h acknowledgment)
  - [ ] Disclosure timeline (90 days)
  - [ ] Supported versions table

- [ ] **Responsible disclosure process** (2h)
  - [ ] Email: security@atelieros.io
  - [ ] Response timeline
  - [ ] Fix → tag → disclose
  - [ ] Document in CHANGELOG

- [ ] **Bug bounty policy** (2h)
  - [ ] "We recognize ethical hackers"
  - [ ] Hall of Fame process (TBD)

**Sign-off:** [ ] Security Lead

### 4.4: Penetration Test Coordination (2h)

- [ ] **ADR-0043: Pen Test Engagement** (2h)
  - [ ] Define scope (API, UI, deployment)
  - [ ] Define timeline (post-launch OK)
  - [ ] Budget estimate
  - [ ] Success criteria

**Sign-off:** [ ] Security Lead (planning only; execution post-launch)

### 4.5: Compliance Documentation (8h)

- [ ] **DPA template** (2h)
  - [ ] Data Processing Agreement
  - [ ] Link from website

- [ ] **Privacy Notice** (3h)
  - [ ] GDPR Art. 13/14 format
  - [ ] What data we collect, why, retention
  - [ ] User rights (access, deletion, portability)

- [ ] **Risk Register** (2h)
  - [ ] Operational risks
  - [ ] Mitigation strategies

- [ ] **Regulatory status** (1h)
  - [ ] EU AI Act: Art. 50 disclosure ✓
  - [ ] GDPR: Art. 6/7/17/30/32 ✓

**Sign-off:** [ ] Compliance Lead

### 4.6: CVE/Advisory Process (4h)

- [ ] **docs/security-response.md** (4h)
  - [ ] When a CVE affects us
  - [ ] Triage process
  - [ ] Response timeline
  - [ ] Disclosure process

**Sign-off:** [ ] Security Lead

### 4.7: Final Security Review (15h)

- [ ] **Code security review** (8h)
  - [ ] Review all changes from Streams 1–3
  - [ ] Check for new vulns

- [ ] **Architecture security review** (4h)
  - [ ] Threat model still valid?
  - [ ] New attack surfaces?

- [ ] **Security sign-off** (3h)
  - [ ] Sign-off memo
  - [ ] Known limitations documented
  - [ ] Acceptable risk register

**Sign-off:** [ ] Security Lead + CISO

---

## Stream 5: Community & Support (Lead: _______)

**Duration:** Week 6  
**Hours:** 6  
**Status:** [ ] NOT STARTED · [⏳] IN PROGRESS · [ ] COMPLETE  

### 5.1: GitHub Community Setup (4h)

- [ ] **Enable Discussions**
  - [ ] Categories: Announcements, General, Q&A, Showcase

- [ ] **Issue template** (`.github/ISSUE_TEMPLATE/bug.md`)
  - [ ] Title, description, steps, expected behavior, logs
  - [ ] Link to troubleshooting

- [ ] **PR template** (`.github/pull_request_template.md`)
  - [ ] Checklist: tests, docs, no TODOs
  - [ ] Link to CONTRIBUTING.md

- [ ] **Code of Conduct** (`.github/CODE_OF_CONDUCT.md`)
  - [ ] Contributor Covenant or similar

- [ ] **First pinned thread**
  - [ ] Welcome message
  - [ ] FAQ link
  - [ ] Getting Started link

**Sign-off:** [ ] Product Lead

### 5.2: Support SLA & Tiers (2h)

- [ ] **docs/support.md**
  - [ ] Community support (GitHub, best-effort)
  - [ ] Managed hosting (future)
  - [ ] Email contacts (hello@, security@)

**Sign-off:** [ ] Product Lead

---

## Launch Gates (Week 6–7)

### Gate 1: Code Quality ✓ SIGN-OFF

**All items must be complete:**

- [ ] Zero TODO/FIXME in production code
  - [ ] Verify: `grep -r "TODO\|FIXME\|HACK" core operator | grep -v "Deferred" | wc -l` = 0
  
- [ ] All 183 tests passing
  - [ ] Verify: `bash operator/bridges/run-all-tests.sh` exits 0
  
- [ ] Zero pytest collection errors
  - [ ] Verify: `pytest --collect-only -q 2>&1 | grep error` = (empty)
  
- [ ] All P0/P1 bugs fixed
  - [ ] Verify: CODE_REVIEW_FINDINGS_2026_05_20.md marked complete
  
- [ ] Security test suite complete
  - [ ] Verify: `pytest test_security_mitigations.py -v` passes

**Sign-off:** 
- [ ] Code Audit Lead
- [ ] Security Lead
- [ ] Maintainer

---

### Gate 2: Documentation ✓ SIGN-OFF

**All items must be complete:**

- [ ] Website live and functional
  - [ ] Verify: all pages load (200 status)
  - [ ] Verify: links work (no 404s)
  - [ ] Verify: responsive on mobile
  
- [ ] Getting Started tested
  - [ ] Verify: tested on macOS fresh install
  - [ ] Verify: tested on Ubuntu 22.04 fresh install
  - [ ] Verify: user can send first message in <10 min
  
- [ ] CLI reference complete
  - [ ] Verify: every command documented
  - [ ] Verify: examples run without error
  
- [ ] Troubleshooting guide (50+ issues)
  - [ ] Verify: covers common bridge issues
  - [ ] Verify: covers common operator errors
  
- [ ] Upgrade guide tested
  - [ ] Verify: can upgrade 1.0.0-rc1 → 1.0.0
  - [ ] Verify: can rollback 1.0.0 → 1.0.0-rc1
  
- [ ] Docs sync with code
  - [ ] Verify: no deprecated flags in docs
  - [ ] Verify: all config options documented

**Sign-off:**
- [ ] Tech Writer
- [ ] Product Lead
- [ ] Maintainer

---

### Gate 3: Operations ✓ SIGN-OFF

**All items must be complete:**

- [ ] Healthcheck 100% pass
  - [ ] Verify: `docker run … bash healthcheck.sh` → success
  
- [ ] Backup + restore tested
  - [ ] Verify: backup created successfully
  - [ ] Verify: restore to temp container successful
  - [ ] Verify: audit chain integrity after restore
  
- [ ] Disaster recovery runbook written
  - [ ] Verify: document covers 4 scenarios
  - [ ] Verify: RTO/RPO specified for each
  - [ ] Verify: runbook reviewed by ops team
  
- [ ] Monitoring dashboard viewable
  - [ ] Verify: Prometheus scraping metrics
  - [ ] Verify: Grafana dashboards display data
  
- [ ] Load test baseline established
  - [ ] Verify: baseline results documented
  - [ ] Verify: regression test passes
  
- [ ] Upgrade path tested
  - [ ] Verify: 1.0.0-rc1 → 1.0.0 → (hypothetical 1.0.1)
  - [ ] Verify: rollback works

**Sign-off:**
- [ ] DevOps Lead
- [ ] Maintainer

---

### Gate 4: Security ✓ SIGN-OFF

**All items must be complete:**

- [ ] Threat model tests complete
  - [ ] Verify: `test_security_mitigations.py` 100% pass
  
- [ ] Container image signed + SBOM generated
  - [ ] Verify: image signed with Cosign
  - [ ] Verify: SBOM generated and published
  
- [ ] Dependency audit passes
  - [ ] Verify: `pip-audit` → 0 vulns
  - [ ] Verify: `npm-audit` → 0 vulns (if applicable)
  
- [ ] SECURITY.md published
  - [ ] Verify: security@atelieros.io monitored
  - [ ] Verify: response SLA defined
  
- [ ] No CRITICAL findings
  - [ ] Verify: final code review complete
  - [ ] Verify: all findings either fixed or deferred with ADR

**Sign-off:**
- [ ] Security Lead
- [ ] CISO (if available)
- [ ] Maintainer

---

### Gate 5: Community ✓ SIGN-OFF

**All items must be complete:**

- [ ] GitHub Discussions enabled
  - [ ] Verify: categories visible
  - [ ] Verify: welcome thread pinned
  
- [ ] Issue + PR templates configured
  - [ ] Verify: templates appear when user creates issue/PR
  
- [ ] Code of Conduct published
  - [ ] Verify: .github/CODE_OF_CONDUCT.md exists
  
- [ ] FAQ with 20+ Qs
  - [ ] Verify: Discussions FAQ thread has 20+ Q&As
  
- [ ] Support SLA published
  - [ ] Verify: docs/support.md exists and is linked from website

**Sign-off:**
- [ ] Product Lead
- [ ] Community Lead
- [ ] Maintainer

---

## Release Day (T+0)

**1 hour before release:**

- [ ] Final healthcheck on prod image
  - [ ] Verify: `docker run ghcr.io/.../atelieros:latest bash healthcheck.sh` → success

- [ ] Website staging test
  - [ ] Verify: all links work
  - [ ] Verify: download link to latest release works

- [ ] GitHub release page ready
  - [ ] CHANGELOG excerpt prepared
  - [ ] Release notes written

**At release (Maintainer):**

- [ ] Tag v1.0.0
  - `git tag -a v1.0.0 -m "AtelierOS v1.0.0 — production ready"`
  - `git push origin v1.0.0`

- [ ] CI/CD triggers
  - [ ] GitHub Actions build image
  - [ ] Push to ghcr.io
  - [ ] Verify image published

- [ ] Create GitHub Release
  - [ ] Title: "AtelierOS v1.0.0 — Production Ready"
  - [ ] Body: CHANGELOG excerpt + highlights
  - [ ] Attach: SBOM .json file

- [ ] Website announcement
  - [ ] Add release card to homepage
  - [ ] Update "Latest Version" to 1.0.0

- [ ] Email announcement
  - [ ] Send to hello@atelieros.io mailing list (if exists)

- [ ] Social announcement
  - [ ] Tweet from @AtelierOS (or equivalent)
  - [ ] Post on relevant forums/communities

**1 hour after release:**

- [ ] Monitor error rates
  - [ ] Check application logs for errors
  - [ ] Check GitHub Issues for crash reports

- [ ] Monitor community
  - [ ] Monitor GitHub Discussions
  - [ ] Check security@atelieros.io for reports

**24 hours after release:**

- [ ] Stability check
  - [ ] No critical issues reported?
  - [ ] If yes → continue
  - [ ] If no → evaluate hotfix (v1.0.1)

---

## Blockers (Track Here)

| Issue | Severity | Owner | Status | ETA |
|---|---|---|---|---|
| (example) | P0 | @code-lead | BLOCKED on external dependency | 2026-06-01 |
| | | | | |

---

## Weekly Standup Template

**Every Monday 10:00 UTC:**

```markdown
## Stream 1: Code Quality (@code-lead)
- [ ] Week X: TODO cleanup — started, 10/35 items resolved
- [ ] Blockers: none
- [ ] ETA to complete: Week 2

## Stream 2: Documentation (@tech-writer)
- [ ] Week X: Website homepage done, getting started in progress
- [ ] Blockers: waiting for logo finalization
- [ ] ETA to complete: Week 5

## Stream 3: Operations (@devops-lead)
- [ ] Week X: Monitoring stack deployed to staging
- [ ] Blockers: none
- [ ] ETA to complete: Week 5

## Stream 4: Security (@security-lead)
- [ ] Week X: Threat model tests 80% complete
- [ ] Blockers: waiting for pen test vendor response
- [ ] ETA to complete: Week 5 (pen test post-launch)

## Stream 5: Community (@product-lead)
- [ ] Week X: deferred (starts Week 6)
- [ ] Blockers: N/A
- [ ] ETA to complete: Week 6
```

---

## Sign-Offs

### Pre-Launch (Week 6)

- [ ] **Code Quality Gate** — Signed: __________________ Date: __________
- [ ] **Documentation Gate** — Signed: __________________ Date: __________
- [ ] **Operations Gate** — Signed: __________________ Date: __________
- [ ] **Security Gate** — Signed: __________________ Date: __________
- [ ] **Community Gate** — Signed: __________________ Date: __________

### Launch Approval (Week 7)

- [ ] **Maintainer:** __________________ Date: __________ (GO/NO-GO)
- [ ] **CISO (if required):** __________________ Date: __________
- [ ] **Product Lead:** __________________ Date: __________

---

**Questions?** Tag issues with `#release-blockers` and flag in standup.

**Timeline questions?** Review the full [`ADR-0042-production-ready-roadmap.md`](docs/decisions/ADR-0042-production-ready-roadmap.md).

---

*Last updated: 2026-05-21*  
*Release Target: 2026-06-21 (12 weeks from planning start)*
