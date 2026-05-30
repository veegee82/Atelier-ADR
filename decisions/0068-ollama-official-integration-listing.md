# ADR-0068 — Ollama Official Integration: AtelierOS Listed as an Ollama Assistant

**Status:** Proposed — 2026-05-30
**Date:** 2026-05-30
**Authors:** Claude Code (maintainer session)
**Type:** Ecosystem — Community listing + upstream contribution + launch-protocol implementation
**Depends on:** ADR-0066 (HermesEngine / Ollama HTTP integration), ADR-0067 (HermesEngine production parity), L35 (Egress Lockdown)
**Scope:** `ollama/ollama` upstream repo · new `atelier-launcher` Python package · `ops/` deployment assets

---

## Context

### The Ollama integrations directory

`docs.ollama.com/integrations` is the canonical discovery surface for tools that
work with Ollama. The directory is structured in tiers inside the `ollama/ollama`
GitHub repository:

```
docs/integrations/
  index.mdx           ← overview with category lists
  <tool>.mdx          ← one page per integration

docs/docs.json        ← sidebar navigation (controls "Assistants" group)

cmd/launch/
  <tool>.go           ← Go implementation of `ollama launch <tool>`
  <tool>_test.go      ← required test file

app/ui/app/public/launch-icons/
  <tool>.svg          ← icon shown in Ollama desktop app
```

The **Assistants** group currently contains two entries:
- **OpenClaw** — messaging-bridge AI assistant (Discord, WhatsApp, Telegram, etc.)
- **Hermes Agent** — self-improving agentic assistant by NousResearch

Both entries use `ollama launch <name>` as the quick-start command. This is not
a documentation convention — it is backed by Go code in `cmd/launch/` that
handles install detection, configuration, and subprocess management.

### AtelierOS relevance

AtelierOS already has **deep Ollama integration** via HermesEngine (ADR-0066/0067):
- `ATELIER_OLLAMA_BASE_URL` wires to any Ollama instance
- `ATELIER_HERMES_MODEL` selects the model tag
- HermesEngine passes L34/L35 compliance gates (`locality=local, egress=none`)
- `/engine hermes` switches the active engine per-chat

AtelierOS satisfies the same category description as the existing assistants:
*"AI assistants that help with everyday tasks"* — it bridges Discord, Telegram,
Slack, WhatsApp, and Email to AI backends through a compliant, audited gateway.

### The gap

Despite functional Ollama integration, AtelierOS is **not discoverable** from
the Ollama ecosystem. Operators searching for an Ollama-compatible assistant
find OpenClaw and Hermes but not AtelierOS — even though AtelierOS has stronger
compliance properties (GDPR / EU AI Act) and a more mature audit layer.

### Current install story

AtelierOS is distributed exclusively as a Docker image:
```
ghcr.io/veegee82/atelieros:latest
```
This creates a mismatch with the Ollama launch protocol, which is built around
CLI tools installed via `npm`, `pip`, or a `curl | bash` installer script.

---

## Decision

Implement Ollama integration listing in **four milestones**, separating the
doc-level presence (M1) from the full `ollama launch` protocol (M2–M4).

### Variant analysis

**Variant A — Docs-only PR (no `ollama launch`)**
Submit `docs/integrations/atelieros.mdx` with manual setup instructions and
no `Quick start` section. Low friction, no Go code required.

*Rejected as the sole approach.* The Ollama team shows strong preference for
launch-enabled integrations in the Assistants tier. A docs-only entry is
likely to be deprioritized or declined for the Assistants group specifically.

**Variant B — Docker-native launcher (novel pattern)**
The `cmd/launch/atelieros.go` checks for Docker, pulls the image, and starts
`docker run ghcr.io/veegee82/atelieros:latest`. Would be the first
Docker-based `ollama launch` integration.

*Retained as a possible M3 path.* Requires Ollama maintainer buy-in. Must be
raised in a GitHub issue before the PR, per `CONTRIBUTING.md`. Risk: the team
may reject Docker-based launchers as too heavyweight or a maintenance burden.

