# Title

Web: task list/create/detail/archive pages

## Summary

Implement `web/routes/tasks.py` + templates: task list with filters, the
new-task form (POST create with validation re-render), task detail with
prompt display and the run-trigger form, and archive/unarchive actions.

## Context

DESIGN §11.2 rows 2–6. The task detail page hosts the run-trigger form whose
POST handler is issue 26 — this issue renders the form; 26 consumes it.

## Scope

- `src/mybench/web/routes/tasks.py`
- `templates/tasks_list.html`, `task_form.html`, `task_detail.html`
- `tests/test_web_tasks.py`

## Detailed Requirements

1. `GET /tasks`: table (id, display title, category, run count, created,
   archived badge) via `tasks.list_` + `tasks.run_counts`; filters
   `?category=` (exact) and `?archived=1`; category chips linking to the
   filter; `New task` button → `/tasks/new`; empty state text.
2. `GET /tasks/new`: form fields (names exact — the POST contract):
   `title`, `category` (text input, placeholder `general`), `system_prompt`
   (textarea), `user_prompt` (textarea, required), `temperature`, `top_p`,
   `max_tokens` (number inputs, blank = unset), `csrf_token` hidden.
3. `POST /tasks` (CSRF-protected via `Depends(csrf_protect)`, issue 22):
   - Normalization (exact): strip all fields; `title`/`system_prompt` empty
     → `None`; `category` empty → `"general"`; numeric fields empty →
     `None`, else parse (`float` for temperature/top_p, `int` for
     max_tokens) — unparsable value adds violation `"{field}: must be a
     number"` instead of raising.
   - Validation-first flow: run `safety.validate_task_params(...)` (issue 06
     contract) plus the parse violations; any violations → re-render
     `task_form.html` with status 400, field values preserved, ordered
     violations listed in a `.banner`, nothing persisted. Only when clean →
     `tasks.create` (its own `ValueError` is a defensive 400 via the same
     re-render).
   - Success → 303 `/tasks/{id}`.
4. `GET /tasks/{id}`:
   - Full prompt display: system + user prompts rendered via
     `markdown_safe` inside cards **plus** raw `<details><pre>` via
     `escape_pre` (user text is semi-trusted but rendered like model text —
     one pipeline, DESIGN §12).
   - Params table (only non-None); created/archived stamps; runs-of-task
     table via `runs.list_runs(conn, task_id=id)` (issue 08's
     `RunListItem`: status, `succeeded/total`, votable badge from
     `.votable`, created, link to `/runs/{id}`).
   - Run trigger form (rendered here, handled by issue 26):
     `POST /tasks/{id}/runs`, checkbox per **enabled** config model
     (name `models`, value = model id; disabled models not listed), optional
     override inputs `temperature`, `top_p`, `max_tokens`, `csrf_token`.
     When fewer than 2 enabled models exist, render exactly the text
     `at least 2 enabled models are required to run a comparison — edit
     your config and run 'mybench models check'` instead of the form (no
     external links).
   - Archive/unarchive button (`POST /tasks/{id}/archive` / `.../unarchive`,
     CSRF) with archived state banner; archived tasks show no run form.
   - 404 for unknown id via issue 22 error page.
5. `POST /tasks/{id}/archive|unarchive`: repo call, 303 back to
   `/tasks/{id}`; unknown id → 404.
6. mypy-strict clean.

## Acceptance Criteria

TestClient:

- [ ] List filters/badges/empty state (fixtures); XSS title escaped.
- [ ] Create happy path → 303, row exists; normalization verified (empty
      strings → None/general; `"0.7"` parses; empty `max_tokens` → NULL).
- [ ] Create with junk number / oversized prompt / bad category → 400
      re-render, values preserved, all violations listed in order, nothing
      persisted (task count unchanged).
- [ ] Create without CSRF token / with bad Origin → 403, nothing persisted.
- [ ] Detail: prompt with `<script>` markdown renders sanitized (no
      `<script` in body); raw details block present (escaped); params table
      only non-None; runs table matches a `RunListItem` fixture incl.
      votable badge.
- [ ] Run form lists exactly enabled models; <2 enabled → no form, the exact
      guidance text shown; archived task → no form.
- [ ] Archive→unarchive round-trip via POSTs (with CSRF) reflected in UI.
- [ ] `tests.helpers.assert_secure_response(resp, html=True)` passes on all
      three pages (list/form/detail), which also enforces no inline
      script/style/handlers in the new templates.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_web_tasks.py -q`; manual create-and-browse with
fake config.

## Dependencies

07, 08 (`runs.list_runs` for the runs-of-task table), 22, 23.

## Non-goals

Run POST handling and status polling (26); task deletion (v1 non-goal); task
editing (v1 non-goal — archive & recreate).

## Design References

DESIGN §11.2 rows 2–6, §5.4, §12, §13.4.
