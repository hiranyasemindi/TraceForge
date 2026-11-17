# TraceForge - Milestones & Roadmap

Ten milestones from empty repository to production-deployable 1.0. Each has a hard **exit criteria**
list: the milestone is closed only when every item has automated evidence. Milestones are created on
GitHub with the same titles so issues can be filtered by milestone.

| # | Milestone | Theme | Depends on |
| --- | --- | --- | --- |
| M0 | Foundations & Developer Experience | repo, tooling, CI, contracts | - |
| M1 | Real-Time Log Streaming | agent, TCP/HTTP ingest, backpressure | M0 |
| M2 | Parsing & Storage Engine | pipeline, WAL, segments, recovery | M1 |
| M3 | Indexing, Retention & Compaction | pruning, lifecycle of data | M2 |
| M4 | Custom Query Engine (TFQL) | lexer, parser, planner, executor | M3 |
| M5 | REST API | public contract, pagination, errors | M4 |
| M6 | Web Dashboard | overview, explorer, services, ingest, settings | M5 |
| M7 | Security & Multi-Tenancy | keys, scopes, tenant isolation, limits | M5 |
| M8 | Observability & Performance | metrics, benchmarks, load tests | M6, M7 |
| M9 | Packaging, Deployment & Integration (1.0) | Docker, compose, Spring Boot demo, docs | M8 |

---

## M0 - Foundations & Developer Experience

Scope: `backend/` and `agent/` packaging (uv/pyproject), ruff + mypy strict, pytest with coverage
gate, pre-commit, GitHub Actions (lint, type, test matrix), issue/PR templates, CODEOWNERS,
`CONTRIBUTING.md`, conventional commits, `docs/adr/` with ADR-0001 (storage approach),
canonical `LogRecord` model, config layer, structured internal logging, import-boundary test.

Exit criteria
- [ ] `make check` runs lint + types + tests locally and in CI, green on `main`.
- [ ] mypy strict passes with no `Any` escapes in `models.py` and `config.py`.
- [ ] Coverage gate enforced (≥ 85% on new code) and visible in the PR.
- [ ] Import-boundary test fails if `storage` imports `api`/`query`/`ingest`.
- [ ] ADR-0001 merged; `docs/` published in the repo and linked from `README.md`.

## M1 - Real-Time Log Streaming

Scope: `tf-agent` file tailer (inode-aware rotation, truncation, partial last line, offset
checkpoint file, local spool with retry/backoff), newline-framed TCP protocol with ACKs, asyncio TCP
receiver (per-connection limits, idle timeout, graceful drain), HTTP `POST /logs` + `/logs/bulk`
skeleton, bounded queue with explicit backpressure, DLQ sink, ingest counters.

Exit criteria
- [ ] Tailing survives `logrotate` (rename+create and copytruncate) with zero duplicate or lost lines, asserted by an integration test.
- [ ] Agent spools to disk while the server is down and drains in order on reconnect.
- [ ] Receiver handles 10k connections/s open-close churn and 10 MB lines without unbounded memory.
- [ ] Queue-full returns 429 (HTTP) / pauses reads (TCP); no silent drops, asserted by a saturation test.
- [ ] Soak test: 30 minutes continuous ingest, flat RSS, zero errors.

## M2 - Parsing & Storage Engine

Scope: parser registry (JSON, logfmt, regex-pattern, Spring Boot/logback default, syslog),
normalization (level aliases, timestamp parsing, attribute flattening, truncation), optional PII
redaction, WAL (framing, fsync policy, roll, replay), segment writer (block accumulation,
compression codec interface, gzip), segment reader, footer/seal, checkpointing, crash recovery,
`tf-admin verify`.

Exit criteria
- [ ] Property test: normalize(parse(x)) is total: no input produces an unhandled exception.
- [ ] Round-trip test: 1M synthetic records written and read back byte-identical in field values.
- [ ] Kill-9 matrix (during WAL append, block flush, seal, index write): after recovery, every acknowledged record is present exactly once.
- [ ] Compression ratio ≥ 6× on the sample corpus, asserted in CI.
- [ ] Truncated/corrupted segment is detected by CRC, truncated safely, and counted, never crashing startup.

## M3 - Indexing, Retention & Compaction

Scope: time-bucket index writer/reader, per-segment summaries, bloom filters for `trace_id`/
`request_id`, numeric ranges, manifest with atomic updates and rebuild, retention sweeper (age and
size, per tenant/service), two-phase delete, compactor, disk watermarks, `tf-admin rebuild-index`/
`compact`, storage stats.

Exit criteria
- [ ] Planner-level test proves a 1h query over 24h of data opens only the overlapping segments.
- [ ] Bloom filter false-positive rate measured ≤ 1% at configured parameters.
- [ ] Retention deletes exactly the expired partitions; a crash mid-delete leaves manifest and disk consistent.
- [ ] Compaction of 100 small segments produces identical query results before and after (differential test).
- [ ] Manifest rebuild from segments reproduces the original manifest exactly.

## M4 - Custom Query Engine (TFQL)

