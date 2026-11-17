# TraceForge - Issue Backlog

Every item below exists as a GitHub issue, assigned to its milestone and labelled by type, area
and priority. Keep this page and the GitHub issues in step: when an issue is added, split or
retired, update it here in the same pull request.

Total: **81 issues** across 10 milestones.

## M0 - Foundations & Developer Experience

Repo layout, tooling, CI, canonical models, config, logging, ADRs. Exit criteria in docs/roadmap.md#m0.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | Bootstrap backend package, tooling and Makefile | type:infra, priority:P0 |
| 2 | Set up GitHub Actions CI with coverage gate | type:infra, priority:P0 |
| 3 | Define canonical LogRecord model and enums | type:feature, area:processing, priority:P0 |
| 4 | Configuration layer with env and YAML overlay | type:feature, priority:P0 |
| 5 | Structured internal logging foundation | type:feature, priority:P1 |
| 6 | Enforce module import boundaries with a test | type:test, priority:P1 |
| 7 | Repo hygiene: templates, CODEOWNERS, CONTRIBUTING | type:docs, priority:P2 |
| 8 | ADR-0001: custom append-only storage engine | type:docs, area:storage, priority:P1 |
| 9 | Resolve architecture open questions as ADRs | type:docs, priority:P1 |

<details><summary>Bootstrap backend package, tooling and Makefile</summary>

Create backend/ and agent/ packages (pyproject, src layout, Python 3.12), ruff, black-compatible formatting, mypy strict, pytest + coverage, pre-commit hooks, and a Makefile exposing fmt/lint/type/test/check.

Acceptance criteria:
- [ ] `make check` runs format check, ruff, mypy strict and pytest
- [ ] Lockfile committed; `pip install -e backend[dev]` works from a clean venv
- [ ] pre-commit installs and blocks a lint error

</details>

<details><summary>Set up GitHub Actions CI with coverage gate</summary>

Workflow running lint, mypy, pytest with coverage on push and PR; cache dependencies; upload coverage artifact; separate job for the web build once it exists.

Acceptance criteria:
- [ ] CI green on main
- [ ] Coverage below the configured threshold fails the build
- [ ] Workflow runs in under 5 minutes on a warm cache

</details>

<details><summary>Define canonical LogRecord model and enums</summary>

Implement the canonical record from docs/architecture.md#4: typed pydantic model, LogLevel enum with severity ordering and alias table, ULID ids, microsecond timestamps, attribute value typing.

Acceptance criteria:
- [ ] Model matches the documented field table exactly
- [ ] Level alias normalization covered by tests (warn/warning/W/30 -> WARN)
- [ ] Serialization round-trip is lossless for every field type

</details>

<details><summary>Configuration layer with env and YAML overlay</summary>

pydantic-settings based config: data dir, fsync policy, block/segment limits, retention, queue sizes, limits, auth, logging. Env overrides YAML; validated at startup with actionable errors; effective config exposed for /admin/config with secrets redacted.

Acceptance criteria:
- [ ] Invalid config fails startup with a message naming the field
- [ ] Every documented setting has a default and a range check
- [ ] Secrets never appear in the dumped config

</details>

<details><summary>Structured internal logging foundation</summary>

JSON structured logging for TraceForge itself with request id propagation, configurable level, and an explicit guard against self-ingestion loops.

Acceptance criteria:
- [ ] All logs are single-line JSON with timestamp, level, logger, message, request_id
- [ ] Log level configurable by env without code change
- [ ] Documented that TraceForge does not ingest its own logs by default

</details>

<details><summary>Enforce module import boundaries with a test</summary>

Add an import-linter (or equivalent AST test) contract encoding the dependency rule: storage must not import query/ingest/api; query must not import api/ingest.

Acceptance criteria:
- [ ] Test fails when a forbidden import is introduced
- [ ] Contract documented in docs/architecture.md#3
- [ ] Runs as part of `make check`

</details>

<details><summary>Repo hygiene: templates, CODEOWNERS, CONTRIBUTING</summary>

Issue templates (bug, feature, chore), PR template with a checklist referencing the readiness checklist, CODEOWNERS, CONTRIBUTING.md with the branching and commit conventions, SECURITY.md.

Acceptance criteria:
- [ ] Templates render correctly on GitHub
- [ ] CONTRIBUTING documents conventional commits and the review rule
- [ ] SECURITY.md states how to report a vulnerability

</details>

<details><summary>ADR-0001: custom append-only storage engine</summary>

Record the decision to build a custom block-compressed append-only engine instead of embedding Elasticsearch/SQLite/Lucene, with the trade-offs and the exit conditions that would reverse it.

Acceptance criteria:
- [ ] ADR follows the template (context, decision, consequences, alternatives)
- [ ] Linked from docs/architecture.md#7
- [ ] Reviewed and merged before M2 starts

</details>

<details><summary>Resolve architecture open questions as ADRs</summary>

Write ADRs for the five open questions in docs/architecture.md#14: compression codec, multi-worker write topology, SSE in 1.0, retention by size, numeric attribute indexing.

Acceptance criteria:
- [ ] One ADR per question, each with a decision and consequences
- [ ] docs/architecture.md#14 updated to point at the ADRs
- [ ] No open question remains unanswered before its dependent milestone

</details>

## M1 - Real-Time Log Streaming

