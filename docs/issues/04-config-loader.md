# Title

Config schema, TOML loader, validation

## Summary

Implement `mybench/config.py`: frozen dataclasses `Settings`, `ModelConfig`,
`Config`; TOML loading with the full validation matrix of DESIGN §5.3; helpful
`config error:` messages.

## Context

Every provider, the runner, the CLI, and the web server consume this config.
The `api_key_env`-only rule (ADR-004) is enforced here — this module is the
gate that keeps key material out of files.

## Scope

- `src/mybench/config.py`
- `tests/test_config.py`

## Detailed Requirements

1. Dataclasses (frozen, slots):
   - `Settings(port: int = 8137, timeout_seconds: int = 120,
     max_concurrency: int = 4, bootstrap_samples: int = 200)`
   - `ModelConfig(id: str, provider: str, model: str | None,
     base_url: str | None, api_key_env: str | None, enabled: bool = True)`
   - `Config(settings: Settings, models: tuple[ModelConfig, ...],
     source_path: Path | None)` with properties
     `enabled_models -> tuple[ModelConfig, ...]` (file order preserved) and
     `model_by_id(model_id) -> ModelConfig | None`.
2. `load(path: Path | None = None) -> Config`:
   - `path=None` → use `paths.config_path()`; explicit paths normalized with
     `Path(path).expanduser().resolve(strict=False)` (must not require
     existence).
   - Missing file → `Config(Settings(), (), source_path=resolved_path)` (no
     error; commands needing models produce their own guidance — DESIGN §5.3).
   - Unreadable file or TOML syntax error → raise `ConfigError` (subclass of
     `Exception`, message prefixed `config error: `).
   - Document-shape validation before field rules: only `settings` (table)
     and `models` (array of tables) allowed at top level (unknown top-level
     key → error naming it); wrong container types (e.g. `models` not an
     array, `[settings]` not a table) → error; per-model required keys
     (`id`, `provider`) missing → `models[N].id: required` style errors;
     wrong scalar types (e.g. `enabled = "yes"`, `port = "8137"`) → error
     naming field and expected type.
3. Validation (raise `ConfigError` listing **all** violations, one per line,
   each prefixed with the TOML location like `models[2].id`):
   - Every rule in the DESIGN §5.3 table, including: id regex + uniqueness;
     provider enum `openai_compat|anthropic|fake`; model required except fake;
     base_url required/optional/forbidden per provider; scheme http/https;
     http restricted to loopback hosts (`localhost`, `127.0.0.1`, `::1` —
     parse with `urllib.parse.urlsplit`, compare hostname); `api_key_env`
     name regex and requiredness rules; `enabled` bool; settings ranges
     (`port` 1024–65535, `timeout_seconds` 1–3600, `max_concurrency` 1–16,
     `bootstrap_samples` 0–1000).
   - Unknown keys anywhere in `[settings]` or a `[[models]]` entry → error
     naming the key (typo protection).
   - A literal `api_key` key in a model entry → dedicated error:
     `models[N].api_key: storing keys in config is not supported; set
     api_key_env to the name of an environment variable instead`.
4. Env var **values** are not read here (lazy at provider build; DESIGN §5.3).
5. Anthropic `base_url` omitted → **normalized to
   `https://api.anthropic.com` at load time** (DESIGN §5.3), so adapters and
   snapshots always see a concrete URL.
6. Error-message hygiene (DESIGN §5.3, §13.6): messages contain the TOML
   location and the violated rule only — never the raw value of
   secret-adjacent fields (`api_key`, `api_key_env`). Example: the `api_key`
   error names the key and the fix, not the pasted value.
7. mypy-strict clean; no dependencies beyond stdlib (`tomllib`).

## Acceptance Criteria

- [ ] Valid example config from DESIGN §5.2 loads; `enabled_models` order
      matches file order.
- [ ] Table-driven tests cover every §5.3 rule plus every document-shape rule
      of requirement 2 with at least one invalid case each (≥ 20 invalid
      fixtures), asserting the offending location string appears.
- [ ] Multiple violations are reported together in one raise.
- [ ] `api_key` literal produces the exact guidance message above, and a
      fixture with `api_key = "sk-canary-123"` produces an error message NOT
      containing `sk-canary-123`.
- [ ] Anthropic model with omitted base_url loads with
      `base_url == "https://api.anthropic.com"`.
- [ ] `http://192.168.1.10:11434/v1` rejected; `http://127.0.0.1:11434/v1`
      accepted; `https://` non-loopback accepted.
- [ ] Missing file returns empty-models Config; malformed TOML raises
      `ConfigError`.
- [ ] ruff, mypy strict, pytest green.

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_config.py -q`; manual load of the §5.2 example via
`python -c`.

## Dependencies

01, 03.

## Non-goals

Reading key values / provider construction (10); task param validation (06);
`mybench config validate` CLI (16).

## Design References

DESIGN §5.2, §5.3; ADR-004 (env-only keys).
