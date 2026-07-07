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
- `src/mybench/cli/serve_cmd.py`
- `tests/test_web_security.py`, `tests/test_serve_cmd.py`

## Detailed Requirements

1. `create_app(config: Config, db_path: Path) -> FastAPI` (DESIGN §11.1):
   - `app.state`: `config`, `db_path`, `runs_manager=None` placeholder
     (issue 26 wires it).
   - Startup handler: `connect` + `apply_migrations` + `sweep_orphans` +
     `install_redaction_filter` (§14), then close the startup connection.
   - Request-scoped DB: dependency `get_conn(request)` yielding a new
     connection per request, closed after response (FastAPI dependency with
     `yield`).
   - Jinja2 env: loader from package resources
     (`jinja2.PackageLoader("mybench.web", "templates")`), autoescape True,
     `render(request, template, **ctx)` helper injecting `csrf_token` and
     nav state into every context.
   - Static mount `/static` from package resources; `Cache-Control:
     public, max-age=3600` for static, `no-store` for HTML (DESIGN §13.5).
   - Routers included from `web/routes/*` — this issue registers only `GET /`
     placeholder returning a minimal page via `base.html` (replaced in 24)
     and the error handlers: 404 and 400/validation → `error.html` with
     status + message (no stack traces ever; DEBUG logs get the traceback).
2. `security.py`:
   - `HostValidationMiddleware(allowed_hosts: frozenset[str])`: computed at
     factory time from the port: `{"127.0.0.1", "localhost", "[::1]"} ×
     {"", ":{port}"}`. Non-matching `Host` → 403 plain-text `invalid host`
     (before any routing; applies to /static too — DESIGN §13.3).
   - `SecurityHeadersMiddleware`: sets, on **every** response including
     errors and static, exactly the DESIGN §13.5 header set (CSP string
     verbatim; `Cache-Control: no-store` only on `text/html`).
   - CSRF (DESIGN §13.4): `ensure_csrf_cookie(request, response)` — on HTML
     GET without valid `mybench_csrf` cookie, set one
     (`secrets.token_urlsafe(32)`, HttpOnly, SameSite=Strict, path=/);
     `require_csrf(request, form)` dependency for every POST route:
     403 unless cookie exists, form field `csrf_token` exists, and
     `hmac.compare_digest` passes; when an `Origin` header is present it must
     equal `http://{host}` for an allowed host:port, else 403. Provide a
     FastAPI dependency `CsrfProtect` that page issues declare on POST
     routes — grep-checkable pattern for issue 32.
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
5. `serve_cmd.py` — `mybench serve [--port N] [--open]` (DESIGN §10.7):
   builds app, `uvicorn.run(app, host="127.0.0.1", port=port,
   workers=1, log_level="warning" or "debug" with -v, access_log=verbose)`.
   `--open` → `webbrowser.open(url)` after a short readiness wait (poll the
   port ≤ 3 s in a thread). Prints `mybench ui: http://127.0.0.1:{port}`.
   There is no host flag; assert in test that the CLI rejects `--host`.
6. mypy-strict clean.

## Acceptance Criteria

TestClient-based unless noted:

- [ ] Host allowlist: `Host: localhost:{port}` etc. pass; `Host: evil.com`,
      `Host: 127.0.0.1.evil.com`, `Host: 192.168.1.5:{port}` → 403 (incl. on
      `/static/app.css`).
- [ ] Header set golden-asserted on: 200 HTML, 404, 403, static (CSP/nosniff/
      frame/referrer on all; no-store on HTML only).
- [ ] CSRF: GET sets cookie once (not re-set when valid); POST without
      cookie / without field / mismatched → 403; matching → passes;
      `Origin: http://localhost:{port}` passes; `Origin: https://evil.com`
      → 403 even with valid token.
- [ ] Error pages contain no traceback text (raise a deliberate error in a
      test-only route; body lacks `Traceback`).
- [ ] Startup applies migrations + orphan sweep on a fresh tmp DB.
- [ ] base.html has no `style=`/`onclick=`/`<script>` without `src` (assert
      via template source scan in a test).
- [ ] `serve` binds 127.0.0.1 (assert uvicorn invocation args via monkeypatch;
      real-bind smoke lives in issue 30).
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_web_security.py tests/test_serve_cmd.py -q`;
manual: `mybench serve` then `curl -H "Host: evil.com" -i
http://127.0.0.1:8137/` → 403, and browser devtools header inspection.

## Dependencies

04, 05, 06, 16.

## Non-goals

All product pages (24–29), runs manager (26), markdown rendering (23),
authentication (v2 per ADR-005), TLS.

## Design References

DESIGN §11.1, §11.3–11.4, §10.7, §13.2–13.5, §14; ADR-005.
