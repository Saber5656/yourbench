# Title

Votes repository, pair selection, games extraction

## Summary

Implement `mybench/db/votes.py`: `Vote` dataclass, vote creation with
integrity checks, deletion, least-voted pair selection, unvoted-pair count,
vote history listing, and the `games()` extraction feeding the rating layer.

## Context

This module implements the scheduling policy of DESIGN §8 and produces the
rating input of §9.1. Its integrity checks are the last line of defense for
vote validity (boundary B7).

## Scope

- `src/mybench/db/votes.py`
- `tests/test_votes_repo.py`

## Detailed Requirements

1. Dataclasses (frozen, slots):
   - `Vote`: `id, run_id, output_a_id, output_b_id, winner, blind, note,
     created_at`.
   - `Pair`: `run_id: int, task_id: int, output_lo_id: int,
     output_hi_id: int, vote_count: int`.
   - `Game`: `model_lo: str, model_hi: str, score_lo: float,
     kind: Literal["decisive", "tie", "both_bad"]`
     (score 1.0 / 0.0 / 0.5 per DESIGN §9.1; `model_lo < model_hi` lexically;
     `kind` preserves the tie/both_bad distinction for display rates).
   - `VoteListItem`: `Vote` fields + `task_id`, `task_title`, `category`,
     `model_a: str`, `model_b: str`.
2. `create(conn, *, run_id, output_a_id, output_b_id, winner, note,
   now=None) -> Vote`:
   - Integrity checks (each violation → `ValueError`): both outputs exist,
     belong to `run_id`, are distinct, both `succeeded`; `winner ∈
     {'a','b','tie','both_bad'}`; `note` ≤ `safety.MAX_NOTE_CHARS`.
   - `blind` is computed **inside** the same transaction: `1` if the run's
     `revealed_at IS NULL` else `0` (caller does not pass it).
3. `delete(conn, vote_id) -> bool` (False if missing).
4. `next_pair(conn, *, category=None, task_id=None, run_id=None) ->
   Pair | None` — implement exactly the normative SQL of DESIGN §8.3 with the
   optional filters appended (`t.category = ?`, `t.id = ?`, `r.id = ?`).
5. `unvoted_pair_count(conn, *, same filters) -> int` (§8.3).
6. `list_votes(conn, *, limit=100) -> list[VoteListItem]` — newest first,
   joining model ids of both outputs (history page; models are revealed there
   by design since the vote already happened).
7. `games(conn, *, category=None, include_nonblind=False) -> list[Game]` —
   one game per vote (join votes→outputs→runs→tasks): map winner through
   display order to `score_lo` with models ordered lexically
   (`model_lo < model_hi`); `blind=1` only unless `include_nonblind`;
   excludes nothing else (archived tasks still count — votes are history).
8. Parametrized SQL only; `votes` table touched only here.
9. mypy-strict clean.

## Acceptance Criteria

- [ ] `create` rejects: cross-run outputs, failed output, identical outputs,
      bad winner, oversized note (each tested).
- [ ] `blind` auto-set: vote before reveal → 1; after `runs.reveal` → 0.
- [ ] `next_pair` ordering: with pairs voted {0,0,1} times → returns one of
      the zero-vote pairs; after voting them once each, the once-voted pair
      becomes eligible (least-voted-first over 20 iterations never returns a
      pair with count > minimum).
- [ ] `next_pair` respects each filter and skips archived tasks and pairs
      with a failed side; returns None when nothing eligible.
- [ ] `unvoted_pair_count` matches a brute-force recount on a random fixture.
- [ ] `games` mapping: fixture where left output belongs to lexically-larger
      model verifies the score flip; tie and both_bad both → 0.5 with `kind`
      set to `tie`/`both_bad` respectively; decisive votes → `kind="decisive"`;
      blind filter verified.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_votes_repo.py -q`.

## Dependencies

05, 07, 08.

## Non-goals

Rating math (14, 15); HTTP surfaces (27, 29); left/right coin flip (view
concern, issue 27).

## Design References

DESIGN §4.3, §4.4, §8.1–8.3, §8.5, §9.1.
