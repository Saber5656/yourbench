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
   api_key: str | None)`; owns one `httpx.AsyncClient` obtained from
   `providers.base.default_client(settings)` (issue 10 — verify on, no
   redirects, §6.8 timeouts); `async aclose()` closes it (protocol method the
   runner calls in `finally`).
   **Key scrubbing (§6.2, B4):** the adapter keeps the resolved `api_key`
   and applies `text.replace(api_key, "[REDACTED]")` to every
   provider-derived string (error bodies, debug-log payloads) *before*
   constructing `ProviderError`s or log records — a hostile server may echo
   the `Authorization` header back.
2. Request (DESIGN §6.5 + research §2): POST `{base_url}/chat/completions`
   (join with a single slash regardless of trailing slash in config), JSON
   body: `model`, `messages` (system message only when `system_prompt` is not
   None), `stream: false`; include `temperature`/`top_p`/`max_tokens` only
   when not None. Header `Authorization: Bearer {key}` only when a key was
   resolved. No other headers beyond content-type/user-agent
   (`mybench/{__version__}`).
3. Response parsing:
   - 3xx (any) → `ProviderError(kind=connection, retryable=False,
     message="redirect response not allowed", status_code=...)`; single
     attempt, no follow (DESIGN §6.8).
   - Other non-2xx → `ProviderError.from_status` with message from
     `error.message` (dict), else body text; body read capped (req. 4);
     message key-scrubbed per requirement 1.
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
   retry on timeout: catch `httpx.TimeoutException` **before**
   `httpx.TransportError` (it is a subclass) → `ProviderError(timeout,
   retryable=False)` directly. Other `httpx.TransportError` → `connection`,
   retryable (one retry). Redirect responses are non-retryable per req. 3.
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

- [ ] Happy path: full body → result fields exact; system message
      included/excluded correctly; params omitted when None.
- [ ] Application-header contract asserted on the captured request: keyed
      config sends exactly `Authorization: Bearer …` + `Content-Type:
      application/json` + `User-Agent: mybench/{version}` as
      application-set headers; loopback keyless config sends no
      `Authorization`; no OpenRouter attribution headers (`HTTP-Referer`,
      `X-Title`) are ever sent (v1 rule, research §3).
- [ ] Key scrubbing: error body echoing `Bearer sk-test-123` → resulting
      `ProviderError.message` contains `[REDACTED]` and not the key.
- [ ] Content as parts array → concatenated; part without `text` →
      invalid_response.
- [ ] Missing `usage` → tokens None; partial usage → partial None.
- [ ] 401 → auth, non-retryable, message from `error.message`.
- [ ] 429 with `Retry-After: 0` → one retry then success (2 requests
      recorded); 429 twice → rate_limit error after exactly 2 attempts.
- [ ] 500 then 200 → success (2 requests); 400 → api_error, single attempt.
- [ ] Connection error then 200 → success (2 requests); two connection
      errors → `connection` error after exactly 2 attempts.
- [ ] Redirect (301) → connection error, non-retryable, single request.
- [ ] Timeout → timeout error, single attempt (no retry).
- [ ] Size cap: (a) `Content-Length: 6MB` → `content_too_large` without the
      stream being read (custom transport that fails the test if its stream
      is consumed); (b) chunked stream without Content-Length aborts once
      past 5 MB; (c) oversized non-2xx error body → error raised with at
      most 5 MB read.
- [ ] Invalid JSON 2xx → invalid_response; empty content → invalid_response.
- [ ] Error body with ANSI escapes → message stripped (via base ctor).
- [ ] `check()` ok and failure paths.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_openai_compat.py -q`. optional manual QA (non-gating) (optional, documented
in PR if run): one real call against Ollama or OpenRouter.

## Dependencies

10.

## Non-goals

Anthropic wire format (12); streaming (v1 non-goal); `max_completion_tokens`
fallback (Known Unknown U1 — do not implement speculatively).

## Design References

DESIGN §6.5, §6.8, §10.4, §13.1 B4, §14; research/provider-api-variance.md.
