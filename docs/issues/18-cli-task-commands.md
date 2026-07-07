# Title

CLI: task add / list / show

## Summary

Implement `mybench/cli/task_cmd.py`: `mybench task add` with the three prompt
sources (arg / file / stdin) and param options, `task list` with filters, and
`task show`.

## Context

DESIGN §10.5. `task add` is the primary CLI ingestion path (Q2-A); stdin
support is what lets users pipe real prompts in from their workflow.

## Scope

- `src/mybench/cli/task_cmd.py`
- `src/mybench/cli/main.py` (one `cli.add_command(task)` registration line
  per issue 16's contract)
- `tests/test_cli_task.py` (+ golden fixtures
  `tests/fixtures/golden_task_list.txt`, `golden_task_show.txt`)

All commands run through issue 16's `require_context()` (config load,
migrations, orphan sweep — DESIGN §10.1); do not reimplement startup here.

## Detailed Requirements

1. `mybench task add [PROMPT] [--file PATH | --stdin] [--title T]
   [--category C] [--system TEXT | --system-file PATH] [--temperature F]
   [--top-p F] [--max-tokens N] [--json]`:
   - Prompt source resolution (DESIGN §10.5): more than one of
     {positional, `--file`, `--stdin`} → `error: provide the prompt via
     exactly one of: argument, --file, --stdin` exit 1. None given: if stdin
     is not a TTY → read stdin; else error with the same guidance.
   - `--file`: read UTF-8 (errors → exit 1 with message); same for
     `--system-file`; `--system` and `--system-file` are mutually exclusive.
   - Inputs decoded then passed to `db.tasks.create` (which enforces §5.4 via
     the issue-06 validator) — a `ValueError` prints each violation on its
     own line prefixed `error: ` and exits 1.
   - Success: `task {id} created` (or `--json` → exactly `{"id": N}`).
2. `mybench task list [--category C] [--archived] [--json]`:
   - Columns: `ID`, `TITLE` (display_title, control-stripped, truncated to
     60 chars **including** the `…` marker via `safety.truncate(s, 60,
     "…")`), `CATEGORY`, `RUNS` (via `tasks.run_counts`, issue 07),
     `CREATED` (date part of created_at, `YYYY-MM-DD`). Formatting rule =
     issue 17's (left-aligned, two-space join, no trailing spaces); golden
     fixture `golden_task_list.txt`.
   - `--archived` includes archived rows and adds an `ARCHIVED` column
     (date part or `-`).
   - Empty → `no tasks yet; add one with 'mybench task add'` stderr, exit 0.
   - `--json` (aligned with the DESIGN §10.5 list fields): array of
     `{"id": int, "title": str | null, "display_title": str,
     "category": str, "created_at": str, "archived_at": str | null,
     "runs": int}` — no prompt bodies in list JSON (use `task show`).
   - Titles are user text: control-stripped in text mode, raw in JSON.
3. `mybench task show ID [--json]`:
   - `ID` is a required int argument (missing/non-int → click usage error,
     exit 2, click's standard message); **nonexistent** id →
     `error: task {id} not found` exit 1.
   - Text layout (golden fixture `golden_task_show.txt`): one `{label}:
     {value}` line each for id, title (omit when NULL), category, created,
     archived (omit when NULL), params (compact `k=v` space-joined, only
     non-None, omit line when all None), then `--- system ---` block (when
     present) and `--- prompt ---` block. **Title and both bodies through
     `safety.strip_terminal_controls`** (B3).
   - `--json`: full task object (`id, title, category, system_prompt,
     user_prompt, params{...}, created_at, archived_at`) with raw
     (unstripped) text — JSON is not a terminal-rendering context, and
     consumers need fidelity.
4. mypy-strict clean.

## Acceptance Criteria

CliRunner tests:

- [ ] `mybench task --help` works from the root group (registration line).
- [ ] add via positional / `--file` / piped stdin all create identical tasks
      (fixture prompt with unicode + newlines round-trips).
- [ ] Multiple-source and no-source-TTY errors exact.
- [ ] Validation failure (oversized prompt, bad category, temperature 9)
      → exit 1 listing each violation on its own `error: ` line.
- [ ] `--json` outputs equal the exact documented objects (full comparison)
      for add/list/show; list JSON contains no prompt bodies.
- [ ] list: golden fixture match; category filter; archived flag adds
      column; run counts from a fixture with runs; truncation at 60 chars
      incl. marker; ANSI in a title stripped in text mode; empty-state
      message.
- [ ] show: golden fixture match for a full-featured task; nonexistent id →
      exit 1 with exact message; missing arg → click usage error (exit 2);
      ANSI in title and prompt body stripped in text mode but preserved in
      `--json`.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_cli_task.py -q`; optional manual QA (non-gating):
`echo "explain X" | uv run mybench task add --category demo`.

## Dependencies

07, 16.

## Non-goals

task edit/delete/archive from the CLI (archive is a web-only action in v1 —
route table DESIGN §11.2); run triggering (19).

## Design References

DESIGN §10.5, §5.4, §13.1 B3.
