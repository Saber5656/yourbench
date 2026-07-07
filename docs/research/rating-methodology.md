# Research: Rating Methodology for Pairwise Preference Leaderboards

Date: 2026-07-08
Status: informs ADR-003, DESIGN.md §9

## 1. Problem

mybench collects blind pairwise votes (`a` / `b` / `tie` / `both_bad`) over model
outputs and must turn them into a ranked leaderboard. Constraints that shape the
choice:

- **Tiny sample sizes.** A single user produces tens to low thousands of votes,
  not millions. The method must behave sanely at n < 100.
- **Order independence.** Votes arrive in bursts (user rates a backlog). The
  rating must not depend on vote order.
- **Recomputability.** The raw vote log is the source of truth; ratings are a
  derived view that can be recomputed with a different algorithm later.
- **No heavy numeric dependencies.** Pure Python preferred (personal-scale data).

## 2. Options considered

| Method | Order-independent | Small-n behavior | Implementation cost | Notes |
|---|---|---|---|---|
| Sequential Elo | No — depends on vote order | Noisy; K-factor tuning needed | Trivial | LMArena originally used Elo, later abandoned it for BT because online updates add path dependence. |
| **Bradley-Terry MLE (chosen)** | Yes — computed from aggregate win matrix | Well-defined MLE; needs regularization for degenerate cases | Small (MM algorithm, ~50 lines pure Python) | What LMArena/Chatbot Arena uses for its published leaderboard (arXiv:2403.04132). |
| TrueSkill / Glicko | Partially (designed for online updates) | Good uncertainty modeling | Medium; less transparent | Overkill; Bayesian machinery not needed for a recompute-from-scratch design. |
| Raw win rate | Yes | Misleading — ignores opponent strength | Trivial | Kept as a display column only, never as the ranking key. |

## 3. Chosen design (specified for implementation in DESIGN.md §9)

### 3.1 Vote → game conversion

- `a` wins → 1 game: winner gets 1 win over loser.
- `tie` and `both_bad` → 1 game counted as half-win for each side (standard
  ties-as-half-games treatment). `both_bad` is additionally surfaced as a
  separate "bad rate" display metric; for rating purposes it is a tie.

### 3.2 Estimator: Bradley-Terry via MM algorithm

Maximum-likelihood BT strengths `p_i` computed with the
Minorization-Maximization iteration (Hunter, "MM algorithms for generalized
Bradley-Terry models", Annals of Statistics 2004):

```
p_i ← W_i / Σ_{j≠i} (n_ij / (p_i + p_j))
```

where `W_i` = total (fractional) wins of model i, `n_ij` = games between i and
j. Normalize `p` (geometric mean = 1) each iteration; stop when max relative
change < 1e-8 or 1000 iterations.

### 3.3 Degenerate cases and regularization

- **Zero-win model diverges** (`p_i → 0`, rating −∞). Fix: for every pair with
  ≥ 1 real game, add **one virtual tied game** (0.5 win each). Bias is small,
  deterministic, and shrinks as real votes accumulate.
- **Disconnected comparison graph** (two groups of models never compared): BT
  ratings are only identifiable within a connected component. Fix: detect
  components (union-find over pairs with ≥ 1 game) and rate/anchor each
  component independently; the UI must label components.

### 3.4 Scale and anchoring

Display ratings on the familiar Elo-like scale:

```
rating_i = 400 · log10(p_i) + C,  C chosen so mean(rating) = 1000 per component
```

### 3.5 Uncertainty

95% confidence intervals via nonparametric bootstrap: resample the vote log
with replacement B times (default B = 200, configurable), recompute BT each
time, take the 2.5/97.5 percentiles per model. This matches LMArena's published
approach and is cheap at personal scale (measured target: < 2 s at 5,000 votes
/ 15 models in pure Python). Models with < 10 games are flagged *provisional*.

## 4. Rejected alternatives (recorded for future reconsideration)

- **Style/length controls** (LMArena's style-controlled BT regression):
  meaningful only at large n; deferred, the raw vote log keeps this possible later (v2+).
- **Per-category separate ratings as the primary view**: v1 computes the global
  leaderboard as default and recomputes BT on a per-category vote subset when a
  category filter is applied. No cross-category blending.

## 5. Sources

- Chatbot Arena: <https://arxiv.org/abs/2403.04132> (BT + bootstrap CI)
- Hunter, D.R. (2004), "MM algorithms for generalized Bradley-Terry models",
  Annals of Statistics 32(1) — MM iteration used above
- LMArena leaderboard methodology notes (Elo → Bradley-Terry transition),
  <https://blog.lmarena.ai/>
