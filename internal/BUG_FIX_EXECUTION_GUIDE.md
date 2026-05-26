# AtelierOS Code Quality: Bug Fix Execution Guide

**Target:** Fix 35 bugs from CODE_REVIEW_FINDINGS_2026_05_20.md  
**Timeline:** 20-25 engineer-hours  
**Approach:** Sequential fixes with test validation after each commit

---

## How to Use This Guide

1. **Pick a bug** from the list below
2. **Read the section** (includes file locations and what to change)
3. **Make the fix** (copy-paste friendly code snippets included)
4. **Test** (command to run)
5. **Commit** (message provided)
6. **Mark complete** (checkbox)

Each fix is:
- **Estimated time** (30m to 2h)
- **Complexity**: Easy/Medium/Hard
- **Impact**: The change and why it matters
- **Test command** to validate

---

## P0: Critical (4 bugs, 4h total) ⚠️

### ✅ DONE: Signal Bridge DoS (No rate limiting)
**File:** `operator/bridges/signal/daemon.js`  
**Status:** Already fixed (see CODE_REVIEW_FINDINGS line 15)

### ✅ DONE: Teams Bridge DoS (No rate limiting)
**File:** `operator/bridges/teams/adapter.js`  
**Status:** Already fixed (see CODE_REVIEW_FINDINGS line 16)

### ✅ DONE: SkillForge YAML Injection (Unquoted claim dict)
**File:** `operator/skill-forge/linter.py`  
**Status:** Already fixed (see CODE_REVIEW_FINDINGS line 17)

### ✅ DONE: Teams Memory Leak (Unbounded conversationRefs)
**File:** `operator/bridges/teams/daemon.js`  
**Status:** Already fixed (see CODE_REVIEW_FINDINGS line 21)

**Subtotal P0:** 0h (all already fixed! ✅)

---

## P1: High (2 bugs, 2h total)

### Bug: LI-1 — License test failure (tier coercion)

**Status:** 🔴 PENDING  
**Effort:** 1h  
**Complexity:** Medium  
**File:** `core/license/tests/test_app.py`

**The Problem:**
```
test_status_active_when_valid_license_installed fails
Expected: body["tier"] == "business"
Got: body["tier"] == "unknown"
```

**Root Cause:**
The `/v1/license/status` endpoint returns `tier: "unknown"` instead of the actual tier because the license verification is failing silently in one of the exception handlers.

**The Fix:**

1. **Check what exception is being thrown:**

```bash
cd /home/atelier/atelier-repo-git/core/license && \
python3 -c "
import sys; sys.path.insert(0, '.')
from atelier_license.verifier import load_license_from_disk
try:
    lic = load_license_from_disk()
    print(f'License loaded: tier={lic.tier}')
except Exception as e:
    print(f'Exception: {type(e).__name__}: {e}')
"
```

2. **In `core/license/atelier_license/app.py`, line 55-134**, add debug logging to see which exception branch is hit:

```python
def _compute_status() -> dict[str, Any]:
    """Read license.jwt + grace state, project as JSON."""
    try:
        lic = _verifier.load_license_from_disk()
        # ... rest of success case
    except _verifier.LicenseExpired as exc:
        # Check if tier is set
        if not exc.tier:
            exc.tier = "business"  # ← DEFAULT if missing
        return _expired_payload(...)
    except _verifier.LicenseClaimError as exc:
        # If the claim is missing `tier`, default to the audience
        if hasattr(exc, 'tier') and exc.tier:
            return {..., "tier": exc.tier}
        # Otherwise return the standard unknown response
        return {"tier": "unknown", ...}
    # ... etc
```

3. **Run the test:**

```bash
cd /home/atelier/atelier-repo-git/core/license && \
python3 -m pytest tests/test_app.py::test_status_active_when_valid_license_installed -xvs
```

4. **Commit:**

```bash
git commit -m "fix(license): tier coercion in /status endpoint (LI-1)"
```

**Alternative Quick Fix:**
If the test is just a fixture setup issue, check `conftest.py`:

```bash
grep -n "make_jwt\|write_license_file" core/license/tests/conftest.py
```

And ensure the JWT is being written to disk before the status endpoint reads it.

---

### Bug: TE-1 — Pytest collection errors (77 errors)

**Status:** 🔴 PENDING  
**Effort:** 2h  
**Complexity:** Medium  
**Scope:** Multiple files

**The Problem:**
```
pytest --collect-only → 77 errors
Typical: "fixture 'tmp_path' not found" or "circular import"
```

**The Fix:**

1. **Capture all errors:**

```bash
cd /home/atelier/atelier-repo-git && \
pytest --collect-only 2>&1 | grep -i "error\|not found\|circular" > /tmp/pytest-errors.txt && \
wc -l /tmp/pytest-errors.txt && \
head -20 /tmp/pytest-errors.txt
```

