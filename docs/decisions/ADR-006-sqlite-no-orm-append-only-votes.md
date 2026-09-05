# ADR-006: stdlib sqlite3 with hand-written migrations; no ORM; append-only votes with explicit delete

Date: 2026-07-08
Status: Accepted

## Context

Storage needs are modest (4 core tables, one writer at a time, personal data
volume). Implementation agents execute better against explicit SQL than ORM
abstractions, and the OSS dependency tree should stay minimal (ADR-002).

## Decision

1. **stdlib `sqlite3`** with a thin connection helper: `Row` row factory,
   `PRAGMA journal_mode=WAL`, `PRAGMA foreign_keys=ON`,
   `PRAGMA busy_timeout=5000`. CLI process and server process may write
   concurrently; WAL + busy_timeout is the concurrency story.
2. **Migrations:** ordered list of `(version, sql)` pairs in
   `mybench/db/migrations.py`, tracked in a `schema_migrations` table, applied
   automatically at startup inside a transaction per version. Downgrades are
   not supported.
3. **Repository modules, not ORM models:** `db/tasks.py`, `db/runs.py`,
   `db/votes.py` expose typed functions (dataclasses in, dataclasses out).
   Every SQL statement lives in these modules only.
4. **Votes are append-only** in normal operation; the single mutation allowed
   is explicit user-initiated deletion of a vote (data ownership). Tasks are
   soft-archived (`archived_at`), never hard-deleted in v1. Runs/outputs are
   immutable after completion.
5. **Timestamps:** UTC ISO-8601 strings with `Z` suffix, generated in Python
   (`datetime.now(timezone.utc)`), never `CURRENT_TIMESTAMP`, so tests can
   inject clocks.

## Consequences

- Zero ORM dependency; schema is auditable in one file; agents copy exact SQL
  from DESIGN.md §4.
- Cross-process locking edge cases (CLI run + server run simultaneously) are
  bounded by WAL semantics; write transactions are kept short by design.
- Schema changes always ship as new migration entries; issue drafts must not
  edit migration 0001 after it lands (enforced in review checklist, issue 32).
