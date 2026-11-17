# TraceForge - System Architecture

Status: baseline design (v0.1, 2026-11-17)

## 1. Design goals and non-goals

### Goals
- **Independence.** TraceForge never requires the log-producing application to import TraceForge code.
  Integration is a text protocol (HTTP/TCP) or a file the agent tails.
- **Durability over throughput.** An accepted log record must survive a process crash. Acceptance is
  only reported after the record is durable in the write-ahead log.
- **Bounded memory.** Query and ingestion paths stream; memory use is a function of batch size and
  page size, never of total stored volume.
- **Index-first reads.** A query must never scan a segment whose metadata proves it cannot match.
- **Replaceable parts.** Storage, parsing, ingestion transports and auth are behind interfaces so a
  component can be swapped (e.g. gzip → zstd, local disk → object store) without touching callers.

### Non-goals (explicitly out of scope for 1.0)
- Distributed clustering, sharding or replication across nodes (single-node, vertically scaled).
- Full-text inverted indexing / relevance ranking (TFQL is field-predicate based, with substring match).
- Metrics and tracing storage - TraceForge stores logs and correlates by `trace_id` only.
- Alerting engine (hooks are designed for, but not implemented in, 1.0).

## 2. Component map

```
            ┌──────────────────────── external applications ────────────────────────┐
            │  Spring Boot app      Python service      nginx/API gateway            │
            └───────┬───────────────────────┬────────────────────────┬───────────────┘
      log files     │           HTTP POST   │            TCP stream  │
                    │                       │                        │
          ┌─────────▼─────────┐   ┌─────────▼─────────┐   ┌──────────▼─────────┐
          │  tf-agent          │   │  HTTP ingest      │   │  TCP ingest        │
          │  (file tailer,     │──▶│  (FastAPI)        │   │  (asyncio server)  │
          │   separate process)│   └─────────┬─────────┘   └──────────┬─────────┘
          └────────────────────┘             │                        │
                                             └───────┬────────────────┘
                                                     ▼
                                      ┌──────────────────────────────┐
                                      │  Ingestion Gateway            │
                                      │  auth · rate limit · envelope │
                                      └──────────────┬───────────────┘
                                                     ▼
                                      ┌──────────────────────────────┐
                                      │  Write-Ahead Log (durability) │
                                      └──────────────┬───────────────┘
                                                     ▼
                                      ┌──────────────────────────────┐
                                      │  Processing Pipeline          │
                                      │  parse → normalize → enrich   │
                                      │  → validate → route (DLQ)     │
                                      └──────────────┬───────────────┘
                                                     ▼
       ┌─────────────────────────────────────────────────────────────────────────┐
       │  Storage Engine                                                          │
       │  segment writer (append-only, block-compressed)                          │
       │  time-bucket index · segment manifest · field summaries · bloom filters  │
       │  retention & compaction workers                                          │
       └───────────────────────────┬─────────────────────────────────────────────┘
                                   ▼
       ┌─────────────────────────────────────────────────────────────────────────┐
       │  Query Engine:  lexer → parser → AST → planner → executor (streaming)   │
       └───────────────────────────┬─────────────────────────────────────────────┘
                                   ▼
       ┌─────────────────────────────────────────────────────────────────────────┐
       │  API Layer (FastAPI): ingest · query · services · stats · health · SSE  │
       └───────────────────────────┬─────────────────────────────────────────────┘
                                   ▼
       ┌─────────────────────────────────────────────────────────────────────────┐
       │  Web Dashboard (React + TS): overview · explorer · services · ingest    │
       └─────────────────────────────────────────────────────────────────────────┘
```

