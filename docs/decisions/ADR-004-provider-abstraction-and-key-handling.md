# ADR-004: Provider abstraction — generic OpenAI-compatible adapter + native Anthropic adapter; API keys only via environment variables

Date: 2026-07-08
Status: Accepted

## Context

Owner selected a generic OpenAI-compatible client covering OpenAI, OpenRouter,
Ollama, vLLM, etc., plus native adapters where wire formats differ (Q3-A).
API variance research: `docs/research/provider-api-variance.md`. The tool will
hold real API keys on the user's machine; leaked keys are the single most
damaging failure the tool can cause.

## Decision

1. **Provider types in v1:** `openai_compat`, `anthropic`, `fake`
   (deterministic, for tests/demos; no network). Registry maps the config
   `provider` string to an adapter class implementing one interface
   (DESIGN.md §6): `complete(request) -> result` and `check() -> result`,
   both async.
2. **Model identity** = user-chosen config `id` slug. Ratings attach to that
   id. The exact provider/model/params in effect are snapshotted per output row.
3. **API keys are referenced, never stored.** Config supports only
   `api_key_env = "ENV_VAR_NAME"`. There is no `api_key = "sk-..."` field; a
   config containing one fails validation with a pointed error. Keys never
   appear in the DB, exports, logs, or error messages (redaction rule,
   DESIGN.md §14).
4. **Transport hardening** (both adapters): TLS verification always on;
   plain `http://` only for loopback hosts; connect/read timeouts; 5 MB
   response cap; no redirect following; single retry on 429/5xx/connection
   errors with jittered backoff honoring `Retry-After` (cap 30 s); no retry on
   timeout.
5. **Streaming is out of scope for v1** — runs happen in the background and
   votes happen later, so time-to-first-token adds no product value yet.

## Consequences

- One adapter covers the long tail of OpenAI-compatible servers; only truly
  different wire formats (Anthropic) cost a new adapter. Gemini native, etc.
  are v2 candidates — most users can reach them through OpenRouter in v1.
- `api_key_env`-only is mildly less convenient (user must export variables or
  use a secrets manager/direnv) and is the deliberate secure default.
- The `fake` provider makes every layer above testable and demoable without
  keys or network.
