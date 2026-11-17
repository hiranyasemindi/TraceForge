# TraceForge - On-Disk Storage Format

Status: baseline design (v0.1). All integers little-endian. All timestamps epoch microseconds UTC.

## 1. Directory layout

```
$TF_DATA_DIR/
├── wal/
│   ├── 000000000001.wal            # active write-ahead log
│   └── 000000000002.wal
├── segments/
│   └── <tenant>/<service>/<YYYY>/<MM>/<DD>/<HH>/
│       ├── seg-<start_us>-<seq>.tfl   # data (blocks)
│       ├── seg-<start_us>-<seq>.tfi   # sparse index
│       └── seg-<start_us>-<seq>.tfm   # per-segment metadata (summaries, bloom)
├── manifest/
│   ├── manifest.json               # current catalog of sealed segments
│   └── manifest.json.bak
├── dlq/
│   └── <YYYY-MM-DD>.dlq.jsonl      # dead-lettered raw records + reason
└── meta/
    ├── version                     # storage format version
    └── checkpoint.json             # last durable (wal_file, offset) applied to segments
```

Partitioning by `tenant/service/hour` makes retention a file unlink and lets the planner prune on
path alone before touching any file.

## 2. Write-ahead log

Purpose: a record that has been acknowledged to a producer must survive `kill -9`.

Frame:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 4 | magic `TFWL` |
| 4 | 2 | format version |
| 6 | 1 | flags (bit0: payload compressed) |
| 7 | 8 | sequence number (monotonic) |
| 15 | 8 | wall clock of append |
| 23 | 4 | payload length |
| 27 | 4 | CRC32C of payload |
| 31 | n | payload: raw ingest batch (JSON lines or msgpack) |

- Append-only; rolled at `wal_max_bytes` (default 64 MB).
- `fsync` policy: `always` (default, safest), `interval_ms`, or `never` (bench only).
- Acknowledge to the producer only after the frame is durable under the active policy.
- A truncated trailing frame (short read or CRC mismatch) ends replay cleanly: the file is
  truncated to the last valid frame and the event is counted.
- A WAL file is deleted only when `checkpoint.json` proves every frame in it has been flushed into
  a sealed or durably-appended segment.

## 3. Segment file (`.tfl`)

```
┌──────────────┐
│ FileHeader   │ 64 bytes
├──────────────┤
│ Block 0      │  compressed records
│ Block 1      │
│ ...          │
├──────────────┤
│ Footer       │ block directory + stats
├──────────────┤
│ FooterLen(4) │
│ Magic (4)    │ "TFLE"
└──────────────┘
```

### FileHeader (64 bytes)

| Size | Field |
| --- | --- |
| 4 | magic `TFLS` |
| 2 | format version |
| 1 | codec id (0 none, 1 gzip, 2 zstd) |
| 1 | flags |
| 16 | segment ULID |
| 8 | partition start (us) |
| 8 | partition end (us) |
| 24 | reserved |

### Block

| Size | Field |
| --- | --- |
| 4 | magic `TFBK` |
| 4 | record count |
| 8 | min timestamp |
| 8 | max timestamp |
| 4 | uncompressed length |
| 4 | compressed length |
| 4 | CRC32C of compressed payload |
| 1 | codec id (per-block, allows mixed codecs during migration) |
| 3 | reserved |
| n | compressed payload: records framed as `uvarint length + msgpack(record)` |

Block targets: `block_max_records` (default 1000) or `block_max_bytes` uncompressed
(default 1 MiB), whichever comes first; also flushed on `block_max_idle_ms` so low-traffic
services stay queryable in near real time.

The block is the unit of compression, CRC and read amplification. A query decompresses exactly the
blocks whose `[min_ts, max_ts]` overlaps the plan interval.

### Footer

