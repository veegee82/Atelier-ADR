# ADR-0026 — Atelier Compute Fabric (General-Purpose ML Compute, Plugin-Extensible, Async Oracle, Big-Data Parallel)

**Status:** Accepted
**Date:** 2026-05-17
**Companion to:** ADR-0013 (Compute-Worker Plugin — prerequisite),
ADR-0012 (Large-Data Snapshot Layer — `data_handle` integration),
ADR-0007 Phase 3.1 (`tenant.atelier.yaml` schema extension),
ADR-0022 (Engine-agnostic Forge/SkillForge — engine-interface precedent),
Layer 6 (Forge — `run_tool()` sandbox primitive),
Layer 7 (Skill-Forge — strategy promotion precedent),
Layer 10 (path-gate — compute tree protection),
Layer 22 (`WorkerEngine` — engine-agnostic MCP-call shape),
Layer 25 (current narrow Compute Worker — extended, not replaced)
**Implements:** `core/compute/` (extension of ADR-0013 plugin tree,
three new sub-packages: `fabric/backends/`, `fabric/oracle/`, `fabric/parallel/`)
**Replaces subsystem:** Layer 25's strategy-only iteration loop is subsumed as
the `grid` / `random` / `bayesian` built-in backends of the new protocol.
Existing `compute_run` / `compute_status` / `compute_result` / `compute_abort`
MCP tool signatures remain byte-identical (backward-compat).

---

## Context

ADR-0013 (Layer 25) delivered an opt-in iteration driver that takes parameter
sweeps out of the LLM context loop. It solved the narrow problem well: three
strategies (grid, random, Bayesian), a thread-pool batch runner, convergence
enforcement, audit-safe iteration history. But it is structurally narrow:

1. **Only parameter sweeps.** A tool that trains a scikit-learn model must
   still be written by the operator as a Forge tool; the compute layer then
   sweeps its hyperparameters. There is no concept of "the compute layer itself
   owns and drives the training loop".

2. **In-memory data only.** The Forge sandbox receives data via ro-bind mount,
   but the Compute Worker has no concept of data chunking, shard-level
   parallelism, or incremental learning. A 100 GB dataset either fits in RAM
   or the job crashes.

3. **Single backend.** The only ML "backend" is whatever Python the Forge tool
   author wrote. sklearn and XGBoost are indistinguishable from a `curl` call
   — the driver has no knowledge of the learning algorithm's capabilities
   (does it support `partial_fit`? Can it resume a checkpoint? Does it run
   distributed natively?).

4. **Steering is the LLM's job.** Between iterations the LLM can read
   `compute_status`, observe the loss curve, and submit a second `compute_run`
   with adjusted params. This costs LLM context and LLM turns — exactly the
   bottleneck ADR-0013 was designed to eliminate.

5. **Not engine-agnostic at the semantic level.** Any WorkerEngine (Claude Code,
   Codex, OpenCode/Ollama) can call `compute_run` today — the MCP interface is
   engine-neutral. But the cognitive model ("I am sweeping parameters") is
   Claude-Code-shaped. A Codex worker or a local Ollama-backed OpenCode session
   should be able to submit complex ML jobs without understanding the sweep
   internals.

The compliance baseline (CLAUDE.md) adds three hard constraints:

- **No SDK imports.** All LLM interaction MUST go through `claude -p --max-turns 1
  --no-tools` subprocess (subscription-native, zero Anthropic API key).
- **Audit-chain metadata-only.** No parameter values, no training samples, no
  model weights enter the chain. Only fingerprints and scalar metrics.
- **Opt-in per tenant.** The plugin starts off; operator activates per tenant in
  `tenant.atelier.yaml`.

---

## Decision

Extend `core/compute/` with three new sub-systems that together
constitute the **Atelier Compute Fabric**:

**A — `ComputeBackend` plugin protocol** (engine/framework abstraction)
**B — Async Gradient Oracle** (LLM-steered mid-training guidance, non-blocking)
**C — Inter-Job Parallelism** (ShardManager + ResourceManager + Aggregator)

The three sub-systems are orthogonal and independently shippable but form a
coherent whole. The existing `compute_run` MCP surface is extended in a
backward-compatible way; a job that does not declare a `backend` behaves
byte-identically to today's ADR-0013 behavior.

---

### A — ComputeBackend Plugin Protocol

A `ComputeBackend` is a Python object satisfying this Protocol:

```python
class ComputeBackend(Protocol):
    name: str                        # e.g. "sklearn", "xgboost", "acme_internal"
    version: str
    supports_partial_fit: bool       # can train on chunks incrementally?
    supports_checkpointing: bool     # can save + restore mid-run?
    supports_distributed: bool       # manages its own distribution (e.g. Spark)?

    def create_session(
        self, spec: JobSpec, cursor: DataCursor
    ) -> BackendSession: ...

    def train_epoch(self, session: BackendSession) -> EpochMetrics: ...

    def translate_steering(
        self, vector: SteeringVector
    ) -> BackendParams: ...          # Oracle speaks abstract ML; backend translates

    def checkpoint(self, session: BackendSession, path: Path) -> None: ...
    def restore(self, session: BackendSession, path: Path) -> None: ...
    def finalize(self, session: BackendSession) -> ArtifactManifest: ...
    def cleanup(self, session: BackendSession) -> None: ...
```

