# ADR-0007 Phase 7 — Durable queue + rate-limit + gRPC parity

**Status:** Active
**Started:** 2026-05-11
**Predecessors:** Phase 1–6 (closed)

Phase 7 finishes the ADR-0007 roadmap with three structural
hardenings:

1. **Durable run queue** — SQLite-backed FIFO so an accepted run
   survives a gateway process crash and gets picked up after
   restart. Phase 2.3's in-memory dispatcher is the fast path;
   the queue is the recovery path.
2. **Per-tenant rate-limit** — token-bucket gate on
   `POST /v1/tenants/{tid}/runs` driven by
   `tenant.atelier.yaml::spec.budget.max_runs_per_day` (Phase 3.1).
   Integrates with Layer 20's quota concept structurally; the
   Gateway's gate fires before any engine work.
3. **gRPC proto definition** — `atelier_gateway/grpc/atelier.proto`
   declares a streaming gRPC surface that mirrors the REST endpoints.
   Phase 7 ships the proto + a documented server skeleton; full
   grpcio code-gen is opt-in for operators who want gRPC.

## Sub-phases

| Sub | Scope |
|---|---|
| **7.1** | `durable_queue.py` — SQLite WAL queue; `enqueue`, `next_pending`, `mark_terminal`; recovery sweep on dispatcher startup |
| **7.2** | `rate_limit.py` — token bucket per tenant; reads `tenant.atelier.yaml`; emits `gateway.rate_limited` audit event |
| **7.3** | gRPC: `atelier.proto` + server skeleton + opt-in test |
| **7.4** | Closure narrative + final PR update + push |

What Phase 7 deliberately does NOT do:
- Multi-process worker pool. The asyncio worker is sufficient for
  Phase 2's traffic envelope; a horizontal worker pool requires
  external coordination (e.g. Redis pub-sub) which is operator
  territory.
- Replace the Phase 2.3 in-memory dispatch path. The queue is
  additive: every accepted run goes into the queue AND is
  immediately dispatched in-memory. Recovery sweep picks up runs
  that never reached terminal because of a crash.
- gRPC code generation in CI. The proto file is the contract;
  `protoc` invocation belongs to the operator's build.
