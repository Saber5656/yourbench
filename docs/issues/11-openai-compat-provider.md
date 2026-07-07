# Title

OpenAI-compatible provider adapter

## Summary

Implement `mybench/providers/openai_compat.py`: the generic
`/chat/completions` adapter with defensive parsing, the §6.8 transport policy
(timeouts, size cap, no redirects, single retry with backoff), and full error
mapping.

## Context

This one adapter covers OpenAI, OpenRouter, Ollama, vLLM, LM Studio (ADR-004).
Known wire variance and the rules for handling it are catalogued in
`docs/research/provider-api-variance.md` §2–3; hostile-response handling is
boundary B4.

## Scope

- `src/mybench/providers/openai_compat.py`
- `tests/test_openai_compat.py` (httpx.MockTransport)

## Detailed Requirements

1. `class OpenAICompatProvider` implementing the `Provider` protocol;
   constructor `(model_cfg: ModelConfig, settings: Settings,
   api_key: str | None)`; owns one `httpx.AsyncClient` created with
   `verify=True`, `follow_redirects=False`,
   `timeout=httpx.Timeout(connect=10.0, read=settings.timeout_seconds,
   write=30.0, pool=10.0)`; `async close()` closes it (runner/CLI call it via
   `contextlib.aclosing`-style usage; also implement `__aenter__/__aexit__`).
2. Request (DESIGN §6.5 + research §2): POST `{base_url}/chat/completions`
   (join with a single slash regardless of trailing slash in config), JSON
   body: `model`, `messages` (system message only when `system_prompt` is not
   None), `stream: false`; include `temperature`/`top_p`/`max_tokens` only
   when not None. Header `Authorization: Bearer {key}` only when a key was
   resolved. No other headers beyond content-type/user-agent
   (`mybench/{__version__}`).
3. Response parsing:
   - Non-2xx → `ProviderError.from_status` with message from
     `error.message` (dict), else body text; body read capped (req. 4).
   - 2xx: JSON parse failure / missing `choices[0].message` →
     `invalid_response`. `content` string → use; list → concatenate
     `part["text"]` for `part["type"] == "text"` (missing text key →
     `invalid_response`); other types → `invalid_response`. Empty/whitespace
     result → `invalid_response` with message `empty completion content`.
   - `usage.prompt_tokens`/`usage.completion_tokens`: ints when present and
     non-negative, else None (no exception). `finish_reason` passthrough.
     `raw_model` from top-level `model` when present.
   - `latency_ms` measured around the HTTP call (monotonic clock), including
     the retried attempt only (last attempt's latency).
4. Size cap (DESIGN §6.8): if `Content-Length` > 5,242,880 → abort without
   reading; otherwise stream-read chunks accumulating length, abort past the
   cap → `ProviderError(kind=content_too_large, message="response exceeded 5
   MB cap")`. Applies to error bodies too (read at most the cap).
5. Retry (DESIGN §6.8): at most one retry, only when the mapped error has
   `retryable=True`; sleep = `Retry-After` seconds when the header parses as
   a number (cap 30) else `2 + random.uniform(0, 1)`; `asyncio.sleep`. No
   retry on timeout (`httpx.TimeoutException` → `ProviderError(timeout,
   retryable=False)` directly). Connection errors
   (`httpx.TransportError`) → `connection`, retryable.
6. `check()` (DESIGN §10.4): `complete(CompletionRequest(system_prompt=None,
   user_prompt="ping", temperature=None, top_p=None, max_tokens=1))` →
   `CheckResult(ok=True, message=f"ok ({latency_ms} ms)")`; on ProviderError
   → `CheckResult(ok=False, message=str(err), latency_ms=None)`.
7. No logging of request/response bodies at INFO; DEBUG logs truncated to 500
   chars via `safety.truncate` (DESIGN §14).
8. mypy-strict clean (parse boundary may use narrow `# type: ignore` or
   `typing.cast` with comments).

## Acceptance Criteria

MockTransport tests, each asserting behavior AND (where relevant) the outgoing
request:

- [ ] Happy path: full body → result fields exact; auth header present;
      system message included/excluded correctly; params omitted when None.
- [ ] Loopback keyless config sends no Authorization header.
- [ ] Content as parts array → concatenated; part without `text` →
      invalid_response.
- [ ] Missing `usage` → tokens None; partial usage → partial None.
- [ ] 401 → auth, non-retryable, message from `error.message`.
- [ ] 429 with `Retry-After: 0` → one retry then success (2 requests
      recorded); 429 twice → rate_limit error after exactly 2 attempts.
- [ ] 500 then 200 → success; 400 → api_error, single attempt.
- [ ] Redirect (301) → connection error, no follow (single request).
- [ ] Timeout → timeout error, single attempt.
- [ ] 6 MB body → content_too_large (both via Content-Length and via chunked
      stream without Content-Length).
- [ ] Invalid JSON 2xx → invalid_response; empty content → invalid_response.
- [ ] Error body with ANSI escapes → message stripped (via base ctor).
- [ ] `check()` ok and failure paths.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_openai_compat.py -q`. Manual (optional, documented
in PR if run): one real call against Ollama or OpenRouter.

## Dependencies

10.

## Non-goals

Anthropic wire format (12); streaming (v1 non-goal); `max_completion_tokens`
fallback (Known Unknown U1 — do not implement speculatively).

## Design References

DESIGN §6.5, §6.8, §10.4, §13.1 B4, §14; research/provider-api-variance.md.
