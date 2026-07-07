# Title

Tasks repository

## Summary

Implement `mybench/db/tasks.py`: the `Task` dataclass and all task CRUD
(create, get, list with filters, archive/unarchive) with server-side
enforcement of §5.4 limits.

## Context

Tasks are the root entity (DESIGN §4.3). CLI (issue 18) and web (issue 25)
call these functions; both rely on this layer rejecting invalid data even if a
UI check was bypassed (boundary B7).

## Scope

- `src/mybench/db/tasks.py`
- `tests/test_tasks_repo.py`

## Detailed Requirements

1. `@dataclass(frozen=True, slots=True) class Task`: `id: int`,
   `title: str | None`, `category: str`, `system_prompt: str | None`,
   `user_prompt: str`, `params: TaskParams`, `created_at: str`,
   `archived_at: str | None`, plus property `display_title -> str`:
   `title` when set; otherwise collapse all whitespace runs (incl. newlines)
   in `user_prompt` to single spaces, strip, then take the first 80
   characters (collapse first, then cut).
2. `@dataclass(frozen=True, slots=True) class TaskParams`:
   `temperature: float | None`, `top_p: float | None`,
   `max_tokens: int | None`; helpers `to_json() -> str` (compact, sorted
   keys, omit Nones) and `from_json(s: str) -> TaskParams` (unknown keys in
   stored JSON are ignored with a DEBUG log — forward compatibility).
3. Functions (all take `conn: sqlite3.Connection` first; write functions
   manage their own `tx`):
   - `create(conn, *, title, category, system_prompt, user_prompt, params,
     now: str | None = None) -> Task` — calls
     `safety.validate_task_params(...)` (issue 06 contract: keyword args,
     returns `list[str]` of exact normative messages); non-empty list →
     raise `ValueError` with the messages joined by newlines;
     `category=None` → `"general"`.
   - `get(conn, task_id: int) -> Task | None`
   - `list_(conn, *, category: str | None = None,
     include_archived: bool = False) -> list[Task]` — newest first.
   - `run_counts(conn, task_ids: Sequence[int]) -> dict[int, int]` — for list
     views (read-only join against `runs`, allowed per DESIGN §4.4).
   - `list_categories(conn) -> list[str]` — distinct categories, sorted
     (DESIGN §4.4; used by the leaderboard filter, issue 28).
   - `archive(conn, task_id, now=None) -> bool` /
     `unarchive(conn, task_id) -> bool` — return False when task missing;
     both idempotent: already-archived archive → True with unchanged
     timestamp; already-active unarchive → True, no change.
4. Parametrized SQL only. Writes to `tasks` happen only in this module;
   read-only joins from other repo modules are allowed where DESIGN mandates
   them (§4.4 ownership rule — e.g. pair selection).
5. mypy-strict clean.

## Acceptance Criteria

- [ ] create→get round-trips every field including params JSON.
- [ ] `create` rejects every §5.4 task rule (parametrized over issue 06's
      normative violations: missing/oversized user_prompt, oversized
      system_prompt, oversized title, bad category slug, temperature out of
      range, top_p out of range, max_tokens out of range), each raising
      `ValueError` containing the exact issue-06 message; `category=None`
      persists as `general`.
- [ ] `list_` filters by category, excludes archived by default, includes with
      flag, orders newest first (fixture with distinct created_at values).
- [ ] `list_categories` returns distinct sorted categories.
- [ ] `display_title` fallback: multi-line prompt collapses whitespace before
      the 80-char cut (fixture where order matters proves collapse-then-cut).
- [ ] archive/unarchive round-trip; missing id → False for both;
      double-archive keeps original `archived_at`; unarchive of active task
      → True, no change.
- [ ] `params from_json` ignores unknown keys.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_tasks_repo.py -q` against a `tmp_path` DB created
via issue 05's `apply_migrations`.

## Dependencies

05, 06.

## Non-goals

CLI/web surfaces (18, 25); run creation (08); deletion of tasks (v1 has only
archive — DESIGN ADR-006).

## Design References

DESIGN §4.3 (tasks table), §4.4, §5.4; ADR-006.
