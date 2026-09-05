# Title

SQLite core: connection policy, migration runner, schema 0001

## Summary

Implement `mybench/db/__init__.py` (connect + transaction helper) and
`mybench/db/migrations.py` (numbered migration runner + migration 0001 with
the full v1 schema).

## Context

ADR-006 fixes the storage approach (stdlib sqlite3, hand-written migrations,
WAL). DESIGN §4.3 is the authoritative schema; repositories (issues 07–09)
build on this.

## Scope

- `src/mybench/db/__init__.py`
- `src/mybench/db/migrations.py`
- `tests/test_db_core.py`

## Detailed Requirements

1. `connect(db_path: Path) -> sqlite3.Connection` (DESIGN §4.1):
   - parent dir assumed to exist (paths module guarantees data dir);
   - `row_factory = sqlite3.Row`; `PRAGMA journal_mode=WAL`,
     `PRAGMA foreign_keys=ON`, `PRAGMA busy_timeout=5000`;
   - after creating a **new** DB file, `chmod 0600` (DESIGN §13.7);
   - `check_same_thread=False` must NOT be set (each worker uses its own
     connection).
2. `tx(conn)` context manager: `BEGIN IMMEDIATE` on enter; commit on clean
   exit; rollback on exception (re-raises).
3. `utc_now_iso(now: datetime | None = None) -> str` helper exactly per
   DESIGN §4.1 format (millisecond precision, `Z` suffix). `now` must be
   timezone-aware: aware datetimes are converted to UTC; a naive datetime
   raises `ValueError`.
4. `migrations.MIGRATIONS: list[tuple[int, str]]` with entry `(1, <SQL>)`
   containing **exactly** the DDL of DESIGN §4.3 (all four tables, all CHECK
   constraints, all five indexes) plus nothing else.
5. `apply_migrations(conn) -> int` (returns number applied): creates
   `schema_migrations(version INTEGER PRIMARY KEY, applied_at TEXT NOT NULL)`
   if absent; applies each pending version **atomically** — `BEGIN
   IMMEDIATE`, execute the migration SQL, insert the version row (bound
   parameters for version/applied_at values), `COMMIT`; any error → rollback
   of both the DDL and the version row, then re-raise. `applied_at` via
   `utc_now_iso()`.
6. `_validate(migrations: Sequence[tuple[int, str]]) -> None` — raises
   `ValueError` on duplicate or non-ascending version numbers; called at
   module import on `MIGRATIONS` and directly testable with synthetic lists.
7. mypy-strict clean.

## Acceptance Criteria

- [ ] Fresh DB: `apply_migrations` returns 1; re-run returns 0 (idempotent);
      `schema_migrations` holds version 1.
- [ ] Schema-structure assertions, not just names: per table,
      `PRAGMA table_info` matches DESIGN §4.3 exactly (column names, order,
      types, NOT NULL, defaults); `PRAGMA foreign_key_list` matches FK
      targets; `PRAGMA index_list`/`index_info` match the five indexes and
      their columns.
- [ ] CHECK constraints enforced (invalid `runs.status`, `votes.winner`,
      `outputs.status`, `votes.blind = 2`, `votes` with `output_a_id ==
      output_b_id` → `IntegrityError`).
- [ ] FK enforcement on (`outputs.run_id` → missing run raises).
- [ ] Failing-migration atomicity: appending a synthetic migration whose SQL
      errors mid-way leaves no partial DDL and no `schema_migrations` row
      for that version.
- [ ] `_validate` raises on duplicate and on descending versions (direct
      unit tests).
- [ ] `utc_now_iso` rejects naive datetimes; converts aware non-UTC to UTC.
- [ ] WAL mode active (`PRAGMA journal_mode` returns `wal`).
- [ ] New DB file has mode `0600`.
- [ ] `tx` commits on success, rolls back on exception (tested both ways).
- [ ] `utc_now_iso()` matches regex
      `^\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{3}Z$`.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_db_core.py -q`; optional manual QA (non-gating):
`python -c "...connect(tmp); apply_migrations(...)"` then
`sqlite3 file .schema` inspection.

## Dependencies

01, 03.

## Non-goals

Repository functions (07–09); orphan sweep SQL (08); any data access beyond
migrations.

## Design References

DESIGN §4.1 (connection), §4.2 (migrations), §4.3 (schema), §13.7; ADR-006.
