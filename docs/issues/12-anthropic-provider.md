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

1. `class AnthropicProvider` implementing `Provider` (incl. `aclose()`);
   same constructor shape as issue 11, client from
   `providers.base.default_client(settings)` (issue 10), same size cap,
   retry rules (timeout caught before TransportError; 3xx → non-retryable
   connection error), latency measurement, and **key scrubbing** of all
   provider-derived text as issue 11 requirement 1 — with `x-api-key` as the
   echoed-header threat.
2. Request: POST `{model_cfg.base_url}/v1/messages` — `base_url` is always
   present (config load normalizes the Anthropic default, issue 04; the
   loopback-only `http://` rule is likewise enforced at config validation,
   which `build()` cannot bypass since it only accepts a validated
   `ModelConfig`). Headers `x-api-key: {key}` (key required — enforced by
   `build()`, issue 10), `anthropic-version: 2023-06-01`, user-agent
   `mybench/{__version__}`. Body: `model`, `max_tokens` = request value or
   **4096** (API requires it; DESIGN §6.6),
   `messages=[{"role":"user","content": user_prompt}]`, top-level `system`
   only when not None, `temperature`/`top_p` only when not None, no `stream`
   key.
3. Response: content = concatenation of `block["text"]` for blocks with
   `type == "text"` (empty result → invalid_response);
   `usage.input_tokens → prompt_tokens`,
   `usage.output_tokens → completion_tokens` (optional-safe);
   `stop_reason → finish_reason` passthrough (`max_tokens` value indicates
   truncation; UI mapping is issue 26/27 concern); `raw_model` from `model`.
4. Errors: non-2xx JSON `{"type":"error","error":{"type","message"}}` →
   message extracted best-effort, then key-scrubbed (the
   `ProviderError` constructor from issue 10 handles control-stripping and
   300-char truncation); status mapping via `ProviderError.from_status`
   (issue 10); 429 honors `retry-after` header (cap 30 s).
5. `check()`: `complete(CompletionRequest(system_prompt=None,
   user_prompt="ping", temperature=None, top_p=None, max_tokens=1))` →
   same result semantics as issue 11.
6. Logging rules identical to issue 11 (DESIGN §14).
7. mypy-strict clean.

## Acceptance Criteria

- [ ] Happy path: headers exact (`x-api-key`, `anthropic-version`,
      `User-Agent: mybench/{version}`), system top-level (not a message),
      `max_tokens` defaulted to 4096 when request has None, params omitted
      when None; request URL uses the config-normalized default base_url.
- [ ] `build()` integration: `PROVIDERS`-mapped construction succeeds with
      `ANTHROPIC_API_KEY` set and raises the issue-10 auth error naming the
      var when unset.
- [ ] Multi-block content concatenated; non-text blocks (e.g.
      `{"type":"tool_use"}`) skipped; all-non-text → invalid_response.
- [ ] `stop_reason: "max_tokens"` surfaces as finish_reason `max_tokens`.
- [ ] usage mapping including missing-usage → None.
- [ ] Error matrix: 401 → auth; 429 with retry-after → retried once (2
      requests); 500 then 200 → success; 529 → retryable api_error; 400 →
      api_error non-retryable with extracted message; timeout → no retry,
      single attempt; connection error then 200 → success; invalid JSON 2xx
      → invalid_response; malformed content blocks → invalid_response.
- [ ] Key scrubbing: error body echoing the `x-api-key` value → message
      contains `[REDACTED]`, not the key; ANSI in error body stripped.
- [ ] Size cap and redirect behavior (same three size-cap cases and the 3xx
      case as issue 11, adapted).
- [ ] `check()` ok/failure paths with the exact request body of req. 5.
- [ ] Issue 11 test suite still green if a shared transport refactor was done.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_anthropic_provider.py tests/test_openai_compat.py -q`.

## Dependencies

10 (11 recommended first for the shared-transport decision).

## Non-goals

Tool use, multi-turn, streaming, prompt caching (all v1 non-goals); Gemini or
other native adapters (v2).

## Design References

DESIGN §6.6, §6.8, §13.1 B4, §14; research/provider-api-variance.md §4.
