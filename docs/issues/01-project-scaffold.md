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

1. `pyproject.toml` — must match this skeleton (add nothing else without an
   ADR-002 amendment):

   ```toml
   [build-system]
   requires = ["hatchling"]
   build-backend = "hatchling.build"

   [project]
   name = "mybench"
   description = "A private leaderboard for blind A/B testing LLMs on your own real tasks."
   requires-python = ">=3.11"
   license = "MIT"
   dynamic = ["version"]
   dependencies = [
     "click", "fastapi", "uvicorn", "jinja2", "httpx", "markdown-it-py", "nh3",
   ]

   [project.scripts]
   mybench = "mybench.cli.main:cli"

   [dependency-groups]
   dev = ["pytest", "pytest-asyncio", "ruff", "mypy", "pip-audit", "coverage[toml]"]

   [tool.hatch.version]
   path = "src/mybench/__init__.py"

   [tool.hatch.build.targets.wheel]
   packages = ["src/mybench"]
   ```

   (Version pins/floors may be added to `dependencies` if `uv lock` requires
   them; the package *set* is fixed by ADR-002. Tool config sections are added
   in requirement 3.)
2. Module tree: create every file listed in DESIGN §3.1. Each placeholder
   module contains exactly one docstring line:
   `"""Implements DESIGN §<n> (placeholder)."""` where `<n>` is the section
   number annotated for that file in the §3.1 tree (e.g. `paths.py` → §5.1).
   Exceptions:
   - `src/mybench/__init__.py`: docstring + `__version__ = "0.1.0"`.
   - `src/mybench/cli/main.py`: minimal importable click group named `cli`
     (no options, no commands) so the console script resolves;
     `mybench --help` must exit 0. All real CLI behavior (including
     `version`) is issue 16.
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
5. `tests/test_smoke.py`: (a) `import mybench` exposes `__version__ ==
   "0.1.0"`; (b) `CliRunner` invokes the group with `--help`, exit code 0.
6. Do not implement any product logic in this issue.

## Acceptance Criteria

- [ ] `uv sync --dev` succeeds on a clean checkout; `uv.lock` committed.
- [ ] `uv run mybench --help` exits 0.
- [ ] `uv run pytest -q` passes (smoke test).
- [ ] `uv run ruff check .` and `uv run ruff format --check .` pass.
- [ ] `uv run mypy src` passes.
- [ ] Every path in DESIGN §3.1 exists (modules or directories).
- [ ] Runtime dependencies in `pyproject.toml` are exactly the ADR-002 list.
- [ ] `uv build` produces sdist + wheel without error.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

From a clean clone, run in order; all must succeed:

```sh
uv sync --dev
uv run mybench --help
uv run ruff check . && uv run ruff format --check .
uv run mypy src
uv run pytest -q
uv build
python -c "import zipfile,glob; names=zipfile.ZipFile(glob.glob('dist/mybench-*.whl')[0]).namelist(); assert any(n.startswith('mybench/') and n.endswith('__init__.py') for n in names), names"
```

## Dependencies

None (first issue).

## Non-goals

CI workflow (issue 02); any real command/module behavior; LICENSE/README
content (issue 31).

## Design References

DESIGN §3.1 (module layout), §16.1 (packaging), §17 (conventions); ADR-002.
