# Title

Markdown rendering + sanitization pipeline

## Summary

Implement `mybench/web/render.py`: the single `markdown_safe()` function
(markdown-it-py with raw HTML disabled → nh3 allowlist clean → Markup) plus
the `escape_pre()` helper for raw views, with an adversarial XSS test corpus.

## Context

Model outputs are untrusted attacker-controlled content rendered into the
user's browser — boundary B2 (DESIGN §12, §13.1). This function is the only
sanctioned path from model/user text to HTML; every content page (25–29)
imports it.

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
- [ ] Table/list/heading markdown renders with allowed tags.
- [ ] Nested/malformed HTML fuzz strings (at least 5 hand-picked) never raise.
- [ ] `escape_pre` escapes `<`, `>`, `&`, quotes.
- [ ] Property: output of both functions never contains the substrings
      `<script`, `onerror=`, `javascript:` for any corpus input.
- [ ] ruff, mypy strict, pytest green (this module: ≥ 95% coverage per §15).

## Validation

`uv run pytest tests/test_render.py -q`.

## Dependencies

01 (deps only — parallelizable with 22).

## Non-goals

Syntax highlighting (v2, §2.3); where rendering is *called* (page issues);
CSP headers (22).

## Design References

DESIGN §12, §13.1 B2, §15 coverage gates.