`translate_steering` is the key abstraction boundary: the Oracle always
emits a generic `SteeringVector` (`{"lr": "↓0.3", "max_depth": "↑1"}`).
Each backend translates to its own native parameter names. The Oracle has
no knowledge of backend internals; backends have no knowledge of the Oracle.

**Built-in backends** (bundled with the plugin):

| Backend name | Framework | Chunk strategy | Checkpoint |
|---|---|---|---|
| `sklearn` | scikit-learn | `partial_fit` API | joblib dump |
| `xgboost` | XGBoost | `DMatrix` streaming + `xgb_model=prev` | native `.ubj` |
| `lightgbm` | LightGBM | `init_model=` incremental boosting | native `.txt` |
| `statsmodels` | statsmodels | rolling-window OLS/ARIMA | pickle |
| `polars_transform` | Polars | lazy LazyFrame eval | Parquet intermediate |

**Plugin discovery** — four-tier hierarchy, highest-specificity wins:

```
1. System:  /etc/atelier/compute/plugins/       enterprise RPM/DEB packages
2. Tenant:  <atelier_home>/tenants/<id>/compute/plugins/   per-tenant enterprise plugins
3. User:    <atelier_home>/compute/plugins/      operator experiments
4. Bundle:  core/compute/backends/   shipped with AtelierOS
```

A tenant-level plugin with the same `name` as a bundle plugin **replaces** it
for that tenant. This allows enterprises to ship a hardened `sklearn` variant
without forking AtelierOS.

**Plugin manifest** (`compute_plugin.yaml` at the plugin root):

```yaml
name: acme-compute-backend
version: 1.0.0
author: ACME Corp
backend_class: acme_compute.AcmeBackend

capabilities:
  - distributed          # manages its own cluster distribution
  - gpu                  # uses GPU resources
  - custom_loss          # supports non-standard loss functions

parallel:
  intra_backend:
    model: distributed   # thread | process | distributed | gpu
    max_workers: auto    # auto = nproc or GPU count
    gpu_aware: true
  inter_job:
    compatible: false    # ShardManager must NOT spawn multiple instances
    max_concurrent_instances: 1  # if compatible: true, operator cap

sandbox:
  network: allow         # MUST be explicitly approved in tenant policy
  extra_mounts:
    - /etc/acme/cluster.conf:ro

audit_events:            # additional event names this plugin emits
  - acme.session_started
  - acme.cluster_error

steering_map:            # Oracle abstract key → backend-native param name
  lr:           learning_rate
  max_depth:    tree_max_depth
  subsample:    row_sample_rate
  custom_reg:   acme_l3_penalty  # firm-specific extension key
```

**Security gate for plugins:**
- Installation is operator-only (`compute_plugin_enable` requires Owner role).
- `sandbox.network: allow` requires explicit `compute.allow_network_plugins: true`
  in `tenant.atelier.yaml` (off by default).
- Plugin code runs inside bwrap like built-in backends — never in the
  adapter/bridge process.
- Plugin MUST NOT `import anthropic` / `import openai` (CI AST lint extended
  to `<tenant>/compute/plugins/**/*.py` and all bundle backends).
- Audit events declared in the manifest are validated against the unified
  chain schema before emission; extra fields raise `AuditFieldNotAllowed`.

---

### B — Async Gradient Oracle

The Oracle is a second subprocess that runs **concurrently** with the Worker,
never blocking it. The communication is via two in-memory asyncio queues:

```
Worker process (bwrap):
  training_loop()
    for epoch in range(max_epochs):
      metrics = backend.train_epoch(session)
      oracle_queue.put_nowait(metrics)        # fire-and-forget
      steer = steer_queue.get_nowait_or_none()  # non-blocking peek
      if steer:
        session.apply_params(backend.translate_steering(steer))
      if converged(metrics): break

oracle_loop()                                 # runs concurrently via asyncio.gather
  while not done:
    metrics = await oracle_queue.get()        # blocks until Worker pushes
    raw = await _call_oracle_subprocess(metrics)
    steer_queue.put_nowait(parse_steering(raw))

asyncio.gather(training_loop(), oracle_loop())
```

**Steering lag:** Oracle output for iteration N is applied at iteration N+1
or N+2 (depending on queue drain timing). This is a documented property, not
a bug. The Worker never waits; the Oracle never blocks training.

**Oracle subprocess** calls `claude -p --max-turns 1 --no-tools` with a
structured prompt containing only the metric history (no training data, no
model weights). It emits a **structured JSON steering vector** — not prose:

```json
{
  "lr":        "↓0.3",
  "max_depth": "↑1",
  "subsample": "↑0.05"
}
```

Numeric delta interpretation: `"↓0.3"` means "multiply current value by 0.7";
`"↑1"` means "increment integer param by 1". The `translate_steering` backend
method receives this generic vector and returns backend-native `BackendParams`.

**Parallel-worker Oracle aggregation:** When N workers run in parallel
(Section C), the Oracle receives metrics from all N workers per batch.
It aggregates before generating the steering vector:

```json
{
  "global_steer": {"lr": "↓0.2"},
  "divergence_detected": true,
  "worker_specific_steer": {
    "worker_0": {"subsample": "↑0.1"},
    "worker_2": {"max_depth": "↑1"}
  }
}
```

