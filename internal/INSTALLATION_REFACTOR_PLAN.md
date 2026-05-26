# Installation Refactor — Website-First Setup (v0.30.0 → v1.0.0)

**Goal:** From CLI-first to website-first. Website loads WITHOUT Claude Code. User can configure engines and auth directly in UI.

**Timeline:** 4-5 weeks (60-80 engineer-hours)  
**Platforms:** macOS · Windows · Linux · Docker

---

## Current State

| Component | Status | Problem |
|---|---|---|
| `setup.sh` | ✅ Works | Requires Claude Code installed first |
| `ops/bootstrap/install.sh` | ✅ Works | Linux/Debian only |
| Website | ❌ Missing | No discovery, no config UI |
| macOS installer | ❌ Missing | No native support |
| Windows installer | ❌ Missing | No native support |
| OAuth/Auth UI | ❌ Missing | Keys entered via env vars only |
| Engine selection UI | ❌ Missing | No way to switch engines in UI |

---

## Architecture Vision

```
┌─────────────────────────────────────────────────────────┐
│                    INSTALLER.EXE / .PKG / .SH           │
│     (downloads + launches local web server)             │
└──────────────────────┬──────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│              localhost:8080 (Setup Website)             │
│  ┌───────────────────────────────────────────────────┐  │
│  │ 1. Welcome (no CC needed!)                        │  │
│  │ 2. System Check (Docker? Node? Python?)          │  │
│  │ 3. Pick Engine (Claude Code / OpenAI / OpenCode) │  │
│  │ 4. Enter Keys (API keys, bridge tokens)          │  │
│  │ 5. Authorize Engine (OAuth → Anthropic/OpenAI)   │  │
│  │ 6. Pick Bridges (Discord, Telegram, etc.)        │  │
│  │ 7. Configure Each Bridge (tokens, whitelist)     │  │
│  │ 8. Start Stack (Docker Compose up)               │  │
│  │ 9. Status Dashboard (healthcheck, logs)          │  │
│  └───────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────┐
│            Backend (Setup Daemon, 5-10 min)             │
│  - System detection (OS, Docker, Node, Python)         │
│  - Config generation (.env, docker-compose.yml)        │
│  - Container lifecycle (pull, start, health-check)     │
│  - Secrets management (secure key storage)             │
└─────────────────────────────────────────────────────────┘
```

---

## Phase 1: Website Foundation (20h)

### 1.1: Setup Website UI (12h)
- **Tech:** HTML + vanilla JS (no build step — embed in binary)
- **Features:**
  - Welcome screen (branding, intro video)
  - System requirements check (Docker installed? Python? Node?)
  - Engine picker (Claude Code / OpenAI / Ollama / OpenRouter)
  - API key input forms (secure, never logged)
  - OAuth redirect flow for each engine
  - Bridge selector (checkboxes for 7 bridges)
  - Per-bridge config (token, whitelist, rate limits)
  - Status dashboard (healthcheck, container logs)
  - Settings panel (change theme, ports, advanced options)

**Files to create:**
```
ops/setup-website/
├── index.html          (main page, router)
├── css/
│   └── setup.css       (tailwind-like, lightweight)
├── js/
│   ├── app.js          (router, state management)
│   ├── api-client.js   (fetch to localhost:8089)
│   ├── oauth.js        (Claude Code, OpenAI flow)
│   └── ui-components.js (forms, status, alerts)
└── assets/
    ├── logo.svg
    └── spinner.svg
```

**Effort breakdown:**
- HTML structure + CSS: 4h
- UI state management (Vue-lite): 3h
- OAuth flow + API integration: 3h
- Error handling + UX polish: 2h

### 1.2: Setup Backend Daemon (8h)
- **Tech:** Node.js Express (same stack as bridges)
- **Port:** 8089 (internal, not exposed)
- **Features:**
  - `/api/system/check` — OS, Docker, deps
  - `/api/engines/list` — available engines
  - `/api/config/validate` — validate .env before apply
  - `/api/config/apply` — write files, restart containers
  - `/api/oauth/callback` — handle OAuth redirects
  - `/api/health/status` — container status
  - `/api/logs/tail` — live logs for debugging

**Files to create:**
```
ops/setup-daemon/
├── server.js           (Express app)
├── routes/
│   ├── system.js       (OS detect, Docker check)
│   ├── engines.js      (engine list, OAuth)
│   ├── config.js       (validate, apply)
│   └── health.js       (status, logs)
├── lib/
│   ├── docker-client.js (start/stop containers)
│   ├── config-writer.js (generate .env)
│   ├── system-check.js (dependencies)
│   └── oauth-handler.js (Claude, OpenAI flows)
└── package.json
```

