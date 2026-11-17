## What this changes

<!-- One or two sentences. Link the issue: Closes #123 -->

Closes #

## Why

<!-- The problem being solved, or the milestone exit criterion this moves. -->

## How

<!-- Approach, and anything a reviewer would otherwise have to reverse engineer.
     If this changes an architectural decision, link or add the ADR under docs/adr/. -->

## Evidence

<!-- Tests, benchmark numbers, explain output, screenshots. "Works locally" is not evidence. -->

- [ ] Unit tests added or updated
- [ ] Integration test added or updated (if this crosses a module boundary)
- [ ] Benchmark or explain output attached (if this touches the storage or query hot path)

## Checklist

- [ ] `make check` passes locally (format, lint, mypy strict, tests)
- [ ] Public functions and classes are typed and documented
- [ ] No module boundary violated (`storage` does not import `query`, `ingest` or `api`)
- [ ] Config changes documented with default, range and reload behaviour
- [ ] New failure paths are observable (metric or structured log) and recoverable
- [ ] Docs updated: architecture, storage format, query language, API contract, as applicable
- [ ] Relevant items in `docs/production-readiness-checklist.md` ticked, with evidence linked
- [ ] No secrets, credentials or personal data in code, tests, fixtures or logs

## Risk and rollback

<!-- What breaks if this is wrong, and how to revert. Mention storage format or API
     contract changes explicitly: both are compatibility surfaces. -->