`divergence_detected` fires when per-worker loss variance exceeds a
configurable threshold (`oracle.divergence_threshold`, default 0.05).

**Failure modes:**
- Oracle subprocess crash → `oracle.subprocess_failed` WARNING audit event.
  Worker continues unsteered (graceful degrade). `oracle.consecutive_failures`
  counter; after 3 consecutive failures the Oracle loop exits cleanly.
- Oracle timeout (configurable, default 30s per call) → same as crash.
- Queue full (> 8 buffered metric batches) → oldest batch dropped;
  `oracle.queue_dropped` WARNING emitted.

**Cost contract:** Oracle subprocess is authenticated via subscription-native
`claude -p`, never via `ANTHROPIC_API_KEY`. The plugin MUST NOT `import
anthropic`. Tenant `disallow_llm_strategies: true` disables the Oracle
globally (Oracle is classified as an LLM strategy).

---

### C — Inter-Job Parallelism (ShardManager, ResourceManager, Aggregator)

**Design decision (dialectical):**
Two fundamentally different kinds of parallelism exist in ML training:

- *Intra-backend* parallelism (threads, GPUs, Spark executors): each backend
  knows best how to parallelize internally. This belongs in the plugin manifest
  (`parallel.intra_backend`). The Fabric does not interfere.

- *Inter-job* parallelism (multiple independent jobs on data shards or different
  hyperparams): this is orthogonal to which backend is used. This belongs in
  the Fabric core as a generic primitive.

Backends declare `parallel.inter_job.compatible` in their manifest. When
`compatible: false` (e.g. `spark-backend` which already distributes internally),
the ShardManager does not spawn multiple instances of that backend — instead it
hands the full DataCursor to the backend and trusts its native distribution.

**ShardManager** — built-in core component:

Receives a `data_ref` handle (Layer 24 `DataCursor`) and horizontally
partitions it into N `ShardCursor` objects. Backends receive a `ShardCursor`
indistinguishable from a full `DataCursor` — they are unaware of sharding.

Sharding strategies (all built-in, not pluggable):

| Strategy | Partition logic | Use case |
|---|---|---|
| `hash` | modular hash of row index → N buckets | i.i.d. assumption, general |
| `range` | sequential row ranges | time-series, ordered data |
| `stratified` | class-balanced buckets | prevents class imbalance per shard |
| `time_window` | fixed calendar windows | online learning, drift detection |

**ResourceManager** — built-in core component:

Reads the host process's cgroup limits (`cpu_quota`, `memory_limit`) and the
backend manifest's `parallel.inter_job.resource_per_instance`. Computes the
maximum number of concurrent Worker instances that fit within the resource
envelope. Allocates `ResourceSlot` objects before spawning; releases on
completion or error.

Each Worker process spawned by the ShardManager receives cgroup limits set at
bwrap invocation time — it cannot consume more than its allocated slot.

**Aggregator** — built-in core component:

Collects `ArtifactManifest` objects from all completed parallel Workers and
combines them according to the `aggregation_strategy`:

| Strategy | Mechanism | Applicable to |
|---|---|---|
| `best` | pick by primary metric | any model type |
| `average` | weighted parameter averaging | linear models, neural nets |
| `vote` | majority vote on predictions | classifiers |
| `stack` | train a meta-learner on parallel outputs | any |
| `federated_avg` | FedAvg algorithm (privacy-preserving) | when raw data must not leave shards |
| `custom` | plugin supplies `aggregate(manifests) -> ArtifactManifest` | firm-specific |

`federated_avg` is structurally notable: it allows shard-local training without
ever aggregating raw data — only model updates travel between workers.

**Backend Negotiation** — optional, activated when `backend` is omitted from
`compute_job_create`:

1. Layer 24 metadata: data size, column types, sparsity, format.
2. Plugin registry: available backends + their capabilities.
3. Haiku-4.5 subprocess (structured output, no prose):
   `{"backend": "spark", "reason": "dataset >50GB, distributed cap required"}`.
4. Operator confirms or overrides before the job starts.

This is compute-layer routing — the same philosophy as Layer 5 auto-routing
for personas, applied to ML backends.

---

### D — DataSourceAdapter System (no-code data connectivity)

Layer 24 (`data_register`) works with local files. A company whose data lives
in S3, PostgreSQL, BigQuery, or a REST API must today extract it to a local
Parquet file before any compute job can touch it. That extraction step:

- Requires operator code and infrastructure
- Moves potentially sensitive data out of its governed location
- Breaks data-residency guarantees (EU data copied to a local disk that may
  not be in the approved zone)
- Makes incremental training impossible (full re-extract for every run)

ADR-0026 Section D closes this gap with a **DataSourceAdapter** system:
a declarative YAML manifest covers the common 90 % of source types (S3,
GCS, Azure Blob, PostgreSQL, BigQuery, Snowflake, REST API, Kafka) with zero
Python code from the operator. For the remaining 10 % a plugin Protocol is
available.

#### Design principle: declarative-first

A company connects their S3 bucket by writing one YAML file. No Python. The
adapter code is already bundled; the manifest is just configuration:

