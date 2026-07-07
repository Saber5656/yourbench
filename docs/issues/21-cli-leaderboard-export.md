# Title

CLI: leaderboard / export commands

## Summary

Implement `mybench/cli/leaderboard_cmd.py` and `mybench/cli/export_cmd.py`:
terminal rendering of `compute_leaderboard` and the CLI wrapper over the
export module.

## Context

DESIGN §10.8–10.9. The CLI leaderboard is the scriptable/glanceable
counterpart of the web page (28) and must render the identical computation.

## Scope

- `src/mybench/cli/leaderboard_cmd.py`
- `src/mybench/cli/export_cmd.py`
- `tests/test_cli_leaderboard.py`, `tests/test_cli_export.py`

## Detailed Requirements

1. `mybench leaderboard [--category C] [--include-nonblind] [--json]`:
   - Calls `compute_leaderboard(conn, category=..., include_nonblind=...,
     bootstrap_samples=settings.bootstrap_samples,
     known_model_ids=[m.id for m in config.models])`.
   - Text output: header `leaderboard — {N} blind votes` (`+ non-blind` when
     included; `category: {c}` line when filtered); table columns exactly
     `RANK`, `MODEL`, `RATING`, `95% CI`, `GAMES`, `WIN%`, `TIE%`, `BAD%`,
     with `CI` as `+{hi−rating}/−{rating−lo}` (or `-` when disabled),
     `WIN%` `-` when None, provisional rows suffixed `*` on RANK and a
     footnote `* provisional: fewer than 10 games`; multiple components →
     per-component section headers `component {i} ({n} models)`; then
     `no games yet: {comma list}` when `unrated_models` non-empty; empty
     leaderboard entirely → `no votes yet; run tasks and vote first` stderr,
     exit 0.
   - `--json`: serialize the full `Leaderboard` dataclass tree
     (rows with all §9.6 fields, components_count, unrated_models,
     total_votes).
2. `mybench export --out PATH [--format json|csv] [--force]`:
   - Default format `json`; formats map to issue 20 functions.
   - Success: stderr reminder `note: export contains your prompt and output
     text — handle accordingly` (DESIGN §13.6) and stdout
     `exported {tasks} tasks, {runs} runs, {outputs} outputs, {votes} votes
     to {path}` using `export_stats`.
   - `FileExistsError` → `error: {path} exists (use --force to overwrite)`
     exit 1.
3. Both commands run through the standard startup context (16); model text
   never appears in these outputs, so no extra stripping beyond titles
   (titles ARE user text → strip in text mode).
4. mypy-strict clean.

## Acceptance Criteria

- [ ] Golden text rendering for: normal single component; provisional rows;
      CI disabled (`bootstrap_samples=0` config); two components; unrated
      list; empty state.
- [ ] `--json` parses; row fields complete; matches computed values.
- [ ] Filters passed through verified with spy on `compute_leaderboard`.
- [ ] export json/csv happy paths create files (contents delegated to issue
      20 tests — here assert existence + summary line/stats correctness).
- [ ] Overwrite refusal exit 1 with exact message; `--force` succeeds.
- [ ] Sensitivity reminder on stderr, summary on stdout (stream separation).
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_cli_leaderboard.py tests/test_cli_export.py -q`;
manual demo-DB run for visual table check.

## Dependencies

15, 16, 20.

## Non-goals

Web leaderboard (28); rating math (14/15); import.

## Design References

DESIGN §10.8, §10.9, §9.6, §13.6.
