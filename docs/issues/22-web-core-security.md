# Title

Web core: app factory, security middleware, base templates, serve cmd

## Summary

Implement the web foundation: `web/app.py` (FastAPI factory), `web/security.py`
(host allowlist, CSRF double-submit, security headers), base/error templates,
static CSS, and `mybench serve` (loopback-only uvicorn).

## Context

This issue is boundary B1 and parts of B2/B6 (DESIGN §13.1–13.5, ADR-005).
Every page issue (24–29) mounts on this foundation, so the middleware contract
must be complete and tested before pages exist.

## Scope

- `src/mybench/web/app.py`, `src/mybench/web/security.py`
- `src/mybench/web/templates/base.html`, `error.html`
- `src/mybench/web/static/app.css`
- `src/mybench/cli/serve_cmd.py` (+ its `cli.add_command` line in
  `cli/main.py`)
- `tests/test_web_security.py`, `tests/test_serve_cmd.py`
- `tests/helpers.py` — shared assertion helper
  `assert_secure_response(response, *, html: bool)` used by every page
  issue (24–29): asserts the full §13.5 header set, `Cache-Control:
  no-store` iff html, and (for html) body contains no inline `<script>`
  without `src`, no `<style>`, no `on*=` attributes, no `|safe` artifacts.

## Detailed Requirements