```yaml
# <atelier_home>/tenants/acme/compute/datasources/crm_events.yaml
name: crm_events
adapter: s3_parquet             # built-in; no code to write

source:
  bucket: acme-datalake-prod
  prefix: events/2024/
  format: parquet
  region: eu-central-1          # validated against tenant.data_residency zone

auth:
  method: vault                 # values come from Layer 16 secret vault
  secret_keys:
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY

schema_hint:
  timestamp_col: created_at
  pii_columns: [user_id, email, phone]   # Layer 24/32 redaction applied

pii_handling: redact            # Layer 24 strategy override for this source

filters:                        # pushed down to S3 (prefix filter / Parquet predicate)
  - col: created_at
    op: ">="
    value: "2024-01-01"

incremental:                    # only fetch rows newer than watermark
  mode: timestamp
  cursor_col: created_at
  watermark: "2024-01-01T00:00:00Z"   # auto-updated after each successful run
```

A PostgreSQL source looks identical — same manifest structure, different
`adapter:` value, different `source:` fields:

```yaml
name: orders_db
adapter: postgresql

source:
  host: db.acme.internal
  port: 5432
  database: production
  table: orders
  region: eu-central-1

auth:
  method: vault
  secret_keys: [DB_USER, DB_PASSWORD]

schema_hint:
  primary_key: order_id
  pii_columns: [customer_name, customer_email, shipping_address]

incremental:
  mode: sequence_id
  cursor_col: order_id
  watermark: 0                  # start from 0; auto-advances after each run
```

#### DataSourceAdapter Protocol

For sources beyond the built-in set, a company implements this Protocol —
exactly the same plugin model as `ComputeBackend`:

```python
class DataSourceAdapter(Protocol):
    name: str                          # e.g. "acme_internal_datalake"
    version: str
    supports_streaming: bool           # can chunk without full download
    supports_pushdown: bool            # can push filters/sharding to source
    supports_schema_discovery: bool    # can auto-detect schema
    supports_incremental: bool         # supports watermark / CDC

    def connect(
        self, config: SourceConfig, secrets: SecretEnv
    ) -> SourceSession: ...

    def discover_schema(self, session: SourceSession) -> SourceSchema: ...

    def create_cursor(
        self, session: SourceSession, query: SourceQuery
    ) -> DataCursor: ...

    def estimate_rows(
        self, session: SourceSession, query: SourceQuery
    ) -> int: ...

    def read_watermark(self, session: SourceSession) -> Any: ...
    def write_watermark(self, session: SourceSession, value: Any) -> None: ...

    def close(self, session: SourceSession) -> None: ...
```

`secrets: SecretEnv` is the bwrap-injected environment from the Layer 16
vault — the adapter reads credentials via `os.getenv()`. Credential values
NEVER appear in manifests, in Python code, or in the audit chain.

#### SourceQuery — injection-safe, no raw SQL

The LLM (or operator YAML) never passes raw SQL. Queries are expressed
as structured `FilterExpr` objects that each adapter translates to its
native query language:

```python
@dataclass
class FilterExpr:
    col: str
    op: str     # "=" | "!=" | ">" | ">=" | "<" | "<=" | "in" | "like"
    value: Any  # scalar or list for "in"

@dataclass
class SourceQuery:
    columns: list[str] | None = None    # None = all; pushed down when supported
    filters: list[FilterExpr] = field(default_factory=list)
    shard_index: int | None = None      # ShardManager passes this for pushdown
    n_shards: int | None = None
    limit: int | None = None
    order_by: str | None = None
```

A `postgresql` adapter translates `FilterExpr` to parameterized
`WHERE` clauses (`psycopg2` `%s` placeholders). An `s3_parquet` adapter
translates to Parquet predicate pushdown or S3 prefix filters. No adapter
ever interpolates user values into a string — SQL injection is structurally
impossible.

#### Built-in adapters (bundled, zero code for operator)

| Adapter name | Source type | Auth methods | Pushdown | Incremental |
|---|---|---|---|---|
| `local_file` | Local filesystem (CSV/JSON/Parquet) | None | No | No |
| `s3_parquet` | AWS S3 — Parquet files | Vault keys / IAM role | Prefix + predicate | Timestamp / partition |
| `s3_csv` | AWS S3 — CSV files | Vault keys / IAM role | Prefix filter | Timestamp |
| `gcs_parquet` | Google Cloud Storage — Parquet | Service account (vault) | Prefix + predicate | Partition |
| `azure_blob` | Azure Blob Storage — Parquet/CSV | Connection string (vault) | Prefix filter | Timestamp |
| `postgresql` | PostgreSQL / PgBouncer | username+password (vault) | Full SQL pushdown | Timestamp / sequence |
| `mysql` | MySQL / MariaDB | username+password (vault) | Full SQL pushdown | Timestamp / sequence |
| `bigquery` | Google BigQuery | Service account (vault) | Full SQL + partition | Timestamp / partition |
| `snowflake` | Snowflake | username+password / key (vault) | Full SQL pushdown | Timestamp |
| `redshift` | AWS Redshift | username+password (vault) | Full SQL pushdown | Timestamp |
| `delta_lake` | Delta Lake (local or S3) | Inherits from storage | Partition filter | Delta log |
| `http_rest` | Paginated REST API | Bearer / API key / OAuth (vault) | URL param filter | Cursor / timestamp |
| `kafka_batch` | Apache Kafka (batch window) | SASL / SSL (vault) | Topic + partition | Offset |