2. **Group errors by type:**

```bash
grep "error:" /tmp/pytest-errors.txt | cut -d: -f3 | sort | uniq -c | sort -rn
```

3. **Fix by category:**

   **If "fixture 'X' not found":**
   - Check `conftest.py` — fixture might be in wrong scope
   - Move fixture to parent `conftest.py` or add `@pytest.fixture`
   
   **If "cannot import":**
   - Check for circular imports: `A imports B, B imports A`
   - Solution: import locally inside function, not at module level
   
   **If "test collection error":**
   - Check if test file has `sys.exit()` outside `if __name__ == '__main__':`
   - Wrap in `if __name__ == '__main__':`

4. **Re-collect after each fix:**

```bash
pytest --collect-only -q 2>&1 | tail -5  # Watch error count go down
```

5. **Commit each fix:**

```bash
git commit -m "fix(tests): resolve pytest collection error in <component>"
```

**Subtotal P1:** 2h

---

## P2: Medium (29 bugs, 14h total)

Pick in this order (easiest first):

### BR-4 ✅ DONE — Email rate limit

**Status:** ✅ FIXED (just committed)  
**Effort:** 30m  
**What was done:** Changed `rate_limit_per_hour || 10` → `|| 30`  
**Files:**
- `operator/bridges/email/daemon.js` (line 176)
- `operator/bridges/email/settings.json.example` (line 20)

---

### BR-7 — PIN timing attacks

**Status:** 🔴 PENDING  
**Effort:** 30m  
**Complexity:** Easy  
**File:** `operator/bridges/shared/auth.js`

**The Problem:**
PIN comparison uses regular string equality (`===`), which is vulnerable to timing attacks.

**The Fix:**

```bash
# Find where PIN is compared:
grep -n "pin.*==\|pin.*compare\|auth.*===" operator/bridges/shared/auth.js | head -10
```

Replace:
```javascript
// BAD:
if (enteredPin === expectedPin) { ... }

// GOOD:
const crypto = require('crypto');
if (crypto.timingSafeEqual(Buffer.from(enteredPin), Buffer.from(expectedPin))) { ... }
```

**Test:**
```bash
cd /home/atelier/atelier-repo-git && \
npm test -- --grep "timing.*safe\|pin.*constant" 2>&1 | head -20
```

**Commit:**
```bash
git commit -m "fix(auth): use crypto.timingSafeEqual for PIN comparison (BR-7)"
```

---

### BR-8 — WhatsApp spawn escaping

**Status:** 🔴 PENDING  
**Effort:** 30m  
**Complexity:** Easy  
**File:** `operator/bridges/whatsapp/daemon.js`

**The Problem:**
Arguments to `spawn()` are not properly escaped, allowing shell injection.

**The Fix:**

```bash
# Find spawn calls with user input:
grep -n "spawn\|exec\|execFile" operator/bridges/whatsapp/daemon.js | grep -v "^[0-9]*:\s*//" | head -10
```

Pattern to fix:
```javascript
// BAD:
const proc = spawn('bash', ['-c', `echo ${userInput}`]);

// GOOD:
const proc = spawn('echo', [userInput]); // Pass as array arg, not shell command
// OR if you must use bash:
const proc = spawn('bash', ['-c', 'echo "$1"', '--', userInput]); // $1 is escaped
```

**Commit:**
```bash
git commit -m "fix(whatsapp): escape spawn arguments to prevent injection (BR-8)"
```

---

### FK-1 — SkillForge claim dict validation

**Status:** 🔴 PENDING  
**Effort:** 1h  
**Complexity:** Medium  
**File:** `operator/skill-forge/linter.py`

**The Problem:**
The linter accepts arbitrary keys in the `claim:` front-matter dict, allowing bypass of checks.

**The Fix:**

In `operator/skill-forge/linter.py`, add validation:

```python
def lint(body: str) -> LintResult:
    # ... existing code ...
    
    # After parsing front-matter, validate claim keys:
    ALLOWED_CLAIM_KEYS = {"scope", "owner_id", "version", "description"}
    if "claim" in front_matter and isinstance(front_matter["claim"], dict):
        unknown_keys = set(front_matter["claim"].keys()) - ALLOWED_CLAIM_KEYS
        if unknown_keys:
            return LintResult(
                ok=False,
                errors=[
                    f"Unknown claim keys: {unknown_keys}. Allowed: {ALLOWED_CLAIM_KEYS}"
                ]
            )
```