**Variant C — Thin `atelier-launcher` PyPI package (chosen)**
Create a lightweight `atelieros` package on PyPI that:
- Requires only Python 3.10+ (no heavy dependencies)
- Provides an `atelier` CLI entry point
- Wraps Docker (calls `docker run ...`) or runs natively if the full repo is present
- Exposes the same commands the Go launcher calls: `atelier setup` / `atelier gateway start`

This follows the Hermes pattern (curl installer) and the OpenClaw pattern (npm
package), and makes the Go code in `ollama/ollama` straightforward. It also
decouples the launcher from the container distribution: operators who run
AtelierOS natively get the same `atelier` command as Docker-based operators.

*Chosen for M2.* Keeps the Ollama-side Go code simple; puts the
Docker/native decision in the Python package where it belongs.

---

## Milestones

### M1 — GitHub Issue + Docs PR (no `ollama launch`)

**Goal:** Establish presence in the Ollama ecosystem and get maintainer feedback.

**Changes to `ollama/ollama`:**

1. **Open GitHub issue** — full text below, to be posted at `github.com/ollama/ollama/issues/new`

---

#### GitHub Issue: `integration: add AtelierOS as an Ollama Assistant`

> **Title:** `integration: add AtelierOS as an Ollama Assistant`
>
> **Labels to request:** `enhancement`, `integrations`

```markdown
## What is AtelierOS?

AtelierOS is an open-source AI assistant gateway that bridges messaging platforms
(Discord, Telegram, Slack, WhatsApp, Email) to local and cloud AI backends through
a single compliant, audited layer. It is MIT/Apache-2.0 licensed and designed for
self-hosted deployment.

Key properties:
- Multi-bridge: one running instance covers all connected channels simultaneously
- GDPR + EU AI Act compliant out of the box: hash-chained audit log, per-user consent
  gate (deny-by-default), one-time AI-disclosure card, data-residency routing
- Distributed as a Docker image: `ghcr.io/veegee82/atelieros:latest`
- Repo: https://github.com/veegee82/AtelierOS

## Existing Ollama integration

AtelierOS already ships a first-class Ollama integration called HermesEngine:

- Speaks Ollama's `/api/chat` streaming API
- Configurable via `ATELIER_OLLAMA_BASE_URL` (defaults to `http://localhost:11434`)
- Model selection via `ATELIER_HERMES_MODEL` (aliases: `hermes-fast` / `hermes-balanced`
  / `hermes-capable` / `hermes-large`, or any direct Ollama tag)
- Passes the system's data-classification and egress-lockdown compliance gates
  (`locality=local, network_egress=none`) — the only AtelierOS engine that qualifies
  for CONFIDENTIAL tasks without a cloud-egress exception
- Users switch to it per-chat with `/engine hermes`

## What we're proposing

We'd like AtelierOS listed under **Integrations → Assistants** on docs.ollama.com,
alongside OpenClaw and Hermes Agent.

We've looked at the existing `cmd/launch/` implementations (hermes.go, openclaw.go)
and understand the full scope of work required:

| File | Status |
|---|---|
| `docs/integrations/atelieros.mdx` | Draft ready (see below) |
| `docs/integrations/index.mdx` | +1 line under Assistants |
| `docs/docs.json` | +1 entry in Assistants group |
| `app/ui/app/public/launch-icons/atelieros.svg` | Logo ready to export |
| `cmd/launch/atelieros.go` | Ready to write once install path confirmed |
| `cmd/launch/atelieros_test.go` | Ready to write alongside Go code |

## Install path question (main thing we need your input on)

AtelierOS is currently Docker-only. To support `ollama launch atelieros` in the
same style as Hermes (curl installer) and OpenClaw (npm), we plan to publish a
thin `atelieros` package on PyPI:

```bash
pip install atelieros   # installs the `atelier` CLI
atelier setup           # wizard: detect Ollama, pull Docker image, configure bridge
atelier gateway start   # starts the daemon
```

Internally, `atelier setup` and `atelier gateway start` call `docker run
ghcr.io/veegee82/atelieros:latest` with the appropriate env vars
(`ATELIER_OLLAMA_BASE_URL`, `ATELIER_HERMES_MODEL`, bridge toggles). On hosts
where the full repo is present natively, it falls back to calling `bridge.sh`
directly (no Docker required).

