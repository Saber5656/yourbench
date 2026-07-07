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
   `archived_at: str | None`, plus property
   `display_title -> str` (title, else first 80 chars of user_prompt with
   newlines collapsed to spaces).
2. `@dataclass(frozen=True, slots=True) class TaskParams`:
   `temperature: float | None`, `top_p: float | None`,
   `max_tokens: int | None`; helpers `to_json() -> str` (compact, sorted
   keys, omit Nones) and `from_json(s: str) -> TaskParams` (unknown keys in
   stored JSON are ignored with a DEBUG log — forward compatibility).
3. Functions (all take `conn: sqlite3.Connection` first; write functions
   manage their own `tx`):
   - `create(conn, *, title, category, system_prompt, user_prompt, params,
     now: str | None = None) -> Task` — calls
     `safety.validate_task_params`; violations → raise `ValueError` with the
     joined messages; `category=None` → `"general"`.
   - `get(conn, task_id: int) -> Task | None`
   - `list_(conn, *, category: str | None = None,
     include_archived: bool = False) -> list[Task]` — newest first.
   - `run_counts(conn, task_ids: Sequence[int]) -> dict[int, int]` — for list
     views.
   - `archive(conn, task_id, now=None) -> bool` /
     `unarchive(conn, task_id) -> bool` — return False when task missing;
     archive is idempotent (already archived → True, unchanged timestamp).
4. Parametrized SQL only; no SQL outside this module for the `tasks` table.
5. mypy-strict clean.

## Acceptance Criteria

- [ ] create→get round-trips every field including params JSON.
- [ ] `create` rejects: oversized user_prompt, bad category slug, temperature
      3.0 (each raising `ValueError` containing the §5.4 message).
- [ ] `list_` filters by category, excludes archived by default, includes with
      flag, orders newest first (fixture with distinct created_at values).
- [ ] `display_title` fallback: multi-line prompt collapses; ≤ 80 chars.
- [ ] archive/unarchive round-trip; missing id → False; double-archive keeps
      original `archived_at`.
- [ ] `params from_json` ignores unknown keys.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_tasks_repo.py -q` against a `tmp_path` DB created
via issue 05's `apply_migrations`.

## Dependencies

05, 06.

## Non-goals

CLI/web surfaces (18, 25); run creation (08); deletion of tasks (v1 has only
archive — DESIGN ADR-006).

## Design References

DESIGN §4.3 (tasks table), §4.4, §5.4; ADR-006.
