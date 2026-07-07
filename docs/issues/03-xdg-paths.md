# Title

Path resolution module with XDG + env overrides

## Summary

Implement `mybench/paths.py`: resolution of config file, data directory, and
DB file paths per DESIGN §5.1, including secure creation (0700) of the data
directory.

## Context

Config loading (issue 04), the DB (issue 05), and both entry points depend on
deterministic path resolution. Filesystem permissions are security boundary
B5 (DESIGN §13.1, §13.7).

## Scope

- `src/mybench/paths.py`
- `tests/test_paths.py`

## Detailed Requirements

1. Public functions (exact signatures):
   - `config_path() -> pathlib.Path`
   - `data_dir() -> pathlib.Path`
   - `db_path() -> pathlib.Path`
2. Resolution order (DESIGN §5.1):
   - config: `$MYBENCH_CONFIG` (used verbatim as a file path) →
     `$XDG_CONFIG_HOME/mybench/config.toml` → `~/.config/mybench/config.toml`.
   - data dir: `$MYBENCH_DATA_DIR` → `$XDG_DATA_HOME/mybench` →
     `~/.local/share/mybench`.
   - `db_path() = data_dir() / "mybench.db"`.
   - Env value normalization per DESIGN §5.1: empty-string env vars are
     treated as unset (fall through to the next candidate); values are
     normalized with `Path(v).expanduser().resolve(strict=False)`.
   - The same XDG rules apply on macOS (deliberate; DESIGN §5.1).
3. `data_dir()` side effects (DESIGN §5.1/§13.7): create the final directory
   with mode `0700` (missing parents created as needed with default mode);
   when the final directory exists with permissions wider than `0700`,
   `chmod` **only the final directory** to `0700` — never chmod ancestors;
   raise `NotADirectoryError` if the path exists and is not a directory.
   Return the path. `config_path()` has **no** side effects (does not create
   the config dir; issue 16's `init` does).
4. No reads of the config file here (config layer is issue 04); module must
   not import `config.py` (no cycles).
5. Type hints, mypy-strict clean.

## Acceptance Criteria

- [ ] Resolution test matrix (each its own test, for both config and data
      paths, using `monkeypatch.setenv/delenv` + `tmp_path` as fake `$HOME`):
      (a) `MYBENCH_*` set → wins; (b) `MYBENCH_*` unset + `XDG_*` set → XDG;
      (c) both unset → home fallback; (d) `MYBENCH_*` empty string → falls
      through to XDG; (e) `XDG_*` empty string → falls through to home;
      (f) relative `MYBENCH_*` path resolves against CWD; (g) `~/`-prefixed
      value expands.
- [ ] `data_dir()` creates missing dir with mode `0700` (assert
      `stat.S_IMODE == 0o700`), tightens an existing `0755` dir to `0700`,
      leaves ancestor permissions untouched, and raises `NotADirectoryError`
      when the path is a regular file.
- [ ] `config_path()` never creates directories (tested).
- [ ] `db_path()` is `data_dir()/mybench.db` (tested).
- [ ] ruff, mypy strict, pytest all green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_paths.py -q`. optional manual QA (non-gating): `MYBENCH_DATA_DIR=/tmp/mbtest
uv run python -c "from mybench.paths import db_path; print(db_path())"`.

## Dependencies

01.

## Non-goals

Config parsing (04); `mybench init` behavior (16); Windows path conventions
(§2.2 non-goal).

## Design References

DESIGN §5.1 (paths), §13.7 (permissions), §13.1 B5.
