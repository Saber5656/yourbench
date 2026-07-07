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
- `tests/test_cli_models.py`

## Detailed Requirements

1. `mybench models list [--json]`:
   - Table columns exactly: `ID`, `PROVIDER`, `MODEL`, `BASE_URL`, `ENABLED`,
     `KEY`. `BASE_URL` shows `-` when None; `KEY` shows `set` /
     `missing` / `n/a` (`n/a` when `api_key_env` is None) — presence check
     via `os.environ.get`, never printing values (DESIGN §10.3).
   - Plain aligned text (two-space padding, header row; no box-drawing);
     zero models → `no models configured; run 'mybench init' and edit the
     config` to stderr, exit 0.
   - `--json`: array of objects with keys `id, provider, model, base_url,
     enabled, key` (same string semantics).
2. `mybench models check [MODEL_ID ...] [--json]`:
   - Default targets: all enabled models in config order; explicit ids may
     include disabled models (checking a disabled model is allowed); unknown
     id → `error: unknown model id '{id}'` exit 1 before any network call.
   - Sequential execution (rate-limit friendly, DESIGN §10.4) via
     `asyncio.run` over `providers.build(...).check()`; a `ProviderError`
     from `build` (missing env var) becomes a failed check row, not a crash.
   - Output per model: `{id}: ok (312 ms)` or `{id}: error ({sanitized
     message})`; summary line `{ok}/{total} ok`. Exit 0 all ok; exit 1
     otherwise. `--json`: array of `{id, ok, message, latency_ms}`.
   - All provider-originated text printed through
     `safety.strip_terminal_controls` (B3).
3. Provider construction must be injectable for tests
   (`provider_factory` param on the underlying function, mirroring runner).
4. mypy-strict clean.

## Acceptance Criteria

CliRunner + fake/stub providers:

- [ ] `models list` renders the §5.2 example config exactly as specified
      (golden-string test) with `set`/`missing`/`n/a` all exercised via env
      manipulation.
- [ ] `models list --json` round-trips through `json.loads`.
- [ ] `check` happy path (fake provider) exit 0 with `ok (…ms)` lines.
- [ ] `check` with one failing stub → exit 1, failing line contains the
      sanitized error, summary counts correct.
- [ ] `check` unknown id → exit 1, no provider constructed (assert via
      factory spy).
- [ ] `check X` where X is disabled works.
- [ ] Missing env var check → failed row naming the env var.
- [ ] ANSI in stub error message stripped from terminal output.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_cli_models.py -q`; manual against a real provider
if available (optional, note in PR).

## Dependencies

10, 16 (11/12 land before real-provider usefulness but are not build deps).

## Non-goals

Parallel checks; model listing from provider `/models` endpoints (research
doc §3 rules it out for v1); cost display.

## Design References

DESIGN §10.3, §10.4, §13.1 B3, §14.
