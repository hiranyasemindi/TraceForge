# TraceForge Documentation

TraceForge is a lightweight, pluggable real-time log aggregation, storage and query engine.
These documents are the planning and design baseline for taking it from an empty repository to a
production-deployable 1.0.

## Index

| Document | Purpose |
| --- | --- |
| [architecture.md](architecture.md) | System architecture: components, data flow, module boundaries, failure modes |
| [storage-format.md](storage-format.md) | On-disk binary format: WAL, segments, blocks, indexes, manifests, recovery |
| [query-language.md](query-language.md) | TFQL grammar, operators, precedence, semantics, planner behaviour |
| [api-contract.md](api-contract.md) | REST surface, request/response shapes, error model, pagination, auth |
| [roadmap.md](roadmap.md) | Milestones M0-M9 with scope, exit criteria and dependencies |
| [production-readiness-checklist.md](production-readiness-checklist.md) | The master checklist gating the 1.0 release |
| [issue-backlog.md](issue-backlog.md) | The 81 issues tracked on GitHub, with scope and acceptance criteria |

## How planning is tracked

1. **Milestones** M0 to M9 live in [roadmap.md](roadmap.md) and exist as GitHub milestones. A
   milestone closes only when every exit criterion listed for it has automated evidence.
2. **Issues** are the work items in [issue-backlog.md](issue-backlog.md), one GitHub issue each,
   assigned to a milestone and labelled by type, area and priority. Each carries its scope and
   acceptance criteria as checkboxes.
3. **Release gating** is the [production readiness checklist](production-readiness-checklist.md).
   An item is only ticked when a test, a CI job, a recorded benchmark or an executed procedure
   proves it.
4. New work is filed as a GitHub issue against the milestone it belongs to, and the backlog
   document is updated in the same pull request so the two do not drift.

## Conventions

- Language: Python 3.12 (backend), TypeScript + React (dashboard).
- Every module gets type hints, docstrings and unit tests; no module merges without tests.
- Architectural decisions are recorded as ADRs under `docs/adr/NNNN-title.md`.
- Nothing is "done" until it is observable (metric or log) and recoverable (restart-safe).