msgpack map, written on seal:
```
{ "blocks": [{ "off": int, "len": int, "n": int, "min_ts": int, "max_ts": int }, ...],
  "records": int, "bytes_raw": int, "bytes_stored": int,
  "min_ts": int, "max_ts": int, "min_ingested": int, "max_ingested": int,
  "levels": {"ERROR": 12, "INFO": 900}, "hosts": [...], "sealed_at": int }
```

An unsealed segment has no footer; readers fall back to scanning block headers (cheap: header-only
seeks), and recovery rewrites the footer. This keeps the active segment queryable.

## 4. Sparse index (`.tfi`)

Time-bucket index, one entry per bucket (default 1 minute):

```
header: magic "TFIX" | version | bucket_width_us | entry_count
entries: [ bucket_start_us (8) | first_block_index (4) | block_count (4) ]
```

Resolving a time range is then a binary search over buckets → a contiguous block range, instead of a
linear block scan. Index files are derivable from the segment, so corruption is recoverable
(`tf-admin rebuild-index`) and the index is never the system of record.

## 5. Segment metadata (`.tfm`)

msgpack:
```
{ "segment_id": str, "tenant": str, "service": str,
  "levels": set[str], "environments": set[str], "hosts": set[str],
  "attr_keys": set[str],
  "numeric_ranges": { "http.status_code": [200, 503] },
  "bloom": { "trace_id": <bitarray>, "request_id": <bitarray> },
  "bloom_params": { "m": int, "k": int, "n": int, "fp": 0.01 } }
```

Pruning order (cheapest first): path/time → `.tfm` equality sets → bloom probe → numeric ranges →
block time ranges → residual predicate per record.

## 6. Manifest

`manifest/manifest.json` is the catalog the planner reads at query time:

```
{ "version": 1, "updated_at": int,
  "segments": [ { "id": ..., "path": ..., "tenant": ..., "service": ...,
                  "min_ts": ..., "max_ts": ..., "records": ..., "bytes": ...,
                  "sealed": true, "state": "active|sealed|compacted|deleting" } ],
  "stats": { "records": int, "bytes_raw": int, "bytes_stored": int } }
```

- Updated atomically: write `manifest.json.tmp` → `fsync` → `rename` → keep one `.bak`.
- Held in memory and refreshed on change; the planner never walks the filesystem on the hot path.
- Rebuildable by scanning segment footers.

## 7. Retention and compaction

- **Retention sweeper** (interval, default 5 min): delete segments whose `max_ts` is older than
  `retention_age` per tenant/service; if `retention_max_bytes` is exceeded, evict oldest first.
  Deletion is two-phase (`state: deleting` in manifest → unlink → remove entry) so a crash mid-delete
  leaves no dangling catalog entry.
- **Compactor**: merges many small segments in one closed hour-partition into one, re-blocking to
  target size. Writes a new segment, atomically swaps manifest entries, then unlinks sources. Never
  rewrites in place.
- Compaction is optional for correctness and only affects read efficiency, so it must be safe to kill.

## 8. Recovery algorithm (startup)

1. Read `meta/version`; refuse to start on a newer format (explicit, loud failure).
2. Load manifest (fall back to `.bak`, then to a full rebuild from the segments tree).
3. For each segment marked `active` or lacking a footer: scan block headers, verify CRCs, truncate
   the file at the first invalid block, rewrite footer and index.
4. Replay WAL from `checkpoint.json` forward: re-run the processing pipeline for each frame, writing
   records that are not already present (dedup by record ULID within the replay window).
5. Update checkpoint, delete fully-applied WAL files, open a new active segment per partition.
6. Expose `tf_recovery_duration_seconds` and `tf_recovery_records_replayed_total`.

Invariant to be asserted by tests: **for any kill point, after recovery the store contains every
record that was acknowledged, and no duplicates.**

## 9. Versioning and migration

- `meta/version` is bumped only for incompatible changes.
- Readers support the current and previous format version.
- Per-block codec ids allow codec changes without rewriting history.
- `tf-admin verify` walks every segment, checks CRCs and cross-checks the manifest; it runs in CI
  against a generated corpus.
