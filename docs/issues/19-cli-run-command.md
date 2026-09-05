# Title

CLI: run

## Summary

Implement `mybench/cli/run_cmd.py`: `mybench run TASK_ID` with model
selection, live progress lines, a result summary table, the votable/not
votable exit-code contract, and `--json` output.

## Context

DESIGN §10.6. This wraps the run engine (13) for terminal use; its exit codes
make mybench scriptable (cron-style batch runs feeding later voting sessions).

## Scope

- `src/mybench/cli/run_cmd.py`
- `tests/test_cli_run.py`

## Detailed Requirements

1. Signature per DESIGN §10.6: `mybench run TASK_ID (--models a,b,c | --all)
   [--temperature F] [--top-p F] [--max-tokens N] [--json]`:
   - Exactly one of `--models` / `--all` required; `--models` parses a
     comma-separated list (trim whitespace, drop empties); `--all` = all
     enabled models in config order.
   - Param flags form the `param_overrides` (issue 13 semantics).
   - **Command-level pre-validation for friendly errors** (DESIGN §10.6:
     task exists, not archived, every id known+enabled, ≥ 2 after dedup) →
     `error: ...` exit 1 **without invoking the runner**; the engine's own
     validation (§7.1) remains the authoritative backstop.
2. Execution: `asyncio.run(runner_fn(...))` with a `progress` callback
   printing to **stderr** as each model finishes:
   `[{done}/{total}] {model_id} ok {latency_s:.1f}s {completion_tokens}tok`
   or `[{done}/{total}] {model_id} FAILED {error}` (error control-stripped,
   truncated 120; `-` substituted for None latency/tokens).
3. Summary to **stdout** after completion (skip in `--json` mode):
   - Table columns `MODEL`, `STATUS`, `LATENCY`, `TOKENS` (completion),
     `ERROR` (empty for succeeded, stripped/truncated 80); `-` for None
     latency/tokens; issue-17 formatting rule.
   - Footer lines: `run {run_id}: {succeeded}/{total} succeeded` and, when
     votable, `vote: mybench serve → http://127.0.0.1:{port}/vote?run={run_id}`
     (port from config), else `not votable: fewer than 2 outputs succeeded`.
4. Exit codes (DESIGN §10.6): 0 when votable; **2** when the run completed
   with < 2 successes; 1 for pre-run errors. `--json` prints the
   `RunSummary` as JSON (`{run_id, task_id, succeeded, failed, votable,
   outputs: [{model_id, status, latency_ms, completion_tokens, error}]}`,
   None → `null`) with the same exit codes.
5. Terminal-safety: progress/summary text via
   `safety.strip_terminal_controls`. JSON mode: values are semantically raw
   but emitted via `json.dumps` (control chars escaped as `\uXXXX`), so
   stdout never carries literal control bytes.
6. Injectable seam (normative): the command function takes
   `runner_fn: RunExecutor = runner.execute_run` where `RunExecutor` is a
   module-level type alias matching `execute_run`'s signature; tests invoke
   the command with a stub via the factory
   `build_run_command(runner_fn=...) -> click.Command` (the default command
   registered in `main.py` uses the default seam).
7. mypy-strict clean.

## Acceptance Criteria

CliRunner tests with stub runner/providers:

- [ ] `--models` parsing (spaces, empties) and `--all` resolution verified by
      asserting the model_ids passed to the runner.
- [ ] Mutually-exclusive/missing selection flags → exit 1 with guidance.
- [ ] Happy path: progress lines on stderr in completion order, summary table
      golden-tested, exit 0, vote URL includes the run id and configured port.
- [ ] Partial failure fixture → failed row rendered, exit 0 while ≥ 2 ok.
- [ ] 1-of-3 success fixture → exit 2 + `not votable` line.
- [ ] Pre-run validation errors (missing task, archived, unknown id,
      disabled id, 1 model) → exit 1, runner spy never invoked.
- [ ] `--json` schema exact (golden), exit codes preserved, nothing else on
      stdout; None fields are `null`.
- [ ] ANSI in stub error → stripped in text output; in JSON mode stdout
      contains no literal `\x1b` byte while the parsed error string equals
      the original.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_cli_run.py -q`; optional manual QA (non-gating):
in a temp `MYBENCH_CONFIG`/`MYBENCH_DATA_DIR` with a config defining two
enabled fake models, run `uv run mybench task add "smoke"` then
`uv run mybench run 1 --all` and expect exit 0 with a 2-row summary.

## Dependencies

13, 16 (18 practically precedes for creating tasks).

## Non-goals

`--then-vote` browser opening (only `serve --open` exists, §10.7); parallel
multi-task batch runs; retry flags (engine policy is fixed, §6.8).

## Design References

DESIGN §10.6, §7, §13.1 B3.
