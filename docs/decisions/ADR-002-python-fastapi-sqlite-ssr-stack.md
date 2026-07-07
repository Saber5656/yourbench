# ADR-002: Implementation stack — Python/uv, FastAPI, SQLite, server-side rendering

Date: 2026-07-08
Status: Accepted

## Context

mybench is a local, single-user tool (CLI + local web UI). Requirements
confirmed with the owner: Python stack (Q4-A), CLI + web hybrid (Q1-A),
fully local storage (Q7-A). Implementation will be executed by lower-capability
agents, so the stack must be mainstream, explicit, and low-magic. The project
will be published as OSS, so the dependency tree is a supply-chain surface and
must stay small.

## Decision

| Concern | Choice |
|---|---|
| Language / packaging | Python ≥ 3.11, `uv` managed, `src/mybench/` layout, `uv.lock` committed |
| CLI framework | `click` |
| Web framework | FastAPI + uvicorn (single process, single worker) |
| Templates | Jinja2 server-side rendering; plain HTML forms work with JavaScript disabled |
| Frontend JS | Minimal vanilla JS files served from `static/` (progressive enhancement only: keyboard shortcuts, status polling). No frontend framework, no CDN, no external requests |
| HTTP client | httpx (async) |
| Storage | SQLite via stdlib `sqlite3`, WAL mode; hand-written SQL and a numbered migration runner. No ORM |
| Markdown | markdown-it-py (raw HTML disabled) + nh3 sanitization |
| Rating math | Pure Python (no numpy) |
| Tests / QA | pytest, ruff, mypy, pip-audit; httpx MockTransport for provider tests |

Runtime dependency allowlist (v1): `click`, `fastapi`, `uvicorn`, `jinja2`,
`httpx`, `markdown-it-py`, `nh3`. Adding any runtime dependency beyond this
list requires an ADR update.

## Consequences

- No ORM/framework magic: every query and migration is visible SQL, which is
  easier for implementation agents to write correctly and for reviewers to audit.
- Single-process assumption (uvicorn `workers=1`) is a hard constraint;
  in-process run tracking relies on it (DESIGN.md §7).
- SSR + no-CDN keeps the privacy promise ("no network traffic except configured
  providers") verifiable and makes a strict CSP trivial.
- Pure-Python rating math bounds performance; acceptable at personal scale
  (see DESIGN.md §9 performance target), revisit only if profiling fails it.