#### ShardManager integration with pushdown

When a datasource adapter declares `supports_pushdown: true`, the
ShardManager passes `shard_index` / `n_shards` directly in the `SourceQuery`.
The adapter translates these into source-native sharding:

| Adapter | Native sharding mechanism |
|---|---|
| PostgreSQL | `WHERE MOD(ctid::text::point[0]::int, n_shards) = shard_index` |
| BigQuery | `WHERE MOD(FARM_FINGERPRINT(primary_key), n_shards) = shard_index` |
| S3 Parquet | One S3 prefix segment per shard (object-level splitting) |
| REST API | `?page_shard=<shard_index>&page_shards=<n_shards>` (configurable param names) |

Adapters without pushdown (`supports_pushdown: false`) receive the full
cursor; client-side row filtering is applied by the ShardCursor wrapper.

#### Data-residency gate (EU AI Act / GDPR, load-bearing)

Every manifest MUST declare `source.region`. At `datasource_register` time:

1. `region` is validated against the tenant's `data_residency` zone
   (ADR-0007 Phase 3.3 compliance-zone routing).
2. A mismatch raises `DataResidencyViolation` — registration is rejected.
3. The violation is audited: `datasource.residency_violation` (WARNING) with
   `declared_region` and `tenant_zone` (no source credentials or data).

This is a structural guarantee: data in `eu-central-1` cannot be pulled
into a compute job running under a tenant whose zone is `us-east-1`, and
vice versa.

#### Incremental / CDC — only new rows per run

For large tables (hundreds of millions of rows), re-reading the full table
for each training run is prohibitively slow. Incremental mode fetches only
rows newer than the last successful run's watermark:

```
Run 1: watermark = 0           → reads rows 0..4,999,999  → trains → writes watermark = 5,000,000
Run 2: watermark = 5,000,000   → reads rows 5,000,000..9,999,999 → trains → writes watermark = 9,999,999
```

Three incremental modes (all built-in adapters support at least one):

| Mode | How | When to use |
|---|---|---|
| `timestamp` | `WHERE cursor_col > watermark` | Append-only tables with a reliable timestamp |
| `sequence_id` | `WHERE cursor_col > watermark` | Append-only tables with a monotonic integer key |
| `cdc_log` | Read from change-data-capture log (Kafka / Debezium) | Tables with updates/deletes |

Watermark state: stored in `<datasources>/<name>.checkpoint.json`
(path-gate protected — LLM cannot write it). Written only after the
compute job completes successfully (audit-first: `datasource.watermark_advanced`
event before filesystem write).

#### Layer 24 / 32 pipeline integration

`datasource_register` triggers the full Layer 24 / 32 pipeline on the
schema — not on raw data:

```
1. DataSourceAdapter.discover_schema()   → SourceSchema (column names + types)
2. PII detector (Layer 24)               → tags columns from manifest.pii_columns
                                             + auto-detection (regex + Presidio opt-in)
3. Redactor (Layer 24)                   → applies manifest.pii_handling strategy
4. L32 strict anonymisation (opt-in)     → k-anonymity bucketing of distinct counts
5. Token-cap gate (Layer 24)             → schema snapshot for LLM context
```

The LLM sees a PII-redacted schema snapshot. Raw data stays in the source
system and only reaches the backend inside bwrap via the adapter's
`create_cursor` call. The adapter's bwrap environment contains the vault
credentials; the LLM context never does.

---

### Extended MCP Surface

All existing `compute_run` / `compute_status` / `compute_result` / `compute_abort`
signatures remain byte-identical. New parameters on `compute_run` and new tools:

```
compute_run (extended):
  + backend?               string          backend name; omit → Backend Negotiation
  + convergence?           ConvSpec        {metric, threshold, patience, max_epochs}
  + oracle_enabled?        bool            default false; activates Async Oracle
  + oracle_divergence_thr? float           default 0.05
  + n_workers?             int             default 1; > 1 → ShardManager
  + shard_strategy?        string          hash | range | stratified | time_window
  + aggregation_strategy?  string          best | average | vote | stack | fedavg | custom

compute_job_create(
  type, model_spec, data_ref, backend?, hyperparams?,
  convergence?, oracle_enabled?, n_workers?, shard_strategy?,
  aggregation_strategy?, tags?
) → job_id

compute_parallel_run(
  job_spec, n_workers, aggregation_strategy,
  param_sampling?         # {key: grid_list | "log_uniform(lo,hi)"} for HP fan-out
) → run_id

compute_shard_plan(data_ref, strategy, n_shards?) → {
  shard_sizes, estimated_wall_s, available_slots, recommended_n
}

compute_resource_status() → {
  available_cpu_cores, available_memory_gb,
  active_workers, queued_jobs, per_backend_max_slots
}

compute_plugin_list() → [{name, version, status, capabilities}]
compute_plugin_enable(name, *, tenant_id)   # Owner-only
compute_plugin_disable(name, *, tenant_id)  # Owner-only
compute_backend_caps(name) → {capabilities, parallel, sandbox, steering_map}

compute_artifact_list(run_id?) → [{artifact_id, model_type, metric, path_hash}]

# DataSourceAdapter MCP tools:
datasource_register(name, manifest_yaml?)
  → {connection_tested, schema_snapshot, pii_columns_detected, handle}
  # manifest_yaml is optional: if absent, reads from the on-disk manifest file

datasource_list() → [{name, adapter, status, row_estimate, last_watermark}]

datasource_schema(name) → schema_snapshot  # PII-redacted, same L24/32 pipeline

datasource_test(name) → {ok, latency_ms, error?}  # connectivity health check

datasource_unregister(name)  # removes manifest + checkpoint; does NOT touch source

datasource_preview(name, n_rows?) → sample  # L24-redacted sample, max 20 rows

compute_job_create (extended):
  + datasource?  string   # datasource name; alternative to data_ref (Layer 24 handle)
                          # exactly one of data_ref / datasource must be provided
```

