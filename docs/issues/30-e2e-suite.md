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
2. CLI journey (CliRunner) — the journey covers every §10 command at least
   once, in this exact flow: `version` → `config path` → `init` (on
   pre-written config: prints exists-message) → `config validate` →
   `models list` → `models check` → `task add` (stdin) ×2 categories →
   `task list` → `task show 1` → `run 1 --all` and `run 2 --all` (exit 0)
   → `leaderboard` before votes (stderr `no votes yet; run tasks and vote
   first`) → inject votes via the votes repo (voting is web-only) →
   `leaderboard` (rows present; `total_votes` line) → `leaderboard
   --category` → `export --out x.json` + `--format csv` → assert the JSON
   document top-level keys per DESIGN §10.9, the exact CSV header per
   §10.9, and row counts equal to the objects created in this journey
   (2 tasks, 2 runs, 6 outputs, N injected votes). (`serve` is exercised by
   the real-server test below.)
3. Web journey (TestClient over `create_app`) — expected counts asserted at
   each step from the known fixture (1 task, 3 fake models):
   dashboard (0 tasks state) → create task via `POST /tasks` → task detail
   → trigger run `POST /tasks/{id}/runs` (fake provider completes quickly;
   poll `status.json` until `completed`, bounded 5 s; expect `succeeded=3,
   votable=true`) → vote loop over all 3 pairs: GET `/vote?run={id}` →
   POST → follow redirect, assert reveal banner names the just-voted pair →
   run detail auto-reveals → `/leaderboard` shows 3 models with ratings and
   `3 votes counted` → `/votes` lists 3 votes → delete one (leaderboard
   then shows `2 votes counted`) → dashboard shows 1 task, 1 run, `2`
   votes, 1 unvoted pair (the deleted one is votable again).
   Route-coverage note: `/tasks` list/new/archive/unarchive and `/runs`
   list get one GET/POST each within this journey (single assertion each —
   deep per-route matrices live in issues 24–29, DESIGN §15).
4. Real-server test (`test_real_server.py`):
   - Setup: temp env with fake-provider config; seed one task + one
     completed 3-model run **via the repos** before server start (so votable
     pairs exist without racing the runner).
   - Start `uvicorn` in a subprocess via `mybench serve --port {free_port}`
     (find a free port by binding 0 first); wait for readiness (poll ≤ 5 s);
     drive with httpx + cookie jar: dashboard 200 with §13.5 headers;
     `Host: evil.com` → 403; CSRF over real HTTP: GET `/vote` (cookie set),
     parse the hidden `csrf_token` and output ids from the HTML, POST
     `/votes` with them → 303; negative cases: POST without the cookie →
     403, POST with `Origin: https://evil.com` → 403; graceful `SIGTERM`
     shutdown, exit code 0, bounded 10 s.
5. Blindness invariant at journey level: a shared helper
   `assert_no_blind_leaks(body, fixture)` implementing the full §8.6 list
   with the fixture's distinctive values (model ids, provider, base_url
   host, raw_model, latency, token counts — issue 27's technique). Applied
   to every `/vote` response in the web journey **after stripping the
   reveal banner element** (the banner names the *previous*, already-voted
   pair and is identified by a stable `id="reveal-banner"` attribute).
6. Journey assertions prefer user-visible surfaces (rendered HTML/CLI output)
   over DB peeking, except where noted (vote injection).
7. mypy-strict clean (tests included in mypy scope? No — mypy covers `src/`
   only per DESIGN §17; tests need only ruff).

## Acceptance Criteria

- [ ] All three journey files pass locally and in CI on Linux + macOS.
- [ ] CLI journey executes every §10 command per the requirement-2 flow
      (checklist comment at the top of the file mapping command → test
      line; `serve` maps to the real-server test).
- [ ] Web journey hits every §11.2 route at least once per the
      requirement-3 flow (same checklist style; status codes asserted).
- [ ] Expected counts of requirements 2/3 asserted literally at each marked
      step.
- [ ] Real-server test proves: loopback bind, Host 403, CSRF token
      round-trip over real HTTP incl. both negative cases, clean shutdown.
- [ ] `assert_no_blind_leaks` applied to every vote-loop response, with the
      reveal banner stripped by its `id`.
- [ ] Total e2e wall time ≤ 60 s locally (fake provider — no sleeps beyond
      polling).
- [ ] ruff, pytest green; coverage gates still met with e2e included.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/e2e -q` and full `uv run pytest -q`.

## Dependencies

16–19, 21, 22–29 (i.e., waves 4 and 5 complete).

## Non-goals

Browser automation (no JS engine — vote.js/runstatus.js behavior is manual QA
+ their own unit-testable simplicity); load testing; real-provider e2e
(manual, documented in issue 31's release checklist).

## Design References

DESIGN §15, §10, §11.2, §8.6; ISSUE_PLAN §1 (completion statement).
