# Priority Roadmap: v0.30.0 → v1.0.0

**Current Status:** v0.30.0 tagged (Planning complete, docs ready)  
**Release Target:** v1.0.0 on 2026-06-21  
**Working Team:** Assume 1-2 full-time engineers  

---

## The Three Paths

### Path A: "Minimum Viable Release" (8 weeks, ~200h)
**Focus:** Code quality + Installation simplicity  
**Exclude:** Monitoring, fancy docs, multi-engine support  
**Result:** Production-ready for early adopters who know what they're doing

### Path B: "Production Release" (12 weeks, 284h) ← **RECOMMENDED**
**Focus:** Everything needed for launch  
**Exclude:** Fancy features, enterprise support, managed hosting  
**Result:** Solid v1.0.0 that normal users can install and use

### Path C: "Premium Release" (16 weeks, 400h)
**Focus:** All bells and whistles  
**Include:** Enterprise features, premium docs, marketing site  
**Result:** Wow everyone, but take longer

**My recommendation:** **Path B** (12 weeks)

---

## v0.30.0 → v1.0.0 Roadmap (Path B)

### Week 1-2: Code Quality ⚡
**Status:** 3% started (1/35 bugs fixed)  
**Focus:** Get tests green  

**Must-do:**
- [ ] Fix 34 remaining bugs (18h)
  - Use BUG_FIX_EXECUTION_GUIDE.md
  - Pick easy ones first (BR-7, BR-8, FK-1)
  - Commit after each fix
- [ ] Fix pytest collection errors (8h)
  - `pytest --collect-only` → 0 errors
- [ ] Write 7 security tests (8h)
  - Threat model validation suite

**Target:** All 183 tests passing ✅

**Effort:** 34h  
**Owner:** 1 engineer (full-time)  
**Blocker:** None (start immediately)

---

### Week 3-4: Installation Refactor — Phase 1 ⚡
**Status:** 0% started  
**Focus:** Website + Core Setup Daemon  

**Must-do:**
- [ ] Build setup website (12h)
  - HTML + vanilla JS (no build step)
  - Pages: Welcome, System Check, Engine Picker, Keys, OAuth, Bridges
  - Make it pretty but simple
- [ ] Build setup daemon (8h)
  - Express backend on port 8089
  - API endpoints for system check, config, health
  - OAuth callback handler
- [ ] Claude Code OAuth (5h)
  - Redirect to claude.ai/auth
  - Handle callback
  - Store API key in vault

**Target:** Website loads, OAuth works locally  

**Effort:** 25h  
**Owner:** 1 engineer (full-time)  
**Blocker:** None

---

### Week 5-6: Installation Refactor — Phase 2 ⚡
**Status:** Will start after Phase 1  
**Focus:** Platform Installers  

**Must-do:**
- [ ] Linux installer (4h)
  - Improve existing ops/bootstrap/install.sh
  - Support Debian, Ubuntu, Fedora
- [ ] macOS installer (6h)
  - .pkg or Homebrew formula
  - Auto-launch setup website
- [ ] Windows installer (7h)
  - .exe via Inno Setup
  - Check for Docker Desktop
  - Auto-launch setup website

**Target:** `bash install.sh` / `.exe` / `.pkg` all work  

**Effort:** 17h  
**Owner:** 1 engineer + platform expertise (macOS/Windows)  
**Blocker:** None (Phase 1 must be done first)

---

### Week 7-8: Bridge Auto-Setup + Docs ⚡
**Status:** Will start after Phase 2  
**Focus:** Bridge configuration automation  

**Must-do:**
- [ ] Discord auto-setup (3h)
  - Website shows bot invite link
  - Auto-detect token
- [ ] Telegram / WhatsApp / Slack (5h)
  - Per-bridge instruction cards
  - Token verification
- [ ] Getting Started guide (6h)
  - Screenshot walkthrough
  - Troubleshooting common errors
  - For each platform (macOS, Windows, Linux)

