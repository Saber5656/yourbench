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
   - Query-param validation (DESIGN §11.2): `category` empty or `all` →
     `None`; a value failing the §5.4 slug regex (oversized, HTML-like,
     SQL-like) → 400 error page before any DB use; a valid-but-unknown
     category → normal render with an empty state (not 500).
   - Filter bar: category `<select>` populated via
     `tasks.list_categories(conn)` (issue 07) + `all`; GET form (no CSRF
     needed — safe method, no state change); `include non-blind votes`
     checkbox; current selections preserved.
   - Table columns: `#` (rank within component; `*` suffix when provisional),
     `model`, `rating` (bold), `95% CI` (`+x/−y` or `—` when disabled),
     `games`, `win%`, `tie%`, `bad%`. Rates one decimal; `win%` `—` when None.
   - Multiple components → one table per component with header `component
     {i+1} — {n} models ({component_games} games)` where `component_games =
     sum(row.games for row in component_rows) // 2` (each game counts both
     participants), plus an explanatory note (models never compared across
     components).
   - Below the table: `no games yet: {list}` when unrated_models non-empty;
     total line `{total_votes} votes counted (blind only|including
     non-blind)`; method footnote: `Bradley-Terry ratings, mean 1000; 95%
     bootstrap CI; * = provisional (<10 games)` with a plain navigation
     link (permitted per DESIGN §13.5 — no subresource load) to
     `https://github.com/Saber5656/mybench/blob/main/docs/research/rating-methodology.md`.
   - Empty (zero games): friendly empty state linking `/vote` and `/tasks`.
2. Model ids/categories are plain autoescaped text (no markdown).
3. No caching; page renders live per request (ADR-003). If render exceeds
   the §9.5 budget in practice, that is Known Unknown U4 — do not add caching
   in this issue.
4. mypy-strict clean.

## Acceptance Criteria

TestClient with vote fixtures (reuse rating test fixtures):

- [ ] Deterministic golden render with `bootstrap_samples=0`: every
      displayed row field equals the corresponding `LeaderboardRow` from a
      direct `compute_leaderboard` call in the test (same args); CI column
      shows `—`; provisional markers correct. CI formatting verified
      separately with a fixed `seed` fixture.
- [ ] Category validation matrix: `all`/empty → unfiltered; valid unknown
      slug → empty state; `x` * 200, `<b>x</b>`, `a' OR 1=1` → 400.
- [ ] Non-blind toggle includes flagged votes (counts change per fixture).
- [ ] Two-component fixture renders two sections with `component_games`
      computed per the formula and the explanatory note.
- [ ] unrated_models listed; empty DB → empty state page.
- [ ] Footnote link href is exactly the documented GitHub URL and is the
      only external URL on the page.
- [ ] `tests.helpers.assert_secure_response(resp, html=True)` passes.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_web_leaderboard.py -q`. Web numbers are asserted
against a direct `compute_leaderboard` call (acceptance criterion 1);
CLI-vs-web consistency on one shared journey is issue 30's job. Optional
manual QA (non-gating): visual pass on demo data.

## Dependencies

15, 22.

## Non-goals

Charts/history-over-time (v2); per-task drill-down (history pages cover raw
data); rating recompute controls.

## Design References

DESIGN §9.6, §11.2, §9.5 (perf), §18 U4; ADR-003.
