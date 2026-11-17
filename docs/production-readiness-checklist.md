# TraceForge - Production Readiness Checklist

The gate for `v1.0.0`. An item is checked only when there is **automated evidence** (a test, a CI
job, a recorded benchmark) or a documented, executed procedure. "It works on my machine" is not
evidence. Each group names the milestone that owns it.

## 1. Project & process (M0)
- [ ] Repository layout matches `docs/architecture.md#13-proposed-project-structure`
- [ ] Dependency management pinned and lockfiles committed (backend + web + agent)
- [ ] `make check` = lint + format + types + tests, identical locally and in CI
- [ ] CI on every PR: lint, mypy strict, pytest, coverage gate, web build, e2e
- [ ] Branch protection on `main`: green CI + review required
- [ ] Conventional commits and automated changelog
- [ ] Issue/PR templates, CODEOWNERS, `CONTRIBUTING.md`
- [ ] ADRs recorded for every decision in the architecture decision table
- [ ] Open questions in `docs/architecture.md#14-open-questions` all resolved by an ADR

## 2. Architecture & code quality
- [ ] Module boundaries enforced by an import-linter test
- [ ] Public functions and classes typed and docstringed
- [ ] No module over ~400 lines without justification; no circular imports
- [ ] Interfaces (codec, parser, sink, index, auth backend) are swappable and have a second implementation or a fake
- [ ] Dead code and TODOs triaged into issues before release

## 3. Ingestion (M1)
- [ ] HTTP single, bulk, ndjson and gzip paths tested
- [ ] TCP framing handles partial lines, oversized lines, binary garbage, abrupt disconnects
- [ ] Agent handles rotation (rename+create and copytruncate), truncation, deletion, permission loss
- [ ] Agent offset checkpoint survives restart with no duplicates or gaps
- [ ] Agent spools locally with bounded disk use and drains in order
- [ ] Backpressure: queue-full produces 429 / read pause, never a silent drop
- [ ] Payload, batch-size and per-record limits enforced and tested
- [ ] Malformed input routed to DLQ with reason, counted, and replayable

## 4. Processing (M2)
- [ ] Parsers for JSON, logfmt, regex, logback, syslog with fixtures
- [ ] Level alias normalization table covered by tests
- [ ] Timestamp parsing covers ISO-8601 variants, epoch s/ms/us, missing and future timestamps
- [ ] Clock skew clamped and counted
- [ ] Attribute flattening depth/count limits enforced; oversized records truncated not dropped
- [ ] Optional PII redaction rules applied before storage, with tests

## 5. Storage (M2, M3)
- [ ] Append-only writes only; no in-place mutation anywhere
- [ ] Per-block CRC verified on read; corruption isolated to the block
- [ ] WAL fsync policy configurable; `always` is the default
- [ ] Zero loss of acknowledged records under the kill-9 matrix
- [ ] Recovery is idempotent: running it twice changes nothing
- [ ] Manifest updates are atomic (tmp + fsync + rename) with a `.bak`
- [ ] Index and manifest rebuildable from segments (`tf-admin rebuild-index`)
- [ ] `tf-admin verify` runs clean against a generated corpus in CI
- [ ] Retention by age and by total size, per tenant/service, with two-phase delete
- [ ] Compaction is crash-safe and result-preserving (differential test)
- [ ] Disk watermarks: warn metric, then 507 on ingest, then early retention
- [ ] Storage format version recorded; reader rejects newer versions loudly

## 6. Query (M4)
- [ ] Full documented operator set implemented with golden tests
- [ ] Precedence, parentheses and implicit AND correct
- [ ] Parse errors report position and expectation
- [ ] Fuzzing yields only `QueryError` - no crash, no hang
- [ ] Planner pushes down time, equality, IN, EXISTS, BETWEEN and bloom probes
- [ ] Residual-only operators (regex, substring) are documented as such
- [ ] Query timeout, scan budget and `limit` cap enforced
- [ ] Streaming execution: bounded memory proven for a 5M-match query
- [ ] Keyset cursors stable under concurrent writes
- [ ] `explain` reports segments pruned, blocks read, records scanned

