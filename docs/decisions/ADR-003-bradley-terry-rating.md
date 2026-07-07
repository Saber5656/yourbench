# ADR-003: Bradley-Terry rating with virtual-tie regularization; raw votes are the source of truth

Date: 2026-07-08
Status: Accepted

## Context

Blind pairwise votes must become a leaderboard. Owner selected Bradley-Terry
recomputation over sequential Elo (Q5-A). Personal-scale data (tens to low
thousands of votes) makes order-independence and small-sample stability the
dominant concerns. Comparison of methods: `docs/research/rating-methodology.md`.

## Decision

1. **Votes are append-only source of truth.** The `votes` table stores every
   vote verbatim (`a`/`b`/`tie`/`both_bad`). Ratings are always recomputed from
   the full vote log; no incremental rating state is persisted.
2. **Estimator:** Bradley-Terry maximum likelihood via Hunter's MM algorithm,
   pure Python. Ties and `both_bad` count as half-win for each side.
3. **Regularization:** one virtual tied game (0.5/0.5) added per model pair
   that has at least one real game — prevents zero-win divergence
   deterministically.
4. **Identifiability:** connected components of the comparison graph are rated
   and anchored independently (mean rating 1000, scale `400·log10`); the UI
   labels components.
5. **Uncertainty:** 95% bootstrap confidence intervals (default B=200,
   configurable); models with < 10 games are flagged *provisional*.
6. **No caching in v1:** every leaderboard render recomputes from votes.
   Performance target: < 2 s at 5,000 votes / 15 models / B=200.

## Consequences

- Vote deletion (user owns their data) is trivially consistent — the next
  recompute reflects it.
- The rating algorithm can be replaced later (style control, category
  weighting) without data migration.
- `win_rate` is display-only and never the ranking key.
- Bootstrap cost grows linearly with votes × B; if the performance target
  fails, lower default B or add a cache keyed on the vote log state — decision
  deferred until measured (DESIGN.md §18).
