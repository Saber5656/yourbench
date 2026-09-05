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
     revealed_at, outputs: tuple[Output, ...]` — `outputs` is filled by
     `create_with_outputs` (creation order) and by `get(...,
     with_outputs=True)`; empty tuple otherwise.
   - `Output`: `id, run_id, model_id, model_snapshot_json, status, content,
     error, finish_reason, prompt_tokens, completion_tokens, latency_ms,
     created_at, finished_at`.
   - `RunListItem` (frozen, slots): `run_id: int, task_id: int,
     task_title: str` (display-title fallback, issue 07 rule),
     `category: str, status: str, created_at: str, revealed_at: str | None,
     model_ids: tuple[str, ...]` (output creation order),
     `succeeded_count: int, failed_count: int, pending_count: int`, property
     `votable -> bool` (`status == "completed" and succeeded_count >= 2`).
2. Functions (all mutation targets missing → `ValueError`):
   - `create_with_outputs(conn, *, task_id, params_json: str,
     model_snapshots: Sequence[tuple[str, str]], now=None) -> Run` — one
     transaction inserting the `running` run and one `pending` output per
     `(model_id, snapshot_json)`; `len >= 2` else `ValueError`; returns the
     `Run` with its `outputs` tuple populated (the run engine maps models to
     output ids from it — DESIGN §4.4).
   - `finish_output(conn, output_id, *, status: Literal["succeeded","failed"],
     content=None, error=None, finish_reason=None, prompt_tokens=None,
     completion_tokens=None, latency_ms=None, now=None) -> None` — one
     transition only: `ValueError` if the output is not `pending` or missing.
     Enforced field pairing: `succeeded` requires non-None `content` and
     forces `error=NULL`; `failed` requires non-None `error` and forces
     `content=NULL`; both set `finished_at=now`. `error` is persisted as
     given — callers pass already-sanitized `ProviderError.message`
     (DESIGN §6.2/§7.2); this function additionally truncates to 1000 chars
     as defense in depth.
   - `complete(conn, run_id, now=None) -> None` — transition guard: only
     from `status='running'` (already completed → `ValueError`, timestamp
     unchanged); raises `ValueError` if any output is still `pending`.
   - `reveal(conn, run_id, now=None) -> bool` — sets `revealed_at` if NULL
     (True); False if already revealed (timestamp unchanged); missing run →
     `ValueError`.
   - `get(conn, run_id, *, with_outputs=False) -> Run | None`;
     `outputs_of(conn, run_id) -> list[Output]` (ordered by id);
     `list_runs(conn, *, task_id=None, limit=50) -> list[RunListItem]`
     (newest first); `status_counts(conn, run_id) -> dict[str, int]` — keys
     exactly `pending/succeeded/failed`; `count(conn) -> int` (dashboard).
   - `sweep_orphans(conn, *, older_than_hours: int = 1, now=None) -> int` —
     per DESIGN §7.4: pending outputs whose **run** `created_at <
     now - older_than_hours` become `failed` with
     `error='orphaned: process terminated'`; their runs get completed.
     Returns count of outputs swept. Uses ISO-string comparison (valid for the
     §4.1 format).
3. Parametrized SQL only. Writes to `runs`/`outputs` happen only in this
   module; read-only joins elsewhere per the DESIGN §4.4 ownership rule.
4. mypy-strict clean.

## Acceptance Criteria

- [ ] `create_with_outputs` is atomic: passing a snapshots list whose second
      entry has `model_id=None` (NOT NULL violation) rolls back the run row
      too (run count unchanged); happy path returns `Run.outputs` in input
      order with ids ascending.
- [ ] `finish_output` state guard: completing twice raises; unknown id
      raises; succeeded-without-content and failed-without-error raise;
      succeeded row has `error IS NULL`, failed row has `content IS NULL`;
      `finished_at` set on both; 2000-char error stored truncated to 1000.
- [ ] `complete` with a pending output raises; after all outputs terminal it
      sets `completed_at`; second `complete` raises with timestamp unchanged;
      unknown run id raises.
- [ ] `reveal` True-then-False semantics; timestamp unchanged on second call;
      unknown run id raises.
- [ ] `sweep_orphans`: fixture with a 2-hour-old running run (2 pending) and a
      fresh running run → returns 2; old run completed with failed outputs
      carrying the exact orphan message; fresh run untouched.
- [ ] `list_runs` returns `RunListItem`s with correct counts/model ids
      (creation order), display-title fallback for NULL titles, votable
      property, newest first, respects `task_id` filter and `limit`;
      `count` and `status_counts` match fixtures.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_runs_repo.py -q`.

## Dependencies

05.

## Non-goals

Provider calls / async orchestration (13); pair selection (09); UI (26, 29);
run deletion (v1 non-goal).

## Design References

DESIGN §4.3, §4.4, §7.3, §7.4, §8.5.