## 7. API (M5)
- [ ] Every endpoint in the contract implemented and integration-tested
- [ ] Uniform error envelope with `request_id` on every response
- [ ] All documented status codes exercised, including 413, 429, 503, 507
- [ ] Pagination and time-range filtering verified end-to-end
- [ ] OpenAPI committed; CI fails on an unreviewed breaking change
- [ ] SSE live tail has a concurrency cap and clean client-disconnect handling
- [ ] Postman collection committed and run in CI
- [ ] Request/response size limits and timeouts set at the server, not just the proxy

## 8. Dashboard (M6)
- [ ] API types generated from OpenAPI, not hand-written
- [ ] Overview, Explorer, Services, Ingestion, Settings implemented per the plan
- [ ] Empty, loading, error and unauthorized states for every view
- [ ] TFQL bar: autocomplete, syntax error highlighting, query history, shareable URL
- [ ] Large result pages virtualized; no jank at 1000 rows
- [ ] Playwright e2e covers the core investigation flow against a live backend
- [ ] Keyboard navigation, focus order and contrast verified
- [ ] Production build served with cache headers and a CSP; no secrets in the bundle

## 9. Security (M7)
- [ ] API keys hashed at rest; plaintext returned exactly once
- [ ] Scopes enforced per endpoint (`ingest` / `query` / `admin`)
- [ ] Tenant isolation proven by cross-tenant tests on every read path, including SSE
- [ ] User queries cannot override the injected tenant predicate
- [ ] Rate limits per key and per IP
- [ ] TLS supported for HTTP and TCP ingest; no plaintext default in the deploy docs
- [ ] Auth and admin events written to an append-only audit log
- [ ] Secrets only from env/files, validated at startup, never logged
- [ ] Dependency and container scans in CI; high severity blocks release
- [ ] Threat model documented with each mitigation mapped to a test
- [ ] No stack traces or internal paths in any client-facing error

## 10. Observability (M8)
- [ ] `/health` (liveness) and `/ready` (dependency checks) behave correctly under load and shutdown
- [ ] Prometheus metrics for ingest, queue, WAL, blocks, segments, DLQ, query latency, errors
- [ ] Index-efficiency and compression-ratio metrics exposed
- [ ] Internal logs structured JSON with request id and configurable level
- [ ] Grafana dashboard JSON committed and verified against `/metrics`
- [ ] Alert rules documented for disk watermark, DLQ rate, queue depth, error rate, recovery events
- [ ] Optional OpenTelemetry tracing behind a flag

## 11. Performance & reliability (M8)
- [ ] Every target in the performance table measured and recorded in `docs/benchmarks.md`
- [ ] Nightly benchmark job with a regression threshold
- [ ] 24h soak: stable RSS, queue depth and DLQ rate
- [ ] Burst test at 5× sustained rate degrades via backpressure, not loss
- [ ] Graceful shutdown drains the queue and seals segments within the configured deadline
- [ ] Restart under continuous load loses no acknowledged record
- [ ] Capacity planning guide: bytes per record, disk per day, memory per partition

## 12. Deployment & operations (M9)
- [ ] Multi-stage, non-root, pinned, multi-arch images with healthchecks for backend, agent, web
- [ ] `docker compose up` works from a clean clone; documented in the quickstart
- [ ] Optional compose profile for Prometheus + Grafana
- [ ] All configuration documented with defaults, ranges and reload behaviour
- [ ] Volume layout and permissions documented; data survives container replacement
- [ ] Backup and restore procedure written and executed at least once
- [ ] Upgrade path and storage-format migration notes
- [ ] Runbook: disk full, corruption, recovery, key rotation, retention change, hot service
- [ ] Spring Boot example app integrated via both file and HTTP ingestion
- [ ] Quickstart proven by a fresh-machine walkthrough
- [ ] `v1.0.0` tagged with changelog, SBOM and published images
- [ ] LICENSE, SECURITY.md and support expectations stated

## Release sign-off
- [ ] All milestones M0-M8 closed
- [ ] No open `severity:critical` or `severity:high` issues
- [ ] Benchmarks and soak results attached to the release
- [ ] Docs reviewed against the shipped behaviour (no aspirational documentation)
