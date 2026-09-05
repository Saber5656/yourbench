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
- `tests/fixtures/expected_config_template.toml`

## Detailed Requirements

1. Click group `cli` with global options:
   - `--verbose/-v` (flag): DEBUG logging, else INFO.
   - `--config PATH` / `--data-dir PATH`: explicit overrides. Precedence
     (DESIGN §5.1): CLI option > `MYBENCH_*` env > `XDG_*` > home default.
     Implement via a helper `resolve_paths(config_opt: Path | None,
     data_dir_opt: Path | None) -> ResolvedPaths` (frozen dataclass:
     `config_path`, `data_dir`, `db_path`) that falls back to the
     `paths.py` functions when an option is None — never mutate `os.environ`.
     `db_path = data_dir / "mybench.db"` always.
   - `click.version_option(__version__, prog_name="mybench")`.
   - Subcommand registration contract for later issues: each command module
     exposes click commands/groups; `cli/main.py` registers them with
     explicit `cli.add_command(...)` lines (issues 17–19/21/22 each add
     their line to `main.py`).
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
3. `mybench init` (DESIGN §10.2): create the config parent dir (`0700` for
   the `mybench` dir), **create the data dir via `paths`/`resolve_paths`
   (side effect: 0700, §13.7)**, and write the commented template
   **exactly** matching DESIGN §5.2 with all `[[models]]` entries commented
   out, only when the file does not exist; print created path + `next: edit
   the config, export your API key env vars, then run 'mybench models
   check'`. Existing file → print `config already exists at {path}` and
   exit 0. Never overwrite. The expected template ships as a test fixture
   `tests/fixtures/expected_config_template.toml` (in Scope) and the
   implementation must byte-match it.
4. `mybench config path`: print three lines `config: {path}`,
   `data: {dir}`, `db: {db_path}` (resolved, honoring global overrides).
5. `mybench config validate`: load config; success → print
   `ok: {N} models ({M} enabled)` exit 0; `ConfigError` → message to stderr,
   exit 1.
6. `mybench version`: print `mybench {__version__}`.
7. Exit-code/error helper (DESIGN §10.1): provide `class CliError(
   click.ClickException)` overriding `show()` to print `error: {message}`
   to stderr (lowercase prefix — click's default `Error:` casing is NOT
   acceptable) with `exit_code = 1`; all later command issues raise it.
8. mypy-strict clean.

## Acceptance Criteria

All via `CliRunner` with `tmp_path` HOME/env isolation:

- [ ] `mybench version` and `--version` print version, exit 0, without any
      config present.
- [ ] `init` creates the template (byte-compare against the checked-in
      `tests/fixtures/expected_config_template.toml`); re-run exits 0 without
      modifying mtime/content; template parses as valid TOML and loads as
      zero-model config; the resolved data dir exists afterwards with mode
      `0700`, honoring `--data-dir` over a set `MYBENCH_DATA_DIR`.
- [ ] `resolve_paths` precedence matrix tested: option beats env; env beats
      XDG; XDG beats home (config and data both).
- [ ] `config path` honors `--config`/`--data-dir` overrides.
- [ ] `config validate` ok and error paths (exit codes, stderr).
- [ ] `require_context()` tested directly: on a fresh data dir it connects,
      applies migrations (DB file exists, `schema_migrations` row present)
      and sweeps orphans (fixture DB with an orphaned run → INFO log
      captured, output rows flipped to failed).
- [ ] Redaction: with `FAKEKEY=sekret` in env and a config naming it, a
      DEBUG log emitting the value prints exactly `[REDACTED:FAKEKEY]` and
      `sekret` is absent from all captured output.
- [ ] `CliError` prints `error: boom` to stderr with exit code 1 (golden).
- [ ] Broken config + `mybench config validate` → exit 1; `mybench init`
      still works with the broken config present.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

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