**Effort breakdown:**
- Express setup + routes: 3h
- Docker lifecycle + health checks: 2h
- Config generation + validation: 2h
- OAuth callbacks: 1h

---

## Phase 2: Installers (20h)

### 2.1: macOS (6h)
- **Tool:** Homebrew formula or notarized .pkg
- **Flow:**
  1. `curl … | bash` → downloads binary
  2. Binary contains setup website + daemon
  3. Opens `http://localhost:8080` in browser
  4. User goes through setup wizard
  5. Binary starts Docker containers in background
  6. Status dashboard shows when ready

**Deliverable:**
```bash
# User runs:
bash <(curl -fsSL https://raw.githubusercontent.com/…/install-macos.sh)

# OR:
brew install atelieros/tap/atelieros
atelieros setup
```

### 2.2: Windows (7h)
- **Tool:** Inno Setup or WiX (generates .exe)
- **Prerequisites installer:** Docker Desktop, WSL2 (if not present)
- **Flow:** Same as macOS but with Windows-specific paths
- **Code signing:** Optional (for Microsoft Defender warning)

**Deliverable:**
```
AtelierOS-Setup-1.0.0.exe
→ checks for Docker Desktop
→ offers to install if missing
→ launches setup website
→ configures paths for Windows
```

### 2.3: Linux (4h)
- **Keep existing:** `ops/bootstrap/install.sh` (Ubuntu/Debian)
- **Add:** Generic systemd unit + launchers
- **Improve:** Support for Fedora, RHEL, Alpine

**Deliverable:**
```bash
curl -fsSL https://get.atelieros.io | sudo bash
# Detects distro, picks right package manager, runs setup
```

### 2.4: Docker image (3h)
- **New:** `docker run atelieros:setup` → localhost:8080
- **Use:** For testing, CI/CD, serverless deployment

---

## Phase 3: Engine Integration (15h)

### 3.1: Claude Code OAuth (5h)
**Flow:**
1. User clicks "Activate Claude Code"
2. Opens `https://claude.ai/auth?return_to=localhost:8080/oauth/callback`
3. User logs in + approves
4. Browser redirects to localhost with code
5. Daemon exchanges code for API key
6. Key stored in `.env` + vault

**Files:**
```
ops/setup-daemon/lib/
├── oauth-providers/
│   ├── claude-code.js   (Claude auth flow)
│   ├── openai.js        (OpenAI auth flow)
│   └── ollama.js        (Ollama local setup)
└── vault.js             (encrypt keys)
```

### 3.2: OpenAI / OpenRouter (5h)
**Similar OAuth flow but for external APIs.**

### 3.3: Local engines (Ollama, OpenCode) (5h)
**No OAuth — just detect local server, verify endpoint reachable.**

---

## Phase 4: Bridge Auto-Setup (10h)

### 4.1: Discord Bot Setup (3h)
- Website shows: "Copy this URL, paste in Discord server settings"
- Auto-detects bot token from paste
- Configures permissions + webhook

### 4.2: Telegram / WhatsApp / Slack (5h)
- Per-bridge instruction cards
- Auto-verify each token works
- Generate QR codes where needed (WhatsApp)

### 4.3: Test Each Bridge (2h)
- Send test message from setup UI
- Confirm round-trip works

---

## Phase 5: macOS / Windows Polish (15h)

### 5.1: Native App Shell (5h)
- **macOS:** Wrap binary in .app bundle with icon
- **Windows:** Add system tray icon (minimize to tray)
- **Both:** Auto-start on login checkbox

### 5.2: Auto-Update (5h)
- Check for new versions weekly
- Download + restart seamlessly
- Changelog shown on upgrade

### 5.3: Uninstall (2h)
- Clean removal of containers, volumes, config
- Optional: keep backups of audit chain

### 5.4: UX Polish (3h)
- Keyboard shortcuts (Cmd+Q to quit, etc.)
- Dark mode
- Progress indicators
- Error recovery suggestions

---

## Phase 6: Testing & Docs (10h)

### 6.1: E2E Test Suite (5h)
```bash
# Test each platform:
npm run test:setup:macos
npm run test:setup:windows
npm run test:setup:linux
npm run test:setup:docker

# Check each engine OAuth works
npm run test:oauth:claude-code
npm run test:oauth:openai
npm run test:oauth:ollama

# Check each bridge auto-setup
npm run test:bridge:discord
npm run test:bridge:telegram
```

### 6.2: Documentation (3h)
- Screenshot walkthrough (Getting Started)
- Troubleshooting (common errors)
- Advanced config (for CLI users)

