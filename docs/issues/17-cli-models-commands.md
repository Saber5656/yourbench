# Title

CLI: models list / models check

## Summary

Implement `mybench/cli/models_cmd.py`: `mybench models list` (config-derived
table with key-presence status) and `mybench models check` (real 1-token
connectivity check per model, sequential).

## Context

DESIGN §10.3–10.4. This is the user's first feedback loop after `init` — the
error texts here determine whether setup feels debuggable.

## Scope

- `src/mybench/cli/models_cmd.py`
- `src/mybench/cli/main.py` (one `cli.add_command(models)` registration line
  per issue 16's contract)
- `tests/test_cli_models.py` (+ golden fixture
  `tests/fixtures/golden_models_list.txt`)

## Detailed Requirements

1. `mybench models list [--json]`:
   - Table columns exactly: `ID`, `PROVIDER`, `MODEL`, `BASE_URL`, `ENABLED`,
     `KEY`. `BASE_URL` shows `-` when None (fake only — config normalizes
     anthropic); `MODEL` shows `-` when None; `ENABLED` shows
     `true`/`false`; `KEY` shows `set` / `missing` / `n/a` (`n/a` when
     `api_key_env` is None) — presence check via `os.environ.get`, never
     printing values (DESIGN §10.3).
   - Formatting rule (mechanical): each column left-aligned and padded to
     its longest cell, columns joined by two spaces, no trailing spaces,
     `\n` line endings. Golden output checked in as
     `tests/fixtures/golden_models_list.txt` for the DESIGN §5.2 example
     config under `OPENAI_API_KEY=set-value`, `ANTHROPIC_API_KEY` unset.
   - Zero models → `no models configured; run 'mybench init' and edit the
     config` to stderr, exit 0, empty stdout.
   - `--json` (stdout only, nothing else): array of
     `{"id": str, "provider": str, "model": str | null,
     "base_url": str | null, "enabled": bool,
     "key": "set" | "missing" | "n/a"}` in config order.
2. `mybench models check [MODEL_ID ...] [--json]`:
   - Default targets: all enabled models in config order; zero targets
     (no models configured / none enabled) → `error: no enabled models
     configured; run 'mybench init' and edit the config` to stderr, exit 1,
     no providers constructed. Explicit ids may include disabled models;
     unknown id → `error: unknown model id '{id}'` exit 1 before any
     network call.
   - Sequential execution (rate-limit friendly, DESIGN §10.4) via
     `asyncio.run` over the injectable helper of requirement 3. A
     `ProviderError` from **either** `build` (e.g. missing env var) **or**
     `check()` becomes a failed row; checking always continues to the next
     model.
   - Output per model: `{id}: ok (312 ms)` or `{id}: error ({sanitized
     message})`; summary line `{ok}/{total} ok`. Exit 0 all ok; exit 1
     otherwise. `--json`: array of `{"id": str, "ok": bool,
     "message": str, "latency_ms": int | null}` in check order
     (`latency_ms` null on failure; `message` is `"ok (312 ms)"`-style on
     success).
   - All provider-originated text printed through
     `safety.strip_terminal_controls` (B3); JSON carries the sanitized
     message as-is (json escaping covers control bytes).
3. Injectable seam (normative):
   `async def check_models(config: Config, model_ids: Sequence[str], *,
   provider_factory: Callable[[ModelConfig, Settings], Provider] =
   providers.build) -> list[CheckRow]` with
   `CheckRow(id: str, ok: bool, message: str, latency_ms: int | None)`;
   the click command is a thin wrapper over it.
4. mypy-strict clean.

## Acceptance Criteria

CliRunner + fake/stub providers:

- [ ] `mybench models --help` works from the root group (registration line
      present in `main.py`).
- [ ] `models list` matches `tests/fixtures/golden_models_list.txt` exactly
      for the §5.2 config with `set`/`missing`/`n/a` all exercised.
- [ ] `models list --json` parses and equals the exact documented objects
      (full-array comparison, not just `json.loads`).
- [ ] `check` happy path (fake provider) exit 0 with `ok (…ms)` lines.
- [ ] `check` with one failing stub (from `check()`) and one build failure
      (missing env var) → exit 1, both failed rows present with sanitized
      messages, remaining models still checked, summary counts correct.
- [ ] `check` zero-configured/zero-enabled → exact error, exit 1, factory
      spy never called; unknown id → exit 1, factory spy never called.
- [ ] `check X` where X is disabled works.
- [ ] `check --json` equals the exact documented objects for a mixed
      ok/failed fixture.
- [ ] ANSI in stub error message stripped from terminal output.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_cli_models.py -q`; manual against a real provider
if available (optional, note in PR).

## Dependencies

10, 16 (11/12 land before real-provider usefulness but are not build deps).

## Non-goals

Parallel checks; model listing from provider `/models` endpoints (research
doc §3 rules it out for v1); cost display.

## Design References

DESIGN §10.3, §10.4, §13.1 B3, §14.
