# Title

CLI skeleton: group, logging/redaction wiring, init/config/version

## Summary

Implement `mybench/cli/main.py` as the real click group with global options,
startup wiring (config load, migrations, orphan sweep, logging + redaction),
and the `init`, `config path`, `config validate`, and `version` commands.

## Context

DESIGN §10.1–10.2, §10.10. Every later CLI command hangs off this group and
inherits its startup behavior. The redaction filter (§14) must be installed
before any command logic can log.

## Scope

- `src/mybench/cli/main.py` (replace scaffold placeholder)
- `src/mybench/cli/__init__.py`
- `tests/test_cli_main.py`

## Detailed Requirements

1. Click group `cli` with global options:
   - `--verbose/-v` (flag): DEBUG logging, else INFO.
   - `--config PATH`: overrides config file path.
   - `--data-dir PATH`: overrides data dir (sets the same resolution the
     paths module would produce — implement by passing explicit paths, not by
     mutating env).
   - `click.version_option(__version__, prog_name="mybench")`.
2. Startup context (a `CliContext` dataclass stored via `ctx.obj`):
   `config: Config`, `db_path: Path`, `data_dir: Path`, `config_path: Path`.
   Built by a `@cli.group`-level callback that: configures logging
   (`logging.basicConfig`, format `%(levelname)s %(name)s: %(message)s`),
   loads config (a `ConfigError` prints `error: {message}` to stderr, exit 1),
   installs the redaction filter with all configured `api_key_env` names
   (safety §14), creates the data dir (paths module), connects+migrates the
   DB and runs `runs.sweep_orphans` (log count at INFO when > 0), then closes
   that connection (commands open their own).
   **Exception:** `init`, `version`, and `config path` must work without a
   valid config/DB — they bypass config-load failure (lazy context: only
   commands that need the DB/config trigger the full startup; implement via
   a `require_context()` helper called by those commands).
3. `mybench init` (DESIGN §10.2): create config parent dir (0700 for the
   mybench dir), write the commented template **exactly** matching DESIGN
   §5.2 with all `[[models]]` entries commented out, only when the file does
   not exist; print created path + `next: edit the config, export your API
   key env vars, then run 'mybench models check'`. Existing file → print
   `config already exists at {path}` and exit 0. Never overwrite.
4. `mybench config path`: print three lines `config: {path}`,
   `data: {dir}`, `db: {db_path}` (resolved, honoring global overrides).
5. `mybench config validate`: load config; success → print
   `ok: {N} models ({M} enabled)` exit 0; `ConfigError` → message to stderr,
   exit 1.
6. `mybench version`: print `mybench {__version__}`.
7. Exit-code policy helper: commands raise `click.ClickException`-compatible
   errors; stderr messages prefixed `error: ` (DESIGN §10.1).
8. mypy-strict clean.

## Acceptance Criteria

All via `CliRunner` with `tmp_path` HOME/env isolation:

- [ ] `mybench version` and `--version` print version, exit 0, without any
      config present.
- [ ] `init` creates the template (byte-compare against a checked-in
      `tests/fixtures/expected_config_template.toml`); re-run exits 0 without
      modifying mtime/content; template parses as valid TOML and loads as
      zero-model config.
- [ ] `config path` honors `--config`/`--data-dir` overrides.
- [ ] `config validate` ok and error paths (exit codes, stderr).
- [ ] Startup applies migrations on a fresh data dir (DB file exists after
      any DB-touching command) and sweeps orphans (fixture DB with an
      orphaned run → INFO log captured).
- [ ] Redaction: with `FAKEKEY=sekret` in env and a config naming it, a DEBUG
      log emitting the value prints `[REDACTED:...]`/`[REDACTED]` instead.
- [ ] Broken config + `mybench config validate` → exit 1; `mybench init`
      still works with the broken config present.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_cli_main.py -q`; manual smoke:
`MYBENCH_CONFIG=/tmp/mb/config.toml MYBENCH_DATA_DIR=/tmp/mb/data uv run
mybench init && uv run mybench config validate`.

## Dependencies

04, 05, 06 (03 transitively).

## Non-goals

models/task/run/leaderboard/export/serve commands (17–19, 21, 22); shell
completion; interactive prompts.

## Design References

DESIGN §10.1, §10.2, §10.10, §7.4 (sweep wiring), §14; ADR-005.