tf-agent file tailing, TCP line protocol, HTTP ingest skeleton, backpressure, DLQ. Exit criteria in docs/roadmap.md#m1.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | tf-agent: file tailer with rotation and truncation handling | type:feature, area:agent, priority:P0 |
| 2 | tf-agent: durable offset checkpoint store | type:feature, area:agent, priority:P0 |
| 3 | tf-agent: shipper with local spool, retry and backoff | type:feature, area:agent, priority:P0 |
| 4 | TCP ingest: line protocol spec and asyncio receiver | type:feature, area:ingest, priority:P0 |
| 5 | HTTP ingest endpoints: /logs and /logs/bulk | type:feature, area:ingest, area:api, priority:P0 |
| 6 | Ingestion gateway: envelope, limits and backpressure | type:feature, area:ingest, priority:P0 |
| 7 | Dead-letter sink for unparseable or invalid records | type:feature, area:processing, priority:P0 |
| 8 | Continuous ingestion soak test harness | type:test, area:ingest, priority:P1 |

<details><summary>tf-agent: file tailer with rotation and truncation handling</summary>

Inode-aware tailer that follows appends, detects rename+create and copytruncate rotation, handles truncation, deletion and re-creation, holds back partial last lines, and supports glob patterns with per-file state.

Acceptance criteria:
- [ ] Integration test covers rename+create and copytruncate with zero duplicates or lost lines
- [ ] Partial final line is never shipped until terminated by a newline
- [ ] Multi-file glob with per-file independent progress

</details>

<details><summary>tf-agent: durable offset checkpoint store</summary>

Persist (path, inode, offset, fingerprint) atomically so a restart resumes exactly where it stopped; fingerprint the first bytes to detect replaced files with reused inodes.

Acceptance criteria:
- [ ] Restart mid-file resumes with no gap and no duplicate
- [ ] Checkpoint write is atomic (tmp+rename) and survives kill -9
- [ ] Stale entries for deleted files are pruned

</details>

<details><summary>tf-agent: shipper with local spool, retry and backoff</summary>

Batch and ship over TCP/HTTP with bounded in-memory batching, exponential backoff with jitter, and a bounded on-disk spool used while the server is unreachable; drain in order on reconnect.

Acceptance criteria:
- [ ] Server down for 5 minutes: nothing is lost, order preserved on drain
- [ ] Spool respects its size cap and reports drops as a counted, logged event
- [ ] Backoff is jittered and capped

</details>

<details><summary>TCP ingest: line protocol spec and asyncio receiver</summary>

Document and implement the newline-framed protocol with batch ACKs, per-connection read limits, max line size, idle timeout, graceful drain on shutdown, and connection metrics.

Acceptance criteria:
- [ ] Protocol documented in docs/api-contract.md or a protocol doc
- [ ] Handles partial lines, oversized lines, binary garbage and abrupt disconnects without leaking connections
- [ ] ACK is only sent after the batch is durable

</details>

<details><summary>HTTP ingest endpoints: /logs and /logs/bulk</summary>

FastAPI endpoints per docs/api-contract.md#2 including ndjson and gzip bodies, partial-success reporting, and the documented limits.

Acceptance criteria:
- [ ] Single, bulk, ndjson and gzip paths tested
- [ ] Partial success returns per-index errors and accepted/rejected counts
- [ ] Payload, batch and record size limits enforced with 413

</details>

<details><summary>Ingestion gateway: envelope, limits and backpressure</summary>

Transport-agnostic gateway producing IngestEnvelope objects: tenant resolution hook, payload limits, bounded queue, and explicit backpressure (429 with Retry-After for HTTP, read pause for TCP).

Acceptance criteria:
- [ ] Saturation test proves no silent drops under sustained overload
- [ ] Queue depth exposed as a metric
- [ ] Backpressure behaviour identical in semantics across transports

</details>

<details><summary>Dead-letter sink for unparseable or invalid records</summary>

Append-only JSONL DLQ with the raw payload, failure reason, transport and timestamp; daily files; bounded by retention; counters per reason.

Acceptance criteria:
- [ ] Nothing is discarded without a DLQ entry and a counter increment
- [ ] DLQ files are rotated daily and subject to retention
- [ ] Reason taxonomy documented

</details>

<details><summary>Continuous ingestion soak test harness</summary>

Load generator producing realistic multi-service logs over file, TCP and HTTP, with configurable rate and duration, plus assertions on delivered counts and resident memory.

Acceptance criteria:
- [ ] 30-minute soak passes with flat RSS and zero errors
- [ ] Delivered count equals produced count
- [ ] Harness is runnable from make and from CI with a short duration

</details>

## M2 - Parsing & Storage Engine

Parser registry, normalization, WAL, segments, compression, recovery. Exit criteria in docs/roadmap.md#m2.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | Parser registry and JSON parser | type:feature, area:processing, priority:P0 |
| 2 | Text parsers: logfmt, regex pattern, logback, syslog | type:feature, area:processing, priority:P1 |
| 3 | Normalization: levels, timestamps, attributes, truncation | type:feature, area:processing, priority:P0 |
| 4 | Optional PII redaction stage | type:feature, area:processing, type:security, priority:P2 |
| 5 | Write-ahead log: framing, fsync policy and rolling | type:feature, area:storage, priority:P0 |
| 6 | WAL replay and checkpointing | type:feature, area:storage, priority:P0 |
| 7 | Block codec interface with gzip implementation | type:feature, area:storage, priority:P0 |
| 8 | Segment writer: block accumulation, roll and seal | type:feature, area:storage, priority:P0 |
| 9 | Segment reader with CRC verification and corruption isolation | type:feature, area:storage, priority:P0 |
| 10 | Crash recovery and kill-9 durability test matrix | type:test, area:storage, priority:P0 |
| 11 | tf-admin verify command | type:feature, area:storage, area:ops, priority:P1 |