### 6.3: QA Checklist (2h)
- ✅ First-time install on clean macOS
- ✅ First-time install on clean Windows
- ✅ First-time install on clean Linux
- ✅ Upgrade from v0.x
- ✅ OAuth flows all work
- ✅ Bridge config auto-detects tokens

---

## Implementation Order (Priority)

### Week 1-2: Core Website + Backend
1. **1.1** Website UI structure + CSS (4h)
2. **1.2** Setup daemon skeleton (3h)
3. **3.1** Claude Code OAuth (5h)
4. **1.1** OAuth integration into website (3h)

### Week 2-3: Installers
5. **2.3** Linux installer (.sh) (4h)
6. **2.1** macOS installer (6h)
7. **2.2** Windows installer (7h)

### Week 3-4: Bridges + Polish
8. **4.1** Discord auto-setup (3h)
9. **4.2** Telegram / WhatsApp / Slack (5h)
10. **5.1** macOS / Windows app shell (5h)

### Week 4-5: Testing + Docs
11. **6.1** E2E tests (5h)
12. **6.2** Documentation (3h)

---

## File Structure (Final)

```
AtelierOS/
├── ops/
│   ├── setup-website/          [NEW]
│   │   ├── index.html
│   │   ├── css/setup.css
│   │   ├── js/{app,oauth,api}.js
│   │   └── assets/{logo,spinner}.svg
│   │
│   ├── setup-daemon/           [NEW]
│   │   ├── server.js
│   │   ├── routes/
│   │   ├── lib/
│   │   └── package.json
│   │
│   ├── installers/             [NEW]
│   │   ├── macos/
│   │   │   └── build.sh → produces AtelierOS.pkg
│   │   ├── windows/
│   │   │   └── setup.iss → Inno Setup config
│   │   └── linux/
│   │       └── install.sh (improved from bootstrap/)
│   │
│   ├── Dockerfile             [MODIFY]
│   │   (add setup daemon sidecar)
│   │
│   └── bootstrap/
│       └── install.sh (kept for backwards compat)
```

---

## Success Criteria

- [ ] User can install on macOS, Windows, Linux with one command
- [ ] Website loads WITHOUT Claude Code installed
- [ ] User can pick engine + authorize in UI (no env vars)
- [ ] Bridge auto-setup works for all 7 bridges
- [ ] First message works end-to-end in <10 minutes
- [ ] Uninstall cleans everything
- [ ] Works offline for local engines (Ollama)
- [ ] E2E tests all pass
- [ ] Zero setup.sh / .env editing needed

---

## Effort Summary

| Phase | Hours | Status |
|---|---|---|
| 1. Website + Backend | 20h | 🔴 NOT STARTED |
| 2. Installers (all 3) | 20h | 🔴 NOT STARTED |
| 3. Engine OAuth | 15h | 🔴 NOT STARTED |
| 4. Bridge Auto-Setup | 10h | 🔴 NOT STARTED |
| 5. macOS/Windows Polish | 15h | 🔴 NOT STARTED |
| 6. Testing + Docs | 10h | 🔴 NOT STARTED |
| **TOTAL** | **90h** | 🔴 **0%** |

---

## Decision Points

**Should we keep setup.sh?**
- A: Replace entirely (cleaner)
- B: Keep for CLI users (backwards compat)
- **Recommendation:** B — add deprecation notice

**Should installers auto-start on first run?**
- A: Yes (one-command ready)
- B: No (safer)
- **Recommendation:** A — but with "disable auto-start" checkbox

**Should we include Docker or assume it's installed?**
- A: Include Docker installer (easier)
- B: Assume pre-installed (simpler)
- **Recommendation:** A — offer to install if missing

**How to handle Claude Code license validation?**
- A: Validate in setup (early error)
- B: Validate on first message (defer)
- **Recommendation:** A — fail fast in setup

---

## Next Steps

1. **Pick start date** (Week of 2026-05-27?)
2. **Assign PM** (Product Lead to drive UX)
3. **Assign Dev** (Full-stack, Node + macOS/Windows familiarity)
4. **Create design** (Figma mockup of setup flow)
5. **Start Phase 1** (Website UI + Backend)

---

**Questions?**
- Why 90h? → Website (20h) + 3 installers (20h) + OAuth (15h) + bridges (10h) + polish (15h) + testing (10h)
- Can this go faster? → Yes, if we skip nice-to-haves (auto-update, dark mode, system tray)
- Can this be done with template? → Partially — use Bootstrap/Tailwind for website, but OAuth + installer logic is custom

---

*Last updated: 2026-05-21*