Scope: lexer, recursive-descent parser, AST nodes, precedence, parentheses, error reporting with
positions, validation (reserved fields, limits), planner (interval extraction, partition filters,
probes, residual), streaming executor, keyset cursors, scan budget and timeout, `explain`.

Exit criteria
- [ ] Golden-file tests for the full documented operator set, including precedence and implicit AND.
- [ ] Fuzz test: 100k random strings: only `QueryError` ever escapes, no crash, no hang.
- [ ] Planner unit tests assert pruning for each pushdown-capable operator.
- [ ] Memory ceiling test: query matching 5M records returns the first page with < 512 MB RSS.
- [ ] Cursor pagination is stable under concurrent writes: no duplicates, no skipped records.

## M5 - REST API

Scope: all endpoints in `docs/api-contract.md`, pydantic schemas, uniform error model, request ids,
gzip/ndjson ingest, SSE live tail, OpenAPI committed and diffed in CI, integration test suite.

Exit criteria
- [ ] Every documented endpoint has an integration test for success and for each documented error code.
- [ ] OpenAPI diff check in CI; breaking changes fail the build.
- [ ] Pagination, time-range filtering and `explain` verified end-to-end against real segments.
- [ ] 413/429/507 paths exercised deliberately in tests.
- [ ] Postman collection for the API committed and runnable (`postman collection run`) in CI.

## M6 - Web Dashboard

Scope: React + TS + Vite, API client with generated types, Overview, Log Explorer (TFQL bar with
autocomplete and error highlighting, level/service/time filters, result table, expandable detail,
pagination, live tail toggle), Service Explorer, Ingestion Monitoring, Settings, dark-first theme,
empty/error/loading states, accessibility pass.

Exit criteria
- [ ] Types are generated from the committed OpenAPI document, not hand-written.
- [ ] Explorer handles 1000-row pages without jank (virtualized), measured.
- [ ] Every view has defined empty, loading, error and unauthorized states.
- [ ] Playwright e2e: ingest → search → open detail → paginate → filter by service, against a live backend.
- [ ] Keyboard navigation and contrast checks pass; no console errors in the e2e run.

## M7 - Security & Multi-Tenancy

Scope: API key issue/hash/verify/revoke, scopes, JWT dashboard sessions, tenant injection in the
query executor, per-key and per-IP rate limits, payload and attribute limits, TLS for TCP ingest,
IP allowlist, audit log, secret handling, dependency and container scanning, threat model doc.

Exit criteria
- [ ] Test proves a tenant A key cannot read tenant B data through any endpoint, including `trace/{id}` and SSE.
- [ ] A user query containing a `tenant` predicate is rejected, not merged.
- [ ] Rate limits enforced per key and per IP with tests for both.
- [ ] Keys are stored hashed; the plaintext appears exactly once, in the creation response.
- [ ] `pip-audit`/`npm audit` and image scan run in CI; high severity fails the build.
- [ ] Threat model documented with mitigations mapped to tests.

## M8 - Observability & Performance

Scope: Prometheus metrics across ingest/storage/query, `/metrics`, readiness checks, optional
OpenTelemetry, Grafana dashboard JSON, benchmark harness (`benchmarks/`), load tests, regression
thresholds in CI, profiling notes, capacity planning guide.

Exit criteria
- [ ] All targets in `docs/architecture.md#10-performance-targets` measured and recorded in `docs/benchmarks.md`.
- [ ] Nightly benchmark job fails on > 20% regression against the recorded baseline.
- [ ] Index efficiency (`records_scanned / records_matched`) asserted for the canonical query set.
- [ ] 24h soak at 50% of target throughput: no leak, no growth in queue depth, no DLQ growth.
- [ ] Grafana dashboard renders ingest, storage and query health from `/metrics` alone.

## M9 - Packaging, Deployment & Integration (1.0)

Scope: multi-stage Docker images (backend, agent, web) with non-root users and healthchecks, docker
compose (full stack + Prometheus/Grafana profile), versioned config samples, upgrade/migration notes,
backup/restore procedure, Spring Boot example app shipping logs by file and HTTP, quickstart,
operations runbook, release automation with changelog, SBOM.

Exit criteria
- [ ] `docker compose up` yields a working stack with logs flowing and the dashboard reachable, from a clean clone, documented in the quickstart.
- [ ] Spring Boot example demonstrates both file-based (agent) and HTTP ingestion, with its own README.
- [ ] Backup and restore procedure executed in a test and documented.
- [ ] Images are non-root, pinned, scanned, multi-arch, and smaller than 250 MB (backend).
- [ ] Tagged `v1.0.0` release with changelog, SBOM and published images.
- [ ] Runbook covers disk-full, corruption, recovery, key rotation and retention changes.

---

## Sequencing notes

- M7 can start in parallel with M6 once M5 lands; both only depend on the API surface.
- Do not start M4 before M3's manifest exists, because the planner is written against it.
- Benchmarks (M8) get stubbed in M2 so storage work has numbers from day one, but the thresholds are
  only enforced in M8.
- Dashboard work must not fork the API contract: if a view needs data, add the endpoint in M5's
  contract first and regenerate types.