The Go code in `cmd/launch/atelieros.go` would then be straightforward:
```
binary() → look for `atelier` in PATH
Configure(model) → atelier config set ollama-url <url> --model <tag>
Run() → atelier gateway start
```

**Two questions before we open a PR:**

1. **Is the Assistants tier the right fit?** AtelierOS is closer to OpenClaw
   (messaging bridge + gateway daemon) than to Hermes (autonomous agent). We're
   happy to be placed in a different category if that fits better.

2. **Is a `pip install` → Docker-wrapping launcher acceptable for `cmd/launch/`?**
   This would be the first launcher in that group where the CLI itself wraps a
   Docker container. If there are concerns about Docker as a hard dependency, we
   can discuss a fully-native Python runner as an alternative.

## Draft documentation page

<details>
<summary>docs/integrations/atelieros.mdx (draft)</summary>

```mdx
---
title: AtelierOS
---

AtelierOS is a privacy-first AI assistant gateway that bridges messaging platforms
(Discord, Telegram, Slack, WhatsApp, and Email) to local and cloud AI models
through a GDPR-compliant, EU AI Act-ready layer.

## Quick start

\```bash
ollama launch atelieros
\```

Ollama handles everything automatically:

1. **Install** — If AtelierOS isn't installed, Ollama prompts to install it via pip
2. **Security** — On first launch, a disclosure card explains AI-generated responses (EU AI Act Art. 50)
3. **Model** — Pick a model from the selector (local or cloud)
4. **Onboarding** — Ollama configures the Ollama provider and sets your model as primary
5. **Gateway** — Starts in the background and guides you through connecting a channel

<Note>AtelierOS requires Docker. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
or Docker Engine before running `ollama launch atelieros`.</Note>

## Connect messaging apps

```bash
atelier gateway setup
```

Link Discord, Telegram, Slack, WhatsApp, or Email to chat with your local models from anywhere.

## Recommended models

**Cloud models:**

- `kimi-k2.5:cloud` — Multimodal reasoning with subagents
- `qwen3.5:cloud` — Reasoning, coding, and agentic tool use with vision
- `glm-5.1:cloud` — Reasoning and code generation

**Local models:**

- `qwen3:8b` — Fast reasoning and instruction-following locally (~5 GB VRAM)
- `gemma4:27b` — Strong general assistant locally (~18 GB VRAM)

More models at [ollama.com/search](https://ollama.com/search).

## EU compliance mode

AtelierOS ships with a built-in GDPR + EU AI Act compliance layer. To run in
strict local-only mode with no cloud egress:

```bash
atelier setup --profile eu-production
```

This configures the Ollama-only tenant preset, disables all cloud engine routes,
and enables the hash-chained tamper-evident audit log.

## Reconfigure

Re-run the full setup wizard at any time:

```bash
atelier setup
```

## Manual setup (without `ollama launch`)

```bash
pip install atelieros
atelier setup
```

During setup, select **Ollama** as the provider and confirm the endpoint
(default: `http://127.0.0.1:11434`).
\```
</details>

Thanks for your time — happy to adjust anything based on your feedback before writing the PR.
```

---

**M1 done-signal:** Issue posted; maintainer response received; PR scope confirmed.

2. **`docs/integrations/atelieros.mdx`** — new file:

```mdx
---
title: AtelierOS
---

AtelierOS is a privacy-first AI assistant gateway that bridges messaging platforms
(Discord, Telegram, Slack, WhatsApp, and Email) to local and cloud AI models through
a GDPR-compliant, EU AI Act-ready layer.

## Quick start

```bash
ollama launch atelieros
```

Ollama handles everything automatically:

1. **Install** — If AtelierOS isn't installed, Ollama prompts to install it via pip
2. **Security** — On first launch, a disclosure card explains AI-generated responses
3. **Model** — Pick a model from the selector (local or cloud)
4. **Onboarding** — Ollama configures the Ollama provider and sets your model as primary
5. **Gateway** — Starts in the background and guides you through connecting a channel

<Note>AtelierOS requires Docker for the gateway daemon. Install Docker Desktop or Docker Engine before running `ollama launch atelieros`.</Note>

## Connect messaging apps

```bash
atelier gateway setup
```

Link Discord, Telegram, Slack, WhatsApp, or Email to chat with your local models from anywhere.

## Recommended models

**Cloud models:**

- `kimi-k2.5:cloud` — Multimodal reasoning with subagents
- `qwen3.5:cloud` — Reasoning, coding, and agentic tool use with vision
- `glm-5.1:cloud` — Reasoning and code generation

**Local models:**

- `qwen3:8b` — Fast reasoning and code locally (~6 GB VRAM)
- `gemma4:27b` — Strong general assistant (~18 GB VRAM)

More models at [ollama.com/search](https://ollama.com/search).

## EU compliance mode

AtelierOS ships with a built-in EU AI Act + GDPR compliance layer. To run in
strict local-only mode (no cloud egress):

```bash
atelier setup --profile eu-production
```

This configures the Ollama-only tenant preset, disables all cloud engine routes,
and enables the hash-chained audit log.

## Reconfigure

```bash
atelier setup
```

## Manual setup (without `ollama launch`)

If you prefer to configure AtelierOS directly:

```bash
pip install atelieros
atelier setup
```

During setup, choose **Ollama** as the provider and set the endpoint to your
Ollama host (default: `http://127.0.0.1:11434`).
```

3. **`docs/integrations/index.mdx`** — add under Assistants:
```markdown
- [AtelierOS](/integrations/atelieros)
```

4. **`docs/docs.json`** — add to Assistants group:
```json
"/integrations/atelieros"
```

**M1 done-signal:** PR submitted to `ollama/ollama`; maintainer feedback received.

---

### M2 — `atelier-launcher` PyPI Package

**Goal:** Provide the `atelier` CLI that the Ollama Go launcher will call.

**New repository / package:** `atelieros/atelier-launcher` (standalone Python package,
separate from the main AtelierOS repo to keep the install tiny).

**Package structure:**
```
atelier-launcher/
  pyproject.toml        ← package: atelieros, entrypoint: atelier
  atelier/
    __main__.py         ← `python -m atelier`
    cli.py              ← argparse: setup | gateway | config
    docker_backend.py   ← wraps `docker run ghcr.io/veegee82/atelieros:latest`
    native_backend.py   ← wraps local `bridge.sh` if repo is present
    backend.py          ← auto-detects which backend to use
    config.py           ← reads/writes ~/.config/atelier-launcher/config.json
    ollama.py           ← ATELIER_OLLAMA_BASE_URL + model detection
```

**Key CLI commands (consumed by the Go launcher):**

| Command | Description |
|---|---|
| `atelier setup` | Interactive wizard: detect Ollama, pull image, configure bridge |
| `atelier setup --yes --model <tag>` | Non-interactive setup (used by `ollama launch --yes`) |
| `atelier gateway start` | Start the configured bridge daemon |
| `atelier gateway stop` | Stop the bridge daemon |
| `atelier gateway setup` | Interactive channel-connection wizard |
| `atelier config set ollama-url <url>` | Set Ollama endpoint |
| `atelier config set model <tag>` | Set active model |

**Docker invocation inside `docker_backend.py`:**
```python
DOCKER_RUN = [
    "docker", "run", "--rm",
    "--name", "atelieros",
    "-e", f"ATELIER_OLLAMA_BASE_URL={ollama_url}",
    "-e", f"ATELIER_HERMES_MODEL={model}",
    "-e", f"ATELIER_BRIDGE_{bridge.upper()}=true",
    "-v", f"{config_dir}:/home/atelier",
    "--network", "host",        # access localhost:11434
    "ghcr.io/veegee82/atelieros:latest",
]
```

**Ollama URL for Docker:**
When Ollama runs on the host, the container must reach it via the host network.
On Linux: `--network host` + `http://127.0.0.1:11434`.
On macOS/Windows: `http://host.docker.internal:11434`.
`ollama.py` auto-detects the platform and adjusts.

