# TraceForge - REST API Contract

Base path `/api/v1`. JSON in, JSON out, UTF-8. All timestamps ISO-8601 UTC in responses.

## 1. Auth

| Header | Use |
| --- | --- |
| `Authorization: Bearer <api-key>` | ingest and query (scoped keys) |
| `Authorization: Bearer <jwt>` | dashboard sessions |

Scopes: `ingest`, `query`, `admin`. A key maps to exactly one tenant. Missing/invalid credentials →
401; valid but wrong scope → 403. Health and readiness are unauthenticated.

## 2. Ingestion

### `POST /logs`
Single event. Body: a raw or partially structured log object.

```json
{ "timestamp": "2026-11-17T10:00:00.123Z", "level": "ERROR", "service": "payment-service",
  "host": "pay-01", "environment": "prod", "trace_id": "9f2b...", "request_id": "req-88",
  "message": "charge failed", "attributes": { "http.status_code": 502, "amount": 42.5 } }
```

`202 Accepted` → `{ "accepted": 1, "rejected": 0, "id": "01JD..." }`.
Only `message` and `service` are required; `timestamp` defaults to receive time, `level` to `INFO`.

### `POST /logs/bulk`
`{ "records": [ ... ] }`, or newline-delimited JSON with `Content-Type: application/x-ndjson`.
Supports `Content-Encoding: gzip`. Partial success is explicit:

```json
{ "accepted": 498, "rejected": 2,
  "errors": [ { "index": 17, "error": "missing_field", "detail": "service" } ] }
```

Limits (configurable): 10 MB body, 5000 records per batch, 256 KB per record.
Queue saturation → `429` with `Retry-After`; disk watermark exceeded → `507`.

### `POST /logs/raw`
`text/plain` body of newline-delimited unstructured lines plus `?service=&host=&parser=`, for
producers that cannot emit JSON.

## 3. Query

### `POST /query`
```json
{ "query": "(level = \"ERROR\" OR level = \"WARN\") AND service = \"payment-service\"",
  "from": "2026-11-17T09:00:00Z", "to": "now",
  "limit": 100, "cursor": null, "order": "desc", "explain": false }
```

`200 OK`:
```json
{ "records": [ { "id": "01JD...", "timestamp": "...", "level": "ERROR", "service": "...",
                 "message": "...", "attributes": { ... } } ],
  "count": 100, "has_more": true, "next_cursor": "eyJzZWciOiJ...",
  "partial": false, "took_ms": 38, "explain": null }
```

- Keyset pagination only; `next_cursor` is opaque and encodes segment/block/record position.
- `partial: true` when a scan budget, timeout or corrupt block limited the result; `warnings[]` says why.
- `limit` capped at `max_query_limit` (default 1000).
- `GET /query?q=...&from=...&to=...` is supported for shareable dashboard links.

### `GET /logs/{id}`
Single record by id, plus `?context=50` to return surrounding lines from the same service.

### `GET /trace/{trace_id}`
All records for a trace across services, ordered by timestamp. This is the cross-service debugging view.

## 4. Metadata and stats

| Endpoint | Returns |
| --- | --- |
| `GET /services` | services seen, with first/last seen, record count, error count, hosts, environments |
| `GET /services/{name}` | per-service detail: level breakdown, recent errors, volume by hour |
| `GET /fields` | known field and attribute keys with inferred types (powers dashboard autocomplete) |
| `GET /stats/overview` | totals, error/warn counts, logs/sec (1m, 5m, 15m), storage bytes, compression ratio |
| `GET /stats/ingest` | throughput series, queue depth, DLQ rate, per-transport counts |
| `GET /stats/storage` | segment count, bytes raw/stored, oldest record, disk usage vs watermark |
| `GET /stats/timeseries?interval=1m&group_by=level` | histogram for the explorer's volume chart |

## 5. Live tail

`GET /stream?query=...` - Server-Sent Events, `event: log` per record plus periodic `event: heartbeat`.
Server-side cap on concurrent streams; the filter is the same TFQL AST evaluated on the write path.

## 6. Admin

| Endpoint | Purpose |
| --- | --- |
| `POST /admin/keys`, `GET /admin/keys`, `DELETE /admin/keys/{id}` | API key lifecycle (key shown once) |
| `GET /admin/config` | effective configuration, secrets redacted |
| `PUT /admin/retention` | retention policy per tenant/service |
| `GET /admin/dlq`, `POST /admin/dlq/replay` | inspect and replay dead-lettered records |
| `POST /admin/compact`, `POST /admin/rebuild-index` | maintenance jobs (async, return a job id) |

## 7. Health

- `GET /health` → `{ "status": "ok", "version": "...", "uptime_s": 1234 }` (liveness, never fails on load)
- `GET /ready` → 200 when WAL is open, the writer task is alive and disk is below the watermark; 503 otherwise with per-check detail.

## 8. Error model

```json
{ "error": "query_parse_error", "message": "human readable",
  "detail": { "position": 24 }, "request_id": "01JD..." }
```

| Status | When |
| --- | --- |
| 400 | malformed body, invalid TFQL, invalid time range |
| 401 / 403 | missing or invalid credentials / wrong scope |
| 404 | unknown record, service or key |
| 409 | conflicting admin operation (e.g. compaction already running) |
| 413 | payload too large |
| 422 | schema validation failure on a structured field |
| 429 | rate limit or queue backpressure (`Retry-After`) |
| 500 | unexpected: logged with `request_id`, never leaks internals |
| 503 | not ready / shutting down |
| 507 | storage watermark exceeded |

Every response carries `X-Request-ID`; every 4xx/5xx is counted by a metric labelled with the code.

## 9. Compatibility rules

- `/api/v1` is additive-only after 1.0: no field removals or type changes.
- Unknown request fields are rejected (`extra="forbid"`) on admin endpoints and ignored on ingest
  (where leniency is the point, since unknown keys become attributes).
- The OpenAPI document generated by FastAPI is committed and diffed in CI to catch accidental breaks.
