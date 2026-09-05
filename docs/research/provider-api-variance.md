# Research: Provider API Variance (OpenAI-compatible + Anthropic)

Date: 2026-07-08
Status: informs ADR-004, DESIGN.md §6; items marked (verify) feed DESIGN.md §18 Known Unknowns

## 1. Problem

Requirement Q3-A: one generic OpenAI-compatible client must cover the official
OpenAI API, OpenRouter, Ollama, vLLM, LM Studio and similar servers, plus a
native Anthropic adapter. "OpenAI-compatible" is a de-facto standard with real
variance; the adapter must be written against the *intersection* we can rely
on, with defensive parsing for the rest.

## 2. OpenAI-compatible `/chat/completions` — reliable intersection

All targeted servers accept:

```
POST {base_url}/chat/completions
Authorization: Bearer {key}          # Ollama ignores it; still safe to send
Content-Type: application/json

{
  "model": "<name>",
  "messages": [
    {"role": "system", "content": "..."},   # optional
    {"role": "user",   "content": "..."}
  ],
  "temperature": 0.7,        # optional
  "top_p": 1.0,              # optional
  "max_tokens": 4096,        # optional; see variance notes
  "stream": false
}
```

and return `choices[0].message.content` (string) plus `choices[0].finish_reason`.

## 3. Known variance to handle defensively

| Area | Variance | Adapter rule (v1) |
|---|---|---|
| `usage` | Officially present; some proxies/older servers omit it or return partial fields | Treat `usage` and each sub-field as optional; store `None` when absent |
| `content` shape | Normally a string; some servers/newer APIs may return an array of content parts | If array: concatenate `part["text"]` for `type == "text"` parts; else reject as invalid_response |
| `max_tokens` | OpenAI newest models prefer `max_completion_tokens`; most compat servers still accept `max_tokens` | v1 sends `max_tokens` only; if a server rejects it (400 mentioning the param), surface the provider error verbatim. Revisit in v2 (verify) |
| Error body | OpenAI: `{"error": {"message", "type", "code"}}`; compat servers vary, some return plain text | Parse best-effort; always keep HTTP status; never assume JSON |
| `finish_reason` | `stop`/`length`/`content_filter`/server-specific values | Store as free-form string; `length` triggers a "truncated" flag in UI |
| Auth | OpenRouter recommends extra headers (`HTTP-Referer`, `X-Title`) but works without (verify) | Send only `Authorization`; no per-provider headers in v1 |
| Base URL shape | Ollama: `http://localhost:11434/v1`; vLLM: `http://host:8000/v1`; OpenRouter: `https://openrouter.ai/api/v1` | Config stores the full base URL ending in `/v1` equivalent; adapter appends `/chat/completions` only |
| Model listing | `GET /models` widely but not universally implemented | Not used for correctness; `mybench models check` uses a 1-token real completion instead |

## 4. Anthropic Messages API differences (native adapter)

| Property | Anthropic Messages API |
|---|---|
| Endpoint | `POST {base_url}/v1/messages` (default base `https://api.anthropic.com`) |
| Auth | `x-api-key: {key}` + `anthropic-version: 2023-06-01` headers |
| System prompt | Top-level `system` string, not a message role |
| `max_tokens` | **Required.** Adapter default when task does not set it: 4096 |
| Response text | `content` is a list of blocks; concatenate `text` of blocks with `type == "text"` |
| Usage | `usage.input_tokens` / `usage.output_tokens` |
| Stop reason | `stop_reason`: `end_turn` / `max_tokens` / `stop_sequence` / ... (`max_tokens` maps to the "truncated" flag) |
| Errors | JSON `{"type": "error", "error": {"type", "message"}}`; 429 carries `retry-after` header |

## 5. Transport-level hardening applicable to both adapters

- TLS verification always on; no option to disable in v1.
- `http://` base URLs allowed only for loopback hosts (`localhost`, `127.0.0.1`,
  `::1`) — startup validation warning/error rule in DESIGN.md §5.
- Explicit timeouts (connect/read/total) — no infinite waits.
- Response body size cap (5 MB) enforced while reading; oversized responses →
  `content_too_large` provider error.
- No redirect following (httpx default) — a redirecting provider is an error.
- Retry once on 429 / 5xx / connection errors with backoff + jitter, honoring
  `Retry-After` (capped 30 s). No retry on timeout (worst-case latency bound).

## 6. Items to re-verify during implementation (feeds §18 Known Unknowns)

- OpenRouter behavior without optional attribution headers under load.
- Content-part arrays from newer OpenAI-compatible servers actually observed in
  the wild (which `type` values need support).
- Whether target servers reject `max_tokens` (vs `max_completion_tokens`) for
  any model the user actually configures.

## 7. Sources

- OpenAI API reference (chat completions), OpenRouter docs, Ollama OpenAI
  compatibility docs, vLLM OpenAI-compatible server docs, Anthropic Messages
  API reference — consulted 2026-07-08; exact field lists re-checked against
  official docs during implementation of issues 11/12 (acceptance criteria
  require doc links in code comments).
