# Title

Security acceptance sweep: verify every §13 control

## Summary

Systematically verify that every security control in DESIGN §13 is
implemented and tested; add the missing tests found; add static grep-checks
that lock in the codebase-wide invariants; produce
`docs/security-verification.md` mapping each control to its evidence.

## Context

DESIGN §13.9 mandates this gate. Individual issues each carry their own
security criteria; this issue is the adversarial re-check that nothing
slipped between them (the classic failure mode of per-module security).

## Scope

- `docs/security-verification.md` (new)
- `tests/test_security_invariants.py` (new static/grep checks)
- Gap-fix tests in existing test files (small code fixes allowed **only** for
  found violations of already-specified behavior; behavior changes need a
  DESIGN update first)

## Detailed Requirements

1. Build the verification matrix `docs/security-verification.md`: **one row
   per atomic control with a stable ID** (not one per boundary) →
   implementing module → test id(s) (file::test) or written justification.
   Every row must end `VERIFIED` or `ACCEPTED-RISK (owner-approved)`; no
   third state. Row inventory (minimum; derive any further atomic controls
   found in §13.2–§13.8 while writing the matrix, including one row per
   §13.8 abuse case):
   - B1-host (incl. static routes), B1-csrf-cookie-flags, B1-csrf-token,
     B1-origin, B1-403-paths, B1-clickjacking, B1-nosniff.
   - B2-pipeline-constants, B2-autoescape, B2-no-safe-filter,
     B2-error-escape, B2-vote-leak-list (§8.6), B2-headers-all-responses.
   - B3-cli-echo — one sub-row per CLI command that prints model/task text
     (models check, task list/show, run progress+summary, leaderboard),
     each pointing at its stripping test.
   - B4-tls-verify, B4-http-loopback-only, B4-size-cap (incl. error
     bodies), B4-no-redirect, B4-retry-bounds, B4-timeout-no-retry,
     B4-key-not-in-db/exports/logs (redaction installed in BOTH entry
     points), B4-auth-error-names-env, B4-key-scrub-in-adapters.
   - B5-datadir-0700, B5-db-0600, B5-export-0600-excl.
   - B6-dep-allowlist, B6-lockfile, B6-pip-audit, B6-no-external-
     subresources.
   - B7-post-csrf-coverage, B7-validators-at-write-paths, B7-param-sql.
2. `tests/test_security_invariants.py` — automated, permanent checks (these
   outlive this issue):
   - No `|safe` in `templates/**`; no inline `<script>` without `src`, no
     `<style>`, no `on*=` attributes in templates.
   - No `verify=False`, no `follow_redirects=True` under `src/`.
   - `MarkdownIt`/`nh3` imported only in `web/render.py`.
   - FastAPI app introspection: the set of POST routes equals the DESIGN
     §11.2 POST table, and every one declares the issue-22 dependency
     (`from mybench.web.security import csrf_protect`; assert
     `csrf_protect` in each route's dependency list). Behavioral backstop:
     parametrized test POSTs to every POST route without a token → all 403.
   - External-subresource invariant: no `src=`/`href=` attribute in
     templates/static references an external origin **except** `<a href>`
     navigation links, which are allowlisted exactly to
     `https://github.com/Saber5656/mybench/...` (DESIGN §13.5).
   - `pyproject.toml` runtime deps set equality with ADR-002 list.
   - No `datetime.now()` without `timezone.utc` in `src/` (ADR-006).
   - SQL discipline via AST walk over **all of `src/`**: every call to
     `execute`/`executemany`/`executescript` must have a first argument
     that is a string literal or a name bound to a module-level string
     constant (rejects JoinedStr, `%`, `.format`, concatenation, and
     locally built strings); and `sqlite3` may be imported only under
     `mybench/db/`.
3. Manual adversarial pass — the appendix in `docs/security-verification.md`
   is written from a fixed template committed with this issue, one section
   per check with: exact setup commands, exact attack command, expected
   result, actual transcript, artifact path. Checks: (a) DNS-rebinding curl
   matrix (`curl -H "Host: evil.com" -i http://127.0.0.1:{port}/` and
   variants incl. `/static/app.css`); (b) CSRF cross-origin POST from a
   local `file://` HTML page with a prefilled form (expected: 403, no vote
   row); (c) XSS: fake-provider task whose output embeds the issue-23
   corpus, screenshot of rendered vote page (artifact under
   `docs/assets/security/`); (d) ANSI injection: task titled with
   `\x1b]0;pwned\x07` displayed via every B3 CLI command, transcripts
   showing stripped output; (e) oversized response: local mock server
   (committed under `tests/mock_servers/`) returning 100 MB, expect
   `content_too_large` within the cap.
4. Each gap found: file a fix within this issue **only if** the behavior was
   already specified (regression); otherwise open a new issue + DESIGN
   amendment and record it in the matrix as a linked exception. Zero silent
   scope changes.

## Acceptance Criteria

- [ ] `docs/security-verification.md` complete: every atomic-control row of
      requirement 1 present with evidence; zero unresolved rows; every
      §13.8 abuse case has its own row.
- [ ] `tests/test_security_invariants.py` passes and is wired into the normal
      pytest run (not optional).
- [ ] Manual adversarial appendix filled per the template with actual
      command transcripts and artifacts.
- [ ] Any fixes landed have accompanying regression tests.
- [ ] Exact §15 coverage gates confirmed by CI output linked in the PR:
      total ≥ 85%; ≥ 95% each for `mybench/rating/*`,
      `mybench/web/security.py`, `mybench/web/render.py`,
      `mybench/safety.py`.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

Full `uv run pytest -q` + CI link; reviewer replays at least the
DNS-rebinding and CSRF manual checks from the appendix.

## Dependencies

30 (whole product assembled; 31 may proceed in parallel).

## Non-goals

New security features (e.g. auth, encryption — v2/ADR territory); external
pentest; fuzzing infrastructure.

## Design References

DESIGN §13 (all), §8.6, §15; ADR-002, ADR-004, ADR-005, ADR-006.
