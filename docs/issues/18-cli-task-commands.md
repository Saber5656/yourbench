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
- `tests/test_cli_task.py`

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
     safety) — a `ValueError` prints each violation on its own line prefixed
     `error: ` and exits 1.
   - Success: `task {id} created` (or `--json` → `{"id": N}` only).
2. `mybench task list [--category C] [--archived] [--json]`:
   - Columns: `ID`, `TITLE` (display_title, control-stripped, truncated 60),
     `CATEGORY`, `RUNS` (run_counts), `CREATED` (date part only).
   - `--archived` includes archived rows and adds an `ARCHIVED` column.
   - Empty → `no tasks yet; add one with 'mybench task add'` stderr, exit 0.
   - `--json`: full field objects (untruncated, raw text).
3. `mybench task show ID [--json]`:
   - Missing id → `error: task {id} not found` exit 1.
   - Prints: id, title, category, created, archived (if set), params (only
     non-None), then `--- system ---` block (when present) and
     `--- prompt ---` block; **both bodies through
     `safety.strip_terminal_controls`** (B3).
   - `--json`: full task object with raw (unstripped) text — JSON is not a
     terminal-rendering context, and consumers need fidelity.
4. mypy-strict clean.

## Acceptance Criteria

CliRunner tests:

- [ ] add via positional / `--file` / piped stdin all create identical tasks
      (fixture prompt with unicode + newlines round-trips).
- [ ] Multiple-source and no-source-TTY errors exact.
- [ ] Validation failure (oversized prompt, bad category, temperature 9)
      → exit 1 listing each violation.
- [ ] `--json` outputs parse and contain only documented keys.
- [ ] list: category filter, archived flag, run counts (fixture with runs),
      truncation at 60 chars, empty-state message.
- [ ] show: golden output for a full-featured task; ANSI in prompt body
      stripped in text mode but preserved in `--json`.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_cli_task.py -q`; manual:
`echo "explain X" | uv run mybench task add --category demo`.

## Dependencies

07, 16.

## Non-goals

task edit/delete/archive from the CLI (archive is a web-only action in v1 —
route table DESIGN §11.2); run triggering (19).

## Design References

DESIGN §10.5, §5.4, §13.1 B3.
