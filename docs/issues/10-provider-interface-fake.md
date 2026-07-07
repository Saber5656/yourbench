# Title

Provider interface, error taxonomy, registry, fake provider

## Summary

Implement `mybench/providers/base.py` (request/result/check types,
`ProviderError` taxonomy, `Provider` protocol), `providers/__init__.py`
(registry + `build()` with lazy API-key resolution), and `providers/fake.py`
(deterministic offline adapter).

## Context

DESIGN §6.1–6.4 and §6.7. Everything above the wire (runner, CLI, web) is
written against these types; the fake provider makes all of it testable and
demoable without keys or network (ADR-004).

## Scope

- `src/mybench/providers/base.py`
- `src/mybench/providers/__init__.py`
- `src/mybench/providers/fake.py`
- `tests/test_providers_base.py`

## Detailed Requirements

1. `base.py`: implement `CompletionRequest`, `CompletionResult`,
   `CheckResult`, `ProviderErrorKind`, `ProviderError`, `Provider` protocol
   **exactly** as typed in DESIGN §6.1–6.3 (field names/order normative).
   `ProviderError.__init__(kind, message, *, status_code=None,
   retryable=False)`; `__str__` = `f"{kind.value}: {message}"`.
   `retryable` must be forced `True` only for `rate_limit`, `connection`, and
   `api_error` with `status_code >= 500` — provide classmethod
   `ProviderError.from_status(status_code, message)` implementing the §6.5
   mapping (401/403→auth, 429→rate_limit, ≥500→api_error retryable, other
   4xx→api_error non-retryable).
2. Message hygiene: `ProviderError` messages must pass through
   `safety.strip_terminal_controls` and `safety.truncate(_, 300)` in the
   constructor (defense against hostile error bodies; §13.1 B4).
3. `__init__.py`:
   - `PROVIDERS: dict[str, type]` = `{"openai_compat": OpenAICompatProvider,
     "anthropic": AnthropicProvider, "fake": FakeProvider}` — import lazily
     inside `build()` to avoid import cycles while 11/12 are unimplemented
     placeholders (placeholders raise `NotImplementedError` on construction
     until their issues land; registry itself ships complete).
   - `build(model_cfg: ModelConfig, settings: Settings) -> Provider`:
     resolves the key: if `api_key_env` set → `os.environ.get`; missing/empty
     value when the provider requires one → raise
     `ProviderError(kind=auth, message=f"environment variable
     {model_cfg.api_key_env} is not set")` (name only, never values);
     constructs the adapter with `(model_cfg, settings, api_key)`.
4. `fake.py` per DESIGN §6.7: content
   `f"fake output from {model_id} for prompt sha256:{h12}\n\n{FILLER}"` where
   `h12` = first 12 hex chars of SHA-256 of `user_prompt` and `FILLER` is a
   fixed two-paragraph markdown constant (must include a code block and a
   list, for render testing); `finish_reason="stop"`;
   `prompt_tokens=len(user_prompt)//4`, `completion_tokens=128`,
   `latency_ms=1`, `raw_model=f"fake/{model_id}"`. `check()` → ok, 1 ms.
   No I/O, no randomness.
5. mypy-strict clean (protocol conformance of fake verified by a typed
   assignment in tests).

## Acceptance Criteria

- [ ] `from_status` mapping table-tested (401, 403, 404, 429, 500, 503).
- [ ] Retryable flags: rate_limit/connection/5xx True; auth/timeout/
      invalid_response/4xx False.
- [ ] Error messages containing `\x1b]0;x\x07` or 10k chars come out stripped
      and truncated.
- [ ] `build` with unset env var raises auth error naming the variable and
      not containing any env **value**.
- [ ] `build` returns a working `FakeProvider` for a fake ModelConfig.
- [ ] Fake determinism: same prompt → identical result; different prompts →
      different hash; content contains code block + list.
- [ ] ruff, mypy strict, pytest green.

## Validation

`uv run pytest tests/test_providers_base.py -q`.

## Dependencies

04 (ModelConfig/Settings), 06 (safety).

## Non-goals

Network adapters (11, 12); transport policy enforcement (their issues); runner
(13).

## Design References

DESIGN §6.1–6.4, §6.7, §13.1 B4; ADR-004.
