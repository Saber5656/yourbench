# Title

Bradley-Terry MM estimator with regularization and components

## Summary

Implement `mybench/rating/bradley_terry.py`: pure-Python Bradley-Terry
maximum-likelihood fitting via Hunter's MM algorithm, virtual-tie
regularization, connected-component detection, and the display-scale
conversion.

## Context

ADR-003 and DESIGN §9.2–9.4 fix the algorithm; the math rationale and the
degenerate cases it must survive are in `docs/research/rating-methodology.md`.
This module is deliberately dependency-free and independently testable with
hand-computed fixtures.

## Scope

- `src/mybench/rating/bradley_terry.py`
- `tests/test_bradley_terry.py`

## Detailed Requirements

1. Input type: reuse `Game` from `mybench.db.votes` (`model_lo`, `model_hi`,
   `score_lo` ∈ {1.0, 0.0, 0.5}; the `kind` field is ignored by fitting).
   "No third-party imports" below means exactly that — this first-party
   type import is expected (dependency on issue 09).
2. `def fit_bradley_terry(games: Sequence[Game], *,
   regularization_virtual_ties: float = 1.0, max_iter: int = 1000,
   tol: float = 1e-8) -> dict[str, float]`:
   - Aggregate wins `w[i][j]` (fractional; tie adds 0.5 to each direction)
     and game counts `n[i][j]`.
   - Regularization (DESIGN §9.2): for each unordered pair with
     `n[i][j] > 0`, add `regularization_virtual_ties` tied games:
     `w[i][j] += 0.5 * r; w[j][i] += 0.5 * r`; game counts stay symmetric —
     increment **both** `n[i][j]` and `n[j][i]` by `r` (or store `n` keyed
     by the unordered pair and read symmetrically).
   - MM update per DESIGN §9.2; renormalize strengths to geometric mean 1
     **per component** each sweep; converge when
     `max_i |p_i' − p_i| / p_i < tol`; raise `RuntimeError` if not converged
     after `max_iter` (with the achieved delta in the message).
   - Return strengths for every model appearing in ≥ 1 game.
3. `def connected_components(games: Sequence[Game]) -> list[set[str]]` —
   union-find; deterministic order (components sorted by min member;
   members are sets).
4. `def to_display_ratings(strengths: dict[str, float],
   components: list[set[str]]) -> dict[str, int]` — per component:
   `400 * log10(p) + C` with `C` s.t. arithmetic mean = 1000; convert with
   Python's built-in `round()` (DESIGN §9.4).
5. Edge cases: empty games → `{}` / `[]`; single pair; a model that lost
   every real game must still get a finite rating (regularization proof).
6. Pure stdlib (`math`, `collections`); no numpy (ADR-002). mypy-strict.

## Acceptance Criteria

- [ ] **Known-answer test**: for the 2-model fixture `w_AB=3, w_BA=1`
      (no regularization, `regularization_virtual_ties=0`), fitted odds
      satisfy `p_A/p_B ≈ 3` within 1e-6 (analytic BT solution).
- [ ] Symmetry: A beats B twice, B beats A twice → equal strengths; 3-model
      rock-paper-scissors cycle (1 win each way) → all equal within 1e-6.
- [ ] Order invariance: shuffled game list yields identical strengths.
- [ ] Zero-win model with default regularization gets a finite strength
      strictly below opponents'.
- [ ] Transitive fixture A>B>C (A beats B 5-0, B beats C 5-0): strengths
      strictly ordered A > B > C.
- [ ] Components: {A,B} and {C,D} disjoint fixtures → two components; each
      anchored to mean 1000 independently in display ratings.
- [ ] Display scale: for strengths `{A: 3.0, B: 1.0}` normalized to
      geometric mean 1 in one component, ratings are
      `A = round(1000 + 200*log10(3)) = 1095`,
      `B = round(1000 - 200*log10(3)) = 905` (document the arithmetic in
      the test).
- [ ] Convergence guard: the `w_AB=3, w_BA=1` fixture with
      `regularization_virtual_ties=0, tol=1e-8, max_iter=1` raises
      `RuntimeError` whose message contains the achieved delta.
- [ ] ruff, mypy strict, pytest green; module imports nothing outside the
      stdlib and `mybench.db.votes` (the `Game` type).

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_bradley_terry.py -q`.

## Dependencies

01, 09 (the `Game` type import).

## Non-goals

Bootstrap CIs and leaderboard rows (15); style/length controls (v2); caching.

## Design References

DESIGN §9.1–9.4; ADR-003; research/rating-methodology.md.
