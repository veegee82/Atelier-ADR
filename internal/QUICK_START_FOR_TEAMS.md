# AtelierOS v1.0.0 Release — Quick Start for Teams

**Target Release:** 2026-06-21  
**Total Effort:** 284 engineer-hours  
**Team Size:** 5 leads + contributors  
**Status:** PLANNING → EXECUTION (starting Week 1)  

---

## Your Role & Responsibilities

### If You're the Code Quality Lead

**Duration:** 3 weeks (Weeks 1–3)  
**Hours:** 51  
**Goal:** Zero TODO/FIXME markers, all tests green, all bugs fixed

**Week 1 Actions:**
1. Run `grep -r "TODO\|FIXME\|HACK" core operator --include="*.py" --include="*.ts" > TODOS_audit.txt`
2. Categorize each line: "will fix" / "defer to Phase X" / "obsolete"
3. Create CSV: `TODOS_audit_20260521.csv` with priority + effort estimates
4. Start resolving P0 items (Signal DoS, Teams DoS, SkillForge injection) — 4 items, 4h

**Week 2 Actions:**
1. Continue P0 + P1 bug fixes (11h)
2. Resolve test collection errors (8h)
3. Fix license test (1h)

**Week 3 Actions:**
1. Finish remaining TODO items (20h)
2. Run full test suite: `bash operator/bridges/run-all-tests.sh`
3. Target: **all 183 tests passing**, **zero collection errors**, **zero TODO/FIXME**

**Deliverables:**
- Git commit: `fix(codebase): eliminate 477 TODO/FIXME markers`
- Test suite: all passing
- Security test suite: 100% threat model coverage

**Sign-off Criteria:**
- [ ] `grep -r "TODO\|FIXME" core operator | wc -l` = 0
- [ ] `pytest --collect-only` → 0 errors
- [ ] `bash operator/bridges/run-all-tests.sh` → exit 0

---

### If You're the Tech Writer

**Duration:** 4 weeks (Weeks 3–6)  
**Hours:** 90  
**Goal:** Website live, Getting Started tested, all docs synced

**Week 3–4 Actions (42h: Website):**
1. Choose static site generator (Hugo recommended, or plain HTML)
2. Register domain (atelieros.io)
3. Build 7 pages:
   - Homepage (hero, features, CTAs, testimonials)
   - Getting Started (step-by-step install + first message)
   - Architecture (diagram + explanation)
   - Security (threat model + compliance)
   - Pricing (free tier, managed tier future, enterprise future)
   - FAQ (20+ Q&As)
   - Docs (mirror of docs/ directory)

4. Demo video (30s): "AtelierOS creates tools at runtime"
5. Screenshots: bridge setup, first message, persona selection

**Week 4–5 Actions (25h: In-app docs):**
1. CLI reference (every command, all options, examples)
2. Config schema (persona.json, bridge.json fields)
3. Bridge setup guides (Discord, Telegram, WhatsApp, Slack, Teams, Signal, Email)
4. Troubleshooting (50+ common issues + fixes)
5. Upgrade guide (how to upgrade safely, rollback procedure)

**Week 5 Actions (15h: Validation):**
1. Test Getting Started on fresh machine (macOS + Ubuntu)
2. Audit all docs for sync with code
3. Setup GitHub Discussions + pin FAQ thread
4. Internal review + iterate

**Sign-off Criteria:**
- [ ] Website loads, all links 200, responsive on mobile
- [ ] Getting Started tested on 2 OSes (< 10 min to first message)
- [ ] CLI reference complete (every command documented)
- [ ] Troubleshooting guide covers 50+ issues
- [ ] Upgrade guide tested end-to-end

---

### If You're the DevOps Lead

**Duration:** 3 weeks (Weeks 4–6)  
**Hours:** 80  
**Goal:** Monitoring, backups, disaster recovery, performance baselines

**Week 4 Actions (24h: Monitoring):**
1. Add metrics endpoint to atelier (`/metrics` → Prometheus format)
2. Expose: bridge health, engine latency, forge exec, audit chain
3. Create `docker-compose.monitoring.yml` with Prometheus + Grafana
4. Build 3 Grafana dashboards: System Health, Bridge Activity, Engine Performance
5. Define alert rules (bridge offline > 5 min → page, disk < 5% → warn)