**Target:** "Install in < 10 minutes" ✅  

**Effort:** 14h  
**Owner:** 1 engineer + tech writer  
**Blocker:** None

---

### Week 9-10: Monitoring + Backups ⏳
**Status:** Will start in parallel with Bridge Setup (Week 7)  
**Focus:** Ops reliability  

**Must-do:**
- [ ] Monitoring stack (Prometheus + Grafana) (8h)
- [ ] Backup + restore automation (6h)
- [ ] Health checks + dashboards (4h)

**Target:** Operator can monitor + backup without scripts  

**Effort:** 18h  
**Owner:** 1 ops engineer (can run parallel)  
**Blocker:** None (independent from install work)

---

### Week 11-12: Security + Polish ⏳
**Status:** Will start in parallel (Week 9)  
**Focus:** Final security review + UX polish  

**Must-do:**
- [ ] Security policy + disclosure process (6h)
- [ ] Container signing + SBOM (4h)
- [ ] E2E tests for install flow (5h)
- [ ] macOS/Windows app polish (3h)
  - Auto-start, system tray, uninstall
- [ ] Website launch (8h)
  - Get domain
  - Deploy docs site
  - Add download links

**Target:** Production-ready ✅  

**Effort:** 26h  
**Owner:** 1 security engineer + 1 frontend engineer (parallel)  
**Blocker:** Phase 2 installers must be done

---

## Week-by-Week Reality Check

```
Week 1-2:  Code Quality (34h)           — 1 eng, full-time
Week 3-4:  Installation P1 (25h)        — 1 eng, full-time
Week 5-6:  Installation P2 (17h)        — 1 eng, full-time
Week 7-8:  Bridges + Docs (14h)         — 1 eng + 1 writer
Week 9-10: Monitoring (18h)             — 1 ops eng (parallel)
Week 11-12: Security + Polish (26h)     — 2 eng (parallel)
         + Community (6h)               — 1 product eng (parallel)

Total: 284 hours over 12 weeks
Parallel possible: Code Quality (W1-2) → Then Installation (W3-6) → Then Monitoring+Security (W7-12 parallel)
```

---

## What If We Only Have 1 Engineer?

**Timeline:** 18 weeks (instead of 12)

```
Week 1-4:   Code Quality + Installation P1
Week 5-8:   Installation P2 + Bridges
Week 9-12:  Monitoring + Security
Week 13-16: Polish + Community + Website launch
Week 17-18: Final testing + v1.0.0 tag
```

**Better option:** Hire contractors for macOS/Windows installers (6-8 weeks → 2 weeks).

---

## What If We Want to Ship Faster?

**MVP Release (8 weeks, ~200h):**
- ✅ Code Quality (Week 1-2)
- ✅ Installation Website + Linux (Week 3-5)
- ✅ Claude Code OAuth (Week 3)
- ✅ 1-2 bridges (Discord only) (Week 6)
- ✅ Basic docs (Week 7)
- ✅ Health checks (Week 8)
- ❌ macOS/Windows installers (do later)
- ❌ Monitoring dashboard (do later)
- ❌ Auto-update (do later)

**Result:** v0.31.0 (beta) on 2026-07-02  
Then v1.0.0 with macOS/Windows on 2026-07-30

---

## Critical Path (Can't Skip)

These must happen in order:

1. **Code Quality (Week 1-2)** ← must be done before any release
   - Tests green is non-negotiable
2. **Installation Website (Week 3-4)** ← no Installation P2 without this
3. **Linux Installer (Week 5)** ← proof of concept before Mac/Windows
4. **Claude Code OAuth (Week 3)** ← core value prop
5. **One working bridge (Week 6)** ← need E2E demo

Everything else (Monitoring, Security Policy, Community, fancy docs) can happen in parallel or be deferred to v1.0.1.

---

## Success Metrics (Launch Day)