Cross-cutting: **config** (pydantic-settings), **metrics** (Prometheus registry),
**internal logging** (structured JSON, never written into TraceForge's own store), **lifecycle**
(startup/shutdown ordering, graceful drain).

## 3. Module responsibilities

| Module | Package | Responsibility | Must not |
| --- | --- | --- | --- |
| Agent | `tf_agent` | Tail files, track offsets, handle rotation/truncation, ship over TCP/HTTP with retry + local spool | Parse or interpret log content |
| Ingest transports | `traceforge.ingest.http`, `.tcp` | Decode the wire protocol into raw `IngestEnvelope` objects; apply framing and limits | Touch disk layout or parsing rules |
| Gateway | `traceforge.ingest.gateway` | Authenticate, authorize tenant/service, rate-limit, enforce payload limits, hand to WAL | Know which transport delivered the batch |
| WAL | `traceforge.storage.wal` | Durably record raw accepted batches; replay on startup | Interpret record contents |
| Pipeline | `traceforge.processing` | Parser selection, field extraction, normalization, enrichment, validation, DLQ routing | Perform I/O other than via injected sinks |
| Storage engine | `traceforge.storage` | Segment write/read, block compression, index maintenance, manifest, retention, compaction, recovery | Know about TFQL or HTTP |
| Query engine | `traceforge.query` | Lex, parse, plan, execute; pushdown of time and equality predicates | Know about HTTP or auth |
| API | `traceforge.api` | HTTP contract, validation, pagination, errors, SSE live tail | Contain business logic |
| Dashboard | `web/` | Presentation only; all state from the API | Assume storage internals |

**Dependency rule:** `api → query → storage`, `api → ingest → processing → storage`.
Nothing in `storage` imports from `query`, `ingest` or `api`. Enforced by an import-linter test.

## 4. Canonical log record

Normalized records are the pipeline's output and the storage/query unit.

| Field | Type | Notes |
| --- | --- | --- |
| `id` | str (ULID) | Assigned at ingest; monotonic, sortable, dedup key |
| `timestamp` | int64 | Event time, epoch microseconds UTC |
| `ingested_at` | int64 | Receive time, epoch microseconds UTC (clock-skew safety) |
| `level` | enum | TRACE, DEBUG, INFO, WARN, ERROR, FATAL (normalized from aliases) |
| `service` | str | Logical service name; required |
| `host` | str | Hostname/pod |
| `environment` | str | dev/staging/prod |
| `tenant` | str | Multi-tenancy partition key (default `"default"`) |
| `trace_id`, `span_id`, `request_id` | str \| null | Correlation |
| `logger`, `thread` | str \| null | Source context |
| `message` | str | Human-readable body |
| `attributes` | dict[str, str\|int\|float\|bool] | Flattened, dotted keys, depth-limited |
| `raw` | str \| null | Original line, kept when parsing was lossy |

Rules: unknown fields land in `attributes`; attribute keys are flattened (`http.status_code`),
values coerced to scalars; records exceeding configured size are truncated with
`attributes["tf.truncated"] = true` rather than dropped.

## 5. Data flow (write path)

1. Transport receives bytes, applies framing (newline-delimited for TCP, JSON body for HTTP) and a
   hard max-payload limit.
2. Gateway authenticates the API key → resolves `tenant`, checks per-key rate limit and quota.
3. Batch is appended to the **WAL** and `fsync`ed per policy (`always` | `interval` | `never`).
   Only then does the transport acknowledge (HTTP 202 / TCP `ACK <n>`).
4. Batch is pushed to a bounded `asyncio.Queue`. If full, the gateway applies backpressure
   (HTTP 429 with `Retry-After`, TCP read pause). It never silently drops.
5. Pipeline workers parse/normalize/validate. Unparseable or invalid records go to the
   **dead-letter store** with the failure reason, and are counted, never discarded.
6. Segment writer appends records to the active segment for `(tenant, service, time-partition)`,
   accumulating a block in memory until `block_max_records` or `block_max_bytes`, then compresses,
   writes, and updates the in-memory index.
7. On block flush, index and manifest deltas are persisted. On segment roll (size/time/idle),
   the segment is sealed, its footer written, and WAL entries up to that point are truncated.

## 6. Data flow (read path)

1. API validates the query request (TFQL string, time range, pagination cursor, limit).
2. Lexer → parser → AST. Parse errors return 400 with position and expected-token information.
3. **Planner** rewrites the AST into:
   - a **time interval** (from `timestamp` predicates plus the request's range, intersected),
   - **partition filters** (`tenant`, `service`, `environment`, `level` equality/IN sets),
   - **probe predicates** (`trace_id`/`request_id` equality → bloom filter check),
   - a **residual predicate** evaluated per record.
4. Planner consults the manifest: selects candidate segments by time overlap, prunes by field
   summaries and bloom filters, and orders them newest-first (or oldest-first on request).
5. Executor streams blocks, skipping blocks whose `[min_ts, max_ts]` misses the interval, decompresses
   one block at a time, evaluates the residual predicate, and yields matches until `limit`.
6. Response carries an opaque cursor (`segment_id`, `block_index`, `record_index`, direction) for
   stable keyset pagination, never `OFFSET`.

Explain mode (`?explain=true`) returns the plan, segments considered, segments pruned, blocks read and
records scanned. This is the primary tool for proving the index is doing its job.

## 7. Key architectural decisions (and why)

| Decision | Rationale | Rejected alternative |
| --- | --- | --- |
| Append-only, block-compressed segments | Sequential writes maximize throughput; block granularity lets reads decompress only what they need | Per-record compression (poor ratio), per-file compression (no random access) |
| WAL in front of the pipeline | Requirement "failures must not silently lose records"; lets processing be async without risking acknowledged data | Acknowledge after storage write (slow); acknowledge on receipt (lossy) |
| Time partitioning by hour, directory per tenant/service | Most queries are time + service bounded; makes retention a directory/file unlink instead of a rewrite | Single flat file (no pruning, no cheap retention) |
| Separate sparse index files + manifest | Index can be rebuilt from segments; manifest answers pruning questions without opening segments | Index embedded only in segment footers (forces opening every segment) |
| Bloom filters for high-cardinality IDs | `trace_id` lookups are the main debugging workflow and would otherwise scan everything | Full inverted index (large build cost, out of scope) |
| Recursive-descent parser, hand-written | Precedence and error messages are explicit and teachable; no runtime grammar dependency | Parser generator / regex-based filter |
| Agent as a separate process | Keeps the producing host's failure domain separate; agent can spool while the server is down | In-process library (couples apps to TraceForge) |
| FastAPI + asyncio single node | Matches "lightweight"; async fits I/O-bound ingest; workers via uvicorn processes with a single writer per partition | Celery/Kafka dependency (heavy for the stated goal) |
| Single-writer-per-partition | Avoids file-level locking and interleaved writes; recovery is simpler | Multi-writer with locks |

ADRs for each of these are to be written under `docs/adr/`.

## 8. Concurrency and process model

- One **API/ingest process group** (uvicorn, N workers) handles HTTP.
- The **TCP receiver** runs in the same asyncio loop as a worker or as a dedicated process.
- Writes are funnelled to a **single storage writer task per partition**. With multiple uvicorn
  workers, the writer runs in a dedicated process and workers hand off through the WAL + a
  queue (unix socket / shared directory watch). This boundary is defined so multi-worker
  deployment does not corrupt segments.
- Background workers: retention sweeper, compactor, index checkpointer, metrics collector -
  each a supervised task with jittered intervals and crash-restart.
- Shutdown order: stop accepting → drain queue → flush blocks → seal/checkpoint → close WAL.

## 9. Failure modes and mitigations

| Failure | Behaviour | Mitigation |
| --- | --- | --- |
| Process killed mid-block | Uncommitted block lost from segment | WAL replay rebuilds it; CRC detects partial block and truncates to last valid |
| Disk full | Writes fail | Watermark check: at `disk_high_watermark` reject ingest with 507 and emit alert metric; retention runs early |
| Corrupt segment block | Block unreadable | Per-block CRC32; executor skips and counts `corrupt_blocks_total`, query response flags partial results |
| Index/manifest lost | Pruning impossible | `tf-admin rebuild-index` reconstructs from segment footers |
| Clock skew on producer | Out-of-order timestamps | Store both `timestamp` and `ingested_at`; segments record both ranges; configurable max-skew clamp with counter |
| Slow consumer / burst | Queue saturation | Bounded queue + 429 backpressure + agent local spool with retry |
| Unparseable logs | Risk of silent loss | Dead-letter store + `dlq_records_total` metric + DLQ browse endpoint |
| Query fan-out too wide | Memory/CPU spike | Per-query segment/record scan budget, timeout, and `limit` cap; explain exposes cost |
| Compromised API key | Unauthorized ingest/read | Scoped keys (ingest vs read), per-key rate limits, revocation, audit log |

## 10. Performance targets (single node, 4 vCPU / 8 GB / SSD)

These are the numbers the benchmark suite must assert; revise with evidence, not opinion.

| Metric | Target |
| --- | --- |
| HTTP bulk ingest | ≥ 20k records/s at p99 < 100 ms (batch 500, gzip off) |
| TCP line ingest | ≥ 30k lines/s sustained |
| Compression ratio | ≥ 6× on typical app logs (gzip level 6) |
| Point query by `trace_id` over 100M records | < 500 ms with bloom pruning |
| Filtered scan, 1h window, one service | < 1 s for first page (100 records) |
| Memory ceiling | < 512 MB RSS under sustained ingest |
| Recovery time after kill -9 | < 10 s for a 128 MB WAL |
| Data loss on kill -9 after ACK | zero records |

## 11. Security model

- **AuthN:** API keys (hashed at rest, `tf_<env>_<random>`), scoped `ingest` / `query` / `admin`.
  JWT bearer supported for dashboard sessions.
- **AuthZ:** every record carries `tenant`; query execution injects a mandatory tenant predicate that
  no user query can override.
- **Transport:** TLS terminated at the reverse proxy; TCP ingest supports TLS and an allowlist.
- **Input safety:** payload size caps, max records per batch, max attribute depth/count, UTF-8
  sanitisation, no eval of query strings (AST only), rate limits per key and per IP.
- **Secrets:** never in images or compose files; env/file-based, validated at startup.
- **Audit:** admin and auth events to a separate append-only audit log.
- **PII:** optional redaction rules (regex → mask) applied in the pipeline before storage.

## 12. Observability

- `/api/v1/health` (liveness) and `/api/v1/ready` (WAL open, writer alive, disk ok).
- `/metrics` Prometheus: ingest rate, queue depth, WAL fsync latency, block flush latency,
  segment count/bytes, compression ratio, DLQ rate, query latency histogram by plan class,
  records scanned per match (index-efficiency), retention deletions, error counters.
- Internal structured JSON logs with request id; log level configurable; never self-ingested by default.
- Optional OpenTelemetry traces for ingest and query paths.

## 13. Proposed project structure

```
TraceForge/
├── docs/                     # this directory (+ adr/)
├── backend/
│   ├── pyproject.toml
│   └── src/traceforge/
│       ├── config.py         # pydantic-settings, env + yaml
│       ├── models.py         # LogRecord, IngestEnvelope, enums
│       ├── lifecycle.py      # startup/shutdown orchestration
│       ├── ingest/{http,tcp,gateway,ratelimit}.py
│       ├── processing/{pipeline,parsers,normalize,enrich,redact,dlq}.py
│       ├── storage/{wal,segment,block,codec,index,manifest,writer,reader,retention,compaction,recovery}.py
│       ├── query/{lexer,parser,ast,planner,executor,functions,errors}.py
│       ├── api/{app,routes,schemas,errors,pagination,auth,sse}.py
│       ├── observability/{metrics,logging,tracing}.py
│       └── admin/cli.py      # tf-admin: rebuild-index, verify, compact, keys
├── agent/src/tf_agent/       # tailer, offset store, spool, shipper
├── web/                      # React + TS dashboard
├── tests/{unit,integration,property,load}/
├── benchmarks/
├── deploy/{docker,compose,k8s}/
├── examples/spring-boot-integration/
└── scripts/
```

## 14. Open questions

1. Compression: ship gzip for 1.0 and expose a codec interface, or go straight to zstd for ratio/speed?
2. Multi-worker writes: dedicated writer process for 1.0, or document single-worker as the supported
   production topology?
3. Does the dashboard need live tail via SSE in 1.0, or is polling acceptable?
4. Retention by age only, or also by total size with oldest-first eviction? (Recommend both.)
5. Is attribute-level typed indexing (e.g. numeric ranges on `status_code`) in scope for 1.0, or do we
   accept full-block scans for numeric comparisons?

Each is resolved by an ADR before the dependent milestone starts.