1. `create_app(config: Config, db_path: Path, *, testing: bool = False)
   -> FastAPI` (DESIGN §11.1):
   - The Host allowlist and CSRF Origin check derive from
     `config.settings.port` — **`serve_cmd` resolves `--port` into an
     effective config before calling the factory**
     (`dataclasses.replace(config.settings, port=resolved)`); `create_app`
     itself never reads CLI state.
   - `app.state`: `config`, `db_path`, `runs_manager=None` placeholder
     (issue 26 wires it).
   - Startup handler: `connect` + `apply_migrations` +
     `runs.sweep_orphans(conn, older_than_hours=1)` (imported from issue
     08's repo — never reimplemented) + an idempotent
     `install_redaction_filter` re-assert, then close the startup
     connection. (Primary redaction install happens earlier in `serve_cmd`
     — requirement 5.)
   - Request-scoped DB: dependency `get_conn(request)` yielding a new
     connection per request, closed after response (FastAPI dependency with
     `yield`).
   - Jinja2 env: loader from package resources
     (`jinja2.PackageLoader("mybench.web", "templates")`), autoescape True.
     Exported rendering helper (normative — page issues 24–29 build every
     HTML response through it): `web.app.render(request: Request,
     template_name: str, *, status_code: int = 200, **ctx) ->
     HTMLResponse` — injects `csrf_token` (via `get_or_create_csrf_token`)
     and nav state into the context and attaches the Set-Cookie header when
     the token was newly generated.
   - Static mount `/static` from package resources; `Cache-Control:
     public, max-age=3600` for static, `no-store` for HTML (DESIGN §13.5).
   - Routers included from `web/routes/*` — this issue registers only `GET /`
     placeholder returning a minimal page via `base.html` (replaced in 24)
     and the error handlers: 404, 400/validation, **and a generic
     `Exception` handler** — all render `error.html` with status + safe
     message, never a stack trace (traceback to DEBUG logs only); §13.5
     headers apply to these responses too (middleware ordering must
     guarantee it).
2. `security.py`:
   - `HostValidationMiddleware(allowed_hosts: frozenset[str])`: computed at
     factory time from the port: `{"127.0.0.1", "localhost", "[::1]"} ×
     {"", ":{port}"}`. Non-matching `Host` → 403 plain-text `invalid host`
     (before any routing; applies to /static too — DESIGN §13.3).
   - `SecurityHeadersMiddleware`: sets, on **every** response including
     errors and static, exactly the DESIGN §13.5 header set (CSP string
     verbatim; `Cache-Control: no-store` only on `text/html`).
   - CSRF (DESIGN §13.4), exact contract:
     - Token lifecycle: `get_or_create_csrf_token(request) -> str` — returns
       the valid cookie value if present, else generates
       `secrets.token_urlsafe(32)` and stashes it on `request.state` so (a)
       the `render()` helper injects the SAME value into templates as
       `csrf_token`, and (b) a response hook sets the cookie (`mybench_csrf`,
       HttpOnly, SameSite=Strict, path=/) only when newly generated — a
       valid existing cookie is never rotated.
     - Enforcement: module-level dependency instance in
       `mybench/web/security.py`: `csrf_protect` (an instance of class
       `CsrfProtect`, callable as a FastAPI dependency). Usage pattern
       (normative, grep-checkable by issue 32): every POST route is declared
       as `@router.post(path, dependencies=[Depends(csrf_protect)])` with
       `from mybench.web.security import csrf_protect`. It returns 403
       unless: cookie exists, form field `csrf_token` exists, and
       `hmac.compare_digest(cookie, field)` passes; when an `Origin` header
       is present it must be in `allowed_origins =
       frozenset({f"http://127.0.0.1:{port}", f"http://localhost:{port}",
       f"http://[::1]:{port}"})` (DESIGN §13.4), else 403.
     - Probe routes, mounted only under `create_app(..., testing=True)`
       (DESIGN §11.1) so the mechanisms are testable before any real POST
       route exists and never ship in normal serving:
       | Method | Path | Behavior |
       |---|---|---|
       | GET | `/_csrf_probe` | renders a minimal page through `render()` containing one form with the hidden `csrf_token` |
       | POST | `/_csrf_probe` | `dependencies=[Depends(csrf_protect)]`; returns text `probe ok` |
       | GET | `/_raise_500` | raises `RuntimeError("probe")` (exercises the generic handler) |
3. `base.html`: `<!doctype html>`, `<html lang="en">`, `<meta charset>`,
   viewport meta, `<title>{% block title %}mybench{% endblock %}</title>`,
   `<link rel="stylesheet" href="/static/app.css">`, nav links
   (Dashboard `/`, Tasks `/tasks`, Vote `/vote`, Leaderboard `/leaderboard`,
   History `/runs`), `{% block content %}`, `{% block scripts %}` (for page
   JS via `<script src>` only). **Zero inline styles/scripts/event handlers**
   (CSP will block them; DESIGN §13.5). `error.html` extends base showing
   status code + safe message.
4. `app.css`: minimal neutral styling: nav bar, tables (bordered, padded),
   forms (labels above inputs), buttons, `.card` two-column grid for the vote
   page (`.pair-grid { display:grid; grid-template-columns:1fr 1fr }`),
   `.banner` for reveal/notices, `pre/code` wrapping
   (`white-space:pre-wrap; overflow-wrap:anywhere`). No external imports.
5. `serve_cmd.py` — `mybench serve [--port N] [--open]` (DESIGN §10.7),
   in order: resolve port (`--port` else `config.settings.port`) → build
   the effective config (`dataclasses.replace`) → **install the redaction
   filter immediately** (DESIGN §14: before app creation, so early logs are
   covered) → `create_app(effective_config, db_path)` →
   `uvicorn.run(app, host="127.0.0.1", port=port, workers=1,
   log_level="warning" or "debug" with -v, access_log=verbose)`.
   `--open` → `webbrowser.open(url)` after a short readiness wait (poll the
   port ≤ 3 s in a thread). Prints `mybench ui: http://127.0.0.1:{port}`.
   There is no host flag; assert in test that the CLI rejects `--host`.
6. mypy-strict clean.

## Acceptance Criteria

TestClient-based unless noted:

- [ ] Host allowlist accepts exactly the six values `127.0.0.1`,
      `127.0.0.1:{port}`, `localhost`, `localhost:{port}`, `[::1]`,
      `[::1]:{port}` (each tested); rejects `evil.com`,
      `127.0.0.1.evil.com`, `192.168.1.5:{port}`, `localhost:{other_port}`
      → 403 (incl. on `/static/app.css`).
- [ ] Header set golden-asserted on: 200 HTML, 404, 403, static (CSP/nosniff/
      frame/referrer on all; no-store on HTML only).
- [ ] CSRF via the `/_csrf_probe` route: first HTML GET sets the cookie and
      the page's hidden field equals the cookie value; a second GET with the
      valid cookie does NOT re-set it (no `Set-Cookie`); POST without cookie
      / without field / mismatched → 403 (probe handler not executed);
      matching → 200; `Origin: http://localhost:{port}` passes;
      `Origin: https://evil.com` → 403 even with valid token.
- [ ] 500 handling: `GET /_raise_500` (testing=True) with
      `TestClient(app, raise_server_exceptions=False)` → status 500,
      `error.html` body without `Traceback`, full §13.5 headers present;
      the probe routes are absent (404) when `testing=False`.
- [ ] Startup applies migrations + orphan sweep on a fresh tmp DB (sweep
      spy: the issue-08 function is called, not a local copy).
- [ ] Port override: app built for `--port 9000` accepts
      `Host: localhost:9000` and rejects `Host: localhost:8137` (403).
- [ ] base.html has no `style=`/`onclick=`/`<script>` without `src` (assert
      via template source scan in a test); `tests/helpers.py::
      assert_secure_response` implemented and self-tested.
- [ ] `serve` flow verified by monkeypatching exactly
      `mybench.safety.install_redaction_filter`,
      `mybench.web.app.create_app`, and `uvicorn.run`: call order is
      redaction → create_app → uvicorn.run; create_app receives the
      effective config with the resolved port; uvicorn receives host
      `127.0.0.1`, `workers=1`, the resolved port (real-bind smoke lives in
      issue 30). `serve` consumes the issue-16 `CliContext` (config already
      loaded) rather than re-loading config itself.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_web_security.py tests/test_serve_cmd.py -q`;
optional manual QA (non-gating): `mybench serve` then `curl -H "Host: evil.com" -i
http://127.0.0.1:8137/` → 403, and browser devtools header inspection.

## Dependencies

04, 05, 06, 08 (sweep_orphans import), 16.

## Non-goals

All product pages (24–29), runs manager (26), markdown rendering (23),
authentication (v2 per ADR-005), TLS.

## Design References

DESIGN §11.1, §11.3–11.4, §10.7, §13.2–13.5, §14; ADR-005.
