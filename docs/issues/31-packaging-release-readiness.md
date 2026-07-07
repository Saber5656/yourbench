# Title

Packaging polish: README, CHANGELOG, LICENSE, wheel smoke test

## Summary

Make the repository releasable: user-facing README with an honest quickstart,
MIT LICENSE, CHANGELOG, packaging metadata completeness, and a scripted
clean-venv wheel install smoke test.

## Context

DESIGN §16.1/§16.3. This is the "someone finds the repo and succeeds in 10
minutes" issue — documentation of what exists, not new behavior.

## Scope

- `README.md` (rewrite), `LICENSE`, `CHANGELOG.md`
- `pyproject.toml` metadata completion (classifiers, urls, readme)
- `scripts/smoke_install.sh`
- `docs/` cross-link fixes if paths moved (none expected)

## Detailed Requirements

1. `README.md` sections, in order: name + one-line pitch (from ADR-001
   wording); 60-second quickstart (`uv tool install mybench` → `mybench init`
   → edit config + export key env vars → `mybench models check` →
   `mybench task add "..."` → `mybench run 1 --all` → `mybench serve` →
   vote → leaderboard); screenshot placeholder comment (owner adds real
   screenshots); "How it works" (blind pairs, Bradley-Terry — 2 paragraphs,
   link `docs/research/rating-methodology.md`); "Privacy & security" (bullet
   summary of ADR-005 promises: local-only, loopback bind, env-only keys, no
   telemetry, sanitized rendering — each one sentence); configuration
   reference (the §5.2 example verbatim + the §5.3 rules table condensed);
   FAQ (name vs HF YourBench — one line + link; "why are my two local models
   in separate components"); development (`uv sync --dev`, test/lint
   commands, link `docs/DESIGN.md` + `docs/ISSUE_PLAN.md`); license line.
2. `LICENSE`: MIT, copyright `2026 mybench contributors`.
3. `CHANGELOG.md`: Keep-a-Changelog skeleton with `[0.1.0] – unreleased`
   listing v1 features by wave (one line each).
4. `pyproject.toml`: `readme = "README.md"`, `license = "MIT"` (+ file),
   `keywords`, classifiers (Python versions, Development Status 4 - Beta,
   Environment :: Web Environment, License OSI MIT, Topic scientific/
   benchmark-adjacent), `[project.urls]` Homepage/Issues pointing at the
   GitHub repo.
5. `scripts/smoke_install.sh` (bash, `set -euo pipefail`): `uv build` →
   create temp venv → `pip install dist/*.whl` → in a temp
   MYBENCH_CONFIG/MYBENCH_DATA_DIR: `mybench version`, `mybench init`,
   `mybench config validate`, `mybench task add "smoke" && mybench run 1
   --models fake1,fake2` against a heredoc fake-provider config, `mybench
   export --out ...json`, then assert templates/static load by importing
   `create_app` (`python -c`) — proving package-data completeness (§16.1).
   Wired into CI as a final job on Linux.
6. Release checklist appended to `CHANGELOG.md` as an HTML comment: version
   bump locations, `uv build`, smoke script, tag, manual PyPI publish
   (owner-only), post-publish `uv tool install mybench` verification, and a
   manual real-provider spot check (one OpenRouter + one Anthropic call).

## Acceptance Criteria

- [ ] `scripts/smoke_install.sh` passes locally and in CI (new job green).
- [ ] Wheel contains templates/static (script asserts `create_app` renders
      `/` in the venv — not just import).
- [ ] README quickstart commands copy-paste-run in a clean env (manually
      verified; PR notes the transcript).
- [ ] README privacy section matches ADR-005 exactly (no over-claims; e.g.
      no "encrypted" wording).
- [ ] `uv build` metadata: `twine check dist/*`-equivalent clean (hatchling
      output verified; classifiers valid).
- [ ] LICENSE/CHANGELOG exist with specified content shape.
- [ ] ruff/mypy/pytest untouched-green (no behavior changes).

## Validation

`bash scripts/smoke_install.sh` locally; CI job link in PR.

## Dependencies

30 (product must actually work end-to-end before release polish).

## Non-goals

Actual PyPI publication (owner-manual, §16.3); docs site; screenshots
(owner-supplied); marketing copy beyond the pitch line.

## Design References

DESIGN §16.1, §16.3; ADR-001, ADR-005; ISSUE_PLAN §1.
