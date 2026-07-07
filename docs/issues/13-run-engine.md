# Title

Run execution engine

## Summary

Implement `mybench/runner.py`: `execute_run()` orchestrating concurrent
provider calls with semaphore, per-model timeout, partial-failure tolerance,
per-output DB transitions, and a progress callback; plus the `RunSummary` /
`OutputEvent` types.

## Context

DESIGN §7 end to end. Both the CLI `run` command (19) and the web runs manager
(26) call this one function; correctness of terminal states determines what is
votable.

## Scope

- `src/mybench/runner.py`
- `tests/test_runner.py`

## Detailed Requirements

1. Types (frozen dataclasses):
   - `OutputEvent(model_id: str, status: Literal["succeeded","failed"],
     latency_ms: int | None, completion_tokens: int | None, error: str | None)`
   - `RunSummary(run_id: int, task_id: int, succeeded: int, failed: int,
     outputs: list[OutputEvent])` with property `votable -> bool`
     (`succeeded >= 2`).
2. `async def execute_run(conn, config, *, task: Task, model_ids:
   list[str], param_overrides: TaskParams | None = None, progress:
   Callable[[OutputEvent], None] | None = None,
   on_created: Callable[[int], None] | None = None,
   provider_factory=providers.build) -> RunSummary` (DESIGN §7.1–7.2):
   - Validation (raise `ValueError`): ≥ 2 distinct model ids after dedup
     (order preserved); every id resolves via `config.model_by_id` and is
     `enabled`; task not archived.
   - Effective params: task.params fields overridden by non-None
     `param_overrides` fields; validate via `safety.validate_task_params`
     (prompt fields from task) → violations raise `ValueError`.
   - Snapshot JSON per model (DESIGN §4.3 outputs): compact JSON
     `{"provider","base_url","model","params":{...effective, omit None}}`.
   - Create run + pending outputs via `runs.create_with_outputs`; the
     returned `Run.outputs` tuple (creation order = model order) is the
     model→output-id mapping used for `finish_output` calls — no ad-hoc SQL
     in the runner. Fire `on_created(run_id)` immediately after (the web
     runs manager, issue 26, uses this to learn the id before execution
     finishes).
   - Per model coroutine: `provider_factory(model_cfg, settings)` (errors
     from build, e.g. missing env var, are recorded as that output's
     failure, not raised); build
     `request = CompletionRequest(system_prompt=task.system_prompt,
     user_prompt=task.user_prompt, temperature=eff.temperature,
     top_p=eff.top_p, max_tokens=eff.max_tokens)` (from
     `mybench.providers.base`); `async with asyncio.timeout(
     settings.timeout_seconds)` around `await provider.complete(request)`;
     `finally: await provider.aclose()`. Success →
     `runs.finish_output(succeeded, ...)`; `ProviderError` → failed with
     `error=e.message` (the sanitized contract field, §6.2 — not `str(e)`);
     `TimeoutError` → failed `error=f"timeout: exceeded {n}s"`; any other
     exception → failed `error=f"invalid_response: {type(e).__name__}"`
     (DESIGN §7.2; no traceback in DB — traceback to DEBUG log via the
     `mybench.runner` logger, redaction active). Fire `progress(event)`
     after each terminal transition.
   - Concurrency: `asyncio.Semaphore(settings.max_concurrency)`;
     `asyncio.gather(..., return_exceptions=True)`; a coroutine bug surfacing
     as an exception must still leave its output terminal (wrap the whole
     coroutine body; last-resort finish_output(failed)).
   - Finally `runs.complete(run_id)` and return the summary.
   - The provided `conn` is used from coroutines sequentially only for the
     short write transactions (sqlite3 default thread = same event loop
     thread — no threads involved; document this invariant in the docstring).
3. Determinism for tests: `provider_factory` injection is the seam; no
   sleeps besides provider internals.
4. mypy-strict clean.

## Acceptance Criteria

Tests use stub providers (no network, controllable delays/errors):

- [ ] All-success: outputs succeeded with content/usage/latency persisted;
      run completed; summary counts correct; progress fired once per model.
- [ ] Partial failure (1 of 3 raises ProviderError): others unaffected;
      failed output stores sanitized message; `votable` True.
- [ ] All-fail: `votable` False; run still `completed` (summary exit-code
      mapping is issue 19's concern).
- [ ] Timeout: stub sleeping past a 1 s test timeout → failed with timeout
      message; faster models still succeed.
- [ ] Provider build failure (missing env) → that output failed with the auth
      message; no exception escapes.
- [ ] Unexpected exception in stub → output failed
      `invalid_response: {ClassName}`; run completes; `aclose` called on
      every provider including failing ones (spy assertion).
- [ ] Semaphore honored: with max_concurrency=2 and 4 instrumented stubs,
      peak concurrent `complete()` calls == 2.
- [ ] `on_created` fires with the run id after rows exist and before any
      output completes (gated-stub test).
- [ ] Dedup + validation errors (1 model, unknown id, disabled id, archived
      task) raise ValueError before any DB write (run count unchanged).
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_runner.py -q`.

## Dependencies

04, 06, 08, 10.

## Non-goals

CLI surface (19), web trigger (26), cross-process run registry (§18 U5),
streaming.

## Design References

DESIGN §7.1–7.5, §4.3 (snapshot), §6 (provider contract).
