# Title

Web: dashboard page

## Summary

Implement `web/routes/dashboard.py` + `templates/dashboard.html`: the `GET /`
overview with counts, recent runs, and the "Start voting" call to action.

## Context

DESIGN §11.2 row 1. The dashboard is the landing page after `mybench serve`;
its job is orientation: what exists, what needs votes, where to go.

## Scope

- `src/mybench/web/routes/dashboard.py`
- `src/mybench/web/templates/dashboard.html`
- `tests/test_web_dashboard.py`

## Detailed Requirements

1. Route `GET /` (replaces issue 22's placeholder):
   - Stats row: active task count (`tasks.list_` non-archived), total runs,
     total votes (blind + non-blind shown as `N (+M non-blind)` when M > 0),
     and **unvoted pair count** via `votes.unvoted_pair_count(conn)`.
   - CTA block: when unvoted pairs > 0 → prominent link `Start voting
     ({n} pairs waiting)` → `/vote`; when 0 and tasks exist → link to
     `/tasks` (`run a task to create new pairs`); when no tasks → onboarding
     block: 3 numbered steps (add models via config, `mybench task add` or
     web form link `/tasks/new`, run + vote) — static text, no shell
     execution of course.
   - Recent runs: last 5 via `runs.list_runs(conn, limit=5)` — table with
     task display title (control chars are impossible in HTML context but
     titles pass through Jinja autoescape; no markdown), status,
     `succeeded/total`, created date, link to `/runs/{id}`.
2. Titles/categories rendered as plain autoescaped text (no `markdown_safe` —
   dashboard shows no model output; keep this page free of issue 23 dep).
3. All queries via existing repos; no SQL in the route.
4. mypy-strict clean.

## Acceptance Criteria

TestClient with seeded fixture DB:

- [ ] Counts correct for a fixture (3 tasks 1 archived, 2 runs, 5 votes 1
      non-blind, 2 unvoted pairs) — assert rendered numbers.
- [ ] CTA states: pairs-waiting / no-pairs-but-tasks / empty-onboarding each
      rendered (three fixtures).
- [ ] Recent runs table rows link to run pages; limited to 5; newest first.
- [ ] Task title containing `<b>evil</b>` appears escaped (source contains
      `&lt;b&gt;`).
- [ ] Page passes the shared security assertions (headers present, no inline
      script) — reuse issue 22's helper.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_web_dashboard.py -q`; manual look at
`http://127.0.0.1:8137/` with the fake-provider demo flow.

## Dependencies

09, 22 (07/08 transitively for repos).

## Non-goals

Charts/graphs; per-category breakdowns (leaderboard page's job); task/run
creation from the dashboard.

## Design References

DESIGN §11.2 (route table), §8.3 (unvoted count).