All new tools are advertised only when the worker socket is reachable
(same 5s TTL discovery cache as ADR-0013).

---

### Model Registry and Artifact Store

```
<atelier_home>/tenants/<tid>/compute/
├── runs/<run_id>/               # existing ADR-0013 layout (unchanged)
│   ├── manifest.json
│   ├── summary.json
│   └── iterations/*.json
├── artifacts/<run_id>/          # NEW — model artifacts
│   ├── model.<ext>              # pkl | ubj | txt | parquet | onnx
│   ├── meta.json                # {backend, metric, final_loss, n_epochs,
│   │                            #   n_workers, aggregation_strategy, ...}
│   └── shards/<k>/              # per-shard artifact before aggregation (optional)
├── plugins/<name>/              # NEW — tenant-level plugins
│   ├── compute_plugin.yaml
│   └── <backend_package>/
├── datasources/                 # NEW — DataSourceAdapter manifests
│   ├── <name>.yaml              # connection manifest (operator-written, path-gate protected)
│   ├── <name>.schema.json       # cached schema discovery (L24-redacted)
│   └── <name>.checkpoint.json   # incremental watermark state (path-gate protected)
├── datasource-adapters/<name>/  # NEW — tenant-level custom adapters (plugin Protocol)
│   ├── compute_datasource.yaml  # adapter manifest
│   └── <adapter_package>/
└── worker.sock                  # existing
```

Mode `0o600` on all artifact files. Path-gate (Layer 10) extended:
- `<atelier_home>/**/compute/artifacts/**` — protected from LLM direct writes
- `<atelier_home>/**/compute/plugins/**` — protected; operator installs only

**Model Registry** — SQLite FTS5 at `<atelier_home>/tenants/<tid>/compute/registry.db`:
Schema: `(run_id, backend, primary_metric, metric_value, artifact_path_hash,
tags, created_at)`. FTS5 over `tags`. LLM-queryable via `compute_artifact_list`.
Raw `artifact_path` never enters the LLM context — only `artifact_path_hash`.

---

### Audit Chain (metadata-only, unified chain)

New event types registered in `forge/security_events.py::EVENT_SEVERITY`:

| Event | Severity | Allow-listed fields |
|---|---|---|
| `compute.backend_session_started` | INFO | `run_id`, `backend`, `backend_version`, `shard_index` (if sharded) |
| `compute.epoch_completed` | INFO | `run_id`, `epoch`, `primary_metric`, `metric_value`, `wall_ms`, `shard_index` |
| `compute.oracle_steer_applied` | INFO | `run_id`, `epoch`, `steering_keys` (list of keys, no values), `divergence_detected` |
| `compute.oracle_subprocess_failed` | WARNING | `run_id`, `epoch`, `failure_reason` (curated set) |
| `compute.shard_completed` | INFO | `run_id`, `shard_index`, `total_shards`, `final_metric` |
| `compute.aggregation_completed` | INFO | `run_id`, `strategy`, `n_shards`, `final_metric` |
| `compute.backend_plugin_enabled` | INFO | `tenant_id`, `plugin_name`, `plugin_version` |
| `compute.backend_plugin_disabled` | INFO | `tenant_id`, `plugin_name` |
| `compute.checkpoint_written` | INFO | `run_id`, `epoch`, `checkpoint_path_hash` |
| `compute.artifact_registered` | INFO | `run_id`, `backend`, `artifact_path_hash`, `artifact_size_b` |
| `compute.resource_slot_denied` | WARNING | `run_id`, `backend`, `requested_slots`, `available_slots` |

Mirror of L23 / L25 / L28 rule: parameter values, model weights, training
data, and Oracle output text NEVER enter the audit chain.
`steering_keys` carries only the keys of the steering vector (e.g.
`["lr", "max_depth"]`), never the direction or magnitude.

---

### Engine-Agnostic Design

Every WorkerEngine (Claude Code, Codex CLI, OpenCode/Ollama) reaches the
Compute Fabric identically — via MCP tools on the Forge MCP server.
No engine-specific code path exists in the Fabric.

The practical implications:

- A Codex CLI worker can submit `compute_job_create` + `compute_parallel_run`
  and poll `compute_status` with no knowledge of which backend runs.
- An OpenCode/Ollama session on a local-only deployment can drive the entire
  Fabric, including Backend Negotiation (the Negotiation Oracle subprocess uses
  `claude -p`, which requires a Claude subscription; Negotiation is simply
  skipped when `claude` is not in PATH — job falls back to requiring an explicit
  `backend` parameter).
- The Async Oracle subprocess is always `claude -p` regardless of which engine
  submitted the job. This is intentional: the Oracle is an AtelierOS internal
  helper, not the engine that owns the conversation.

