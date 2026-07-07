# Title

Bootstrap CIs and leaderboard assembly

## Summary

Implement `mybench/rating/bootstrap.py` (percentile bootstrap over the game
log) and `mybench/rating/leaderboard.py` (`LeaderboardRow` /
`Leaderboard` assembly with rates, provisional flags, components, filters),
the shared computation used by web and CLI.

## Context

DESIGN §9.5–9.6 and ADR-003 (no caching; recompute per request; performance
target). Consumes `votes.games()` (09) and `fit_bradley_terry` (14).

## Scope

- `src/mybench/rating/bootstrap.py`
- `src/mybench/rating/leaderboard.py`
- `tests/test_bootstrap.py`, `tests/test_leaderboard.py`

## Detailed Requirements

1. `bootstrap.py`: `def bootstrap_ci(games: Sequence[Game], *,
   samples: int = 200, seed: int | None = None, confidence: float = 0.95)
   -> dict[str, tuple[int, int]]` per DESIGN §9.5:
   - Parameter validation: `samples < 0` or not `0 < confidence < 1` →
     `ValueError`. `samples == 0` or `len(games) == 0` → `{}`.
   - `rng = random.Random(seed)`; resample `len(games)` games with
     replacement; refit (`fit_bradley_terry` defaults) + components +
     display scale per resample (issue 14's `connected_components` /
     `to_display_ratings`); models absent from a resample are skipped for
     that sample.
   - Per model with ≥ 1 sampled rating: the DESIGN §9.5 percentile
     algorithm verbatim (sorted values, `k = q*(n-1)`, linear interpolation,
     `int(round(v))`); `n == 1` → that value for both bounds.
2. `leaderboard.py`: implement `LeaderboardRow` and `Leaderboard` **exactly**
   as the DESIGN §9.6 dataclasses (field names/types normative;
   `unrated_models` sorted lexically), and
   `def compute_leaderboard(conn, *, category: str | None = None,
   include_nonblind: bool = False, bootstrap_samples: int = 200,
   seed: int | None = None, known_model_ids: Sequence[str] | None = None)
   -> Leaderboard`:
   - games via `votes.games(conn, category=..., include_nonblind=...)`.
   - fit via issue 14's `fit_bradley_terry`; components via
     `connected_components`; display ratings via `to_display_ratings`;
     CIs via `bootstrap_ci`.
   - Per-model tallies from the games list using `Game.kind` (DESIGN §9.1).
     For model m: `games(m)` = games where m participates; `wins(m)` = sum
     of m's scores (`score_lo` when m is `model_lo`, else `1 - score_lo`);
     `decisive_games(m)` = games with `kind == "decisive"` involving m;
     `decisive_wins(m)` = decisive games m won (score 1.0);
     `win_rate = decisive_wins / decisive_games` (`None` when 0 decisive);
     `tie_rate = kind=="tie" games / games(m)`;
     `both_bad_rate = kind=="both_bad" games / games(m)`.
   - `provisional = games < 10`; `component` index per DESIGN §9.6; rows
     sorted rating desc within component, components ordered by size desc
     then min model id; `known_model_ids` (callers pass config model ids)
     yields `unrated_models` = ids with zero games, sorted.
   - `total_votes` = number of games after filters, before regularization.
3. Performance test, marked `slow` but included in the default pytest run
   (it completes in seconds; no CI-workflow change needed): synthetic 5,000
   games / 15 models / samples=200. Assert `< 4 s` (runner-variance guard,
   DESIGN §9.5) and print the measured time; a measured local value above
   2 s must be reported in the PR as a Known Unknown U4 trigger, not
   silently absorbed.
4. mypy-strict clean; no third-party imports.

## Acceptance Criteria

- [ ] Deterministic: same seed → identical CIs (run twice); `samples=0` →
      no CIs and `ci_low/ci_high = None` in rows; `samples=-1` /
      `confidence=1.5` → `ValueError`.
- [ ] CI sanity: for a lopsided fixture (A beats B 20-0), A's `ci_low` >
      B's `ci_high`.
- [ ] Rates: fixture A-vs-B with A winning 2 decisive, 1 tie, 1 both_bad →
      A: games 4, wins 3.0, win_rate 1.0, tie_rate 0.25, both_bad_rate 0.25;
      B: games 4, wins 1.0, win_rate 0.0, same rates; `total_votes == 4`
      (exact expected table asserted).
- [ ] Blind filter: non-blind votes excluded by default, included with flag
      (fixture from issue 09 semantics).
- [ ] Category filter verified via games() passthrough.
- [ ] Components: disjoint fixture produces component indices and independent
      1000-means; sort order per requirement 2.
- [ ] `known_model_ids` yields `unrated_models` for gameless models.
- [ ] Perf test passes and prints measured duration.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_bootstrap.py tests/test_leaderboard.py -q` and the
`slow` perf test via `uv run pytest -m slow -q`.

## Dependencies

09, 14.

## Non-goals

HTTP/CLI rendering (21, 28); caching (ADR-003 defers until U4 triggers).

## Design References

DESIGN §9.5, §9.6; ADR-003; §18 U4.
