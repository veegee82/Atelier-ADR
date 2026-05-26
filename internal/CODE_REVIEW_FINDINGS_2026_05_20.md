# Code Review Findings — 2026-05-20

## Comprehensive audit of AtelierOS codebase completed

**Review Scope:** Compliance Layer | Bridges | Forge/SkillForge | Docker | Installation | Tests | Documentation

**Status:** 35+ bugs identified and triaged across Compliance, Bridge, Forge, License, and Infrastructure layers.

---

## Fixed This Session (Commit: 5f8f68c, 2cd2c0d)

### CRITICAL

- [x] **Signal Bridge DoS** — No rate limiting on envelope processing → Added `rateAllow()` checks
- [x] **Teams Bridge DoS** — No rate limiting on webhook activity → Added rate limiting gate
- [x] **SkillForge YAML Injection** — Unquoted claim dict values in front-matter → Implemented `_yaml_quote()`

### MEDIUM

- [x] **Teams Memory Leak** — Unbounded conversationRefs Map → LRU eviction (max 1000)
- [x] **Compliance Type Coercion** — Unhandled int() failures in audit_query, gdpr_ropa → Added try/except
- [x] **Deprecated datetime API** — datetime.utcfromtimestamp() → Migrated to timezone.utc variant
- [x] **Bootstrap Repo Name** — Wrong URL (ClaudeClaw vs AtelierOS) → Fixed
- [x] **Test sys.exit()** — pytest collection errors from test_voice_freshness.py → Wrapped in __main__ guard

---

## Outstanding Issues (Next Sprint)

### Bridge Adapters (HIGH PRIORITY)

| ID | Bridge | Severity | Issue | Effort |
|---|---|---|---|---|
| BR-1 | Signal | MEDIUM | No exponential backoff on signal-cli API failures | 1h |
| BR-2 | Signal/Teams | MEDIUM | No audit events for bridge.message_received | 2h |
| BR-3 | Teams | MEDIUM | No chat opt-in system (unlike WhatsApp/Telegram) | 2h |
| BR-4 | Email | MEDIUM | Default rate limit too low (10/h vs 30-60 others) | 30m |
| BR-5 | Discord | LOW | Watchdog ping timeout hard-coded; no network-adaptation | 1h |
| BR-6 | Teams | LOW | No error handling in onTurnError (no retry/audit) | 1h |
| BR-7 | All | LOW | PIN timing attack via string comparison (crypto.timingSafeEqual) | 30m |
| BR-8 | WhatsApp | MEDIUM | spawn() argument escaping audit needed | 30m |

### Forge & SkillForge (HIGH PRIORITY)

| ID | Component | Severity | Issue | Effort |
|---|---|---|---|---|
| FK-1 | SkillForge | MEDIUM | Claim dict not validated by linter (bypass vector) | 1h |
| FK-2 | SkillForge | MEDIUM | Grade carry-over during promotion not transactional | 1h |
| FK-3 | SkillForge | MEDIUM | skill_cleanup.py has no audit trail for skipped items | 30m |
| FK-4 | Forge | MEDIUM | No persona ACL check on meta.secrets at create time | 1h |
| FK-5 | Skill Diff | MEDIUM | No scope-boundary enforcement on diff request | 30m |
| FK-6 | SkillForge | LOW | Plugin-slot mirror writes are best-effort (silent failures) | 1h |
| FK-7 | Forge | LOW | PYTHONPATH manipulation can disable loopback-deny | 1h |
| FK-8 | Linter | LOW | Base64 detection false positives on long hashes | 30m |

### License & Tests (MEDIUM PRIORITY)

| ID | Component | Severity | Issue | Effort |
|---|---|---|---|---|
| LI-1 | license-app | MEDIUM | test_status_active_when_valid_license_installed fails | 1h |
| TE-1 | test suite | MEDIUM | 77 pytest collection errors (fixtures, imports) | 2h |
| TE-2 | conftest | LOW | rs256_keypair scope issues (tmp_path contention) | 30m |

### Compliance & Audit (DONE ✅)

| ID | Component | Severity | Issue | Status |
|---|---|---|---|---|
| CM-1 | audit_query.py | CRITICAL | Type coercion errors on ts field | **FIXED** |
| CM-2 | gdpr_ropa.py | CRITICAL | Type coercion errors on size_b/audio_s | **FIXED** |
| CM-3 | templates.py | MEDIUM | Deprecated datetime API | **FIXED** |

All Compliance tests: **35/35 passing** ✅

---

## Documentation Status

- README.md — accurate, comprehensive ✅
- Architecture diagrams (SVG) — present, need visual audit
- CLAUDE.md — fully documented ✅
- Bridge daemon READMEs — present for all 7 channels ✅
- Compliance layer README — excellent ✅

---

## Next Steps (Priority Order)

### This Week
1. Fix License test failures (LI-1)
2. Resolve pytest collection errors (TE-1)
3. Implement missing rate limits in Email bridge (BR-4)

### Next Week
1. Audit Signal exponential backoff (BR-1)
2. Add bridge audit events (BR-2)
3. Implement claim validation in SkillForge (FK-1)

### Sprint Planning
- Estimate 12 issues × (30m–2h) each = ~20 engineer-hours
- Recommend splitting across 2 sprints (Bridges week 1, Forge week 2)

---

## Test Suite Health

```
✅ Compliance Layer:     35/35 passing
⚠️  License Tests:       1 failing (tier='unknown' instead of 'business')
⚠️  Overall Collection:  77 errors during pytest --collect-only
```

### Priority: Fix License + Collection Errors Before Merge

---

*Audit completed: 2026-05-20 23:40 UTC*
*Reviewer: Claude Code Agent (full codebase pass)*
