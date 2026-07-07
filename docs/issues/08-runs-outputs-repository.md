# Title

Runs/outputs repository incl. reveal + orphan sweep

## Summary

Implement `mybench/db/runs.py`: `Run`/`Output` dataclasses, atomic run
creation with pending outputs, output completion, run completion, reveal
(blindness), orphan sweep, and read queries used by CLI/web.

## Context

The run engine (issue 13) drives these functions; run pages (26) and history
(29) read them. `revealed_at` is the blindness-integrity anchor (DESIGN §8.5).
Orphan sweep (§7.4) is the recovery path for killed processes.

## Scope

- `src/mybench/db/runs.py`
- `tests/test_runs_repo.py`

## Detailed Requirements

1. Dataclasses (frozen, slots):
   - `Run`: `id, task_id, status, params_json, created_at, completed_at,
     revealed_at`.
   - `Output`: `id, run_id, model_id, model_snapshot_json, status, content,
     error, finish_reason, prompt_tokens, completion_tokens, latency_ms,
     created_at, finished_at`.
2. Functions:
   - `create_with_outputs(conn, *, task_id, params_json, model_snapshots:
     Sequence[tuple[str, str]], now=None) -> Run` — one transaction inserting
     the `running` run and one `pending` output per `(model_id,
     snapshot_json)`; `len(model_snapshots) >= 2` else `ValueError`.
   - `finish_output(conn, output_id, *, status: Literal["succeeded","failed"],
     content=None, error=None, finish_reason=None, prompt_tokens=None,
     completion_tokens=None, latency_ms=None, now=None) -> None` — one
     transition only: raises `ValueError` if the output is not `pending`
     (guards double-completion). `succeeded` requires non-None content;
     `failed` requires non-None error (enforced here).
   - `complete(conn, run_id, now=None) -> None` — sets
     `status='completed', completed_at`; raises `ValueError` if any output of
     the run is still `pending`.
   - `reveal(conn, run_id, now=None) -> bool` — sets `revealed_at` if NULL
     (returns True); False if already revealed.
   - `get(conn, run_id) -> Run | None`; `outputs_of(conn, run_id) ->
     list[Output]` (ordered by id);
     `list_runs(conn, *, task_id=None, limit=50) -> list[RunListItem]` where
     `RunListItem` adds `task_title`, `category`, `model_ids: list[str]`,
     `succeeded_count`, `failed_count`, `pending_count`.
   - `status_counts(conn, run_id) -> dict[str, int]` — keys
     `pending/succeeded/failed` (for `status.json`, issue 26).
   - `sweep_orphans(conn, *, older_than_hours: int = 1, now=None) -> int` —
     per DESIGN §7.4: pending outputs whose **run** `created_at <
     now - older_than_hours` become `failed` with
     `error='orphaned: process terminated'`; their runs get completed.
     Returns count of outputs swept. Uses ISO-string comparison (valid for the
     §4.1 format).
3. Parametrized SQL only; the `runs`/`outputs` tables are touched only here.
4. mypy-strict clean.

## Acceptance Criteria

- [ ] `create_with_outputs` is atomic (inject a failure after run insert in a
      test via constraint violation → whole tx rolled back).
- [ ] `finish_output` state guard: completing twice raises; succeeded-without-
      content and failed-without-error raise.
- [ ] `complete` with a pending output raises; after all outputs terminal it
      sets `completed_at`.
- [ ] `reveal` True-then-False semantics; timestamp unchanged on second call.
- [ ] `sweep_orphans`: fixture with a 2-hour-old running run (2 pending) and a
      fresh running run → returns 2; old run completed with failed outputs
      carrying the exact orphan message; fresh run untouched.
- [ ] `list_runs` returns items with correct counts/model ids, newest first,
      respects `task_id` filter and `limit`.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_runs_repo.py -q`.

## Dependencies

05.

## Non-goals

Provider calls / async orchestration (13); pair selection (09); UI (26, 29);
run deletion (v1 non-goal).

## Design References

DESIGN §4.3, §4.4, §7.3, §7.4, §8.5.
