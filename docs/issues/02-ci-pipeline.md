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
   `uv sync --dev` → `uv run ruff format --check .` → `uv run ruff check .` →
   `uv run mypy src` → `uv run coverage run -m pytest -q` →
   `uv run coverage report` (fails under 85 via config) → per-module gate
   (step 4) → `uv run pip-audit` → `uv build`.
4. Per-module coverage gate: a small script `scripts/coverage_gate.py`
   (stdlib only) that reads `coverage json` output and exits 1 unless each of
   `mybench/rating/*`, `mybench/web/security.py`, `mybench/web/render.py`,
   `mybench/safety.py` has ≥ 95% line coverage **when those files exist and
   contain executable code** (files not yet implemented are skipped, so this
   issue can land in wave 0 without failing).
5. All third-party actions pinned to full commit SHAs (not tags).
6. `pip-audit` failure fails the build (no `|| true`).
7. Concurrency group cancels superseded runs for the same ref.

## Acceptance Criteria

- [ ] Workflow runs green on this issue's own PR.
- [ ] A deliberately failing test / lint error / type error on a scratch
      branch each turn the workflow red (verified once, evidence linked in PR).
- [ ] Actions pinned by SHA; `permissions` is read-only contents.
- [ ] Coverage < 85% fails the job (verify by temporarily lowering test scope
      on a scratch branch or by config inspection).
- [ ] `scripts/coverage_gate.py` enforces the 95% list from DESIGN §15 and
      skips not-yet-implemented modules.
- [ ] macOS job runs the same steps on Python 3.12.

## Validation

Push a scratch branch with an intentional ruff violation and confirm failure;
revert. Confirm green run URL is linked in the PR description.

## Dependencies

01 (scaffold with tool configs must exist).

## Non-goals

Release/publish automation (manual, §16.3); CodeQL/Dependabot configuration
(owner may add separately); Windows CI (§2.2 non-goal).

## Design References

DESIGN §16.2 (CI), §15 (gates), §13.1 B6 (supply chain), §17 (conventions).