**M2 done-signal:** `pip install atelieros && atelier setup && atelier gateway start` works
end-to-end on Linux and macOS with a local Ollama instance.

---

### M3 — Go Launcher in `ollama/ollama`

**Goal:** Implement `ollama launch atelieros` in Go.

**New files:**

**`cmd/launch/atelieros.go`:**
```go
package launch

import (
    "errors"
    "fmt"
    "os/exec"
    "runtime"
)

// AtelierOS is intentionally not an Editor integration: launch owns one
// primary model and the local Ollama endpoint; AtelierOS keeps its own
// bridge/channel selection UX after startup.
type AtelierOS struct{}

func (a *AtelierOS) String() string { return "AtelierOS" }

var (
    atelierGOOS      = runtime.GOOS
    atelierLookPath  = exec.LookPath
    atelierCommand   = exec.Command
    atelierOllamaURL = envconfig.ConnectableHost
)

const (
    atelierPipPackage   = "atelieros"
    atelierInstallHint  = "pip install atelieros"
    atelierGatewaySetup = "atelier gateway setup"
)

func (a *AtelierOS) binary() (string, error) {
    bin, err := atelierLookPath("atelier")
    if errors.Is(err, exec.ErrNotFound) {
        return "", fmt.Errorf(
            "AtelierOS is not installed. Install it with:\n\n  pip install atelieros\n",
        )
    }
    return bin, err
}

func (a *AtelierOS) Configure(model string) error {
    bin, err := a.binary()
    if err != nil {
        return err
    }
    ollamaURL := atelierOllamaURL()
    return atelierCommand(bin,
        "config", "set", "ollama-url", ollamaURL,
        "--model", model,
    ).Run()
}

func (a *AtelierOS) Paths() []string {
    home, _ := os.UserHomeDir()
    return []string{
        filepath.Join(home, ".config", "atelier-launcher", "config.json"),
    }
}

func (a *AtelierOS) Run(_ string, _ []LaunchModel, args []string) error {
    bin, err := a.binary()
    if err != nil {
        return err
    }
    return atelierCommand(bin, append([]string{"gateway", "start"}, args...)...).Run()
}
```

**Registration in `cmd/launch/launch.go`** (in the integration lookup table):
```go
"atelieros": {Name: "atelieros", Runner: &AtelierOS{}},
```

**`cmd/launch/atelieros_test.go`** — mirrors structure of `hermes_test.go`:
- Unit tests for binary detection (found / not found)
- Unit tests for Configure() path construction
- Unit tests for Run() arg forwarding
- Integration tests gated on `ATELIER_AGENTS_SKIP_LIVE != 1`

**Launch icon: `app/ui/app/public/launch-icons/atelieros.svg`**
The existing AtelierOS logo exported as a clean SVG (dark-mode friendly, square viewport).

**M3 done-signal:** `go test ./cmd/launch/...` passes; PR to `ollama/ollama` submitted.

---

### M4 — Upstream Merge + Official Listing

**Goal:** Merged into `ollama/ollama` main and visible on `docs.ollama.com`.

**Process:**
1. Address all PR review feedback from Ollama maintainers
2. `atelieros` added to `cmd/launch/integrations_test.go` expected-integrations list
3. Ollama docs deploy picks up `docs/integrations/atelieros.mdx`
4. AtelierOS appears in sidebar: `Integrations → Assistants → AtelierOS`

**M4 done-signal:** `docs.ollama.com/integrations/atelieros` is live and
`ollama launch atelieros` works in an upstream Ollama release.

---

## Implementation checklist

### AtelierOS side (this repo + `atelier-launcher` new repo)

- [ ] **M2.1** Create `atelieros/atelier-launcher` repo (GitHub, Apache-2.0)
- [ ] **M2.2** Implement `atelier setup` wizard (Ollama URL detection, model selector, Docker preflight)
- [ ] **M2.3** Implement `atelier gateway start|stop` (Docker backend, Linux/macOS platform handling)
- [ ] **M2.4** Implement `atelier gateway setup` (interactive channel-connection wizard)
- [ ] **M2.5** Publish `atelieros` package to PyPI (`pip install atelieros`)
- [ ] **M2.6** Write installer script: `curl -fsSL https://atelier-labs.net/install.sh | bash`
- [ ] **M2.7** Export AtelierOS SVG logo (square, dark-mode safe)
- [ ] **M2.8** Update `ops/docker-compose.yml` to support `--network host` mode for Ollama access
- [ ] **M2.9** Update `docs/` (atelier-labs.net) with Ollama setup guide

