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

1. Signature: `mybench run TASK_ID (--models a,b,c | --all) [--temperature F]
   [--top-p F] [--max-tokens N] [--json]`:
   - Exactly one of `--models` / `--all` required; `--models` parses a
     comma-separated list (trim whitespace, drop empties); `--all` = all
     enabled models in config order.
   - Param flags form the `param_overrides` (issue 13 semantics).
   - Task lookup failure → `error: task {id} not found` exit 1; archived task
     / unknown/disabled model / < 2 models → the runner's `ValueError`
     message with exit 1 (no run row created — engine guarantee).
2. Execution: `asyncio.run(execute_run(...))` with a `progress` callback
   printing to **stderr** as each model finishes:
   `[{done}/{total}] {model_id} ok {latency_s:.1f}s {completion_tokens}tok`
   or `[{done}/{total}] {model_id} FAILED {error}` (error control-stripped,
   truncated 120).
3. Summary to **stdout** after completion (skip in `--json` mode):
   - Table columns `MODEL`, `STATUS`, `LATENCY`, `TOKENS` (completion),
     `ERROR` (empty for succeeded, stripped/truncated 80).
   - Footer lines: `run {run_id}: {succeeded}/{total} succeeded` and, when
     votable, `vote: mybench serve → http://127.0.0.1:{port}/vote?run={run_id}`
     (port from config), else `not votable: fewer than 2 outputs succeeded`.
4. Exit codes (DESIGN §10.6): 0 when votable; **2** when the run completed
   with < 2 successes; 1 for pre-run errors. `--json` prints the
   `RunSummary` as JSON (`{run_id, task_id, succeeded, failed, votable,
   outputs: [{model_id, status, latency_ms, completion_tokens, error}]}`)
   with the same exit codes.
5. Progress/summary text via `safety.strip_terminal_controls`; JSON raw.
6. Testability: the command must accept an injectable runner
   (`runner_fn=execute_run` default) so tests use stub engines without
   network; provider stubs via issue 13's seams otherwise.
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
- [ ] Pre-run validation errors → exit 1, runner not invoked past validation
      (spy assertion).
- [ ] `--json` schema exact (golden), exit codes preserved, nothing else on
      stdout.
- [ ] ANSI in stub error → stripped in text output, raw in JSON.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_cli_run.py -q`; manual e2e with the fake provider:
`uv run mybench run 1 --all` on a fake-only config.

## Dependencies

13, 16 (18 practically precedes for creating tasks).

## Non-goals

`--then-vote` browser opening (only `serve --open` exists, §10.7); parallel
multi-task batch runs; retry flags (engine policy is fixed, §6.8).

## Design References

DESIGN §10.6, §7, §13.1 B3.
