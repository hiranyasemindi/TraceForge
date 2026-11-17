# TraceForge

A real-time log aggregator and query engine built from scratch in Python.

TraceForge collects logs from many applications over files, TCP and HTTP, parses them into
structured records, stores them in a custom append-only compressed segment store with time-based
indexing, and makes them searchable through TFQL, its own query language, with a REST API and a
developer-focused web dashboard on top.

> Status: planning complete, implementation starting at milestone M0.

## Documentation

| Document | Purpose |
| --- | --- |
| [docs/architecture.md](docs/architecture.md) | Components, data flow, decisions, failure modes, performance targets |
| [docs/storage-format.md](docs/storage-format.md) | WAL, segment/block layout, indexes, manifest, recovery |
| [docs/query-language.md](docs/query-language.md) | TFQL grammar, operators, precedence, planning |
| [docs/api-contract.md](docs/api-contract.md) | REST surface, error model, pagination, auth |
| [docs/roadmap.md](docs/roadmap.md) | Milestones M0-M9 with exit criteria |
| [docs/production-readiness-checklist.md](docs/production-readiness-checklist.md) | The 1.0 release gate |
| [docs/issue-backlog.md](docs/issue-backlog.md) | The 81-issue backlog |

Start with [docs/README.md](docs/README.md).

## Planning workflow

Work is tracked as GitHub milestones M0 to M9, with one issue per work item, labelled by type,
area and priority. Each issue carries its scope and acceptance criteria; each milestone has hard
exit criteria in [docs/roadmap.md](docs/roadmap.md), and the 1.0 release gate is
[docs/production-readiness-checklist.md](docs/production-readiness-checklist.md).

- [Milestones](https://github.com/hiranyasemindi/TraceForge/milestones)
- [Open issues](https://github.com/hiranyasemindi/TraceForge/issues)
- [Backlog in one page](docs/issue-backlog.md)

## License

See [LICENSE](LICENSE).