### `ollama/ollama` upstream (PR)

- [ ] **M1.1** Open GitHub issue `integration: add AtelierOS as an Ollama Assistant`
- [ ] **M3.1** `docs/integrations/atelieros.mdx` — new file
- [ ] **M3.2** `docs/integrations/index.mdx` — add AtelierOS to Assistants list
- [ ] **M3.3** `docs/docs.json` — add `/integrations/atelieros` to Assistants group
- [ ] **M3.4** `cmd/launch/atelieros.go` — Go launcher implementation
- [ ] **M3.5** `cmd/launch/atelieros_test.go` — unit + gated live tests
- [ ] **M3.6** `cmd/launch/launch.go` — register `"atelieros"` in lookup table
- [ ] **M3.7** `cmd/launch/integrations_test.go` — add to expected-integrations list
- [ ] **M3.8** `app/ui/app/public/launch-icons/atelieros.svg` — app icon

---

## Consequences

### Positive

- **Discovery:** AtelierOS becomes findable by the ~52M monthly Ollama users
- **Credibility:** Listing alongside OpenClaw and Hermes signals ecosystem maturity
- **Differentiation:** Only Ollama assistant with a native EU AI Act + GDPR compliance layer
- **Pitch material:** "Ollama official integration" is a concrete credibility marker for
  the Deep Tech Night pitch (2026-06-23) and the KI-Accelerator application
- **Zero-egress story:** `ollama launch atelieros --profile eu-production` → fully local,
  no cloud API key required — unique among the current Assistants

### Negative / risks

- **Docker acceptance risk:** Ollama maintainers may not want a Docker-wrapping
  launcher in `cmd/launch/`. Mitigation: raise in the issue before the PR; fallback
  is a pure-Python native launcher with optional Docker as the runtime backend.
- **Maintenance burden:** Every Ollama API change or new launch interface may require
  an update to `cmd/launch/atelieros.go`. Mitigation: the Go file is thin (delegates
  all logic to the Python package).
- **PyPI package scope creep:** `atelier-launcher` must stay lean (no heavy deps).
  Hard rule: it must not `import anthropic` or pull in the full AtelierOS tree.
- **Platform coverage:** `--network host` does not work on Docker Desktop (Mac/Windows).
  `host.docker.internal` must be used on those platforms. The launcher must
  auto-detect and handle both cases.

### Neutral / deferred

- `ollama launch atelieros` starts the Docker container. For operators who run
  AtelierOS natively (no Docker), the native backend path in `atelier-launcher`
  activates automatically.
- The `atelier-launcher` package is a separate repo/package deliberately — it must
  not depend on the AtelierOS repo to keep install size minimal (target: < 50 KB
  installed).
- A `nemoclaw`-style sandboxing ADR for AtelierOS may follow if Ollama adds a
  sandboxing tier for Assistants (currently only NemoClaw is in that group).

---

## Not in scope

- Changing HermesEngine itself — ADR-0066/0067 covers that
- Publishing AtelierOS to the Ollama model library (`ollama.com/library`) — that is
  for LLM model weights, not agent software
- A Modelfile-based persona packaging — low value without the full plugin stack
  (MCP, Forge, SkillForge are not expressible in a Modelfile)
- Windows-native support in the launcher — deferred; WSL2 path works

---

*References: [ollama/ollama docs/integrations/](https://github.com/ollama/ollama/tree/main/docs/integrations) · [cmd/launch/hermes.go](https://github.com/ollama/ollama/blob/main/cmd/launch/hermes.go) · [cmd/launch/openclaw.go](https://github.com/ollama/ollama/blob/main/cmd/launch/openclaw.go) · [docs.ollama.com/integrations](https://docs.ollama.com/integrations)*
