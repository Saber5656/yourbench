# Title

Anthropic provider adapter

## Summary

Implement `mybench/providers/anthropic.py`: native Messages API adapter with
the same transport policy as issue 11 and Anthropic-specific request/response
mapping.

## Context

DESIGN §6.6; wire differences catalogued in
`docs/research/provider-api-variance.md` §4. Shares transport hardening
requirements with issue 11 (boundary B4).

## Scope

- `src/mybench/providers/anthropic.py`
- `tests/test_anthropic_provider.py` (httpx.MockTransport)
- If (and only if) extracting shared transport helpers from issue 11 avoids
  duplication, a `providers/_transport.py` internal module refactor is
  allowed; behavior of issue 11 must not change (its tests stay green).

## Detailed Requirements

1. `class AnthropicProvider` implementing `Provider`; same constructor shape,
   client policy (timeouts, verify, no redirects), size cap, retry rules, and
   latency measurement as issue 11 (DESIGN §6.8).
2. Request: POST `{base_url or "https://api.anthropic.com"}/v1/messages`;
   headers `x-api-key: {key}` (key required — enforced by `build()`),
   `anthropic-version: 2023-06-01`, user-agent `mybench/{__version__}`.
   Body: `model`, `max_tokens` = request value or **4096** (API requires it;
   DESIGN §6.6), `messages=[{"role":"user","content": user_prompt}]`,
   top-level `system` only when not None, `temperature`/`top_p` only when not
   None, no `stream` key.
3. Response: content = concatenation of `block["text"]` for blocks with
   `type == "text"` (empty result → invalid_response);
   `usage.input_tokens → prompt_tokens`,
   `usage.output_tokens → completion_tokens` (optional-safe);
   `stop_reason → finish_reason` passthrough (`max_tokens` value indicates
   truncation; UI mapping is issue 26/27 concern); `raw_model` from `model`.
4. Errors: non-2xx JSON `{"type":"error","error":{"type","message"}}` →
   message extracted best-effort; mapping via `ProviderError.from_status`;
   429 honors `retry-after` header (cap 30 s). Timeout/connection semantics
   identical to issue 11.
5. `check()` identical semantics to issue 11 (`max_tokens=1`).
6. Logging rules identical to issue 11 (DESIGN §14).
7. mypy-strict clean.

## Acceptance Criteria

- [ ] Happy path: headers exact (`x-api-key`, `anthropic-version`), system
      top-level (not a message), `max_tokens` defaulted to 4096 when request
      has None, params omitted when None.
- [ ] Multi-block content concatenated; non-text blocks (e.g.
      `{"type":"tool_use"}`) skipped; all-non-text → invalid_response.
- [ ] `stop_reason: "max_tokens"` surfaces as finish_reason `max_tokens`.
- [ ] usage mapping including missing-usage → None.
- [ ] 401 → auth; 429 with retry-after → retried once; 400 (e.g. missing
      max_tokens error shape) → api_error non-retryable with extracted
      message; 529 (Anthropic overloaded) → retryable api_error.
- [ ] Size cap and redirect behavior (same tests as 11 adapted).
- [ ] `check()` ok/failure paths.
- [ ] Issue 11 test suite still green if a shared transport refactor was done.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_anthropic_provider.py tests/test_openai_compat.py -q`.

## Dependencies

10 (11 recommended first for the shared-transport decision).

## Non-goals

Tool use, multi-turn, streaming, prompt caching (all v1 non-goals); Gemini or
other native adapters (v2).

## Design References

DESIGN §6.6, §6.8, §13.1 B4, §14; research/provider-api-variance.md §4.
