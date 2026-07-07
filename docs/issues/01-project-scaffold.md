# Title

Project scaffold: uv package, src layout, tooling config

## Summary

Create the installable Python package skeleton for mybench: `pyproject.toml`
(hatchling, console script), `src/mybench/` module tree with empty modules,
tooling configuration (ruff, mypy, pytest, coverage), `.gitignore`, and a
smoke test proving the package imports and the CLI entry point runs.

## Context

Everything else builds on this skeleton. DESIGN §3.1 fixes the module layout;
ADR-002 fixes the stack and the runtime dependency allowlist. The repository
currently contains only documentation.

## Scope

- `pyproject.toml`, `uv.lock`, `.gitignore`, `.python-version` (3.12)
- `src/mybench/` package tree per DESIGN §3.1 with placeholder modules
  (docstring-only; no logic)
- `tests/` with one smoke test
- Tool configuration for ruff, mypy, pytest, coverage (in `pyproject.toml`)

## Detailed Requirements

1. `pyproject.toml`:
   - `[project]`: name `mybench`, description "A private leaderboard for blind
     A/B testing LLMs on your own real tasks.", `requires-python = ">=3.11"`,
     license `MIT`, dynamic version from `mybench.__init__.__version__`
     (hatchling `[tool.hatch.version] path = "src/mybench/__init__.py"`).
   - Runtime dependencies exactly (ADR-002 allowlist): `click`, `fastapi`,
     `uvicorn`, `jinja2`, `httpx`, `markdown-it-py`, `nh3`.
   - `[dependency-groups] dev`: `pytest`, `pytest-asyncio`, `ruff`, `mypy`,
     `pip-audit`, `coverage[toml]`.
   - `[project.scripts] mybench = "mybench.cli.main:cli"`.
   - Build backend hatchling; package data will include
     `web/templates/**` and `web/static/**` (configure
     `[tool.hatch.build.targets.wheel]` to include `src/mybench`).
2. Module tree: create every file listed in DESIGN §3.1 as a module with only
   a docstring line `"""Implements DESIGN §<n> (placeholder)."""`, except:
   - `src/mybench/__init__.py`: `__version__ = "0.1.0"`.
   - `src/mybench/cli/main.py`: minimal working click group `cli` with
     `--version` support (click's `version_option`) so the console script runs.
   - `web/templates/` and `web/static/` as directories with `.gitkeep`.
3. Tooling config in `pyproject.toml`:
   - ruff: `line-length = 100`, `target-version = "py311"`, lint select
     `["E","F","I","UP","B","S"]` (`S` = bandit rules; per-file-ignores
     `"tests/*" = ["S101"]`).
   - mypy: `strict = true`, `mypy_path = "src"`, `packages = ["mybench"]`.
   - pytest: `testpaths = ["tests"]`, `asyncio_mode = "auto"`,
     marker `e2e` declared.
   - coverage: `source = ["mybench"]`; fail-under handled in CI (issue 02).
4. `.gitignore`: standard Python (`__pycache__/`, `.venv/`, `dist/`,
   `.coverage`, `.pytest_cache/`, `.mypy_cache/`, `.ruff_cache/`).
5. `tests/test_smoke.py`: (a) `import mybench` exposes `__version__`;
   (b) `CliRunner` invokes `cli --version` exit code 0 containing `0.1.0`.
6. Do not implement any product logic in this issue.

## Acceptance Criteria

- [ ] `uv sync --dev` succeeds on a clean checkout; `uv.lock` committed.
- [ ] `uv run mybench --version` prints `0.1.0` and exits 0.
- [ ] `uv run pytest -q` passes (smoke test).
- [ ] `uv run ruff check .` and `uv run ruff format --check .` pass.
- [ ] `uv run mypy src` passes.
- [ ] Every path in DESIGN §3.1 exists (modules or directories).
- [ ] Runtime dependencies in `pyproject.toml` are exactly the ADR-002 list.
- [ ] `uv build` produces sdist + wheel without error.

## Validation

Run the five commands above from a clean clone. Inspect the wheel
(`unzip -l dist/*.whl`) to confirm `mybench/` is packaged from `src/`.

## Dependencies

None (first issue).

## Non-goals

CI workflow (issue 02); any real command/module behavior; LICENSE/README
content (issue 31).

## Design References

DESIGN §3.1 (module layout), §16.1 (packaging), §17 (conventions); ADR-002.
