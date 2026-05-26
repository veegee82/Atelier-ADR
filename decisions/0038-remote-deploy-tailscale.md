# ADR-0038 — Remote-Deploy via Tailscale (private operator console)

**Status:** Accepted (2026-05-18)
**Touches:** `ops/tailscale/` (NEW), `ops/docker-compose.yml` (untouched)
**Sister-ADR:** ADR-0037 (console re-launch)

---

## Context

ADR-0037 shipped the Atelier console as a local-only systemd unit on
`127.0.0.1:8765`. The next operational step is "I want my console
running on a Docker host (mini-PC, VPS, NAS) and reachable from my
laptop and phone without exposing anything to the public internet."

The repo already carries a production-deploy template at
`ops/docker-compose.yml` that bundles the atelier container with a
Caddy reverse-proxy sidecar. That template is purpose-built for
**public** access — domain, ACME / Let's Encrypt, port 80/443 open
to the world. Two costs for a single-operator setup:

1. Requires a domain name + DNS pointing at the host before first
   start (the entrypoint waits for the cert).
2. Public ports invite scanners and require ongoing security hygiene
   (fail2ban, log review, automated patching) even when only the
   operator ever connects.

For the single-operator scenario neither cost is justified.

---

## Decision

Add a parallel deployment template at **`ops/tailscale/`** that
replaces the Caddy public-TLS layer with a **Tailscale sidecar**.

- Both atelier and tailscale containers share **one network namespace**
  (`network_mode: "service:tailscale"`). Inside the namespace, the
  gateway listens on `:8000` exactly as today; from outside, it is
  reachable only as `http://<hostname>.<tailnet>.ts.net:8000` over
  the WireGuard mesh.
- No host port is published. The Docker host itself does not need
  any inbound firewall rule beyond Tailscale's UDP (which Tailscale
  handles for the operator).
- The existing `ops/docker-compose.yml` stays the recommended path
  for any deployment that needs to be publicly reachable. The two
  files do not share a compose project name so an operator can have
  both checked out without conflict.

### Why a sidecar, not host-mode Tailscale

A Tailscale daemon on the host with the atelier container on
`network_mode: host` would also work. The sidecar variant is
preferred for three reasons:

1. **Isolation.** Only one Docker compose project needs to run; no
   host-level Tailscale package is required (good fit for minimal
   bases like Container-Optimized OS or Talos).
2. **Tailnet hostname == container name.** With the sidecar, the
   tailnet sees a host called `atelier` (set via `hostname:`) and
   resolves it via MagicDNS regardless of the underlying host's
   hostname. Restoring on a different machine just works.
3. **Lifecycle coupling.** `docker compose down` stops both at once.
   No risk of orphaned tailscale daemon talking to a stopped atelier.

### Hostname + URL contract

| Tailnet device name | URL the operator opens |
|---|---|
| `atelier` (default `hostname:`) | `http://atelier:8000/console/` (MagicDNS short name) |
| | `http://atelier.tail-XXXX.ts.net:8000/console/` (FQDN) |

Operators can override `hostname:` in their own checkout (`atelier-home`,
`atelier-work`, …) without rebuilding the image.

### Auth + transport

- WireGuard inside Tailscale is end-to-end encrypted between peers
  → no public TLS termination needed for the tailnet-only case.
- Console auth contract from ADR-0015 is unchanged: owner-token,
  `atelier_console_sid` cookie, `X-CSRF-Token` for mutations.
- Cookie `secure=false` (plain HTTP) is acceptable here because the
  transport is already encrypted by WireGuard. If the operator later
  publishes the console publicly, the Caddy variant flips `secure=true`
  automatically.

### Bootstrapping the first owner token

The owner-token must be minted on the server and copied to the
operator's laptop through a secure channel **before** the first login
in the browser. The recommended one-shot:

```bash
docker exec atelier /opt/atelier-venv/bin/python -c \
  "import sys; sys.path.insert(0,'/opt/atelier-repo/core/gateway'); sys.path.insert(0,'/opt/atelier-repo/operator/forge'); \
   from atelier_gateway.auth import issue_token; \
   print(issue_token('_default', label='laptop'))"
```

The token is printed exactly once. Subsequent revokes via the same
module's `revoke_token`.

---

## Non-Goals

- Public access from outside the tailnet. The Caddy variant
  (`ops/docker-compose.yml`) covers that.
- ACL-based multi-tenant access. Single operator only.
- Mobile-optimised UI. The console is responsive enough; native apps
  are not in scope.
- Replacing the lokal-systemd path (ADR-0037 Iter 4). That stays the
  recommended entry point for desktop-only operators.

---

## Consequences

**Positive.**
- One-evening setup: install Tailscale on server + laptop, mint an
  auth-key, `docker compose up -d`, mint an owner token, log in.
- No DNS, no certificate, no public port. Lower surface area than
  any other remote-access option.
- Mobile-friendly without extra work: Tailscale's mobile clients
  put the console one tap away on the phone.

**Costs.**
- Tailscale dependency. The operator must trust Tailscale Inc. as
  the coordination server (the data plane is peer-to-peer WireGuard,
  but the control plane is hosted). Headscale (self-hosted) is a
  drop-in replacement if that becomes a concern.
- `network_mode: "service:tailscale"` means the atelier container's
  networking is tied to the sidecar — restarting tailscale resets
  TCP connections (including any open WebSocket).

**Risks.**
- TS_AUTHKEY leakage. The auth-key is a long-lived secret per ADR
  default. Mitigated by Tailscale's "reusable + ephemeral + tagged"
  key options; the README recommends a tagged, non-reusable key.
- DNS collisions. If the operator's tailnet already has a device
  called `atelier`, MagicDNS will refuse the duplicate. Mitigated
  by env-var override.

---

## Migration / rollback

- This ADR adds a new directory only — no change to existing deploys.
- Switching between the public (Caddy) and private (Tailscale) path
  is a `docker compose -f <one> down && docker compose -f <other> up -d`;
  the volume layout is identical (`./data/home/`) so state survives.

---

## References

- ADR-0007 (multi-tenant axis, `<atelier_home>` layout)
- ADR-0015 (console v1 auth contract — unchanged here)
- ADR-0037 (console re-launch — local systemd path)
- `ops/docker-compose.yml` (public-deploy with Caddy)
- `ops/tailscale/docker-compose.yml` (this ADR's new template)
- Tailscale docs: <https://tailscale.com/kb/1282/docker> (sidecar pattern)
