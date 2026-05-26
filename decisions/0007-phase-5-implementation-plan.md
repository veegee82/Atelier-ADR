# ADR-0007 Phase 5 — `.atelier-pkg` package format

**Status:** Active
**Started:** 2026-05-11
**Predecessors:** Phase 1–4 (closed)

Phase 5 ships the **package format** AtelierOS uses to publish and
install skill / persona / forge-tool bundles. The format is a
tar.gz with a strict manifest at the root, plus a detached
signature next to the archive. Sigstore is the documented production
path; the in-tree implementation uses a simpler ed25519 signature
pair to keep the CI hermetic.

## Format (on-disk)

```
acme-customer-support-1.4.2.atelier-pkg          # the archive
acme-customer-support-1.4.2.atelier-pkg.sig      # detached signature
```

The `.atelier-pkg` is a gzip-compressed tar containing exactly:

```
manifest.atelier.yaml
payload/
  skills/      (optional — SkillForge skills)
  personas/    (optional — cowork personas)
  tools/       (optional — forge tools)
```

`manifest.atelier.yaml`:

```yaml
apiVersion: atelier/v1
kind: Package
metadata:
  name: customer-support
  publisher: acme
  version: 1.4.2
  created_at: "2026-05-11T00:00:00Z"
spec:
  contents:
    skills:   [refund-flow, escalation-playbook]
    personas: [agent]
    tools:    []
  dependencies:
    atelier_apiversion: atelier/v1
    runtime_min:        "0.10"
  checksums:
    payload_sha256: <hex>
```

## Sub-phases

| Sub | Scope |
|---|---|
| **5.1** | `atelier_gateway/packaging.py` — pack / unpack / verify; ed25519 signer + verifier |
| **5.2** | CLI: `atelier package {build,verify,install} <path>` |
| **5.3** | Per-subtask E2E: round-trip build → sign → verify → install |

What Phase 5 deliberately does NOT do:
- Run Sigstore against a real Rekor instance. Operators wire that
  themselves; the package format accepts any detached signature
  (`<archive>.sig`) and the verifier exposes a hook so a Sigstore
  client can plug in.
- Solve the marketplace registry. Phase 5 is the *format* +
  publish/install primitive; a hosted registry is Phase 5+ work.
- Cross-sign with multiple keys. One signature per package; key
  rotation is the operator's concern.
