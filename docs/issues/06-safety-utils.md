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
   result length ≤ `limit` (marker included when truncation happened).
4. `redact(s: str, secrets: Sequence[str]) -> str` — replaces each non-empty
   secret value with `[REDACTED]`; longest-first replacement order.
5. `install_redaction_filter(secret_env_vars: Sequence[str]) -> None`
   (DESIGN §14): reads current values of the named env vars (skips
   unset/empty), attaches a `logging.Filter` to the **root logger's handlers
   and the `mybench` logger** that rewrites `record.getMessage()` output via
   `redact` (implementation: set `record.msg = redact(record.getMessage(),
   ...); record.args = None`). Idempotent (calling twice does not stack
   duplicate filters — use a module-level marker attribute).
6. `validate_task_params(*, user_prompt, system_prompt, title, category,
   temperature, top_p, max_tokens) -> list[str]` — returns a list of
   human-readable violations (empty = valid), applying §5.4 rules; `category`
   `None` → treated as `"general"`. Callers (CLI issue 18, web issues 25/26)
   render these messages verbatim.
7. Pure stdlib; mypy-strict clean.

## Acceptance Criteria

- [ ] ANSI corpus test: for each of `\x1b[31mred\x1b[0m`,
      `\x1b]0;title\x07`, `\x1b]8;;http://evil\x1b\\link\x1b]8;;\x1b\\`,
      `a\x07b`, `a\x00b`, `a\x9bXb`, output contains no byte < 0x20 except
      `\n`/`\t` and no `\x1b`/`\x9b`; plain text with `\n`/`\t` passes
      through unchanged.
- [ ] `redact` replaces multiple distinct secrets and overlapping cases
      (longer secret containing a shorter one) correctly.
- [ ] Redaction filter: a logger call containing a fake key value emits
      `[REDACTED]`; installing twice doesn't duplicate output or filters.
- [ ] `validate_task_params` table-driven tests: at least one violation per
      §5.4 rule and a fully-valid case returning `[]`.
- [ ] `truncate` boundary cases (exact limit, limit < marker length).
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_safety.py -q`.

## Dependencies

01.

## Non-goals

Markdown/HTML sanitization (23); where limits are *enforced* (repos, CLI, web
— their issues); provider message sanitization (10).

## Design References

DESIGN §5.4, §5.5, §13.1 B3, §13.6, §14.