**Week 4–5 Actions (18h: Backups):**
1. Write backup strategy doc (what, frequency, encryption, storage)
2. Create `ops/scripts/backup.sh` (tar + gzip + GPG + S3 upload)
3. Create systemd timer: `atelier-backup.timer` (daily 02:00)
4. Write disaster recovery runbook (4 scenarios: disk full, corrupted, etc.)
5. Test: backup → restore → verify audit chain integrity

**Week 5 Actions (20h: Performance):**
1. Create load test suite (N concurrent bridges, M msg/sec)
2. Establish baselines: throughput, latency (p99), CPU/memory
3. Add regression test to CI (fail if p99 latency > 10% vs baseline)
4. Write scaling guide (CPU vs memory vs network bottlenecks)

**Week 5–6 Actions (18h: Health checks):**
1. Enhance `bridge.sh doctor` (show all 7 bridges, audit chain, disk, CPU)
2. Add `GET /v1/health` endpoint (JSON, Kubernetes-compatible)
3. Write logging guide (ELK/Loki options)
4. Test upgrade path: 1.0.0-rc1 → 1.0.0 (with 1M audit entries)

**Sign-off Criteria:**
- [ ] Healthcheck 100% pass on default config
- [ ] Backup + restore tested, audit chain verified
- [ ] DR runbook covers 4 scenarios with RTO/RPO
- [ ] Monitoring dashboard shows real metrics
- [ ] Load baseline established, regression test in CI
- [ ] Upgrade path tested end-to-end

---

### If You're the Security Lead

**Duration:** 3 weeks (Weeks 3–5)  
**Hours:** 57  
**Goal:** Threat model validated, container signed, security policy live

**Week 3 Actions (12h: Threat Model Tests):**
1. For each threat in `docs/security.md`, write a test proving mitigation:
   - Whitelist bypass ← `test_whitelist_blocks_unlisted_chat()`
   - Forge breakout ← `test_forge_filesystem_isolation_prevents_parent_write()`
   - SkillForge injection ← `test_skillforge_injection_reject()`
   - Audit tampering ← `test_audit_chain_verify_detects_tampering()`
   - PIN timing attack ← `test_pin_timing_safe()`
   - PYTHONPATH bypass ← `test_forge_pythonpath_override_denied()`

2. Create `test_security_mitigations.py` with full coverage

**Week 4 Actions (10h: Container Security):**
1. Setup Cosign key pair, sign image on release
2. Setup Syft, generate SBOM on release
3. Enable GitHub container scanning (fail build if CRITICAL vulns)
4. Add `pip-audit` to CI (fail build on vulns)

**Week 4–5 Actions (6h: Security Policy):**
1. Write `SECURITY.md` (vuln reporting, response SLA 72h, disclosure 90d)
2. Setup `security@atelieros.io` monitoring
3. Document responsible disclosure process (receive → patch → tag → disclose)

**Week 5 Actions (2h: Pen Test Coordination):**
1. Write ADR-0043: Pen Test Engagement Plan (scope, timeline, budget)
2. Identify potential vendors (scheduling post-launch OK)

**Week 5 Actions (8h: Compliance):**
1. Write DPA template (link from website)
2. Write Privacy Notice (GDPR Art. 13/14)
3. Create risk register (operational risks + mitigations)

**Week 5 Actions (4h: CVE Response):**
1. Write `docs/security-response.md` (process when a CVE affects us)

**Week 5–6 Actions (15h: Final Security Review):**
1. Review all code changes from Streams 1–3
2. Threat model review
3. Sign-off memo

**Sign-off Criteria:**
- [ ] Threat model tests 100% pass
- [ ] Container image signed, SBOM generated
- [ ] Dependency audit passes
- [ ] SECURITY.md published, security@atelieros.io live
- [ ] No CRITICAL findings in final review

---

### If You're the Product/Community Lead

**Duration:** 1 week (Week 6)  
**Hours:** 6  
**Goal:** GitHub Discussions setup, support SLA, templates

**Week 6 Actions:**
1. Enable GitHub Discussions (Categories: Announcements, General, Q&A, Showcase)
2. Create issue template (`.github/ISSUE_TEMPLATE/bug.md`)
3. Create PR template (`.github/pull_request_template.md`)
4. Create Code of Conduct (`.github/CODE_OF_CONDUCT.md`)
5. Write `docs/support.md` (community support, managed tier future, email contacts)
6. Pin first FAQ thread in Discussions

