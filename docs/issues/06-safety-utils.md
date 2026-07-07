# Title

Safety utilities: limits, ANSI stripping, redaction, param validation

## Summary

Implement `mybench/safety.py`: the single source for input limit constants,
task parameter validation, terminal control-character stripping, string
truncation, secret redaction, and the logging redaction filter installer.

## Context

Terminal escape injection from model outputs (boundary B3) and secret leakage
into logs (§13.6) are both mitigated here. CLI and web forms share the §5.4
validation rules via this module so limits can never drift apart.

## Scope

- `src/mybench/safety.py`
- `tests/test_safety.py`

## Detailed Requirements

1. Limit constants (names exact; values from DESIGN §5.4):
   `MAX_USER_PROMPT_CHARS = 200_000`, `MAX_SYSTEM_PROMPT_CHARS = 50_000`,
   `MAX_TITLE_CHARS = 200`, `MAX_NOTE_CHARS = 2_000`,
   `CATEGORY_RE = re.compile(r"^[a-z0-9][a-z0-9-]{0,31}$")`,
   `TEMPERATURE_RANGE = (0.0, 2.0)`, `TOP_P_RANGE = (0.0, 1.0)`,
   `MAX_TOKENS_RANGE = (1, 128_000)`.
2. `strip_terminal_controls(s: str) -> str`:
   - removes ANSI CSI sequences (`\x1b[` ... final byte `@`–`~`), OSC
     sequences (`\x1b]` ... terminated by `\x07` or `\x1b\\`), other
     `\x1b`-prefixed two-byte escapes, and all C0 controls except `\n` and
     `\t`; also strips `\x7f` and C1 range `\x80–\x9f`.
   - Must be regex-based and total (never raises on any str input).
3. `truncate(s: str, limit: int, marker: str = "…[truncated]") -> str` —
   result length ≤ `limit`; marker included when truncation happened; when
   `limit <= len(marker)` return `marker[:limit]` (so `limit=0` → `""`).
4. `redact(s: str, replacements: Sequence[tuple[str, str]]) -> str` —
   replaces each `(secret_value, replacement)` pair (skip empty secrets),
   applying longest secrets first (DESIGN §5.5).
5. `install_redaction_filter(secret_env_vars: Sequence[str]) -> None`
   (DESIGN §5.5/§14): reads current values of the named env vars (skips
   unset/empty) building pairs `(value, f"[REDACTED:{name}]")`, then wraps
   the formatter of every root-logger handler in a `RedactingFormatter`
   whose `format(record)` = `redact(inner.format(record), pairs)` — this
   covers message, args, **and exception traceback text**. Handlers added
   later are not covered (documented limitation; both entry points install
   before other logging setup). Idempotent via an `_is_redacting` marker
   attribute on wrapped formatters.
6. `validate_task_params(*, user_prompt, system_prompt, title, category,
   temperature, top_p, max_tokens) -> list[str]` — returns human-readable
   violations (empty = valid) applying the §5.4 **task** rules; `category`
   `None` → treated as `"general"`. Exact messages (normative; callers
   render verbatim, tests assert equality):
   - `user_prompt: required` / `user_prompt: exceeds 200000 characters`
   - `system_prompt: exceeds 50000 characters`
   - `title: exceeds 200 characters`
   - `category: must match [a-z0-9][a-z0-9-]{0,31}`
   - `temperature: must be between 0 and 2`
   - `top_p: must be between 0 and 1`
   - `max_tokens: must be an integer between 1 and 128000`
   Order: field order as listed above. The vote-`note` limit constant lives
   here but is enforced by `votes.create` (issue 09), not by this function.
7. Pure stdlib; mypy-strict clean.

## Acceptance Criteria

- [ ] ANSI corpus test: for each of `\x1b[31mred\x1b[0m`,
      `\x1b]0;title\x07`, `\x1b]8;;http://evil\x1b\\link\x1b]8;;\x1b\\`,
      `a\x07b`, `a\x00b`, `a\x9bXb`, output contains no byte < 0x20 except
      `\n`/`\t` and no `\x1b`/`\x9b`; plain text with `\n`/`\t` passes
      through unchanged.
- [ ] `redact` replaces multiple distinct secrets and overlapping cases
      (longer secret containing a shorter one) correctly.
- [ ] Redaction formatter: with `FAKEKEY=sekret` configured, a log record
      whose message contains `sekret` emits `[REDACTED:FAKEKEY]` and never
      `sekret`; an exception logged via `logger.exception` whose traceback
      text contains `sekret` is also redacted; installing twice doesn't
      double-wrap (formatter count unchanged).
- [ ] `validate_task_params` table-driven tests: each normative message of
      requirement 6 asserted by exact string equality, plus a fully-valid
      case returning `[]` and a multi-violation case preserving field order.
- [ ] `truncate` boundary cases: exact limit (no marker), one-over,
      `limit == len(marker)`, `limit < len(marker)`, `limit = 0`.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_safety.py -q`.

## Dependencies

01.

## Non-goals

Markdown/HTML sanitization (23); where limits are *enforced* (repos, CLI, web
— their issues); provider message sanitization (10).

## Design References

DESIGN §5.4, §5.5, §13.1 B3, §13.6, §14.