---

### Tenant Configuration Extension (`tenant.atelier.yaml`)

New fields under `spec.compute` (additive to ADR-0013 fields):

```yaml
spec:
  compute:
    enabled: false                          # existing — unchanged
    # --- ADR-0026 additions ---
    fabric_enabled: false                   # gates ADR-0026 sub-systems; default OFF
    allow_network_plugins: false            # allows sandbox.network: allow manifests
    max_parallel_workers: 4                 # inter-job parallelism cap [1, 32]
    max_artifact_size_mb: 500              # per-artifact cap [10, 10000]
    oracle_enabled: false                   # global oracle on/off per tenant
    oracle_model: null                      # null → claude -p default
    backend_allowlist: []                   # [] = all discovered backends allowed
    backend_denylist: []                    # wins over allowlist
    aggregation_strategies_allowed:
      - best
      - average
      - vote
      - stack
    negotiation_enabled: true              # Backend Negotiation via LLM; false = require explicit backend

    # --- DataSourceAdapter additions (Section D) ---
    datasource_enabled: false              # gates Section D; default OFF
    datasource_adapter_allowlist: []       # [] = all discovered adapters OK
    datasource_adapter_denylist: []        # wins over allowlist
    datasource_max_row_estimate: 0         # 0 = unlimited; cap for safety on untrusted sources
    datasource_incremental_enabled: true   # allow watermark-based incremental reads
    datasource_allow_network_adapters: false  # same gate as allow_network_plugins
    datasource_residency_strict: true      # reject on region/zone mismatch (recommended ON)
```

`extra="forbid"` on the Pydantic model (ADR-0007 Phase 3.1 schema strictness).
`fabric_enabled: false` is the guard — even with `enabled: true`, all
ADR-0026 MCP tools return `FabricNotEnabled` unless `fabric_enabled: true`.

---

### Implementation Plan — Sub-Phases

| Phase | What | Prerequisites |
|---|---|---|
| **26.1** | `ComputeBackend` Protocol + `BackendRegistry` (discovery, manifest validation, 4-tier loader) | ADR-0013 green |
| **26.2** | Built-in backends: `sklearn`, `xgboost`, `lightgbm` | 26.1 |
| **26.3** | `DataCursor` + `ShardCursor` (Layer 24 integration, all 4 sharding strategies) | 26.1, ADR-0012 |
| **26.4** | `ResourceManager` (cgroup-aware slot allocation, bwrap limit injection) | 26.1 |
| **26.5** | `ShardManager` + `Aggregator` (5 built-in strategies + `custom` hook) | 26.3, 26.4 |
| **26.6** | `compute_parallel_run` + `compute_shard_plan` MCP tools | 26.5 |
| **26.7** | Async Oracle (oracle_loop subprocess, steering vector protocol, queue management) | 26.2 |
| **26.8** | Parallel-Oracle aggregation (N-worker metric fan-in, divergence detection) | 26.5, 26.7 |
| **26.9** | Model Registry (SQLite FTS5, artifact store, `compute_artifact_list` MCP tool) | 26.5 |
| **26.10** | Plugin install/enable/disable/list MCP tools + security gate | 26.1 |
| **26.11** | Backend Negotiation (Haiku subprocess, structured output, operator confirm flow) | 26.1, 26.10 |
| **26.12** | Audit events (all new event types, allow-list enforcement, chain regression tests) | 26.6, 26.8, 26.9 |
| **26.13** | Prometheus + Grafana (Fabric metric families, dashboard panels) | 26.12, ADR-0007 Phase 6 |
| **26.14** | Tenant config extension (`fabric_enabled`, all new fields, strict schema) | 26.12 |
| **26.15** | `statsmodels` + `polars_transform` built-in backends | 26.2 |
| **26.D1** | `DataSourceAdapter` Protocol + `DataSourceRegistry` (4-tier discovery, manifest validation, `SourceQuery` dataclass) | 26.1, ADR-0012 |
| **26.D2** | Built-in adapters: `local_file`, `postgresql`, `mysql` (SQLAlchemy, parameterized queries, no raw SQL) | 26.D1 |
| **26.D3** | Built-in adapters: `s3_parquet`, `s3_csv`, `gcs_parquet`, `azure_blob` (boto3 / google-cloud-storage / azure-sdk, vault auth) | 26.D1 |
| **26.D4** | Built-in adapters: `bigquery`, `snowflake`, `redshift` (SQL pushdown, vault auth, region-check) | 26.D1 |
| **26.D5** | Built-in adapters: `http_rest`, `kafka_batch`, `delta_lake` | 26.D1 |
| **26.D6** | Incremental / CDC: watermark state, all three modes (`timestamp`, `sequence_id`, `cdc_log`), checkpoint write (audit-first) | 26.D2, 26.D3 |
| **26.D7** | Pushdown-ShardManager bridge: `SourceQuery.shard_index` / `n_shards` wiring, per-adapter pushdown tests | 26.D2, 26.5 |
| **26.D8** | Layer 24 / 32 pipeline integration: `datasource_register` → schema discovery → PII detection → snapshot | 26.D1, ADR-0012 |
| **26.D9** | Data-residency gate: `DataResidencyViolation`, tenant zone check, audit event | 26.D8, ADR-0007 Phase 3.3 |
| **26.D10** | `datasource_*` MCP tools (register / list / schema / test / unregister / preview) | 26.D8 |
| **26.D11** | Vault integration (Layer 16 v3): `SecretEnv` bwrap injection, no credential in manifest or chain | 26.D1, Layer 16 |
| **26.D12** | Datasource audit events (all types, allow-list, chain regression) | 26.D8, 26.D9 |
| **26.D13** | Prometheus + Grafana: datasource metric families, dashboard panels | 26.D12, ADR-0007 Phase 6 |
| **26.D14** | Tenant config extension (`datasource_*` fields, strict schema) | 26.D12 |
| **26.16** | ADR close, CLAUDE.md Layer 25 update, `layer-data.md` Fabric section | all sub-phases green (incl. D) |