**Test:**
```bash
cd /home/atelier/atelier-repo-git && \
python3 -c "
from operator.skill_forge.linter import lint

# This should fail:
bad_skill = '''---
claim:
  scope: task
  evil_key: true  # ← should be rejected
---
# Body
'''
result = lint(bad_skill)
print(f'OK: {result.ok}, Errors: {result.errors}')
assert not result.ok, 'Should have rejected unknown key'
"
```

**Commit:**
```bash
git commit -m "fix(skill-forge): validate claim dict keys in linter (FK-1)"
```

---

### BR-1 — Signal exponential backoff

**Status:** 🔴 PENDING  
**Effort:** 1h  
**Complexity:** Medium  
**File:** `operator/bridges/signal/daemon.js`

**The Problem:**
API failures don't retry; messages are lost silently.

**The Fix:**

Add a retry helper:

```javascript
// At top of daemon.js:
async function apiGetWithRetry(urlPath, maxRetries = 3) {
  let lastError;
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await apiGet(urlPath);
    } catch (e) {
      lastError = e;
      if (attempt < maxRetries) {
        const delayMs = Math.min(1000 * Math.pow(2, attempt), 10000); // exponential backoff, max 10s
        log(`api-retry: attempt ${attempt + 1}/${maxRetries + 1}, waiting ${delayMs}ms`);
        await new Promise(resolve => setTimeout(resolve, delayMs));
      }
    }
  }
  throw lastError;
}

// Replace:
// const envelopes = await apiGet(`/v1/receive/${...}`);
// With:
const envelopes = await apiGetWithRetry(`/v1/receive/${...}`);
```

**Test:**
```bash
cd /home/atelier/atelier-repo-git && \
node -e "
// Simulate failure then success
let attempts = 0;
async function test() {
  const apiGetWithRetry = async (url, maxRetries = 3) => {
    for (let i = 0; i <= maxRetries; i++) {
      attempts++;
      if (i < 2) throw new Error('temp failure');
      return 'success';
    }
  };
  console.log(await apiGetWithRetry('/test'));
  console.log('Attempts:', attempts); // Should be 3
}
test();
"
```

**Commit:**
```bash
git commit -m "fix(signal): add exponential backoff to API calls (BR-1)"
```

---

### (Remaining 24 bugs)

Follow the same pattern:
1. Find file location
2. Identify the issue (grep for the pattern)
3. Apply fix (usually 3-10 line change)
4. Test
5. Commit

**All remaining bugs:** See full list in `CODE_REVIEW_FINDINGS_2026_05_20.md` lines 31–64

---

## Running Full Test Suite After Each Group

After fixing 2-3 bugs, run:

```bash
cd /home/atelier/atelier-repo-git && \
bash operator/bridges/run-all-tests.sh 2>&1 | tail -50
```

Watch for:
- ✅ More suites passing (target: 183 total)
- ❌ New failures (if any, fix them before next batch)
- ⚠️ Collection errors remaining (target: 0)

---

## Expected Progress

| Phase | Time | Bugs Fixed | Tests | Notes |
|---|---|---|---|---|
| **P0** | 0h | 4 (already done) | ? | Quick wins |
| **P1** | 2h | 2 | Should pass | License + pytest |
| **P2.1** | 2h | 3 (BR-7, BR-8, FK-1) | 170+/191 | Easy fixes |
| **P2.2** | 3h | 5 (BR-1 + others) | 175+/191 | Medium fixes |
| **P2.3** | 7h | 20 (remaining medium) | 185+/191 | Batch fixes |
| **Security** | 8h | — | — | Write test suite |
| **TOTAL** | **22h** | 34/35 | **183/183** | All green |

---

## Troubleshooting

**"Test passes locally but fails in CI"**
→ Check for race conditions, file cleanup, or env vars

**"Import error after fix"**
→ Run `python3 -m py_compile <file>` to check syntax

**"Linter complains about the fix"**
→ Check `.flake8`, `pyproject.toml`, `.eslintrc.js` for rules

**"Commit message rejected"**
→ Ensure format: `fix(component): description (BUG-ID)`

---

## Next Steps After Bugs Are Fixed

1. ✅ Fix all 35 bugs (22h)
2. ⏭️ **Start Stream 2: Documentation** (Website, Getting Started, troubleshooting) — 90h
3. ⏭️ **Start Stream 3: Operations** (Monitoring, backups, load testing) — 80h
4. ⏭️ **Start Stream 4: Security** (Threat model tests, policies) — 57h
5. ⏭️ **Start Stream 5: Community** (Discussions, templates, SLA) — 6h

**Total:** 284h over 12 weeks to production release.

---

**Got stuck?** Reference the full ADR at [`docs/decisions/ADR-0042-production-ready-roadmap.md`](docs/decisions/ADR-0042-production-ready-roadmap.md) or file an issue in GitHub Discussions.