- [ ] Installer works on macOS, Windows, Linux (zero prior setup needed)
- [ ] Website loads without Claude Code installed
- [ ] User can pick engine + get authorized in < 5 minutes
- [ ] User can configure Discord and send first message in < 10 minutes
- [ ] All 183 tests green
- [ ] Security policy published
- [ ] Healthcheck + basic monitoring
- [ ] Getting Started guide + troubleshooting
- [ ] 100+ GitHub stars after launch
- [ ] First 10 issues are feature requests, not bugs

---

## Decision: Which Path?

| Path | Time | Risk | Polish | Cost |
|---|---|---|---|---|
| **A: Minimum Viable** | 8w | Medium | 6/10 | $30k |
| **B: Production** | 12w | Low | 8/10 | $50k |
| **C: Premium** | 16w | Very Low | 10/10 | $80k |

**My recommendation:** **Path B (Production)**
- 12 weeks is reachable with 1-2 engineers
- Covers all must-haves
- Still has 8-10 weeks buffer before June 21
- Good balance of quality + speed

---

## Team Sizing

### Lean Team (1 engineer, 18 weeks)
```
Week 1-4:   Code Quality + Website
Week 5-8:   Installers
Week 9-12:  Bridges + Docs
Week 13-14: Monitoring
Week 15-16: Security + Polish
Week 17-18: Final testing
```
❌ Tight, high risk if blocked

### Recommended Team (2-3 engineers, 12 weeks)
```
Engineer 1: Code Quality (W1-2) → Installation Website (W3-4) → Bridges (W7-8)
Engineer 2: Installation Installers (W3-6 parallel)
Engineer 3: Monitoring + Security (W9-12 parallel)
+ 1 Tech Writer (W7-8)
+ 1 QA (W11-12)
```
✅ Comfortable, low risk

### Maxed-Out Team (5+ engineers, 8 weeks)
```
E1: Code Quality
E2: Website + Backend
E3: macOS/Windows installers
E4: Bridges + Docs
E5: Monitoring + Security
```
⚠️ Too many cooks, coordination overhead

---

## Start Date & Milestones

**Start:** Monday, 2026-05-27  
**Milestones:**
- 2026-06-09: Code Quality ✅ (Week 2)
- 2026-06-16: Installation Website + Daemon ✅ (Week 4)
- 2026-06-23: All Installers ✅ (Week 6)
- 2026-06-30: Bridges + Docs ✅ (Week 8)
- 2026-07-07: Monitoring ✅ (Week 10)
- 2026-07-14: v1.0.0 Tag ✅ (Week 12)
- 2026-07-21: Launch Day

---

## What Should We Do This Week? (Week 0)

1. **Tag v0.30.0**
   - Push all docs/ADRs to main
   - Tag with `git tag v0.30.0`
   - Message: "Planning complete, implementation starts May 27"

2. **Decide on Path** (A/B/C)
   - Quick team meeting
   - Pick one

3. **Recruit Team**
   - Assign Code Quality engineer (start Week 1)
   - Assign Installation engineer (start Week 3)
   - Assign Ops/Security engineer (start Week 9)

4. **Setup Infrastructure**
   - Slack channel: #atelieros-v1-release
   - Weekly standups: Monday 10:00 UTC
   - Track progress in LAUNCH_CHECKLIST.md

---

## Next Steps (After Tag)

```bash
# If Path B (recommended):

# Week 1-2:
# → Assign Code Quality engineer
# → Start: BUG_FIX_EXECUTION_GUIDE.md
# → Target: 183/183 tests green

# Week 3-4:
# → Assign Installation engineer
# → Start: ops/setup-website/
# → Target: Website + daemon running locally

# Week 5-6:
# → Continue Installation engineer
# → Start: Installers (Linux, macOS, Windows)
# → Target: All three installers working

# Week 7-12:
# → Parallel: Bridges, Docs, Monitoring, Security
# → Converge on Week 11 for final testing
# → Tag v1.0.0 on Week 12
# → Launch on 2026-07-21
```

---

*Last updated: 2026-05-21*  
*Status: READY TO START WEEK 1*