Each sub-phase is a self-contained commit with a green test suite. No sub-phase
is merged before the previous is soak-stable (mirror of ADR-0013 + ADR-0024
discipline).

---

## Consequences

**What changes:**
- `core/compute/` gains four sub-packages: `fabric/backends/`,
  `fabric/oracle/`, `fabric/parallel/`, `fabric/datasources/`.
- Four new MCP tools advertised when `fabric_enabled: true` and worker reachable.
- `compute_run` gains optional backward-compatible parameters.
- Path-gate Layer 10 extended with two new protected globs.
- Tenant schema gains new `spec.compute.*` fields.
- CI AST lint extended to `<tenant>/compute/plugins/**/*.py`.

**What does not change:**
- Existing `compute_run` / `compute_status` / `compute_result` / `compute_abort`
  signatures and semantics — byte-identical for callers that don't use new params.
- `disallow_llm_strategies: true` blocks Oracle globally (Oracle = LLM strategy).
- Plugin not bootstrapped → worker not running → MCP tools not advertised.
  Discovery cache TTL unchanged (5s).
- `MUST NOT import anthropic` contract extended to all new Fabric modules.

**What is out of scope (planned follow-on ADRs):**
- ADR-0027: Federated learning cross-tenant protocol (FedAvg across tenants;
  requires consent gate integration, ADR-0007 tenant isolation).
- ADR-0028: ONNX export pipeline (backend-neutral model interchange format).
- Compute-aware quota (Layer 20 integration: cap per-tenant GPU/CPU hours).
- Real-time streaming predictions (model serving; Compute Fabric is training-only).

---

## Must NOT Do (load-bearing constraints)

See `docs/claude-ref/adr-0026.md` for the full annotated must-NOT list.
The five structural absolutes:

1. **Don't bypass `fabric_enabled` gate.** Even `compute_run` extensions must
   check `fabric_enabled`; falling through to ADR-0013 behavior when
   `fabric_enabled: false` is the correct path.
2. **Don't put steering values into the audit chain.** `steering_keys` (list
   of key names) only — never the direction, magnitude, or resulting params.
3. **Don't let the Oracle block the Worker.** Queues are the only communication
   channel; `get_nowait_or_none` is the only Worker-side read call. Any
   `await` in the training loop is a design violation.
4. **Don't spawn multiple ShardManager instances for `inter_job.compatible: false`
   backends.** The manifest flag is the structural guarantee; ignoring it
   against a Spark backend would create nested distribution layers.
5. **Don't auto-start the Fabric worker from bridge or adapter code.** Same
   constraint as ADR-0013: operator action is the gate.
6. **Don't put credential values into manifest files, the audit chain, or
   the LLM context.** Only key names are declared in `auth.secret_keys`;
   values live exclusively in the Layer 16 vault and reach adapters via
   bwrap environment injection. A manifest containing a literal password
   is a GDPR Art. 32 violation.
7. **Don't allow `datasource_register` to succeed when `source.region`
   mismatches the tenant's `data_residency` zone** and
   `datasource_residency_strict: true` (the default). The residency gate
   is structural, not advisory.
8. **Don't pass raw SQL strings through any MCP tool or `SourceQuery` field.**
   `FilterExpr` is the only legal query primitive. Adapters translate to
   parameterized queries internally. SQL injection via LLM-constructed queries
   is the threat; `FilterExpr` is the structural defense.

---

## References

- `docs/claude-ref/adr-0026.md` — full protocol reference + complete must-NOT list
- `docs/decisions/0013-compute-worker-plugin.md` — ADR-0013 (Layer 25, prerequisite)
- `docs/decisions/0012-large-data-snapshot-layer.md` — ADR-0012 (Layer 24)
- `docs/decisions/0022-engine-agnostic-forge-skillforge.md` — ADR-0022
- `core/compute/` — implementation target
- `operator/voice/hooks/path_gate.py` — protection layer (extended)
- `operator/forge/forge/mcp_server.py` — tool advertisement wiring
- Layer 22 (`WorkerEngine`) — engine-interface; Fabric is engine-agnostic by design
- Layer 11 (`dialectic.py`) — `claude -p --max-turns 1 --no-tools` subprocess pattern
- ADR-0007 Phase 3.1 — `tenant.atelier.yaml` schema extension contract
- EU AI Act Art. 15 — accuracy / robustness (Oracle steering must not corrupt
  model training; fail-open on Oracle failure is the structural safety choice)
- GDPR Art. 30 — audit chain completeness (Fabric events extend the chain,
  never bypass it)