<details><summary>Parser registry and JSON parser</summary>

Pluggable parser interface with per-service configuration and auto-detection; first implementation parses structured JSON logs including nested objects.

Acceptance criteria:
- [ ] Registry selects a parser by service config, then by sniffing, then falls back to raw
- [ ] JSON parser maps known keys to canonical fields and the rest to attributes
- [ ] Adding a parser requires no change to the pipeline

</details>

<details><summary>Text parsers: logfmt, regex pattern, logback, syslog</summary>

Configurable parsers for conventional text logs, including a user-supplied named-group regex parser and presets for Spring Boot/logback and RFC5424 syslog.

Acceptance criteria:
- [ ] Fixture-driven tests per parser with real-world sample lines
- [ ] Unmatched lines fall back to raw with the original line preserved
- [ ] Regex parser rejects catastrophic patterns at config time

</details>

<details><summary>Normalization: levels, timestamps, attributes, truncation</summary>

Normalize levels via the alias table, parse all documented timestamp forms, clamp clock skew, flatten attributes with depth and count limits, coerce values, and truncate oversized records with a marker instead of dropping them.

Acceptance criteria:
- [ ] Property test: normalization never raises on arbitrary input
- [ ] Skew clamping is counted and documented
- [ ] Oversized records are stored truncated and flagged

</details>

<details><summary>Optional PII redaction stage</summary>

Configurable regex-to-mask rules applied before storage, with named built-ins (email, card, bearer token) and a metric for redactions applied.

Acceptance criteria:
- [ ] Redaction happens before anything is written to disk
- [ ] Rules are configurable per tenant/service
- [ ] Disabled by default with documented performance cost

</details>

<details><summary>Write-ahead log: framing, fsync policy and rolling</summary>

Implement the WAL frame from docs/storage-format.md#2 with CRC32C, monotonic sequence numbers, configurable fsync policy and size-based rolling.

Acceptance criteria:
- [ ] Frames match the documented layout byte for byte
- [ ] fsync policy always/interval/never implemented with measured latency metrics
- [ ] Roll at the configured size with no gap in sequence numbers

</details>

<details><summary>WAL replay and checkpointing</summary>

Replay from checkpoint.json on startup, truncate a torn trailing frame, dedup by record id within the replay window, and delete WAL files only when fully applied.

Acceptance criteria:
- [ ] Replay is idempotent: running it twice changes nothing
- [ ] Torn trailing frame is truncated and counted, not fatal
- [ ] A WAL file is never deleted before its records are durable in a segment

</details>

<details><summary>Block codec interface with gzip implementation</summary>

Codec abstraction (id, compress, decompress) with per-block codec ids so codecs can change without rewriting history; gzip first, zstd behind the same interface.

Acceptance criteria:
- [ ] Codec id stored per block and honoured on read
- [ ] Compression ratio measured on the sample corpus and asserted >= 6x
- [ ] Swapping the codec requires no reader change

</details>

<details><summary>Segment writer: block accumulation, roll and seal</summary>

Append records into the active segment per (tenant, service, hour), flush blocks at the configured record/byte/idle thresholds, roll segments on size or partition boundary, and write the footer on seal.

Acceptance criteria:
- [ ] Block and segment layout match docs/storage-format.md#3
- [ ] Idle flush keeps low-traffic services queryable within the configured delay
- [ ] Single-writer-per-partition invariant enforced and tested

</details>

<details><summary>Segment reader with CRC verification and corruption isolation</summary>

Streaming reader over blocks with CRC verification, support for unsealed segments (header scan), and corruption confined to the affected block with a counter and a partial-result flag.

Acceptance criteria:
- [ ] A deliberately corrupted block is skipped, counted and surfaced as partial
- [ ] Unsealed active segment is readable
- [ ] Reader decompresses only the blocks whose time range overlaps the request

</details>

<details><summary>Crash recovery and kill-9 durability test matrix</summary>

