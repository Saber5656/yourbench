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
   - Text output per DESIGN §10.8 (normative formatting): header
     `leaderboard — {total_votes} votes (blind only|including non-blind)`
     plus `category: {c}` line when filtered; table columns exactly
     `RANK`, `MODEL`, `RATING`, `95% CI`, `GAMES`, `WIN%`, `TIE%`, `BAD%`;
     `CI` as `+{hi−rating}/−{rating−lo}` (`-` when disabled); rates as
     `rate*100` with one decimal + `%` (`-` when `None`); provisional rows
     suffixed `*` on RANK with footnote `* provisional: fewer than 10
     games`; multiple components → section headers
     `component {row.component + 1} ({n} models)` in component-index order;
     then `no games yet: {comma list}` when `unrated_models` non-empty.
   - Empty-state precedence (DESIGN §10.8): text mode with
     `total_votes == 0` → `no votes yet; run tasks and vote first` to
     stderr, exit 0, before any table/unrated output. `--json` always
     prints the full `Leaderboard` document regardless.
   - `--json`: serialize the full `Leaderboard` dataclass tree
     (rows with all §9.6 fields, components_count, unrated_models,
     total_votes).
2. `mybench export --out PATH [--format json|csv] [--force]`:
   - Default format `json`; call exactly `export.export_json(conn, path,
     force=force)` / `export.export_votes_csv(conn, path, force=force)` —
     no serialization logic in the CLI layer.
   - Success: stderr prints `export.EXPORT_SENSITIVITY_NOTE` (issue 20
     constant, DESIGN §13.6) and stdout
     `exported {tasks} tasks, {runs} runs, {outputs} outputs, {votes} votes
     to {path}` using `export.export_stats(conn)`.
   - `FileExistsError` → `error: {path} exists (use --force to overwrite)`
     exit 1.
3. Both commands run through the standard startup context (16).
   `strip_terminal_controls` applies to terminal-rendered user text (model
   ids/categories in the leaderboard table) only — export **file contents**
   preserve stored values verbatim (DESIGN §13.6 scopes stripping to
   terminal echoes; files rely on the sensitivity reminder).
4. mypy-strict clean.

## Acceptance Criteria

- [ ] Golden text rendering (checked-in fixtures, `bootstrap_samples=0` for
      determinism) for: normal single component; provisional rows; CI
      disabled; two components (1-based headers); unrated list; empty state
      (stderr text, exit 0); rate formatting `62.5%` style verified.
- [ ] `--json` parses; row fields complete; matches computed values.
- [ ] Filters passed through verified with spy on `compute_leaderboard`.
- [ ] export json/csv happy paths create files (contents delegated to issue
      20 tests — here assert existence + summary line/stats correctness).
- [ ] Overwrite refusal exit 1 with exact message; `--force` succeeds.
- [ ] Sensitivity reminder on stderr, summary on stdout (stream separation).
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_cli_leaderboard.py tests/test_cli_export.py -q`;
manual demo-DB run for visual table check.

## Dependencies

15, 16, 20.

## Non-goals

Web leaderboard (28); rating math (14/15); import.

## Design References

DESIGN §10.8, §10.9, §9.6, §13.6.
