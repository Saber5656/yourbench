# Title

CI pipeline: lint, types, tests, coverage, audit, build

## Summary

Add a GitHub Actions workflow that blocks merges unless formatting, lint,
typing, tests with coverage gates, dependency audit, and package build all
pass, on Linux and macOS.

## Context

DESIGN §16.2 specifies the CI contract; §15 specifies the coverage gates. CI
is the enforcement mechanism for the per-issue validation rules in DESIGN §17,
and part of supply-chain posture (§13.1 B6).

## Scope

- `.github/workflows/ci.yml`
- Coverage gate configuration (`[tool.coverage.report] fail_under = 85` plus
  per-module gate script)

## Detailed Requirements

1. Trigger: `push` to `main` and `pull_request` targeting `main`.
2. Top-level `permissions: contents: read`. No other permissions.
3. Jobs:
   - `test-linux`: matrix `python-version: ["3.11", "3.12", "3.13"]` on
     `ubuntu-latest`.
   - `test-macos`: Python `3.12` on `macos-latest`.
   Steps (both jobs): checkout → install uv (official setup action) →
   `uv sync --dev --locked` (fails on stale `uv.lock` — supply-chain gate) →
   `uv run ruff format --check .` → `uv run ruff check .` →
   `uv run mypy src` → `uv run coverage run -m pytest -q` →
   `uv run coverage report` (fails under 85 via
   `[tool.coverage.report] fail_under = 85`) →
   `uv run coverage json -o coverage.json` →
   `uv run python scripts/coverage_gate.py coverage.json` →
   `uv run pip-audit` → `uv build`.
4. Per-module coverage gate `scripts/coverage_gate.py` (stdlib only; exit 0
   pass / exit 1 fail, printing each failing file and its percentage):
   - Reads the coverage-json `files` mapping; matches entries whose
     normalized path (strip any leading `src/`, use `/` separators) equals
     `mybench/safety.py`, `mybench/web/security.py`, `mybench/web/render.py`,
     or starts with `mybench/rating/` (`__init__.py` included).
   - Gate: `percent_covered >= 95.0` per matched file.
   - Skip rule (lets this land in wave 0): a matched file is skipped iff its
     `num_statements` ≤ 1 (docstring-only placeholder) or it is absent from
     the report.
5. Every `uses:` action — including GitHub-owned (`actions/checkout`) and the
   uv setup action — pinned to a full commit SHA with a trailing
   `# vX.Y.Z` comment.
6. `pip-audit` failure fails the build (no `|| true`).
7. Concurrency group cancels superseded runs for the same ref.
8. Merge-blocking note: this issue delivers the status checks; making them
   *required* is a repository ruleset setting the owner manages (out of
   scope; record the handoff in the PR description).

## Acceptance Criteria

- [ ] Workflow runs green on this issue's own PR.
- [ ] Failure-mode evidence in the PR: one scratch-branch push with a ruff
      violation shows a red run (link), plus local transcripts showing
      `uv run mypy src` and `uv run coverage run -m pytest -q` exiting
      non-zero on an intentional type error / failing test.
- [ ] Coverage-threshold evidence: local transcript showing
      `uv run coverage report` exiting non-zero with `fail_under = 85` when
      coverage is forced below 85 (e.g. by deselecting tests).
- [ ] Every `uses:` pinned by full SHA; `permissions` is contents read-only.
- [ ] `uv sync --dev --locked` in the workflow; a stale-lockfile failure mode
      is documented in the PR (transcript or CI link).
- [ ] `scripts/coverage_gate.py` enforces the 95% list from DESIGN §15 per
      requirement 4, including its skip rule (unit-tested with two synthetic
      coverage.json fixtures: one passing, one failing).
- [ ] macOS job runs the same steps on Python 3.12.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

Push a scratch branch with an intentional ruff violation and confirm failure;
revert. Confirm green run URL is linked in the PR description.

## Dependencies

01 (scaffold with tool configs must exist).

## Non-goals

Release/publish automation (manual, §16.3); CodeQL/Dependabot configuration
(owner may add separately); Windows CI (§2.2 non-goal).

## Design References

DESIGN §16.2 (CI), §15 (gates), §13.1 B6 (supply chain), §17 (conventions).