Implement the startup recovery algorithm (docs/storage-format.md#8) and a test matrix killing the process during WAL append, block flush, seal and index write.

Acceptance criteria:
- [ ] For every kill point, every acknowledged record is present exactly once after recovery
- [ ] Recovery duration and replayed-record metrics exposed
- [ ] Recovery of a 128 MB WAL completes in under 10 seconds

</details>

<details><summary>tf-admin verify command</summary>

CLI that walks every segment, verifies CRCs and footers, cross-checks the manifest, and reports inconsistencies with a non-zero exit code.

Acceptance criteria:
- [ ] Runs against a generated corpus in CI and exits 0
- [ ] Detects an injected corruption and reports the segment, block and reason
- [ ] Documented in the operations runbook

</details>

## M3 - Indexing, Retention & Compaction

Time-bucket index, summaries, bloom filters, manifest, retention, compaction. Exit criteria in docs/roadmap.md#m3.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | Time-bucket sparse index writer and reader | type:feature, area:storage, priority:P0 |
| 2 | Per-segment summaries for pruning (.tfm) | type:feature, area:storage, priority:P0 |
| 3 | Bloom filters for trace_id and request_id | type:feature, area:storage, priority:P1 |
| 4 | Manifest: atomic updates, in-memory catalog and rebuild | type:feature, area:storage, priority:P0 |
| 5 | Retention sweeper: age and size based | type:feature, area:storage, priority:P0 |
| 6 | Segment compaction worker | type:feature, area:storage, priority:P1 |
| 7 | Disk watermark enforcement and storage stats | type:feature, area:storage, area:ops, priority:P1 |
| 8 | tf-admin rebuild-index and compact commands | type:feature, area:ops, area:storage, priority:P1 |

<details><summary>Time-bucket sparse index writer and reader</summary>

Write and read the .tfi index from docs/storage-format.md#4; resolve a time range by binary search to a contiguous block range.

Acceptance criteria:
- [ ] Index resolves ranges without scanning block headers
- [ ] Index is rebuildable from the segment and never the system of record
- [ ] Benchmark shows fewer blocks read than a header scan on the same query

</details>

<details><summary>Per-segment summaries for pruning (.tfm)</summary>

Collect and persist level/environment/host sets, attribute key sets and numeric ranges per segment, used by the planner before any block is opened.

Acceptance criteria:
- [ ] Summary written on seal and loaded lazily with caching
- [ ] Planner skips a segment whose summary excludes the predicate
- [ ] Summary rebuildable by tf-admin rebuild-index

</details>

<details><summary>Bloom filters for trace_id and request_id</summary>

Per-segment bloom filters with configurable target false-positive rate, probed by the planner for equality predicates on high-cardinality ids.

Acceptance criteria:
- [ ] Measured false-positive rate within the configured target
- [ ] trace_id lookup over a large corpus opens only candidate segments
- [ ] Parameters (m, k, n) recorded in the .tfm for verification

</details>

<details><summary>Manifest: atomic updates, in-memory catalog and rebuild</summary>

Implement the manifest from docs/storage-format.md#6: atomic tmp+fsync+rename with a .bak, an in-memory catalog refreshed on change, and a full rebuild from segment footers.

Acceptance criteria:
- [ ] Planner never walks the filesystem on the query path
- [ ] Crash during a manifest update leaves a loadable manifest
- [ ] Rebuild reproduces the original manifest exactly

</details>

<details><summary>Retention sweeper: age and size based</summary>

Background sweeper deleting expired partitions by age and evicting oldest-first when a size budget is exceeded, per tenant/service, using a two-phase delete.

Acceptance criteria:
- [ ] Exactly the expired partitions are deleted
- [ ] Crash mid-delete leaves manifest and disk consistent
- [ ] Deletions counted and bytes reclaimed reported as metrics

</details>

<details><summary>Segment compaction worker</summary>

Merge many small segments within a closed hour partition into one re-blocked segment, swapping manifest entries atomically before unlinking sources.

Acceptance criteria:
- [ ] Differential test: identical query results before and after compaction
- [ ] Killing the compactor at any point loses no data and leaves no orphan entries
- [ ] Compaction is throttled and yields to ingest

</details>

<details><summary>Disk watermark enforcement and storage stats</summary>

Track disk usage, warn at the soft watermark, reject ingest with 507 at the hard watermark, trigger early retention, and expose storage statistics for the API.

Acceptance criteria:
- [ ] 507 returned at the hard watermark and recovery is automatic once space is reclaimed
- [ ] Soft watermark raises a metric suitable for alerting
- [ ] Stats expose segment count, raw/stored bytes, compression ratio and oldest record

</details>

<details><summary>tf-admin rebuild-index and compact commands</summary>

Operator commands to rebuild indexes, summaries and the manifest from segments, and to force compaction of a partition, both safe to run on a live system.

Acceptance criteria:
- [ ] Rebuild restores a deleted index and manifest with identical query results
- [ ] Commands are safe to interrupt
- [ ] Documented in the runbook with expected durations

</details>

## M4 - Custom Query Engine (TFQL)

Lexer, parser, AST, planner, streaming executor, cursors, explain. Exit criteria in docs/roadmap.md#m4.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | TFQL lexer | type:feature, area:query, priority:P0 |
| 2 | TFQL recursive-descent parser and AST | type:feature, area:query, priority:P0 |
| 3 | Query validation and DoS guards | type:feature, area:query, type:security, priority:P1 |
| 4 | Query planner: interval extraction and pushdown | type:feature, area:query, priority:P0 |
| 5 | Streaming executor with scan budget and timeout | type:feature, area:query, priority:P0 |
| 6 | Keyset pagination cursors | type:feature, area:query, area:api, priority:P0 |
| 7 | Query explain output | type:feature, area:query, priority:P1 |
| 8 | TFQL fuzz and golden test suites | type:test, area:query, priority:P1 |

<details><summary>TFQL lexer</summary>

Tokenizer for the grammar in docs/query-language.md: identifiers, dotted fields, strings with escapes, numbers, durations, booleans, operators, keywords (case-insensitive), parentheses, with token positions for error reporting.

Acceptance criteria:
- [ ] Every documented token type covered by tests
- [ ] Unterminated string and unknown character produce positioned errors
- [ ] Reserved tokens for post-1.0 syntax are recognised and rejected clearly

</details>

<details><summary>TFQL recursive-descent parser and AST</summary>

Parser producing an immutable AST with the documented precedence (comparison > NOT > AND > OR), parentheses, implicit AND, IN, BETWEEN and EXISTS.

Acceptance criteria:
- [ ] Golden tests for every example query in docs/query-language.md
- [ ] Precedence and implicit AND asserted explicitly
- [ ] Parse errors report position and expected token

</details>

<details><summary>Query validation and DoS guards</summary>

Validate the AST: reject the reserved tenant field, cap query length, AST node count, regex length and IN-list size, and pre-compile regexes with a bounded cache.

Acceptance criteria:
- [ ] Reserved-field predicate is rejected with a clear error
- [ ] All caps are configurable and tested at the boundary
- [ ] Catastrophic regex is rejected or bounded by a timeout

</details>

<details><summary>Query planner: interval extraction and pushdown</summary>

Split the AST into (interval, partition filters, probes, residual); intersect timestamp bounds with the request range; short-circuit unsatisfiable intervals; choose segment order.

Acceptance criteria:
- [ ] Planner is pure and unit-tested: AST in, plan out
- [ ] Pushdown asserted for =, !=, IN, BETWEEN, EXISTS and timestamp comparisons
- [ ] Unsatisfiable interval returns empty without disk access

</details>

<details><summary>Streaming executor with scan budget and timeout</summary>

Generator-based execution over segments, buckets, blocks and records with a per-query record-scan budget, wall-clock timeout, and partial-result reporting.

Acceptance criteria:
- [ ] Query matching 5M records returns the first page under the documented memory ceiling
- [ ] Budget and timeout produce partial:true with a warning, not an error
- [ ] Concurrent queries do not block ingest

</details>

<details><summary>Keyset pagination cursors</summary>

Opaque, signed cursor encoding (segment, block, record, direction) with stable ordering in both directions and validation of stale cursors.

Acceptance criteria:
- [ ] No duplicates or skipped records while writes continue during paging
- [ ] Tampered or stale cursor returns 400 with a clear reason
- [ ] Cursor survives segment compaction or fails cleanly with a documented behaviour

</details>

<details><summary>Query explain output</summary>

Produce the explain structure from docs/query-language.md#9: plan, segments considered/pruned, blocks read, records scanned/matched, duration.

Acceptance criteria:
- [ ] Explain available on the query endpoint behind a flag
- [ ] Numbers verified against instrumented counters in tests
- [ ] Index efficiency (scanned/matched) asserted for the canonical query set

</details>

<details><summary>TFQL fuzz and golden test suites</summary>

Property/fuzz testing over random and mutated query strings plus a golden corpus of query/result pairs over a fixed dataset.

Acceptance criteria:
- [ ] 100k fuzz inputs: only QueryError escapes, no crash or hang
- [ ] Golden results regenerate deterministically from the fixture dataset
- [ ] Suite runs in CI in under two minutes

</details>

## M5 - REST API

Full documented REST surface, error model, pagination, SSE, OpenAPI contract checks. Exit criteria in docs/roadmap.md#m5.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | API app skeleton: error envelope, request ids, middleware | type:feature, area:api, priority:P0 |
| 2 | Query endpoints: /query, /logs/{id}, /trace/{trace_id} | type:feature, area:api, priority:P0 |
| 3 | Metadata endpoints: /services and /fields | type:feature, area:api, priority:P1 |
| 4 | Stats endpoints: overview, ingest, storage, timeseries | type:feature, area:api, priority:P1 |
| 5 | SSE live tail endpoint | type:feature, area:api, priority:P2 |
| 6 | Health and readiness endpoints | type:feature, area:api, area:ops, priority:P0 |
| 7 | Commit OpenAPI document and add a CI diff check | type:infra, area:api, priority:P1 |
| 8 | Postman collection for the API, run in CI | type:test, area:api, priority:P2 |
| 9 | API integration test suite | type:test, area:api, priority:P0 |

<details><summary>API app skeleton: error envelope, request ids, middleware</summary>

FastAPI app factory with the uniform error envelope from docs/api-contract.md#8, X-Request-ID propagation, exception handlers, timeouts, body size limits and CORS configuration.

Acceptance criteria:
- [ ] Every 4xx/5xx response uses the documented envelope and carries a request id
- [ ] No stack trace or internal path reaches a client
- [ ] Error counters labelled by status code

</details>

<details><summary>Query endpoints: /query, /logs/{id}, /trace/{trace_id}</summary>

Implement POST/GET query with TFQL, time range, limit, cursor, order and explain; single record fetch with context; cross-service trace view.

Acceptance criteria:
- [ ] Contract matches docs/api-contract.md#3 exactly
- [ ] Integration tests cover success plus 400/404/429 paths
- [ ] GET form produces shareable URLs used by the dashboard

</details>

<details><summary>Metadata endpoints: /services and /fields</summary>

Service catalog with first/last seen, counts and hosts; per-service detail; known field and attribute keys with inferred types for dashboard autocomplete.

Acceptance criteria:
- [ ] Data derived from the manifest and summaries, not a full scan
- [ ] Response time under 100 ms with 10k segments
- [ ] Autocomplete consumes /fields in the dashboard

</details>

<details><summary>Stats endpoints: overview, ingest, storage, timeseries</summary>

Aggregated counters and windowed rates for the dashboard, plus a grouped histogram endpoint for the volume chart.

Acceptance criteria:
- [ ] Rates for 1m/5m/15m windows are correct under a synthetic load
- [ ] Timeseries supports interval and group_by with bounded bucket counts
- [ ] All values sourced from maintained counters, not recomputed scans

</details>

<details><summary>SSE live tail endpoint</summary>

Server-sent events stream filtered by a TFQL AST evaluated on the write path, with heartbeats, a concurrency cap and clean disconnect handling.

Acceptance criteria:
- [ ] Client disconnect releases resources immediately
- [ ] Concurrency cap enforced with a documented status when exceeded
- [ ] Filter semantics identical to the query path

</details>

<details><summary>Health and readiness endpoints</summary>

Liveness that never fails under load, and readiness checking WAL open, writer alive and disk below watermark, with per-check detail and 503 during shutdown.

Acceptance criteria:
- [ ] Readiness turns 503 before the process stops accepting traffic
- [ ] Each check reported individually
- [ ] Used by the container healthchecks

</details>

<details><summary>Commit OpenAPI document and add a CI diff check</summary>

Generate and commit the OpenAPI document; CI regenerates and fails on an undeclared diff, with a documented approval path for intentional changes.

Acceptance criteria:
- [ ] CI fails on an accidental contract change
- [ ] Intentional changes require the committed document to be updated in the same PR
- [ ] Document is the source for the dashboard's generated types

</details>

<details><summary>Postman collection for the API, run in CI</summary>

Author a Postman collection covering ingest, query, metadata, stats and health with assertions and environment variables, run headlessly in CI against a disposable instance.

Acceptance criteria:
- [ ] Collection runs green against a freshly started stack
- [ ] Secrets come from environment variables, never committed
- [ ] Failure output identifies the failing request and assertion

</details>

<details><summary>API integration test suite</summary>

End-to-end tests against a real instance with real storage: ingest, query, paginate, filter, explain, plus every documented error status.

Acceptance criteria:
- [ ] Every endpoint has a success and an error test
- [ ] 413, 429, 503 and 507 are exercised deliberately
- [ ] Suite runs in CI against a temporary data directory

</details>

## M6 - Web Dashboard

React + TS dashboard: overview, explorer, services, ingestion, settings, e2e. Exit criteria in docs/roadmap.md#m6.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | Scaffold React + TypeScript dashboard with generated API types | type:infra, area:web, priority:P0 |
| 2 | Overview dashboard view | type:feature, area:web, priority:P1 |
| 3 | Log explorer with TFQL search bar | type:feature, area:web, priority:P0 |
| 4 | Service explorer view | type:feature, area:web, priority:P1 |
| 5 | Ingestion monitoring view | type:feature, area:web, priority:P1 |
| 6 | System settings view | type:feature, area:web, priority:P2 |
| 7 | Playwright e2e suite and accessibility pass | type:test, area:web, priority:P1 |

<details><summary>Scaffold React + TypeScript dashboard with generated API types</summary>

Vite + React + TS + component library, routing, query client, theme (dark-first), lint/format, and API types generated from the committed OpenAPI document.

Acceptance criteria:
- [ ] No hand-written API types; generation runs in CI
- [ ] Production build output and bundle size recorded
- [ ] Lint, typecheck and build wired into CI

</details>

<details><summary>Overview dashboard view</summary>

Totals, error and warning counts, logs/sec, service statistics and recent activity, with a volume chart and auto-refresh.

Acceptance criteria:
- [ ] All data from the stats endpoints
- [ ] Empty, loading, error and unauthorized states implemented
- [ ] Refresh interval configurable and paused when the tab is hidden

</details>

<details><summary>Log explorer with TFQL search bar</summary>

Search bar with TFQL autocomplete and syntax error highlighting, level/service/time filters, virtualized result table, expandable record detail, keyset pagination, query history and shareable URLs.

Acceptance criteria:
- [ ] 1000-row pages scroll without jank (measured)
- [ ] Parse errors are highlighted at the reported position
- [ ] Filters and query state are reflected in the URL

</details>

<details><summary>Service explorer view</summary>

List of connected services with per-service statistics, level breakdown, recent errors and a link into a pre-filtered explorer query.

Acceptance criteria:
- [ ] Data from /services and /services/{name}
- [ ] Recent errors link to the record detail
- [ ] Handles hundreds of services without layout breakage

</details>

<details><summary>Ingestion monitoring view</summary>

Incoming volume, throughput by transport, queue depth, DLQ rate, processing status and storage utilisation.

Acceptance criteria:
- [ ] Charts driven by /stats/ingest and /stats/storage
- [ ] Watermark and DLQ growth are visually obvious
- [ ] Explains backpressure state in plain language

</details>

<details><summary>System settings view</summary>

Read effective storage, retention and ingestion configuration; edit retention policy; manage API keys where permitted by scope.

Acceptance criteria:
- [ ] Secrets never rendered
- [ ] Scope-gated controls hidden or disabled for unprivileged sessions
- [ ] Changes confirmed and reflected after save

</details>

<details><summary>Playwright e2e suite and accessibility pass</summary>

End-to-end flow against a live backend: ingest, search, open detail, paginate, filter by service, plus keyboard navigation and contrast checks.

Acceptance criteria:
- [ ] e2e runs in CI against the compose stack
- [ ] No console errors during the run
- [ ] Keyboard-only navigation completes the core flow

</details>

## M7 - Security & Multi-Tenancy

API keys, scopes, tenant isolation, rate limits, TLS, audit log, threat model. Exit criteria in docs/roadmap.md#m7.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | API key issuance, hashing and revocation | type:feature, type:security, area:api, priority:P0 |
| 2 | Scope-based authorization across endpoints | type:feature, type:security, area:api, priority:P0 |
| 3 | Tenant isolation in the query executor | type:feature, type:security, area:query, priority:P0 |
| 4 | Rate limiting per API key and per IP | type:feature, type:security, area:ingest, priority:P1 |
| 5 | TLS and IP allowlist for TCP ingest | type:feature, type:security, area:ingest, priority:P1 |
| 6 | Append-only audit log for auth and admin events | type:feature, type:security, area:ops, priority:P2 |
| 7 | Dependency and container scanning plus threat model | type:security, type:infra, priority:P1 |

<details><summary>API key issuance, hashing and revocation</summary>

Key lifecycle: generate prefixed keys, store only a strong hash, return plaintext exactly once, list metadata, revoke immediately, and support rotation.

Acceptance criteria:
- [ ] Plaintext key is unrecoverable from storage
- [ ] Revocation takes effect on the next request
- [ ] Admin endpoints covered by tests, including 404 and 409 paths

</details>

<details><summary>Scope-based authorization across endpoints</summary>

Enforce ingest/query/admin scopes per route, with 401 versus 403 distinguished and health left unauthenticated.

Acceptance criteria:
- [ ] A matrix test asserts allowed and denied routes for each scope
- [ ] Health and readiness remain unauthenticated
- [ ] Dashboard JWT sessions map to scopes

</details>

<details><summary>Tenant isolation in the query executor</summary>

Inject a mandatory tenant predicate at execution, reject user predicates on the reserved field, and partition data by tenant on disk.

Acceptance criteria:
- [ ] Cross-tenant test proves tenant A cannot read tenant B through any read path, including SSE and /trace/{id}
- [ ] A user query mentioning tenant is rejected, never merged
- [ ] Isolation covered on the ingest path too

</details>

<details><summary>Rate limiting per API key and per IP</summary>

Token-bucket limits for ingest and query, configurable per key and per IP, returning 429 with Retry-After and counted by metrics.

Acceptance criteria:
- [ ] Limits enforced for both dimensions with tests
- [ ] Limits configurable without a restart where documented
- [ ] Backpressure and rate limiting are distinguishable in metrics and responses

</details>

<details><summary>TLS and IP allowlist for TCP ingest</summary>

Optional TLS for the TCP receiver with certificate configuration, plus an IP allowlist, and documentation that plaintext is not a production default.

Acceptance criteria:
- [ ] TLS handshake tested with a self-signed certificate fixture
- [ ] Allowlist rejection is counted and logged
- [ ] Deployment docs show the TLS-enabled configuration

</details>

<details><summary>Append-only audit log for auth and admin events</summary>

Record authentication outcomes, key lifecycle, retention changes and maintenance jobs to a separate append-only log with actor, action, target and outcome.

Acceptance criteria:
- [ ] Audit entries are separate from application logs and from stored log data
- [ ] Entries are append-only and rotated under their own retention
- [ ] Documented fields and retention

</details>

<details><summary>Dependency and container scanning plus threat model</summary>

Add pip-audit and npm audit to CI, image scanning for published images, and docs/threat-model.md mapping each threat to a mitigation and a test.

Acceptance criteria:
- [ ] High severity findings fail the build with a documented exception process
- [ ] Threat model covers ingest spoofing, key compromise, tenant escape, DoS and data at rest
- [ ] Each mitigation names the test that proves it

</details>

## M8 - Observability & Performance

Metrics, benchmarks, regression gates, soak tests, dashboards, capacity guide. Exit criteria in docs/roadmap.md#m8.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | Prometheus metrics across ingest, storage and query | type:feature, area:ops, priority:P0 |
| 2 | Benchmark harness and recorded baselines | type:perf, priority:P0 |
| 3 | Nightly benchmark regression job | type:perf, type:infra, priority:P1 |
| 4 | 24h soak and burst load tests | type:test, type:perf, priority:P1 |
| 5 | Grafana dashboard and alert rules | type:docs, area:ops, priority:P2 |
| 6 | Capacity planning guide and profiling notes | type:docs, area:ops, priority:P2 |

<details><summary>Prometheus metrics across ingest, storage and query</summary>

Instrument the documented metric set (docs/architecture.md#12) and expose /metrics, including latency histograms, queue depth, WAL fsync latency, compression ratio and index efficiency.

Acceptance criteria:
- [ ] Every metric in the architecture doc exists with the documented labels
- [ ] Cardinality reviewed: no unbounded label values
- [ ] /metrics scrape adds negligible overhead under load

</details>

<details><summary>Benchmark harness and recorded baselines</summary>

benchmarks/ suite measuring every target in docs/architecture.md#10 with a fixed corpus and reproducible procedure; record results in docs/benchmarks.md with hardware details.

Acceptance criteria:
- [ ] All documented targets measured and recorded
- [ ] Benchmarks reproducible from a documented command
- [ ] Results include hardware, dataset and configuration

</details>

<details><summary>Nightly benchmark regression job</summary>

Scheduled workflow running the benchmark suite and failing on a regression beyond the configured threshold against the recorded baseline.

Acceptance criteria:
- [ ] Job fails on a deliberate 30% slowdown
- [ ] Baseline updates require an explicit commit
- [ ] Results published as an artifact with trend history

</details>

<details><summary>24h soak and burst load tests</summary>

Long-running soak at half the target rate plus a 5x burst test, asserting flat memory, stable queue depth, no DLQ growth and backpressure instead of loss.

Acceptance criteria:
- [ ] Soak completes with no leak and no error growth
- [ ] Burst degrades through backpressure with zero acknowledged-record loss
- [ ] Restart under load loses no acknowledged record

</details>

<details><summary>Grafana dashboard and alert rules</summary>

Commit a Grafana dashboard JSON covering ingest, storage and query health, and document alert rules for disk watermark, DLQ rate, queue depth, error rate and recovery events.

Acceptance criteria:
- [ ] Dashboard renders from /metrics alone
- [ ] Alert rules documented with thresholds and rationale
- [ ] Imported and verified against the compose monitoring profile

</details>

<details><summary>Capacity planning guide and profiling notes</summary>

Document bytes per record, disk per day per service, memory per partition, CPU per 10k records/s, and how to profile ingest and query hot paths.

Acceptance criteria:
- [ ] Guide includes a worked example sizing a 50-service deployment
- [ ] Numbers traceable to recorded benchmarks
- [ ] Profiling procedure reproducible by a new contributor

</details>

## M9 - Packaging, Deployment & Integration (1.0)

Docker, compose, config reference, backup/restore, Spring Boot demo, runbook, release. Exit criteria in docs/roadmap.md#m9.

| # | Issue | Labels |
| --- | --- | --- |
| 1 | Docker images for backend, agent and web | type:infra, area:ops, priority:P0 |
| 2 | Docker Compose stack with monitoring profile | type:infra, area:ops, priority:P0 |
| 3 | Configuration reference documentation | type:docs, area:ops, priority:P1 |
| 4 | Backup and restore procedure | type:docs, area:ops, priority:P1 |
| 5 | Spring Boot example integration | type:docs, area:ops, priority:P1 |
| 6 | Quickstart and operations runbook | type:docs, area:ops, priority:P0 |
| 7 | Release automation, changelog and SBOM | type:infra, area:ops, priority:P1 |
| 8 | Upgrade path and storage format migration notes | type:docs, area:storage, priority:P1 |

<details><summary>Docker images for backend, agent and web</summary>

Multi-stage, non-root, pinned, multi-arch images with healthchecks, minimal runtime layers and a documented size budget.

Acceptance criteria:
- [ ] Backend image under 250 MB and non-root
- [ ] Healthchecks use /ready and fail correctly when the writer is down
- [ ] Images build for amd64 and arm64 in CI

</details>

<details><summary>Docker Compose stack with monitoring profile</summary>

Compose file bringing up backend, web and agent with named volumes, plus an optional profile adding Prometheus and Grafana.

Acceptance criteria:
- [ ] `docker compose up` from a clean clone yields a working stack with logs flowing
- [ ] Data survives container replacement
- [ ] Monitoring profile is opt-in and documented

</details>

<details><summary>Configuration reference documentation</summary>

Document every setting: purpose, default, range, reload behaviour and production guidance, generated from the config model where possible.

Acceptance criteria:
- [ ] Every config field documented; a test asserts no field is missing from the reference
- [ ] Production-recommended values called out
- [ ] Secrets handling documented

</details>

<details><summary>Backup and restore procedure</summary>

Define and execute a backup strategy for segments, manifest, WAL and keys, including a consistent snapshot method and a verified restore.

Acceptance criteria:
- [ ] Restore executed in a test and documented step by step
- [ ] Procedure states what a backup taken mid-write guarantees
- [ ] tf-admin verify run as part of restore validation

</details>

<details><summary>Spring Boot example integration</summary>

Example application in examples/spring-boot-integration shipping logs both by file (via tf-agent) and by HTTP appender, with its own README and compose wiring.

Acceptance criteria:
- [ ] Both ingestion paths demonstrated end to end
- [ ] Example requires no TraceForge library on the application side
- [ ] README walks through the demo in under 10 minutes

</details>

<details><summary>Quickstart and operations runbook</summary>

Quickstart proven on a fresh machine, plus a runbook covering disk full, corruption, recovery, key rotation, retention change, hot service and upgrade.

Acceptance criteria:
- [ ] Quickstart validated by someone other than the author
- [ ] Each runbook scenario has concrete commands and expected output
- [ ] Linked from README.md and docs/README.md

</details>

<details><summary>Release automation, changelog and SBOM</summary>

Tag-driven release workflow publishing images, generating a changelog from conventional commits and attaching an SBOM.

Acceptance criteria:
- [ ] v1.0.0 release produced by the workflow, not by hand
- [ ] Changelog generated and reviewed
- [ ] SBOM attached for backend, agent and web images

</details>

<details><summary>Upgrade path and storage format migration notes</summary>

Document the storage version policy, the supported upgrade path, and the migration procedure including rollback expectations.

Acceptance criteria:
- [ ] Reader rejects a newer format version loudly, with a test
- [ ] Upgrade procedure documented with a dry run
- [ ] Rollback limitations stated explicitly

</details>
