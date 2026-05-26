# ADR-0007 Phase 4 — Implementation Plan (OCI image + Helm)

**Status:** Active
**Started:** 2026-05-11
**Predecessors:** Phase 1, 2, 3 (closed)

Phase 4 packages the Gateway for container-orchestrated deployment.
Two artefacts:

1. **OCI image** built from `core/gateway/Dockerfile`,
   based on `python:3.12-slim`, runs `uvicorn` as the entry point.
2. **Helm chart** under `core/gateway/chart/` with the
   minimum surface a K8s operator needs: Deployment, Service,
   ConfigMap (tenant home), Secret stub (Gateway bearer tokens),
   and a values.yaml that wires the lot.

E2E gate (hermetic):
- `tests/test_container.py` — validates the Dockerfile parses
  cleanly and the Helm chart renders with `helm template` (when
  `helm` is on PATH; skips otherwise).

What Phase 4 deliberately does NOT do:
- Push to a registry. Operators do that themselves.
- Provide a CI workflow that builds the image. The Dockerfile +
  chart are artefacts; the operator integrates them into their CI.
- Solve the `bwrap` requirement for restrictive K8s clusters. The
  chart's `securityContext` documents the privileges needed; a
  sidecar pattern for namespaces without user-namespaces is
  deferred to a follow-up if demand surfaces.
