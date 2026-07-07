# Title

Web: runs list, votes history, vote delete

## Summary

Implement `web/routes/history.py` + `templates/runs_list.html`,
`votes_list.html`: the browsing surfaces for past runs and votes, including
vote deletion (the one destructive action in the UI).

## Context

DESIGN §11.2 rows for `/runs`, `/votes`, `/votes/{id}/delete`. Data ownership
(§1) requires the user to see and prune their vote history; ratings follow
automatically (ADR-003).

## Scope

- `src/mybench/web/routes/history.py`
- `src/mybench/web/templates/runs_list.html`, `votes_list.html`
- `tests/test_web_history.py`

## Detailed Requirements

1. `GET /runs`:
   - `runs.list_runs(conn, limit=50)` table: run id (link), task display
     title (link to task), category, status, `succeeded/total` (+ `failed`
     count when > 0), votable badge, blind/revealed badge, created.
   - Filter `?task={id}` passthrough to the repo; header shows the task
     scope with a clear-filter link.
   - Empty state message.
2. `GET /votes`:
   - `votes.list_votes(conn, limit=100)` table: vote id, created, task title
     (link), category, `model_a` vs `model_b` (models ARE shown — the vote
     already happened; §8.5 does not apply retroactively), outcome rendered
     as the winning model id / `tie` / `both bad`, blind badge (`non-blind`
     highlighted), note (plain autoescaped text, truncated 80 with full text
     in `title` attribute), delete button.
   - Delete button: inline form `POST /votes/{id}/delete` with CSRF +
     `onsubmit`-free confirm: v1 uses a plain submit (no JS confirm — CSP
     forbids inline handlers; an accidental delete is recoverable only by
     re-voting, accepted for personal tool) — the button label is
     `delete` with `aria-label="delete vote {id}"`.
   - Empty state message.
3. `POST /votes/{id}/delete` (CSRF): `votes.delete`; missing id → 404;
   success → 303 `/votes`.
4. Both pages: no model output content is rendered here (lists only) — no
   `markdown_safe` needed; all text autoescaped; notes are user text
   (autoescape suffices, consistent with §12 which mandates the pipeline for
   *rendered-as-markdown* content only — notes are shown as plain text).
5. mypy-strict clean.

## Acceptance Criteria

TestClient:

- [ ] Runs list fixture renders counts/badges/links; task filter works;
      newest first; limit respected.
- [ ] Votes list renders all outcome variants (a/b/tie/both_bad), blind vs
      non-blind badges, truncated note with full title attr; XSS note
      appears escaped.
- [ ] Delete: CSRF-less POST → 403; valid POST removes the row → 303; the
      pair becomes least-voted again (next `/vote` GET serves it — cross-
      check with issue 09 scheduling); missing id → 404.
- [ ] Deleting a vote changes the leaderboard on next request (integration
      assertion via compute_leaderboard before/after).
- [ ] Security helper assertions pass on both pages.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_web_history.py -q`; manual browse-and-delete pass.

## Dependencies

08, 09, 22, 23 (23 nominally — only if any content rendering sneaks in;
otherwise autoescape only).

## Non-goals

Run detail page (26); bulk deletion; vote editing (delete + re-vote instead);
run deletion (v1 non-goal).

## Design References

DESIGN §11.2, §8.5 (post-vote reveal semantics), §1 (data ownership);
ADR-003, ADR-006.
