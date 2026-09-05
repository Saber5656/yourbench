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
   wording); quickstart split in two (F: post-publish `uv tool install
   mybench` steps are marked as available after the first PyPI release;
   from-source steps `uv sync && uv run mybench …` work today and are what
   gets verified): `init` → edit config + export key env vars →
   `models check` → `task add "..."` → `run 1 --all` → `serve` → vote →
   leaderboard; screenshot placeholder comment (owner adds real
   screenshots); "How it works" (blind pairs, Bradley-Terry — 2 paragraphs,
   link `docs/research/rating-methodology.md`); "Privacy & security" — an
   exact two-part list: guarantees (local-only data in
   `~/.local/share/mybench`, loopback-only server with CSRF/Host/CSP
   defenses, env-var-only API keys never persisted or logged, zero
   telemetry/external assets, sanitized rendering of model output) and
   **limitations** (no encryption at rest — OS/disk encryption assumed; a
   process running as your OS user can read the data; prompts you paste are
   stored verbatim; exports contain prompt/output text) — no claim beyond
   ADR-005/DESIGN §13, forbidden words checked: "encrypted", "secure by
   default" unqualified; configuration reference (the §5.2 example verbatim
   + the §5.3 rules table condensed); FAQ (name vs HF YourBench — one line
   + link; "why are my two local models in separate components");
   development (`uv sync --dev`, test/lint commands, link `docs/DESIGN.md` +
   `docs/ISSUE_PLAN.md`); license line.
2. `LICENSE`: MIT, copyright `2026 mybench contributors`.
3. `CHANGELOG.md`: Keep-a-Changelog skeleton with `[0.1.0] – unreleased`
   listing v1 features by wave (one line each).
4. `pyproject.toml` — apply exactly this metadata block (merged into the
   issue-01 skeleton):

   ```toml
   [project]
   readme = "README.md"
   keywords = ["llm", "benchmark", "leaderboard", "blind-test", "evaluation"]
   classifiers = [
     "Development Status :: 4 - Beta",
     "Environment :: Web Environment",
     "Programming Language :: Python :: 3.11",
     "Programming Language :: Python :: 3.12",
     "Programming Language :: Python :: 3.13",
     "Topic :: Scientific/Engineering :: Artificial Intelligence",
   ]

   [project.urls]
   Homepage = "https://github.com/Saber5656/mybench"
   Issues = "https://github.com/Saber5656/mybench/issues"
   ```

   (License already set by issue 01; SPDX `license = "MIT"` needs no
   classifier.)
5. `scripts/smoke_install.sh` (bash, `set -euo pipefail`; trap-based temp
   cleanup) — exact sequence:
   1. `uv build`; pick `WHEEL=$(ls dist/mybench-*.whl | head -1)`.
   2. `python3 -m venv "$TMP/venv"`; `"$TMP/venv/bin/pip" install "$WHEEL"`.
   3. Export `MYBENCH_CONFIG="$TMP/cfg/config.toml"`,
      `MYBENCH_DATA_DIR="$TMP/data"`; write via heredoc a config with two
      fake models (`fake1`, `fake2`).
   4. Run from the venv: `mybench version`, `mybench init` (exists-message
      path), `mybench config validate`, `TASK_ID=$(mybench task add
      "smoke" --json | python3 -c 'import json,sys;
      print(json.load(sys.stdin)["id"])')`, `mybench run "$TASK_ID"
      --models fake1,fake2` (expect exit 0), `mybench export --out
      "$TMP/x.json"`.
   5. Wheel-completeness + serve smoke: `mybench serve --port "$FREE_PORT"`
      in the background from the installed venv, poll `http://127.0.0.1:
      $FREE_PORT/` until 200 (≤ 10 s) **and** fetch `/static/app.css`
      (200) — proving console script, package-data templates, and static
      files all ship in the wheel; then SIGTERM and assert exit 0.
   CI wiring: a final `smoke` job on ubuntu (needs: test-linux) with
   `permissions: contents: read`, SHA-pinned actions, installing only from
   the locally built wheel — no PyPI publish, no API keys, no publish
   credentials anywhere in the workflow.
6. Release checklist appended to `CHANGELOG.md` as an HTML comment: version
   bump locations, `uv build`, `uvx twine check dist/*` (exact metadata
   validation command), smoke script, tag, manual PyPI publish
   (owner-only), post-publish `uv tool install mybench` verification, and a
   manual real-provider spot check (one OpenRouter + one Anthropic call).
7. Doc cross-links: only links directly introduced or broken by README/
   metadata changes may be touched; no file moves in this issue.

## Acceptance Criteria

- [ ] `scripts/smoke_install.sh` passes locally and in CI (new job green,
      wheel-only install, no keys, read-only permissions, pinned actions).
- [ ] Wheel completeness proven by the served `/` (200) and
      `/static/app.css` (200) from the installed venv.
- [ ] From-source README quickstart verified with a transcript in the PR
      (fake-provider config); post-publish steps clearly marked as
      after-first-release.
- [ ] README privacy section contains both the guarantees and the
      limitations lists of requirement 1; grep-check: no unqualified
      "encrypted"/"secure by default" claims.
- [ ] `uvx twine check dist/*` passes.
- [ ] LICENSE/CHANGELOG exist with specified content shape (incl. the
      release-checklist comment).
- [ ] ruff/mypy/pytest untouched-green (no behavior changes).

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`bash scripts/smoke_install.sh` locally; CI job link in PR.

## Dependencies

30 (product must actually work end-to-end before release polish).

## Non-goals

Actual PyPI publication (owner-manual, §16.3); docs site; screenshots
(owner-supplied); marketing copy beyond the pitch line.

## Design References

DESIGN §16.1, §16.3; ADR-001, ADR-005; ISSUE_PLAN §1.
