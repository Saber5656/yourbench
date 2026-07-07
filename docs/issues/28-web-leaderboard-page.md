# Title

Web: leaderboard page

## Summary

Implement `web/routes/leaderboard.py` + `templates/leaderboard.html`: the
ranked table with CIs, rates, provisional flags, component sections, category
filter, and the non-blind toggle.

## Context

DESIGN §11.2 last row + §9.6. This page is the product's payoff screen; it
must render the same `compute_leaderboard` results as the CLI (issue 21) with
zero recomputation drift.

## Scope

- `src/mybench/web/routes/leaderboard.py`
- `src/mybench/web/templates/leaderboard.html`
- `tests/test_web_leaderboard.py`

## Detailed Requirements

1. `GET /leaderboard?category=&include_nonblind=1`:
   - Calls `compute_leaderboard(conn, category=..., include_nonblind=...,
     bootstrap_samples=settings.bootstrap_samples,
     known_model_ids=[m.id for m in config.models])` — identical arguments to
     issue 21 (shared helper permitted in `rating/leaderboard.py`).
   - Filter bar: category `<select>` populated from distinct task categories
     in the DB (+ `all`); GET form (no CSRF needed — safe method, no state
     change); `include non-blind votes` checkbox; current selections
     preserved.
   - Table columns: `#` (rank within component; `*` suffix when provisional),
     `model`, `rating` (bold), `95% CI` (`+x/−y` or `—` when disabled),
     `games`, `win%`, `tie%`, `bad%`. Rates one decimal; `win%` `—` when None.
   - Multiple components → one table per component with header `component
     {i+1} — {n} models ({games} games)` and an explanatory note (models
     never compared across components).
   - Below the table: `no games yet: {list}` when unrated_models non-empty;
     total line `{total_votes} votes counted (blind only|including
     non-blind)`; method footnote: `Bradley-Terry ratings, mean 1000; 95%
     bootstrap CI; * = provisional (<10 games)` linking to
     `docs/research/rating-methodology.md` on the repo (plain external link).
   - Empty (zero games): friendly empty state linking `/vote` and `/tasks`.
2. Model ids/categories are plain autoescaped text (no markdown).
3. No caching; page renders live per request (ADR-003). If render exceeds
   the §9.5 budget in practice, that is Known Unknown U4 — do not add caching
   in this issue.
4. mypy-strict clean.

## Acceptance Criteria

TestClient with vote fixtures (reuse rating test fixtures):

- [ ] Golden render: ordered rows, ratings match `compute_leaderboard`
      output exactly (compare against direct call in the test), CI/provisional
      markers correct.
- [ ] Category filter changes both the query and the select state; unknown
      category → empty state (not 500).
- [ ] Non-blind toggle includes flagged votes (counts change per fixture).
- [ ] Two-component fixture renders two sections with the note.
- [ ] `bootstrap_samples=0` config → CI column shows `—`.
- [ ] unrated_models listed; empty DB → empty state page.
- [ ] Security helper assertions pass (headers, no inline script).
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_web_leaderboard.py -q`; manual visual pass on demo
data; cross-check numbers against `mybench leaderboard` CLI output.

## Dependencies

15, 22.

## Non-goals

Charts/history-over-time (v2); per-task drill-down (history pages cover raw
data); rating recompute controls.

## Design References

DESIGN §9.6, §11.2, §9.5 (perf), §18 U4; ADR-003.
