# Title

End-to-end test suite over CLI + web with fake provider

## Summary

Add `tests/e2e/`: scripted full-product journeys — CLI-only, web-only, and
mixed — running against the fake provider, including one test that boots a
real uvicorn server on an ephemeral port and drives it over real HTTP.

## Context

DESIGN §15 (E2E row). Unit/route tests validate parts; this issue validates
the assembled product against the v1 completion statement (ISSUE_PLAN §1)
before release issues 31/32.

## Scope

- `tests/e2e/test_journey_cli.py`
- `tests/e2e/test_journey_web.py`
- `tests/e2e/test_real_server.py`
- Shared fixtures in `tests/e2e/conftest.py`

## Detailed Requirements

1. Shared fixture `mybench_env(tmp_path)`: isolated `MYBENCH_CONFIG` /
   `MYBENCH_DATA_DIR`, a config file with 3 fake-provider models
   (`fake-a`, `fake-b`, `fake-c`), yields paths + helpers. All e2e tests
   marked `@pytest.mark.e2e` (runs in CI by default; the marker exists for
   local selection).
2. CLI journey (CliRunner):
   `init` (on pre-written config: prints exists-message) → `config validate`
   → `models list` / `models check` → `task add` (stdin) ×2 categories →
   `task list` → `run --all` per task (exit 0) → `leaderboard` before votes
   (`no votes yet`) → inject votes via repo (voting is web-only) →
   `leaderboard` (rows present, blind counts) → `export` json + csv →
   validate export files parse and counts match `export_stats`.
3. Web journey (TestClient over `create_app`):
   create task via `POST /tasks` → trigger run `POST /tasks/{id}/runs`
   (fake provider completes quickly; poll `status.json` until `completed`,
   bounded 5 s) → vote loop: GET `/vote?run={id}` → POST all 3 pairs
   (following the redirect + asserting reveal banner each time) → run
   auto-reveals on detail page → `/leaderboard` shows 3 models with ratings
   → `/votes` lists 3 votes → delete one → leaderboard reflects it →
   dashboard counts consistent at each step.
4. Real-server test (`test_real_server.py`):
   - Start `uvicorn` in a subprocess via `mybench serve --port {free_port}`
     (find a free port by binding 0 first); wait for readiness (poll ≤ 5 s);
     drive with httpx: dashboard 200 with security headers; `Host: evil.com`
     → 403; full vote POST cycle with cookie jar (CSRF works over real HTTP);
     graceful `SIGTERM` shutdown, exit code 0, bounded 10 s.
   - Skipped on platforms without `SIGTERM` semantics (none in v1 targets).
5. Blindness invariant re-checked at journey level: capture every `/vote`
   response body during the loop and assert no model id appears until the
   reveal banner (which names models of the *previous* pair only).
6. Journey assertions prefer user-visible surfaces (rendered HTML/CLI output)
   over DB peeking, except where noted (vote injection).
7. mypy-strict clean (tests included in mypy scope? No — mypy covers `src/`
   only per DESIGN §17; tests need only ruff).

## Acceptance Criteria

- [ ] All three journey files pass locally and in CI on Linux + macOS.
- [ ] CLI journey covers every §10 command at least once (checklist comment
      at the top of the file mapping command → test line).
- [ ] Web journey covers every §11.2 route at least once (same checklist
      style).
- [ ] Real-server test proves: loopback bind, Host 403, CSRF over real HTTP,
      clean shutdown.
- [ ] Blindness invariant assertion present in the vote loop.
- [ ] Total e2e wall time ≤ 60 s locally (fake provider — no sleeps beyond
      polling).
- [ ] ruff, pytest green; coverage gates still met with e2e included.

## Validation

`uv run pytest tests/e2e -q` and full `uv run pytest -q`.

## Dependencies

16–19, 21, 22–29 (i.e., waves 4 and 5 complete).

## Non-goals

Browser automation (no JS engine — vote.js/runstatus.js behavior is manual QA
+ their own unit-testable simplicity); load testing; real-provider e2e
(manual, documented in issue 31's release checklist).

## Design References

DESIGN §15, §10, §11.2, §8.6; ISSUE_PLAN §1 (completion statement).
