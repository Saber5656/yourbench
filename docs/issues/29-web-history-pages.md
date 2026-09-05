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
   - `runs.list_runs(conn, limit=50)` (issue 08's `RunListItem` — this
     issue adds no repo SQL) table: run id (link), task display title (link
     to task), category, **models** (`model_ids` joined by `, ` — ids only,
     no mapping to content; DESIGN §11.2), status, `succeeded/total`
     (+ `failed` count when > 0), votable badge, blind/revealed badge,
     created.
   - Filter `?task={id}`: non-integer value → 400 error page; integer but
     nonexistent task → 404; valid → passthrough to the repo, header shows
     the task scope with a clear-filter link.
   - Empty state message.
2. `GET /votes`:
   - `votes.list_votes(conn, limit=100)` (issue 09's `VoteListItem` — no
     new repo SQL) table: vote id, created, task title (link), category,
     `model_a` vs `model_b` (models ARE shown — the vote already happened;
     §8.5 does not apply retroactively), outcome rendered as the winning
     model id / `tie` / `both bad`, blind badge (`non-blind` highlighted),
     note (scalar metadata → plain autoescaped text per DESIGN §12,
     truncated 80 with full text in the `title` attribute — attribute
     context is autoescaped by Jinja too), delete button.
   - Delete button: inline form `POST /votes/{id}/delete` with CSRF +
     `onsubmit`-free confirm: v1 uses a plain submit (no JS confirm — CSP
     forbids inline handlers; an accidental delete is recoverable only by
     re-voting, accepted for personal tool) — the button label is
     `delete` with `aria-label="delete vote {id}"`.
   - Empty state message.
3. `POST /votes/{id}/delete` (CSRF): `votes.delete`; missing id → 404;
   success → 303 `/votes`.
4. Both pages: no content bodies are rendered here (lists only), so
   `web.render` is not imported; all text is scalar metadata under the
   DESIGN §12 policy (Jinja autoescape).
5. mypy-strict clean.

## Acceptance Criteria

TestClient:

- [ ] Runs list fixture renders models column (ids only), counts, badges,
      links; newest first; limit respected; `?task=` matrix: valid filter
      works with clear-filter link, non-int → 400, unknown int → 404.
- [ ] Votes list renders all outcome variants (a/b/tie/both_bad), blind vs
      non-blind badges, truncated note with full title attr; XSS note
      appears escaped in both text and attribute contexts.
- [ ] Delete: CSRF-less/bad-Origin POST → 403 and row still present; valid
      POST removes the row → 303; repo-level assertion that the deleted
      pair's vote count dropped (`votes.unvoted_pair_count` +1 for a
      fixture where that pair was the only voted one); missing id → 404.
- [ ] Deleting a vote shrinks `votes.games(conn)` by exactly one game
      (before/after assertion — rating-level effects are covered by
      ADR-003 recomputation and issue 30).
- [ ] `tests.helpers.assert_secure_response(resp, html=True)` passes on
      `/runs` and `/votes`.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_web_history.py -q`; manual browse-and-delete pass.

## Dependencies

08, 09, 22. (23 is not a dependency: these pages render scalar metadata
only, per DESIGN §12.)

## Non-goals

Run detail page (26); bulk deletion; vote editing (delete + re-vote instead);
run deletion (v1 non-goal).

## Design References

DESIGN §11.2, §8.5 (post-vote reveal semantics), §1 (data ownership);
ADR-003, ADR-006.
