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

1. Build the verification matrix `docs/security-verification.md`: one row per
   control below → implementing module → test id(s) (file::test) or written
   justification. Every row must end `VERIFIED` or `ACCEPTED-RISK
   (owner-approved)`; no third state. Controls to cover (from §13):
   - B1: host allowlist incl. static routes; CSRF token + compare_digest +
     Origin check; cookie flags (HttpOnly, SameSite=Strict, path=/); 403
     paths.
   - B2: markdown_safe pipeline constants; autoescape on in the Jinja env;
     zero `|safe` in templates; error text via escape_pre; §8.6 vote-page
     leak list; §13.5 header set on every response class.
   - B3: strip_terminal_controls applied in every CLI echo path of
     model/task text — enumerate every CLI command that prints such text
     (models check, task show/list, run progress+summary, leaderboard
     titles) and point each at its stripping test.
   - B4: TLS verify never disabled; http-loopback-only config rule; 5 MB
     caps incl. error bodies; no-redirect; retry bounds; timeout no-retry;
     key never in DB/exports/logs (redaction filter installed in BOTH entry
     points); auth errors name env var not value.
   - B5: data dir 0700 (create + tighten); DB 0600; export 0600 + O_EXCL.
   - B6: runtime deps == ADR-002 allowlist (parse pyproject in test);
     uv.lock committed; pip-audit in CI; zero external URLs in templates/
     static (grep http:// and https:// in templates+static; allowlist the
     repo doc link from issue 28).
   - B7: every POST route has the CSRF dependency (static route-table
     introspection test); §5.4 limits enforced at repo layer (not only UI).
2. `tests/test_security_invariants.py` — automated, permanent greps/
   introspections (these outlive this issue):
   - No `|safe` in `templates/**`; no inline `<script>`/`<style>`/`on*=` in
     templates.
   - No `verify=False`, no `follow_redirects=True` under `src/`.
   - `MarkdownIt`/`nh3` imported only in `web/render.py`.
   - FastAPI app introspection: every POST route depends on CsrfProtect;
     every route module registered.
   - `pyproject.toml` runtime deps set equality with ADR-002 list.
   - No `datetime.now()` without `timezone.utc` in `src/` (ADR-006).
   - sqlite: no f-string/`%` SQL construction (`grep` for `f"` adjacent to
     `execute` heuristics — implement as AST walk over `db/` for
     `JoinedStr` inside `execute` calls).
3. Manual adversarial pass (documented with commands + results in
   `docs/security-verification.md` appendix): DNS-rebinding curl matrix;
   CSRF cross-origin POST attempt from a file:// page; XSS attempt via a
   fake-provider task crafted with hostile markdown (screenshot); ANSI
   injection via task text through every CLI display command; oversized
   provider response via a local mock server.
4. Each gap found: file a fix within this issue **only if** the behavior was
   already specified (regression); otherwise open a new issue + DESIGN
   amendment and record it in the matrix as a linked exception. Zero silent
   scope changes.

## Acceptance Criteria

- [ ] `docs/security-verification.md` complete: every §13.1–13.8 control has
      a row with evidence; zero unresolved rows.
- [ ] `tests/test_security_invariants.py` passes and is wired into the normal
      pytest run (not optional).
- [ ] Manual adversarial appendix filled with actual command transcripts.
- [ ] Any fixes landed have accompanying regression tests.
- [ ] Coverage gates (§15: security modules ≥ 95%) confirmed by CI output
      linked in the PR.
- [ ] ruff, mypy strict, pytest green.

## Validation

Full `uv run pytest -q` + CI link; reviewer replays at least the
DNS-rebinding and CSRF manual checks from the appendix.

## Dependencies

30 (whole product assembled; 31 may proceed in parallel).

## Non-goals

New security features (e.g. auth, encryption — v2/ADR territory); external
pentest; fuzzing infrastructure.

## Design References

DESIGN §13 (all), §8.6, §15; ADR-002, ADR-004, ADR-005, ADR-006.
