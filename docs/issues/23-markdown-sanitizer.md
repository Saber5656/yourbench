# Title

Markdown rendering + sanitization pipeline

## Summary

Implement `mybench/web/render.py`: the single `markdown_safe()` function
(markdown-it-py with raw HTML disabled → nh3 allowlist clean → Markup) plus
the `escape_pre()` helper for raw views, with an adversarial XSS test corpus.

## Context

Model outputs are untrusted attacker-controlled content rendered into the
user's browser — boundary B2 (DESIGN §12, §13.1). These helpers are the only
sanctioned path from content bodies to HTML; the pages that render
prompts/outputs (25, 26, 27) import them, and scalar metadata everywhere
else relies on Jinja autoescape per the DESIGN §12 policy.

## Scope

- `src/mybench/web/render.py`
- `tests/test_render.py`

## Detailed Requirements

1. `def markdown_safe(text: str) -> markupsafe.Markup` implementing DESIGN
   §12 exactly:
   - `MarkdownIt("commonmark")` with `html=False` (raw HTML escaped, never
     passed through), linkify **off**, typographer off.
   - `nh3.clean(rendered, tags=ALLOWED_TAGS, attributes=ALLOWED_ATTRS,
     url_schemes={"http","https","mailto"},
     link_rel="noopener noreferrer nofollow")` with the §12 constant sets
     (module-level, exported for issue 32's checks):
     `ALLOWED_TAGS = {"p","br","pre","code","blockquote","ul","ol","li",
     "h1","h2","h3","h4","h5","h6","strong","em","a","table","thead",
     "tbody","tr","th","td","hr","del"}`,
     `ALLOWED_ATTRS = {"a": {"href"}}`.
   - Return `Markup` (never a plain str) so Jinja does not double-escape.
2. `def escape_pre(text: str) -> markupsafe.Markup` — HTML-escape + wrap in
   nothing (templates place it inside `<pre>`); used for raw-output
   `<details>` views and provider error strings (DESIGN §12).
3. Both functions total: arbitrary unicode input, `\x00`, 10 MB strings must
   not raise (perf note: no size cap here — caps live at input boundaries).
4. Template usage contract documented in the module docstring: templates must
   never use `|safe`; the only Markup producers are these two functions
   (issue 32 greps for `|safe` and direct `nh3`/`MarkdownIt` use outside this
   module).
5. mypy-strict clean.

## Acceptance Criteria

Adversarial corpus — for each input, assert the dangerous artifact is absent
from the output AND the benign remainder renders:

- [ ] `<script>alert(1)</script>` (raw and inside markdown text).
- [ ] `<img src=x onerror=alert(1)>`.
- [ ] `[x](javascript:alert(1))` → no `javascript:` href (link dropped or
      href removed).
- [ ] `[x](data:text/html;base64,...)` → no `data:` href.
- [ ] `<a href="https://ok" onclick="x">` raw HTML → escaped (html=False).
- [ ] Markdown autolink `<https://example.com>` → anchor with
      `rel="noopener noreferrer nofollow"`.
- [ ] `<iframe>`, `<form>`, `<style>`, `<svg onload=...>` → stripped/escaped.
- [ ] Code block containing `<script>` → rendered escaped **inside**
      `<pre><code>` (visible as text).
- [ ] Heading/list/code/blockquote/emphasis markdown renders with allowed
      tags (no pipe-table criterion: CommonMark does not render tables —
      DESIGN §12; table tags stay allowlisted as defense-in-depth only).
- [ ] Nested/malformed HTML fuzz strings (at least 5 hand-picked) never raise.
- [ ] DOM-level assertions for `markdown_safe` (parse output with
      `html.parser`, not substring checks): every tag ∈ `ALLOWED_TAGS`;
      attributes only `href`/`rel` on `a`; every `href` scheme
      (case-insensitive, after strip) ∈ {http, https, mailto}; every `a` has
      the required `rel`.
- [ ] Separate invariants per function (they differ by design):
      `markdown_safe` output parses with no active content (per the DOM
      assertions above) for every corpus input; `escape_pre(text)` equals
      `markupsafe.escape(text)` exactly (a hostile `<img onerror=…>`
      remains **visible as escaped text** — `&lt;img…` — which is correct),
      returns `Markup`, and contains no raw `<` or `>` characters.
- [ ] ruff, mypy strict, pytest green; `uv run coverage run -m pytest && uv
      run coverage report --include='*/web/render.py'` shows ≥ 95% (the §15
      gate for this module).

## Validation

All per-issue gates (DESIGN §17) must pass:

```sh
uv run ruff check . && uv run ruff format --check . && uv run mypy src && uv run pytest -q
```

Targeted checks:

`uv run pytest tests/test_render.py -q`.

## Dependencies

01 (deps only — parallelizable with 22).

## Non-goals

Syntax highlighting (v2, §2.3); where rendering is *called* (page issues);
CSP headers (22).

## Design References

DESIGN §12, §13.1 B2, §15 coverage gates.
