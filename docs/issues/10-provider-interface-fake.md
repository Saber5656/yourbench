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
   (including `async def aclose(self) -> None`) **exactly** as typed in
   DESIGN §6.1–6.3 (field names/order normative).
   `ProviderError.__init__(kind, message, *, status_code=None,
   retryable=False)`; `__str__` = `f"{kind.value}: {message}"`.
   `retryable` must be forced `True` only for `rate_limit`, `connection`, and
   `api_error` with `status_code >= 500` — provide classmethod
   `ProviderError.from_status(status_code, message)` implementing the §6.5
   mapping (401/403→auth, 429→rate_limit, ≥500→api_error retryable, other
   4xx→api_error non-retryable).
2. Message hygiene: `ProviderError` messages must pass through
   `safety.strip_terminal_controls` and `safety.truncate(_, 300)` in the
   constructor (defense against hostile error bodies; §13.1 B4). Note: the
   constructor cannot know API keys — network adapters (11/12) must scrub
   the resolved key from provider-derived text *before* raising (§6.2).
3. `base.py` also provides the shared transport factory (DESIGN §6.8):
   `default_client(settings: Settings) -> httpx.AsyncClient` — `verify=True`,
   `follow_redirects=False`, `timeout=httpx.Timeout(connect=10.0,
   read=settings.timeout_seconds, write=30.0, pool=10.0)`. Adapters 11/12
   must build their client through it (enforcement of caps/retries stays in
   the adapters).
4. `__init__.py`:
   - `build(model_cfg: ModelConfig, settings: Settings) -> Provider` resolves
     the adapter class lazily by provider key:
     `{"openai_compat": ("mybench.providers.openai_compat",
     "OpenAICompatProvider"), "anthropic": ("mybench.providers.anthropic",
     "AnthropicProvider"), "fake": ("mybench.providers.fake",
     "FakeProvider")}` via `importlib.import_module` + `getattr`. While
     issues 11/12 have not landed, those modules are the issue-01 docstring
     placeholders and `getattr` fails → re-raise as
     `NotImplementedError(f"provider '{key}' not implemented yet")`. Do not
     create placeholder classes in their files.
   - Key requirement predicate (from DESIGN §5.3, restated):
     `anthropic` → required; `openai_compat` → required unless the base_url
     host is loopback (`localhost`/`127.0.0.1`/`::1`); `fake` → never.
     If `api_key_env` is set → `os.environ.get`; a missing/empty value when
     required → raise `ProviderError(kind=auth,
     message=f"environment variable {model_cfg.api_key_env} is not set")`
     (name only, never values); optional-and-unset → pass `api_key=None`.
     Constructs the adapter with `(model_cfg, settings, api_key)`.
5. `fake.py` per DESIGN §6.7, with `model_id = model_cfg.id` (the config
   slug — `model_cfg.model` may be None for fake, ADR-004): content
   `f"fake output from {model_id} for prompt sha256:{h12}\n\n{FILLER}"` where
   `h12` = first 12 hex chars of SHA-256 of `user_prompt` (UTF-8) and
   `FILLER` is a fixed two-paragraph markdown constant (must include a code
   block and a list, for render testing); `finish_reason="stop"`;
   `prompt_tokens=len(user_prompt)//4`, `completion_tokens=128`,
   `latency_ms=1`, `raw_model=f"fake/{model_cfg.id}"`. `check()` → ok, 1 ms.
   `aclose()` is a no-op. No I/O, no randomness.
6. mypy-strict clean (protocol conformance of fake verified by a typed
   assignment in tests).

## Acceptance Criteria

- [ ] `from_status` mapping table-tested (401, 403, 404, 429, 500, 503).
- [ ] Retryable flags: rate_limit/connection/5xx True; auth/timeout/
      invalid_response/4xx False.
- [ ] Error messages containing `\x1b]0;x\x07` or 10k chars come out stripped
      and truncated.
- [ ] Key-requirement predicate table-tested: anthropic (required), remote
      openai_compat (required), loopback openai_compat (optional → None ok),
      fake (never). With `MYBENCH_CANARY_SECRET=sk-canary` set and the
      required var unset, the auth error names the required var and contains
      neither `sk-canary` nor any env value.
- [ ] `build` returns a working `FakeProvider` for a fake ModelConfig;
      `build` for `openai_compat`/`anthropic` against the issue-01
      placeholders raises `NotImplementedError` naming the provider.
- [ ] `default_client` returns a client with verify on, redirects off, and
      the §6.8 timeout tuple (inspect `client.timeout`).
- [ ] Fake determinism: same prompt → identical result; different prompts →
      different hash; content contains code block + list; `raw_model` =
      `fake/{id}` even when `model` is None.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_providers_base.py -q`.

## Dependencies

04 (ModelConfig/Settings), 06 (safety).

## Non-goals

Network adapters (11, 12); transport policy enforcement (their issues); runner
(13).

## Design References

DESIGN §6.1–6.4, §6.7, §13.1 B4; ADR-004.