**Sign-off Criteria:**
- [ ] Discussions visible and categorized
- [ ] Issue/PR templates appear when user creates them
- [ ] Code of Conduct published
- [ ] FAQ thread pinned with 20+ Q&As
- [ ] Support SLA linked from website

---

## The Weekly Standup (Every Monday 10:00 UTC)

**1 minute per lead, ~5 min total:**

```
Code Quality Lead: "TODO cleanup is 30% done (10/35 P0-P1 items). 
  Tests still have 21 failing suites on fixtures. 
  No blockers. ETA Week 2 for P0, Week 3 for all."

Tech Writer: "Homepage done, Getting Started in review. 
  Waiting on logo from design. 
  No blockers. ETA Week 4 for website live."

DevOps Lead: "Monitoring stack deployed to staging, Grafana working. 
  Backup script written, testing restore next. 
  No blockers. ETA Week 5 for all ops."

Security Lead: "Threat model tests 60% done, 4 of 6 written. 
  No blockers. ETA Week 5."

Product Lead: "Deferred to Week 6. On track."
```

---

## Launch Day (Week 7, T+0)

**1 hour before:**
- Final healthcheck on prod image
- Website staging test (all links 200)
- GitHub release page ready

**Release (Maintainer):**
```bash
git tag -a v1.0.0 -m "AtelierOS v1.0.0 — production ready"
git push origin v1.0.0
# → CI builds image, pushes to ghcr.io
# → Create GitHub Release with CHANGELOG excerpt
# → Update website homepage
# → Tweet announcement
```

**1 hour after:**
- Monitor error rates
- Check GitHub Discussions for questions

**24 hours after:**
- If stable → celebrate 🎉
- If critical issues → prepare v1.0.1 hotfix

---

## Key Decisions to Make NOW

1. **Assign 5 stream leads** (or re-distribute if team is smaller)
2. **Pick website domain** (atelieros.io? github.io? custom?)
3. **Pick static site builder** (Hugo? Jekyll? Plain HTML?)
4. **Decide: all 7 bridges in v1.0.0 or just core 3?** (I recommend all 7)
5. **Pen test pre- or post-launch?** (Post is fine)

---

## Common Questions

**Q: Can I parallelize the streams?**  
A: Yes! Code Quality, Docs, Ops, Security can all run Weeks 1–6 in parallel. Community is best left to Week 6.

**Q: What if I miss a deadline?**  
A: Slip is OK. Prioritize in order: Code Quality → Ops → Docs → Security → Community. You can launch with slightly-rough docs.

**Q: What if a critical bug is found post-launch?**  
A: Tag v1.0.1 hotfix, re-release in 24h. Document post-mortem.

**Q: Can we split the work across fewer people?**  
A: Yes, but then:
  - Code Quality lead doubles (Code + Security tests)
  - Tech Writer + Product lead merge (Docs + Community)
  - Total effort stays same; timeline extends to 14–16 weeks

**Q: What if we want to launch sooner?**  
A: Focus on: Code Quality (1 week) → Ops (2 weeks) → Launch. Cut: Docs (use existing), Community (do post-launch). Total: 3 weeks. Trade: less polish.

---

## Resources

- **Full roadmap:** [`docs/decisions/ADR-0042-production-ready-roadmap.md`](docs/decisions/ADR-0042-production-ready-roadmap.md)
- **Executive summary:** [`PRODUCTION_READY_SUMMARY.md`](PRODUCTION_READY_SUMMARY.md)
- **Master checklist:** [`LAUNCH_CHECKLIST.md`](LAUNCH_CHECKLIST.md) (this file)
- **Code review findings:** [`CODE_REVIEW_FINDINGS_2026_05_20.md`](CODE_REVIEW_FINDINGS_2026_05_20.md)

---

## Slack Channel / Standups

Suggest: `#atelieros-v1-release` (private, invite all 5 leads + key contributors)

**Standing meetings:**
- Monday 10:00 UTC: Standup (5 min)
- Wednesday 14:00 UTC: Blocker resolution (if needed)
- Friday 16:00 UTC: Gate reviews (Week 6 only)

---

**Questions?** Tag issues with `#release-v1.0.0` and mention the relevant lead.

**Need to escalate?** Create a GitHub Issue with `[RELEASE BLOCKER]` prefix.

Let's ship this! 🚀

---

*Generated: 2026-05-21*  
*Release Target: 2026-06-21*
